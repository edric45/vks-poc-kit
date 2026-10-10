# vks-poc-kit

ArgoCD on a vSphere Supervisor, managing VKS clusters and what runs on them
from git. Built one step at a time; each step is plain YAML applied by hand
until it is understood.

## Layout

The top-level folder names **where a file is applied**:

```
supervisor/   applied to the Supervisor (context 172.17.10.2); each file names its
              namespace: se-ns-vks (vCenter-made) or se-pais-tp7mm (Automation-made)
  argocd.yaml   the ArgoCD instance
  argocd-se-pais.yaml   a second ArgoCD, in se-pais-tp7mm (VCF Automation namespace, for PAIS)
  argocd-se-pais-vip.yaml   its UI/API address, 172.19.0.23 (operator's own LB is off: stale NSX routes)
  clusters/
    se-cluster-001/
      cluster.yaml               a workload cluster (to be synced by ArgoCD)
      addons/                    VKS add-ons for it, one folder per add-on
        cert-manager/
        gatekeeper/              OPA engine only; policies go under se-cluster-001/
        headlamp/                web UI, HTTPS through an Istio Gateway
        istio/                   sidecar mode
      antrea-egress-test/        NSX subnet + observer for the Antrea Egress test (by hand)
    se-pais-001/                 the PAIS support cluster in se-pais-tp7mm, same add-ons
  applications/
    se-cluster-001.yaml          the ArgoCD app for the cluster (manual sync)
    se-pais-001.yaml             the same for se-pais-001 (Application lives in se-pais-tp7mm)
    se-pais-001-pais-db.yaml     PAIS's database inside se-pais-001 (automatic sync)
  applicationsets/
    se-cluster-001-addons.yaml   one ArgoCD app per folder in addons/
    se-pais-001-addons.yaml      the same for se-pais-001
se-cluster-001/  applied inside the workload cluster (context se-cluster-001:se-cluster-001)
  storageclass-retain.yaml   vSAN default storage with reclaimPolicy Retain
  istio-gatewayclass-defaults.yaml   seccomp for Istio gateway pods (Pod Security)
  headlamp-admin.yaml        Headlamp login (works around the add-on's RBAC bug)
  kube-bench-cis-2.0.yaml    one-off CIS Kubernetes Benchmark scan (docs/cis-scan.md)
  antrea-egress-test/        Antrea Egress test: clients, egress IP pool, Egress
se-pais-001/     applied inside se-pais-001 (context se-pais-001:se-pais-001)
  storageclass-retain.yaml, istio-gatewayclass-defaults.yaml, headlamp-admin.yaml   as for se-cluster-001
  pais-db/postgres.yaml      PostgreSQL + pgvector for PAIS on VIP 172.19.0.24, synced by ArgoCD (password by hand)
docs/         reference for the whole project
  commands.md   finding fields, schemas and supported versions on the Supervisor
  cis-scan.md   CIS benchmark scan: how to run, results on this lab, keeping it current
```

## Step 1: ArgoCD on the Supervisor

**Before you start:** the vSphere Namespace `se-ns-vks` exists (created in
vCenter), and you are logged in to the Supervisor.

```sh
kubectl --context 172.17.10.2 auth whoami                    # logged in?
kubectl --context 172.17.10.2 get argocdversions argocd-supported-versions -o yaml   # version still offered?
```

**Install:**

```sh
kubectl --context 172.17.10.2 apply -f supervisor/argocd.yaml
kubectl --context 172.17.10.2 -n se-ns-vks get argocd,pods -w
```

Wait for the ArgoCD resource to show `Ready`. Expect six pods: the
application controller, the ApplicationSet controller, the repo server, the
server, redis, and a redis init job that ends as `Completed`.

**Get in:**

```sh
# the UI address: EXTERNAL-IP of the argocd-server Service
kubectl --context 172.17.10.2 -n se-ns-vks get svc argocd-server

# the initial admin password
kubectl --context 172.17.10.2 -n se-ns-vks get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Open `https://<EXTERNAL-IP>` and log in as `admin`. Then change the password
(User Info > Update Password). After that, the secret above no longer holds the
current one.

**Remove it again** (if you want to start over):

```sh
kubectl --context 172.17.10.2 delete -f supervisor/argocd.yaml
# the operator leaves these two behind
kubectl --context 172.17.10.2 -n se-ns-vks delete secret argocd-initial-admin-secret argocd-redis
```
