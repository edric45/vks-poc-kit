# vks-poc-kit

ArgoCD on a vSphere Supervisor, managing VKS clusters and what runs on them
from git. Built one step at a time; each step is plain YAML applied by hand
until it is understood.

## Layout

The top-level folder names **where a file is applied**:

```
supervisor/   applied to the Supervisor (context 172.17.10.2), namespace se-ns-vks
  argocd.yaml   the ArgoCD instance
  clusters/
    se-cluster-001/
      cluster.yaml               a workload cluster (to be synced by ArgoCD)
      addons/                    VKS add-ons for it, one folder per add-on
        cert-manager/
        gatekeeper/              OPA engine only; policies go under se-cluster-001/
        headlamp/                web UI, HTTPS through an Istio Gateway
        istio/                   sidecar mode
  applications/
    se-cluster-001.yaml          the ArgoCD app for the cluster (manual sync)
  applicationsets/
    se-cluster-001-addons.yaml   one ArgoCD app per folder in addons/
se-cluster-001/  applied inside the workload cluster (context se-cluster-001:se-cluster-001)
  storageclass-retain.yaml   vSAN default storage with reclaimPolicy Retain
  istio-gatewayclass-defaults.yaml   seccomp for Istio gateway pods (Pod Security)
  headlamp-admin.yaml        Headlamp login (works around the add-on's RBAC bug)
docs/         reference for the whole project
  commands.md   finding fields, schemas and supported versions on the Supervisor
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
