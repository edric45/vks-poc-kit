# CIS Kubernetes Benchmark scan of a VKS cluster

How to check a VKS workload cluster against the CIS Kubernetes Benchmark, what
the results on this lab looked like, and how to keep the scan current. All of
it is read-only on the cluster except the temporary `kube-bench` namespace.

## What this does and does not cover

- **Covers:** CIS Kubernetes Benchmark **2.0**, the generation written for
  Kubernetes 1.34 and 1.35: API server, scheduler, controller manager, etcd,
  kubelet, and the (manual) policy section.
- **Does not cover:** the node operating system. There is no CIS benchmark
  for Photon OS 5.0; Broadcom publishes **STIG** results for VKS node images
  instead, one report per VKr release:
  https://github.com/vmware/dod-compliance-and-automation/tree/master/vcf/9.x/docs/reports
- **Not the newest CIS:** CIS has published 2.1.0. No free tool checks it
  yet; CIS's own CIS-CAT Pro (paid) checks 2.0.1.

Broadcom does not state that VKS is CIS hardened; it states STIG hardening.
A kube-bench result is your own evidence for one cluster, not a vendor
statement.

## Where the pieces come from

Nothing here is a VKS add-on; the add-on catalog has no compliance scanner.

| Piece | Source |
|---|---|
| kube-bench | Aqua Security, open source (Apache 2.0): https://github.com/aquasecurity/kube-bench |
| Container image | `docker.io/aquasec/kube-bench:v0.16.0` (latest release) |
| CIS 2.0 checks | kube-bench `main` branch, commit `e278a59`, embedded as ConfigMaps in the manifest |
| Broadcom's guidance | KB 417426, running kube-bench by hand over SSH on a node |

The manifest is not a downloaded file; it was built from these upstream files
and adapted for VKS (see the next sections):

- Job templates, release v0.16.0:
  [job-master.yaml](https://github.com/aquasecurity/kube-bench/blob/v0.16.0/job-master.yaml)
  (control-plane Job) and
  [job-node.yaml](https://github.com/aquasecurity/kube-bench/blob/v0.16.0/job-node.yaml)
  (worker Job).
- CIS 2.0 checks, commit `e278a59` on `main` (the two ConfigMaps):
  [cfg/cis-2.0/](https://github.com/aquasecurity/kube-bench/tree/e278a5915a982f2756026de6e2f9ca77077ef9f9/cfg/cis-2.0)
  and
  [cfg/config.yaml](https://github.com/aquasecurity/kube-bench/blob/e278a5915a982f2756026de6e2f9ca77077ef9f9/cfg/config.yaml).

The CIS 2.0 checks are not in a kube-bench release yet. Adding them needed
no code change, only new check files, so the released v0.16.0 image runs
them when pointed at those files with `--config-dir`.

## Run the scan

The manifest is `se-cluster-001/kube-bench-cis-2.0.yaml`. It creates a
namespace, two Jobs (one on a control-plane node, one on a worker) and two
ConfigMaps. The cluster pulls the image from Docker Hub.

```sh
C=se-cluster-001:se-cluster-001
kubectl --context $C apply -f se-cluster-001/kube-bench-cis-2.0.yaml
kubectl --context $C -n kube-bench wait --for=condition=complete job --all --timeout=300s
kubectl --context $C -n kube-bench logs job/kube-bench-controlplane > cis-2.0-controlplane.txt
kubectl --context $C -n kube-bench logs job/kube-bench-worker       > cis-2.0-worker.txt
kubectl --context $C delete ns kube-bench
```

Each Job takes under 20 seconds. Keep the result files out of git: they
describe a specific cluster.

## What had to change for VKS

The upstream kube-bench Jobs do not run on VKS as they are:

- **Pod Security.** VKS enforces the `restricted` profile in every
  unlabelled namespace. kube-bench needs `hostPID` and host paths, so the
  manifest labels its own namespace `privileged`, as VKS does for its system
  namespaces. Deleting the namespace removes the exception.
- **Control-plane taint.** VKS taints control-plane nodes. The control-plane
  Job tolerates `node-role.kubernetes.io/control-plane:NoSchedule`, or it
  stays Pending.
- **Benchmark selection.** kube-bench picks a benchmark from the Kubernetes
  version, and its released table stops at 1.34. The manifest names
  `cis-2.0` explicitly.
- **One worker is enough.** Every worker boots from the same VKr image.

## Results on this lab (VKr v1.35.6, Photon OS 5.0, October 2026)

| Scope | Pass | Fail | Warn |
|---|---|---|---|
| Control plane: API server, scheduler, controller manager | 50 | 3 | 7 |
| etcd | 7 | 0 | 0 |
| Control plane configuration: auth, logging | 1 | 0 | 4 |
| Policies: RBAC, pod security, network, secrets | 0 | 0 | 34 |
| Worker: kubelet | 19 | 1 | 5 |
| **Total** | **77** | **4** | **50** |

The four failures are all settings inside the VKr image. VKr images are
immutable (Broadcom's STIG report: changes to them are not supported and do
not persist), so a customer documents them as exceptions or raises them with
Broadcom; they cannot be fixed on the node.

| Check | Finding | Why on VKS |
|---|---|---|
| 1.1.12 | etcd data directory not owned by `etcd:etcd` | etcd runs as a root static pod (kubeadm layout). Broadcom's STIG documents accept the same point. |
| 1.2.5 | API server has no `--kubelet-certificate-authority` | Kubelets use self-signed serving certificates; matches the kubelet TLS exceptions in Broadcom's VKS STIG report. |
| 1.2.30 | `--service-account-extend-token-expiration` not `false` | New in CIS 2.0; VKS keeps the Kubernetes default (`true`). |
| 4.1.1 | kubelet service file more permissive than `600` | Image file permission. |

The warnings are checks kube-bench cannot automate:

- **Policies, 34 warnings:** least-privilege RBAC, pod security, network
  policies, secret handling. These are attested, not scanned. Evidence on
  VKS: Pod Security Admission (`restricted` enforced by default) and
  Gatekeeper (installed as an add-on, policies are your own).
- **Worker, two worth noting:** 4.2.13 (no per-pod PID limit) and 4.2.14
  (kubelet `--seccomp-default` not set). The second is covered in practice
  by Pod Security `restricted`, which requires every pod to set a seccomp
  profile.
- **4.1.3 / 4.1.4** (kube-proxy kubeconfig) are manual in CIS 2.0: kube-proxy
  runs in a container on VKS and has no kubeconfig file on the host.

## Keep it current

**When kube-bench releases CIS 2.0 (or 2.0.1/2.1) support:** change the image
tag in both Jobs, remove the `--config-dir`/`--config` arguments, the
`cfg-2-0` volume and mounts, and the two ConfigMaps at the end of the file.
Check the release notes for the benchmark name to pass to `--benchmark`.

**To refresh the embedded checks from kube-bench `main` before that:**

```sh
SHA=$(curl -sf https://api.github.com/repos/aquasecurity/kube-bench/commits/main \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["sha"])')
mkdir -p /tmp/kb/cis-2.0
curl -sfL -o /tmp/kb/config.yaml https://raw.githubusercontent.com/aquasecurity/kube-bench/$SHA/cfg/config.yaml
for f in config controlplane etcd master node policies; do
  curl -sfL -o /tmp/kb/cis-2.0/$f.yaml https://raw.githubusercontent.com/aquasecurity/kube-bench/$SHA/cfg/cis-2.0/$f.yaml
done
kubectl create configmap kube-bench-cfg -n kube-bench --from-file=config.yaml=/tmp/kb/config.yaml --dry-run=client -o yaml
kubectl create configmap kube-bench-cis-2.0 -n kube-bench --from-file=/tmp/kb/cis-2.0 --dry-run=client -o yaml
```

Replace the two ConfigMaps at the end of the manifest with that output and
update the commit in the header comment.

**Known kube-bench issues to watch:** #2124 (CIS 2.0.1 support), #2151
(kube-proxy checks when the kubeconfig is absent).

## Without access to Docker Hub

Mirror `docker.io/aquasec/kube-bench:v0.16.0` into a registry the cluster can
pull from and change the `image:` lines. Everything else in the manifest is
self-contained.
