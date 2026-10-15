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
h()  { helm --kube-context se-pais-001:se-pais-001 "$@"; }   # Helm otherwise uses the CURRENT kubectl context
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
# its own CRDs only (8 x gateway.envoyproxy.io); Gateway API CRDs stay VKS's.
# Rendered and applied server-side, NOT `helm install`: the bundle is ~2 MB and
# Helm's release Secret is limited to 1 MiB ("Too long: may not be more than
# 1048576 bytes", lab 2026-10-15).
helm template eg-crds oci://docker.io/envoyproxy/gateway-crds-helm --version v1.7.2 \
  --set crds.gatewayAPI.enabled=false --set crds.envoyGateway.enabled=true \
  | k apply --server-side -f -
k get crd | grep envoyproxy.io                     # 8
# the controller, without CRDs (every helm command needs the context)
h install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.2 -n envoy-gateway-system --create-namespace --skip-crds
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
h install infra-ssp ssp-infra --repo $REPO --version 4.1.0+1578 -n agentminder \
  -f se-pais-001/agentminder/values/ssp-infra.yaml --timeout=120m
k -n agentminder wait jobs --selector app.kubernetes.io/name=ssp-infra-create-db-job \
  --for=condition=complete --timeout=10m
k -n agentminder get pods,pvc       # pxc, clickhouse, keeper, operators Running; PVCs Bound
```

## 5. ssp: the platform

```sh
h install ssp ssp --repo $REPO --version 4.1.0+1578 -n agentminder \
  -f se-pais-001/agentminder/values/ssp.yaml --timeout=120m
k -n agentminder get pods -w                       # all Running / Completed
k -n agentminder get httproute                     # ssp-ssp-router-full, -mgmt: Accepted
```

Hazelcast licence: chart 4.1.0+1578 ships `hazelcast-enterprise:5.7.0-p3`,
whose built-in Hazelcast Enterprise licence **expired on 29 Sep 2026** (all
AgentMinder versions; Broadcom, AgentMinder 4.1.1 release notes). The pod
then exits with `InvalidLicenseException: Enterprise License has expired!`
and every service that needs the cache stays unready. `values/ssp.yaml`
therefore pins Broadcom's renewed image, `5.7.1-p3` (release notes, "Steps to
update the Hazelcast license", for 4.0.2 and later).

If an install already runs the old image (as ours did, lab 2026-10-15):

```sh
h upgrade ssp ssp --repo $REPO --version 4.1.0+1578 -n agentminder \
  -f se-pais-001/agentminder/values/ssp.yaml --set ssp.featureFlags.dataseed.enabled=false --timeout=120m
# the StatefulSet (OrderedReady) won't replace a pod that never became Ready:
k -n agentminder delete pod ssp-hazelcast-entp-0
k -n agentminder get pod ssp-hazelcast-entp-0 -w        # 5.7.1-p3, 1/1 Running
```

`dataseed.enabled=false` skips the seeding hook on that upgrade (the
database is already seeded), as in Broadcom's command.

## 6. ssp-data: risk and network data

```sh
h install data-ssp ssp-data --repo $REPO --version 2026.26 -n agentminder \
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
| `ImagePullBackOff` | the `ssp-gcr-registry-creds` secret (step 3), and that `values/ssp.yaml` sets `ssp.global.ssp.registry.existingSecrets` (the guide's `global.image.pullSecretNames` alone leaves AgentMinder's own pods without it; registry answers 403) |
| `helm install ssp` hangs | the pre-install hook job `ssp-ssp-dataseed-job` isn't finishing: `k -n agentminder get pods -l job-name=ssp-ssp-dataseed-job` and its events |
| reinstall refused: "active SSP dataseed process ... do not delete this job" | the chart blocks while that job exists and is active. If its containers NEVER STARTED (init waiting on an image pull), deleting the job is safe: `k -n agentminder delete job ssp-ssp-dataseed-job`, `h uninstall ssp -n agentminder`, install again. If `ssp-db-init` or `ssp-db-dataseed` ran, follow the chart's message instead (save the logs; `dbinitlog-saved-confirmed`) |
| "no matches for kind ServiceMonitor / PodMonitor" | Prometheus Operator objects; VKS's Prometheus has no such types. Both are switched off in our values |
| ssp install fails "No PXC object" / "ClickHouse is not available" | ssp-infra not finished: step 4's checks |
| pods Pending | not enough room: `k describe pod`; workers (step 1) |
| Helm step fails with x509 from a webhook | Gatekeeper's webhook certificate rotates now and then on se-pais-001; retry |
| HTTPRoute not Accepted | `k -n agentminder describe httproute`; host must match the certificate and values |

## Removing it

```sh
h uninstall data-ssp ssp infra-ssp -n agentminder
k delete -f se-pais-001/agentminder/gateway.yaml        # namespace, Gateway, certificate
h uninstall eg -n envoy-gateway-system
helm template eg-crds oci://docker.io/envoyproxy/gateway-crds-helm --version v1.7.2 \
  --set crds.gatewayAPI.enabled=false --set crds.envoyGateway.enabled=true | k delete -f -
```

Left behind on purpose by Helm: the database PVC (cluster default class,
Retain, so its PV and the backing PVC on the Supervisor in se-pais-tp7mm stay),
and the CRDs (Percona, MySQL operator, ClickHouse). Delete them by hand when
done, then remove pool np02 from `supervisor/clusters/se-pais-001/cluster.yaml` and sync.

## Status

2026-10-15, se-pais-001 (lab): np02 pool up; Envoy Gateway v1.7.2 (CRDs via
server-side apply) and class eg Accepted; Gateway agentminder at
172.19.0.31, certificate for agentminder / mgmt-agentminder
.172-19-0-31.sslip.io; ssp-infra deployed (PXC ready, ClickHouse and keeper
Completed); ssp deployed, then upgraded to Broadcom's renewed Hazelcast image
5.7.1-p3 at securityservices.packages.broadcom.com/hazelcast-enterprise (the
built-in licence of 5.7.0-p3 expired 29 Sep 2026; the new one runs to 2099;
step 5). All 25 pods Ready; OIDC discovery answers (build 4.1.0.1578).
After Hazelcast came back, scheduler (~2.5 min start) and opa (waits for the
scheduler) needed a few restarts to settle. ssp-data not installed yet.
AgentMinder 4.1.1 (charts 4.1.1+1673) is in https://ssp_helm_charts.storage.googleapis.com
(the repo root, not /agentminder-charts/, which stops at 4.1.0).
