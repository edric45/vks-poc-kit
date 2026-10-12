# Antrea Egress on VKS with NSX VPC: a fixed source IP per namespace

Tested in the SE lab on 2026-10-08 and 2026-10-12, including packet captures
outside the VPC. All outputs on this page are from those runs.

## What this shows

Antrea Egress gives the pods of one namespace a **fixed source IP** instead of
their worker node's IP. On VKS with NSX VPC, that egress IP lives on its own
NSX subnet, which the worker reaches over an **802.1Q VLAN sub-interface**
Antrea creates on the node.

Who sees the egress IP depends on the NSX access mode of that subnet. The lab
runs both side by side, plus a namespace without Egress:

| Namespace | Egress subnet | Egress IP | Seen inside the VPC | Seen outside the VPC |
|---|---|---|---|---|
| `egress-private` | NSX **Private**, VLAN 101 | 172.30.0.34 | 172.30.0.34 | 172.17.10.24, the VPC's shared outbound address |
| `egress-public` | NSX **Public**, VLAN 102 | 172.17.10.34 | 172.17.10.34 | **172.17.10.34** |
| `egress-control` | none | — | its node IP | 172.17.10.24, the VPC's shared outbound address |

**For a physical firewall rule** (for example `src -> Oracle DB, 1521`) use the
**Public** variant: its egress IP is the only address that reaches the physical
network and identifies just this namespace. 172.17.10.24 is the VPC's
`defaultSNATIP`, shared by every workload in the vSphere Namespace.

| Status | |
|---|---|
| Antrea Egress, EgressSeparateSubnet, Private and Public subnets | **works in our lab** (VKS 3.7, VKr v1.35, NSX VPC) |
| Method | Broadcom's method for VKS on NSX VPC: NSX Subnet + SubnetConnectionBindingMap (VLAN) + SubnetIPReservation |
| Not tested | egress IP failover to another node; `PrivateTGW` subnets |

## Environment

| Item | Value |
|---|---|
| Supervisor / vSphere Namespace | 172.17.10.2 / `se-ns-vks` (NSX VPC `se-ns-vks_sw4oy`, AutoSNAT on, `defaultSNATIP` 172.17.10.24) |
| Workload cluster | `se-cluster-001`, 1 control plane + 2 workers (`…49dq4` 172.30.0.3, `…sntdw` 172.30.0.6) |
| Node network (NSX SubnetSet) | `se-cluster-001-fxfqc`, 172.30.0.0/27 |
| Private egress subnet | `egress-private`, 172.30.0.32/28, gateway .33, reserved .34–.39, VLAN 101 |
| Public egress subnet | `egress-public`, 172.17.10.32/28, gateway .33, reserved .34–.35, VLAN 102 |
| Observer, inside the VPC | test pod `egress-observer` on the Supervisor (vSphere Pod), 172.31.1.3:8080; replies with the caller's source address |
| Outside the VPC | lab jump host 192.168.50.1; `tcpdump` shows the source of incoming connections |

Antrea in VKS already has the needed feature gates (from `antrea-config`):

```
gate Egress: true
gate EgressSeparateSubnet: true
trafficEncapMode: encap
```

## How the traffic flows

```
 se-cluster-001                                              NSX VPC se-ns-vks
 ────────────────────────────────────────────────────        ──────────────────────────────────────────────

 pod in egress-private or egress-public (any worker)
   │ OVS: Egress applies -> Geneve tunnel to the egress node (if not already there)
   ▼
 egress node …49dq4
   egress-private: mark 0x1 -> SNAT 172.30.0.34  -> table 101 -> antrea-ext.101 (VLAN 101) ┐
   egress-public:  mark 0x2 -> SNAT 172.17.10.34 -> table 102 -> antrea-ext.102 (VLAN 102) ┤
                                                                                     eth0 ─┼──► node vNIC segment port
                                                                                           │     binding maps:
 pod in egress-control (any worker)                                                        │       VLAN 101 -> Subnet egress-private (Private)
   SNAT to its node IP 172.30.0.x -> its node's eth0, untagged ────────────────────────────┘       VLAN 102 -> Subnet egress-public  (Public)
                                                                                                 untagged -> node SubnetSet (Private)
                                                                                                   │
                                                                                                   ▼ VPC routing
                                                                     ┌─────────────────────────────┴──────────────────────────┐
                                                                     ▼                                                        ▼
                                                     inside the VPC: egress-observer (test pod)           leaving the VPC: VPC gateway
                                                       egress-private  172.30.0.34                          Private subnets -> AutoSNAT 172.17.10.24
                                                       egress-public   172.17.10.34                          Public subnet   -> routed as is
                                                       egress-control  node IP                                      │
                                                                                                                    ▼
                                                                                              jump host 192.168.50.1 (outside the VPC)
                                                                                                egress-private  172.17.10.24
                                                                                                egress-public   172.17.10.34
                                                                                                egress-control  172.17.10.24
```

Layers involved:

| Layer | Component | What it does here |
|---|---|---|
| Overlay | Antrea OVS + Geneve | carries Egress traffic from any worker to the egress node |
| L3 | Antrea OVS + iptables `ANTREA-POSTROUTING` | marks the packet per Egress (0x1, 0x2) and SNATs it to that Egress's IP |
| L3 | Linux policy routing | each mark has its own table: default route via the egress subnet's NSX gateway |
| L2 | `antrea-ext.<vlan>` (802.1Q) | tags the traffic with the egress subnet's VLAN on the node's vNIC |
| NSX | SubnetConnectionBindingMap | maps that VLAN on the node's port to the egress subnet |
| NSX | Subnet + SubnetIPReservation | owns the egress subnet; reserved addresses are never handed out |
| NSX | VPC gateway | Private subnets: AutoSNAT to `defaultSNATIP`. Public subnet: routed and advertised outside (BGP) |

The VLAN tags exist only between the guest and its NSX segment port; they
never reach the physical switches, so they cannot clash with VLANs there. They
must be unique per target SubnetSet.

## Steps

### 1. NSX side, on the Supervisor

Private egress subnet, with the observer:

```sh
kubectl --context 172.17.10.2 -n se-ns-vks get subnetset          # the cluster's node SubnetSet name
kubectl --context 172.17.10.2 apply -f supervisor/clusters/se-cluster-001/antrea-egress-test/egress-subnet.yaml
```

`supervisor/clusters/se-cluster-001/antrea-egress-test/egress-subnet.yaml`:

```yaml
# Antrea Egress test, PRIVATE variant, Supervisor side (namespace se-ns-vks).
# Public variant: egress-subnet-public.yaml.
# The cluster side is se-cluster-001/antrea-egress-test/.
#
#   kubectl --context 172.17.10.2 apply -f supervisor/clusters/se-cluster-001/antrea-egress-test/egress-subnet.yaml
#   kubectl --context 172.17.10.2 -n se-ns-vks get subnet egress-private \
#     -o jsonpath='{.status.networkAddresses} {.status.gatewayAddresses}{"\n"}'
#   kubectl --context 172.17.10.2 -n se-ns-vks get subnetipreservation egress-private -o jsonpath='{.status.ips}{"\n"}'
#
# Not synced by ArgoCD (the add-on ApplicationSet only reads addons/*, the
# cluster Application only cluster.yaml). Applied by hand for the test.
#
# Why a separate Subnet: on VKS with NSX VPC, egress IPs must be addresses NSX
# knows about, or NSX drops the traffic. Broadcom's method (EgressSeparateSubnet,
# already enabled in the cluster's Antrea config) is: NSX allocates a Subnet,
# a SubnetConnectionBindingMap attaches it to the cluster's SubnetSet on a VLAN,
# and Antrea uses that CIDR/gateway/VLAN in an ExternalIPPool.
apiVersion: crd.nsx.vmware.com/v1alpha1
kind: Subnet
metadata:
  name: egress-private
  namespace: se-ns-vks
spec:
  # Private: the egress IP is seen inside the VPC (the observer below), but
  # traffic leaving the VPC is still translated to the VPC's outbound address.
  # Public would keep the egress IP end to end, but draws from the shared
  # external pool that has stale routes for .27 and .30.
  accessMode: Private
  ipv4SubnetSize: 16
---
# The egress IPs, reserved in NSX. The Subnet has static IP allocation on, so
# NSX would hand its addresses to anything attached to it; reserved addresses
# it never hands out. NSX picks the addresses (you choose only how many) and
# lists them in status.ips, which is what 02-egress.yaml must use:
#   kubectl --context 172.17.10.2 -n se-ns-vks get subnetipreservation egress-private -o jsonpath='{.status.ips}{"\n"}'
apiVersion: crd.nsx.vmware.com/v1alpha1
kind: SubnetIPReservation
metadata:
  name: egress-private
  namespace: se-ns-vks
spec:
  subnet: egress-private
  numberOfIPs: 6
---
apiVersion: crd.nsx.vmware.com/v1alpha1
kind: SubnetConnectionBindingMap
metadata:
  name: egress-private
  namespace: se-ns-vks
spec:
  subnetName: egress-private
  # The cluster's node SubnetSet: kubectl -n se-ns-vks get subnetset
  targetSubnetSetName: se-cluster-001-fxfqc
  # Must be unique on the target SubnetSet; the ExternalIPPool uses the same tag.
  vlanTrafficTag: 101
---
# Observer: a pod running directly on the Supervisor, in the same VPC, that
# replies with the caller's source address. It shows which IP the cluster's
# traffic arrives from. Reach it at <pod IP>:8080.
apiVersion: v1
kind: Pod
metadata:
  name: egress-observer
  namespace: se-ns-vks
  labels:
    app: egress-observer
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 65532
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: whoami
    image: docker.io/traefik/whoami:v1.10.3
    # Non-root cannot bind port 80, whoami's default.
    args: ["--port", "8080"]
    ports:
    - containerPort: 8080
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

Public egress subnet:

```sh
kubectl --context 172.17.10.2 apply -f supervisor/clusters/se-cluster-001/antrea-egress-test/egress-subnet-public.yaml
```

`supervisor/clusters/se-cluster-001/antrea-egress-test/egress-subnet-public.yaml`:

```yaml
# Antrea Egress test, PUBLIC variant, Supervisor side (namespace se-ns-vks).
# Cluster side: se-cluster-001/antrea-egress-test/03-egress-public.yaml
#
#   kubectl --context 172.17.10.2 apply -f supervisor/clusters/se-cluster-001/antrea-egress-test/egress-subnet-public.yaml
#   kubectl --context 172.17.10.2 -n se-ns-vks get subnet egress-public \
#     -o jsonpath='{.status.networkAddresses} {.status.gatewayAddresses}{"\n"}'
#   kubectl --context 172.17.10.2 -n se-ns-vks get subnetipreservation egress-public -o jsonpath='{.status.ips}{"\n"}'
#
# Why Public: a Private subnet's traffic is translated at the VPC gateway to
# the VPC's single outbound address (defaultSNATIP 172.17.10.24, shared by
# everything in se-ns-vks), so a physical firewall can never tell this
# namespace apart. NSX routes a Public subnet without translation and
# advertises it outside the VPC, so the Antrea egress IP itself reaches the
# physical network, and replies route back to it.
#
# Cost: Public subnets are carved from the external IP block that all
# namespaces in this NSX project share (172.17.10.0/24 here); 16 is the
# smallest size NSX accepts. That block has stale routes on .27 and .30
# (vip-placeholder-27 holds .27), so test the egress IP from outside before
# relying on it.
#
# Kept next to the Private variant (egress-subnet.yaml): accessMode cannot be
# changed on an existing Subnet.
apiVersion: crd.nsx.vmware.com/v1alpha1
kind: Subnet
metadata:
  name: egress-public
  namespace: se-ns-vks
spec:
  accessMode: Public
  ipv4SubnetSize: 16
---
# Two egress IPs: one for the Egress, one spare. NSX picks them; read them
# from status.ips and use exactly those in 03-egress-public.yaml.
apiVersion: crd.nsx.vmware.com/v1alpha1
kind: SubnetIPReservation
metadata:
  name: egress-public
  namespace: se-ns-vks
spec:
  subnet: egress-public
  numberOfIPs: 2
---
apiVersion: crd.nsx.vmware.com/v1alpha1
kind: SubnetConnectionBindingMap
metadata:
  name: egress-public
  namespace: se-ns-vks
spec:
  subnetName: egress-public
  targetSubnetSetName: se-cluster-001-fxfqc
  # 101 is taken by the Private binding on this SubnetSet; tags must be unique.
  vlanTrafficTag: 102
```

Read back what NSX assigned; the cluster side must use exactly these values:

```sh
kubectl --context 172.17.10.2 -n se-ns-vks get subnet,subnetipreservation,subnetconnectionbindingmap | grep -E "NAME|egress-"
kubectl --context 172.17.10.2 -n se-ns-vks get subnet egress-public -o jsonpath='{.status.networkAddresses} {.status.gatewayAddresses}{"\n"}'
```

```
subnet.../egress-private    Private   16   172.30.0.32/28     gateway 172.30.0.33
subnet.../egress-public     Public    16   172.17.10.32/28    gateway 172.17.10.33
subnetipreservation.../egress-private   6   172.30.0.34-172.30.0.39
subnetipreservation.../egress-public    2   172.17.10.34-172.17.10.35
subnetconnectionbindingmap.../egress-private   se-cluster-001-fxfqc   101
subnetconnectionbindingmap.../egress-public    se-cluster-001-fxfqc   102
```

NSX advertises the Public subnet outside the VPC straight away; the jump host
learned it over BGP:

```
172.17.10.32/28 nhid 86 via 192.168.50.5 dev eth1 proto bgp metric 20
```

### 2. Test clients and the Private Egress, in the workload cluster

```sh
C=se-cluster-001:se-cluster-001
kubectl --context $C apply -f se-cluster-001/antrea-egress-test/01-clients.yaml
kubectl --context $C apply -f se-cluster-001/antrea-egress-test/02-egress.yaml
```

`se-cluster-001/antrea-egress-test/01-clients.yaml`:

```yaml
# Antrea Egress test, cluster side: one test client per case.
#
#   kubectl --context se-cluster-001:se-cluster-001 apply -f se-cluster-001/antrea-egress-test/01-clients.yaml
#
# Applied INSIDE the workload cluster. Supervisor side (NSX subnets + observer):
# supervisor/clusters/se-cluster-001/antrea-egress-test/
#
#   egress-private/client   gets the Private-subnet Egress (02-egress.yaml)
#   egress-control/control  no Egress: leaves with its own node's IP
#   egress-public/client    is in 03-egress-public.yaml, with its Egress
#
# All pods pass the cluster's default "restricted" Pod Security profile.
apiVersion: v1
kind: Namespace
metadata:
  name: egress-private
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: egress-private
  labels:
    app: egress-private-client
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 100
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: curl
    image: docker.io/curlimages/curl:8.10.1
    command: ["sleep", "infinity"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
---
apiVersion: v1
kind: Namespace
metadata:
  name: egress-control
---
apiVersion: v1
kind: Pod
metadata:
  name: control
  namespace: egress-control
  labels:
    app: egress-control
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 100
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: curl
    image: docker.io/curlimages/curl:8.10.1
    command: ["sleep", "infinity"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

`se-cluster-001/antrea-egress-test/02-egress.yaml`:

```yaml
# Antrea Egress test, PRIVATE variant, cluster side: the egress IP pool and the
# Egress for namespace egress-private.
#
#   kubectl --context se-cluster-001:se-cluster-001 apply -f se-cluster-001/antrea-egress-test/02-egress.yaml
#   kubectl --context se-cluster-001:se-cluster-001 get egress egress-private     # EGRESSIP and NODE
#
# Apply AFTER the Supervisor side is Ready. Every address below comes from NSX,
# and NSX chooses them, so check both each time and edit this file to match:
#   gateway, prefixLength   from the Subnet:
#     kubectl --context 172.17.10.2 -n se-ns-vks get subnet egress-private \
#       -o jsonpath='{.status.networkAddresses} {.status.gatewayAddresses}{"\n"}'
#   ipRanges, egressIP      from the SubnetIPReservation (reserved, so NSX never
#                           hands them to anything else):
#     kubectl --context 172.17.10.2 -n se-ns-vks get subnetipreservation egress-private \
#       -o jsonpath='{.status.ips}{"\n"}'
# Last run (2026-10-08): network 172.30.0.32/28, gateway 172.30.0.33,
# reserved 172.30.0.34-172.30.0.39.
#
# The pool must exist before the Egress (Antrea's webhook rejects an Egress
# that names a missing pool), which is why it comes first in this file.
apiVersion: crd.antrea.io/v1beta1
kind: ExternalIPPool
metadata:
  name: egress-private
spec:
  # Exactly the addresses NSX reserved (SubnetIPReservation status.ips). If
  # NSX returns separate addresses rather than a block, list one range each.
  ipRanges:
  - start: 172.30.0.34
    end: 172.30.0.39
  # EgressSeparateSubnet: the egress IP lives on its own subnet, reached on a
  # VLAN sub-interface. vlan must match vlanTrafficTag in the binding.
  subnetInfo:
    gateway: 172.30.0.33
    prefixLength: 28
    vlan: 101
  # Workers only: keep egress traffic off the control plane.
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/control-plane
      operator: DoesNotExist
---
apiVersion: crd.antrea.io/v1beta1
kind: Egress
metadata:
  name: egress-private
spec:
  appliedTo:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: egress-private
  externalIPPool: egress-private
  # Fixed so firewall rules can name it; omit to let Antrea pick from the pool.
  egressIP: 172.30.0.34
```

### 3. The Public Egress, in its own namespace

```sh
kubectl --context $C apply -f se-cluster-001/antrea-egress-test/03-egress-public.yaml
kubectl --context $C get egress
```

`se-cluster-001/antrea-egress-test/03-egress-public.yaml`:

```yaml
# Antrea Egress test, PUBLIC variant, cluster side: its own namespace
# (egress-public) with its own Egress, so it runs side by side with the
# Private variant (egress-private, 02-egress.yaml) and the no-Egress control
# (egress-control). One Egress per namespace.
#
#   kubectl --context se-cluster-001:se-cluster-001 apply -f se-cluster-001/antrea-egress-test/03-egress-public.yaml
#   kubectl --context se-cluster-001:se-cluster-001 get egress                 # both egresses, IPs and nodes
#
# Needs the Supervisor side first: egress-subnet-public.yaml. Values from NSX
# (2026-10-12): Subnet 172.17.10.32/28, gateway 172.17.10.33, reserved
# 172.17.10.34-172.17.10.35. NSX advertises the subnet to the physical network
# (the jump host learned 172.17.10.32/28 over BGP), so replies route back.
apiVersion: crd.antrea.io/v1beta1
kind: ExternalIPPool
metadata:
  name: egress-public
spec:
  ipRanges:
  - start: 172.17.10.34
    end: 172.17.10.35
  subnetInfo:
    gateway: 172.17.10.33
    prefixLength: 28
    # Must match vlanTrafficTag of the egress-public binding.
    vlan: 102
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/control-plane
      operator: DoesNotExist
---
apiVersion: v1
kind: Namespace
metadata:
  name: egress-public
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: egress-public
  labels:
    app: egress-public-client
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 100
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: curl
    image: docker.io/curlimages/curl:8.10.1
    command: ["sleep", "infinity"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
---
apiVersion: crd.antrea.io/v1beta1
kind: Egress
metadata:
  name: egress-public
spec:
  appliedTo:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: egress-public
  externalIPPool: egress-public
  # The source address a physical firewall rule can name.
  egressIP: 172.17.10.34
```

```
NAME             EGRESSIP       NODE
egress-private   172.30.0.34    se-cluster-001-se-cluster-001-np01-nf8s7-mmc6z-49dq4
egress-public    172.17.10.34   se-cluster-001-se-cluster-001-np01-nf8s7-mmc6z-49dq4
```

### 4. Test

Capture on the jump host (outside the VPC), then let each client connect to it
and to the observer (inside the VPC):

```sh
# on the jump host
tcpdump -n -tttt -i eth1 "dst host 192.168.50.1 and dst port 22 and tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0"

# from the cluster
for p in egress-private/client egress-public/client egress-control/control; do
  kubectl --context $C -n ${p%/*} exec ${p#*/} -- curl -s -m 3 telnet://192.168.50.1:22 </dev/null >/dev/null
  kubectl --context $C -n ${p%/*} exec ${p#*/} -- curl -s -m 5 http://172.31.1.3:8080/ | grep RemoteAddr
done
```

## Results (2026-10-12)

### Side by side

| Namespace | Pod ran on | Observer, inside the VPC | Jump host, outside the VPC |
|---|---|---|---|
| egress-private | `sntdw` (not the egress node) | 172.30.0.34 | 172.17.10.24 |
| egress-public | `49dq4` (the egress node) | 172.17.10.34 | **172.17.10.34** |
| egress-control | `sntdw` | 172.30.0.6 (its node IP) | 172.17.10.24 |

Observer replies:

```
egress-private/client    RemoteAddr: 172.30.0.34:37294
egress-public/client     RemoteAddr: 172.17.10.34:42144
egress-control/control   RemoteAddr: 172.30.0.6:48750
```

Jump host capture, in the order the pods connected (private, public, control;
the jump host's clock runs about 30 s behind):

```
08:00:35.381567 IP 172.17.10.24.50476 > 192.168.50.1.22: Flags [S]    <- egress-private
08:00:39.378622 IP 172.17.10.24.46344 > 192.168.50.1.22: Flags [S]    <- egress-private
08:00:47.757621 IP 172.17.10.34.36486 > 192.168.50.1.22: Flags [S]    <- egress-public
08:00:52.452256 IP 172.17.10.34.36492 > 192.168.50.1.22: Flags [S]    <- egress-public
08:01:00.836571 IP 172.17.10.24.64306 > 192.168.50.1.22: Flags [S]    <- egress-control
08:01:05.525294 IP 172.17.10.24.43763 > 192.168.50.1.22: Flags [S]    <- egress-control
```

Reading it:

- **egress-private** is identified inside the VPC (172.30.0.34), but outside it
  looks exactly like the namespace without Egress: both become the VPC's
  172.17.10.24.
- **egress-public** keeps 172.17.10.34 end to end; that is the address a
  physical firewall rule can name.
- The egress-private pod ran on the *other* worker: Antrea tunnelled its
  traffic to the egress node, which applied the SNAT.

Where 172.17.10.24 comes from:

```sh
kubectl --context 172.17.10.2 -n se-ns-vks get networkinfo -o yaml | grep -B1 -A1 defaultSNATIP
#   - defaultSNATIP: 172.17.10.24
#     name: se-ns-vks_sw4oy
```

### The VLAN interfaces on the egress node

Read through the Antrea agent, which runs on the node's host network:

```sh
A=$(kubectl --context $C -n kube-system get pods -l component=antrea-agent \
      --field-selector spec.nodeName=<node> -o name)
kubectl --context $C -n kube-system exec $A -c antrea-agent -- ip -br addr
kubectl --context $C -n kube-system exec $A -c antrea-agent -- ip -d link show antrea-ext.101
kubectl --context $C -n kube-system exec $A -c antrea-agent -- sh -c 'ip rule; ip route show table 101; ip route show table 102'
kubectl --context $C -n kube-system exec $A -c antrea-agent -- sh -c 'iptables-save -t nat | grep "to-source"'
```

Egress node `…49dq4` carries one VLAN sub-interface per egress subnet, each
with its own packet mark, routing table and SNAT rule:

```
eth0                UP   172.30.0.3/27        <- node IP on the node SubnetSet
antrea-ext.101@eth0 UP   172.30.0.34/28       <- 802.1Q id 101, Private egress subnet
antrea-ext.102@eth0 UP   172.17.10.34/28      <- 802.1Q id 102, Public egress subnet

32763:  from all fwmark 0x1/0xff lookup 101
32764:  from all fwmark 0x2/0xff lookup 102
table 101:
default via 172.30.0.33 dev antrea-ext.101
172.30.0.32/28 dev antrea-ext.101 scope link
table 102:
default via 172.17.10.33 dev antrea-ext.102
172.17.10.32/28 dev antrea-ext.102 scope link

-A ANTREA-POSTROUTING ! -o antrea-gw0 -m comment --comment "Antrea: SNAT Pod to external packets" -m mark --mark 0x1/0xff -j SNAT --to-source 172.30.0.34
-A ANTREA-POSTROUTING ! -o antrea-gw0 -m comment --comment "Antrea: SNAT Pod to external packets" -m mark --mark 0x2/0xff -j SNAT --to-source 172.17.10.34
```

The other worker (`…sntdw`) holds no egress IP and has no `antrea-ext`
interface and no egress routing tables: Antrea creates them only on the node
that currently holds an egress IP.

### Antrea Traceflow (2026-10-09, Private variant)

Traceflow injects a test packet and reports what each node's datapath did with
it. This run predates the renaming: the namespace was then called
`egress-test` and the client ran on worker `hgscj`, which did not hold the
egress IP; destination the observer.

```yaml
apiVersion: crd.antrea.io/v1beta1
kind: Traceflow
metadata:
  name: egress-check
spec:
  source:
    namespace: egress-private      # was egress-test in this run
    pod: client
  destination:
    ip: 172.31.1.3
  packet:
    transportHeader:
      tcp:
        dstPort: 8080
        flags: 2          # SYN
```

```sh
kubectl --context $C apply -f traceflow.yaml
kubectl --context $C get traceflow egress-check -o jsonpath='{.status.phase}'   # Succeeded
kubectl --context $C get traceflow egress-check -o yaml                          # status.results
kubectl --context $C delete traceflow egress-check
```

Result:

```
phase: Succeeded
node: se-cluster-001-se-cluster-001-np01-nf8s7-mmc6z-hgscj        (source node)
    {'action': 'Forwarded', 'component': 'SpoofGuard', 'srcPodIP': '192.168.145.21'}
    {'action': 'ForwardedToEgressNode', 'component': 'Egress', 'egress': 'egress-test',
     'egressIP': '172.30.0.34', 'egressNode': 'se-cluster-001-se-cluster-001-np01-nf8s7-mmc6z-49dq4'}
    {'action': 'Forwarded', 'component': 'Forwarding', 'componentInfo': 'Output', 'tunnelDstIP': '172.30.0.34'}
node: se-cluster-001-se-cluster-001-np01-nf8s7-mmc6z-49dq4        (egress node)
    {'action': 'Received', 'component': 'Forwarding'}
    {'action': 'MarkedForSNAT', 'component': 'Egress', 'egressIP': '172.30.0.34'}
    {'action': 'ForwardedOutOfNetwork', 'component': 'Forwarding', 'componentInfo': 'Output'}
```

Read it as: on the source node the packet matches the Egress and is tunnelled
to the egress node (`ForwardedToEgressNode`); there it is marked for SNAT to
the egress IP (`MarkedForSNAT`) and leaves the cluster (`ForwardedOutOfNetwork`).

## Notes and gotchas

- **Private vs Public egress subnet (the important one).** Private: the egress
  IP identifies the namespace inside the VPC only; leaving the VPC, NSX
  translates it to the VPC's `defaultSNATIP`, shared by everything in the
  vSphere Namespace. Public: NSX routes the subnet without translation and
  advertises it outside, so the egress IP is kept end to end. A Public subnet
  is carved from the external IP block shared by the namespaces of the NSX
  project (172.17.10.0/24 here, with stale routes on .27 and .30); 16 addresses
  is the minimum. `accessMode` cannot be changed on an existing Subnet.
- **NSX chooses the addresses.** Read the Subnet and SubnetIPReservation status
  and put exactly those values into the ExternalIPPool. Without a reservation
  (the first run on 2026-10-08), NSX can hand the same address to something
  else on the subnet.
- **Order matters.** Antrea's webhook rejects an Egress whose ExternalIPPool
  does not exist yet, and an ExternalIPPool whose range overlaps another pool.
  When replacing a pool, delete the old Egress and pool first.
- **One VLAN per binding** on the same SubnetSet (101 Private, 102 Public); the
  same value goes into the ExternalIPPool's `subnetInfo.vlan`.
- **One Egress per namespace.** Several Egresses can share an egress node; each
  gets its own VLAN interface, mark and routing table.
- **Node selection:** `nodeSelector` keeps egress IPs off the control plane.
  With several workers eligible, Antrea moves an egress IP to another worker if
  its node fails (failover not tested here).
- **Pod Security:** all test pods pass VKS's enforced `restricted` profile.

## Cleanup

```sh
# cluster side first
kubectl --context $C delete -f se-cluster-001/antrea-egress-test/03-egress-public.yaml
kubectl --context $C delete -f se-cluster-001/antrea-egress-test/02-egress.yaml
kubectl --context $C delete -f se-cluster-001/antrea-egress-test/01-clients.yaml
# then NSX; the Public subnet frees its 16 shared external addresses
kubectl --context 172.17.10.2 delete -f supervisor/clusters/se-cluster-001/antrea-egress-test/egress-subnet-public.yaml
kubectl --context 172.17.10.2 delete -f supervisor/clusters/se-cluster-001/antrea-egress-test/egress-subnet.yaml
```

Delete the Egresses before the NSX side, so Antrea removes the VLAN interfaces
while the subnets still exist.
