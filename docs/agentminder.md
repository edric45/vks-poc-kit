# AgentMinder 4.1 on se-pais-001

AgentMinder gives every agent its own identity (certificate based; AgentMinder
is an OIDC provider), checks every tool call against the agent's declared
mission, and records it. This page installs it in the lab, following
Broadcom's "Installation Lab Guide" (Technical Course 2), with the changes
our VKS lab needs. Object setup (agents, missions, policies) is the guide's
Part B and comes later.

Labels: **lab** = seen here; **chart** = read in the charts themselves;
**to test** = first seen at install.

## What gets installed

```
se-pais-001 (new worker pool np02: 2 x 16 vCPU / 128 GiB)
  envoy-gateway-system   Envoy Gateway v1.7.2 + GatewayClass "eg"   (next to Istio's "istio")
  agentminder            Gateway "agentminder" (VIP from Avi) + cert-manager certificate
                         ssp-infra: Percona XtraDB DB (1 node), ClickHouse + keeper, operators
                         ssp:       the platform, demo size, in-platform AI Gateway, observability
                         ssp-data:  risk schema + network data (jobs)
  https://agentminder.<ip-with-dashes>.sslip.io        sign-in, OIDC, AI Gateway
  https://mgmt-agentminder.<ip-with-dashes>.sslip.io   admin console
```

Files: `se-pais-001/agentminder/` (envoy-gateway.yaml, gateway.yaml,
values/ssp-infra.yaml, values/ssp.yaml, values/ssp-data.yaml) and the worker
pool np02 in `supervisor/clusters/se-pais-001/cluster.yaml`.

## Where we differ from the lab guide, and why

| Guide | Here | Why |
|---|---|---|
| Envoy Gateway v1.8.1, with Gateway API CRDs | **v1.7.2, Envoy's own CRDs only** | VKS installs and owns Gateway API v1.4.0. v1.8 is built for v1.5 and its CRD bundle includes a policy that refuses Gateway API CRDs older than v1.5 (chart). v1.7.2 targets v1.4.1. Cost: the optional mTLS mode `verifyAtGateway: false` needs 1.8+ |
| the ssp chart creates the Gateway | **we create it** (`existingGateway`) | the platform must know its own name at install; the sslip.io name contains the Gateway's address, which only exists once the Gateway does |
| `helm install ssp-infra` directly | **apply the ClickHouse CRDs first** | the chart refuses otherwise ("Altinity ClickHouse Operator CRDs are not installed"; chart) |
| enclave services (Elasticsearch, Kibana, Prometheus, Grafana) | **skipped**, Fluent Bit off | ~7 cores / 16 GB; needed for a supportable install, not for the lab. Agent telemetry (ClickHouse) stays on |
| a DNS name you own | **sslip.io** | no lab DNS we control; `agentminder.172-19-0-41.sslip.io` resolves to 172.19.0.41 anywhere with internet DNS |
| any Kubernetes | namespace labelled **privileged** | the database and ClickHouse operators, NATS, Hazelcast and the collector set no seccomp profile (chart); VKS enforces "restricted" |
| install with Helm | still Helm, **not ArgoCD** | the charts look up the database, ClickHouse and the DB secret in the cluster at install time; ArgoCD renders charts without a cluster |

Why se-pais-001: it sits in the PAIS project's VPC, next to PAIS, the MCP
servers, agentgateway and Phoenix, which AgentMinder is meant to govern.

Sizing (chart): the ssp chart requests about 9.5 cores / 21 GiB; with the
database and ClickHouse about 12 cores / 26 GiB. se-pais-001's two np01
workers are already busy (one at 99 % of its CPU requests), so AgentMinder
gets its own pool, np02: 2 x best-effort-4xlarge (16 vCPU / 128 GiB).

Storage quota (lab, 2026-10-15): se-pais-tp7mm has 1000 GiB on
vsan-default-storage-policy and already uses ~725 GiB (PAIS's own cluster,
models, se-pais-001). An np01 worker takes 120 GiB, an np02 worker ~60 GiB
(root 20 + containerd 40); with AgentMinder's and Phoenix's volumes (~55 GiB)
the namespace ends near 900 GiB. Check before adding anything:
`kubectl --context 172.17.10.2 -n se-pais-tp7mm get storagepolicyquota -o yaml`.

## Before you start

- **Registry token** for `securityservices.packages.broadcom.com` (Broadcom
  Support Portal > My Downloads > Registry Tokens). Tested 2026-10-15:
  `admin-svc` lists tag `4.1.0.1578`.
- **Helm 3.10+** on the Mac. Latest v3 is 3.22.1:
  ```sh
  curl -LO https://get.helm.sh/helm-v3.22.1-darwin-arm64.tar.gz
  curl -LO https://get.helm.sh/helm-v3.22.1-darwin-arm64.tar.gz.sha256sum
  shasum -a 256 -c helm-v3.22.1-darwin-arm64.tar.gz.sha256sum
  tar xzf helm-v3.22.1-darwin-arm64.tar.gz && sudo install darwin-arm64/helm /usr/local/bin/helm
  helm version
  ```
- Commit and push this repo (step 1 is an ArgoCD sync).

Shorthand used below:

```sh
k()  { kubectl --context se-pais-001:se-pais-001 "$@"; }
REPO=https://ssp_helm_charts.storage.googleapis.com/agentminder-charts/
```

## 1. Worker pool np02

The Application lives in the se-pais ArgoCD (172.19.0.23), CLI context `se-pais`:

```sh
argocd --argocd-context se-pais app diff se-pais-001     # only: a new pool se-pais-001-np02
argocd --argocd-context se-pais app sync se-pais-001
kubectl --context 172.17.10.2 -n se-pais-tp7mm get machines -w   # 2 new np02 machines Running
k get nodes                                                      # 5 Ready (1 control plane + 4)
```

A new pool only adds VMs; np01 and its pods are not touched. If the new
machines stay Provisioning, look for quota events
(`kubectl --context 172.17.10.2 -n se-pais-tp7mm get events --sort-by=.lastTimestamp | tail`):
storage quota (above) or the VCF Automation organization's region quota.

## 2. Envoy Gateway

```sh
# its own CRDs only (gateway.envoyproxy.io); Gateway API CRDs stay VKS's
helm install eg-crds oci://docker.io/envoyproxy/gateway-crds-helm --version v1.7.2 \
  --set crds.gatewayAPI.enabled=false --set crds.envoyGateway.enabled=true
# the controller, without CRDs
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.7.2 \
  -n envoy-gateway-system --create-namespace --skip-crds
k -n envoy-gateway-system wait deployment/envoy-gateway --for=condition=Available --timeout=5m

k apply -f se-pais-001/agentminder/envoy-gateway.yaml
k get gatewayclass eg              # ACCEPTED True
```

## 3. Namespace, Gateway, name, certificate, registry secret

```sh
k apply -f se-pais-001/agentminder/gateway.yaml
k -n agentminder get gateway agentminder -w      # ADDRESS appears, PROGRAMMED True
```

Avi picks the address from the 172.19.0.0/16 block that VCF Automation
organizations share. Check it against the known stale routes (`docs/pais.md`;
placeholders hold the affected 172.19.0.x addresses) and the address another
organization holds (172.19.0.27). Then the name:

```sh
IP=$(k -n agentminder get gateway agentminder -o jsonpath='{.status.addresses[0].value}')
HOST=agentminder.$(echo $IP | tr . -).sslip.io; echo $HOST mgmt-$HOST
nslookup $HOST          # -> $IP
```

Put `$HOST` and `mgmt-$HOST` in place of the two placeholders in
`gateway.yaml` (Certificate dnsNames), and `$HOST` as `ssp.ingress.host` in
`values/ssp.yaml`. Commit, then:

```sh
k apply -f se-pais-001/agentminder/gateway.yaml
k -n agentminder get certificate agentminder-tls          # READY True
k -n agentminder get secret agentminder-tls -o jsonpath='{.data.tls\.crt}' | base64 -d \
  | openssl x509 -noout -ext subjectAltName                # both names
```

Registry secret, without the token in shell history:

```sh
read -rs AM_TOKEN      # paste the token, Enter
k -n agentminder create secret docker-registry ssp-gcr-registry-creds \
  --docker-server=securityservices.packages.broadcom.com \
  --docker-username='<portal e-mail>' --docker-password="$AM_TOKEN"
unset AM_TOKEN
```

## 4. ssp-infra: database and ClickHouse

```sh
cd $(mktemp -d) && helm pull ssp-infra --repo $REPO --version 4.1.0+1578 --untar
# ClickHouse operator CRDs first (the chart refuses to install without them)
k apply --server-side -f ssp-infra/charts/altinity-clickhouse-operator/crds/
cd -
helm install infra-ssp ssp-infra --repo $REPO --version 4.1.0+1578 -n agentminder \
  -f se-pais-001/agentminder/values/ssp-infra.yaml --timeout=120m
k -n agentminder wait jobs --selector app.kubernetes.io/name=ssp-infra-create-db-job \
  --for=condition=complete --timeout=10m
k -n agentminder get pods,pvc       # pxc, clickhouse, keeper, operators Running; PVCs Bound
```

## 5. ssp: the platform

```sh
helm install ssp ssp --repo $REPO --version 4.1.0+1578 -n agentminder \
  -f se-pais-001/agentminder/values/ssp.yaml --timeout=120m
k -n agentminder get pods -w                       # all Running / Completed
k -n agentminder get httproute                     # ssp-ssp-router-full, -mgmt: Accepted
```

Hazelcast Enterprise needs a licence key and the guide sets none; Broadcom's
image presumably carries it (**to test**). If `ssp-hazelcast-*` logs a licence
error, ask the AgentMinder team.

## 6. ssp-data: risk and network data

```sh
helm install data-ssp ssp-data --repo $REPO --version 2026.26 -n agentminder \
  -f se-pais-001/agentminder/values/ssp-data.yaml --timeout=60m
k -n agentminder get jobs | grep -i data          # Completed
```

## 7. Check and first login

```sh
curl -sk "https://$HOST/default/.well-known/openid-configuration?sspinfo=true" | head -c 400
# JSON with "issuer" = AgentMinder is up as an OIDC provider

# bootstrap admin (valid 48 HOURS: create a durable admin within that window)
k -n agentminder get secret ssp-ssp-secret-defaulttenantadminuser -o jsonpath='{.data.userLoginId}' | base64 -d; echo
k -n agentminder get secret ssp-ssp-secret-defaulttenantadminuser -o jsonpath='{.data.userPassword}' | base64 -d; echo
```

Admin console: `https://mgmt-$HOST` (or `https://$HOST`, as in the guide).
The certificate is from the lab's private CA; to drop browser warnings, trust
it: `k -n cert-manager get secret selfsigned-root-ca -o jsonpath='{.data.ca\.crt}' | base64 -d`.

## Troubleshooting

| Symptom | Look at |
|---|---|
| Gateway without ADDRESS | `k -n envoy-gateway-system get pods` (proxy pods rejected by Pod Security? the EnvoyProxy in envoy-gateway.yaml); `k -n agentminder describe gateway agentminder` |
| Certificate not READY | `k -n agentminder describe certificate agentminder-tls`; cert-manager logs |
| `ImagePullBackOff` | the `ssp-gcr-registry-creds` secret (step 3) |
| ssp install fails "No PXC object" / "ClickHouse is not available" | ssp-infra not finished: step 4's checks |
| pods Pending | not enough room: `k describe pod`; workers (step 1) |
| Helm step fails with x509 from a webhook | Gatekeeper's webhook certificate rotates now and then on se-pais-001; retry |
| HTTPRoute not Accepted | `k -n agentminder describe httproute`; host must match the certificate and values |

## Removing it

```sh
helm uninstall data-ssp ssp infra-ssp -n agentminder
k delete -f se-pais-001/agentminder/gateway.yaml        # namespace, Gateway, certificate
helm uninstall eg -n envoy-gateway-system && helm uninstall eg-crds
```

Left behind on purpose by Helm: the database PVC (cluster default class,
Retain, so its PV and the backing PVC on the Supervisor in se-pais-tp7mm stay),
and the CRDs (Percona, MySQL operator, ClickHouse). Delete them by hand when
done, then remove pool np02 from `supervisor/clusters/se-pais-001/cluster.yaml` and sync.

## Status

Written and checked 2026-10-15: the four values/manifest files render
(helm template, chart lookups bypassed) and pass server dry-runs; Envoy
Gateway versions and CRD contents read from its charts; token tested.
Nothing installed yet.
