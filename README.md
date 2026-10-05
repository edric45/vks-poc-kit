# vks-poc-kit

ArgoCD on a vSphere Supervisor, managing VKS clusters and what runs on them
from git. Built one step at a time; each step is plain YAML applied by hand
until it is understood.

## Layout

The top-level folder names **where a file is applied**:

```
supervisor/   applied to the Supervisor (context 172.17.10.2), namespace se-ns-argo
  argocd.yaml   the ArgoCD instance
  clusters/
    se-cluster-01/
      cluster.yaml               a workload cluster (to be synced by ArgoCD)
      addons/                    VKS add-ons for it, one folder per add-on
        cert-manager/
        istio/                   sidecar mode
  applicationsets/
    se-cluster-01-addons.yaml    one ArgoCD app per folder in addons/
docs/         reference for the whole project
  commands.md   finding fields, schemas and supported versions on the Supervisor
```

## Step 1: ArgoCD on the Supervisor

**Before you start:** the vSphere Namespace `se-ns-argo` exists (created in
vCenter), and you are logged in to the Supervisor.

```sh
kubectl --context 172.17.10.2 auth whoami                    # logged in?
kubectl --context 172.17.10.2 get argocdversions argocd-supported-versions -o yaml   # version still offered?
```

**Install:**

```sh
kubectl --context 172.17.10.2 apply -f supervisor/argocd.yaml
kubectl --context 172.17.10.2 -n se-ns-argo get argocd,pods -w
```

Wait for the ArgoCD resource to show `Ready`. Expect six pods: the
application controller, the ApplicationSet controller, the repo server, the
server, redis, and a redis init job that ends as `Completed`.

**Get in:**

```sh
# the UI address: EXTERNAL-IP of the argocd-server Service
kubectl --context 172.17.10.2 -n se-ns-argo get svc argocd-server

# the initial admin password
kubectl --context 172.17.10.2 -n se-ns-argo get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Open `https://<EXTERNAL-IP>` and log in as `admin`. Then change the password
(User Info > Update Password). After that, the secret above no longer holds the
current one.

**Remove it again** (if you want to start over):

```sh
kubectl --context 172.17.10.2 delete -f supervisor/argocd.yaml
# the operator leaves these two behind
kubectl --context 172.17.10.2 -n se-ns-argo delete secret argocd-initial-admin-secret argocd-redis
```
