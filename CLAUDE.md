# CLAUDE.md

ArgoCD on a vSphere Supervisor, managing VKS clusters and what runs on them
from git. README.md is the runbook, one step at a time.

## How the user works here

- **Start basic, evolve.** Plain YAML first. Add templating (Helm,
  ApplicationSets) or automation only when the user asks, or when copying
  files has clearly become the problem. Then propose it; don't add it.
- **The user runs every step against the Supervisor and clusters.** You write
  and validate the files. Do not `kubectl apply/delete`, `argocd app sync` or
  `argocd app delete` unless the user asks for that action in that message.
- Read-only checks are fine and encouraged: `kubectl get/explain`,
  `kubectl apply --dry-run=server`, `argocd app diff`.
- Commit or push only when asked. The repo is public (GitHub
  `edric45/vks-poc-kit`): no credentials, kubeconfigs or customer data.

## Layout rule

The top-level folder names the API server a file is applied to:
`supervisor/` goes to the Supervisor. Workload clusters will get their own
folders. A manifest in the wrong tree either fails or does something
surprising.

## Docs

`docs/commands.md` is the project's command reference: how to find fields,
schemas and supported versions on the Supervisor. When a new step needs a
discovery or check command the user will want again, add it there. Keep
step-specific commands in README.md.

## Environment (SE lab)

- Supervisor context `172.17.10.2`; vSphere Namespace `se-ns-vks`, created in
  vCenter. ArgoCD and the workload clusters share it.
- Logins expire about every 10h. If `kubectl` returns Unauthorized, ask the
  user to log in again.
- The earlier hand-built lab is `~/se-gitops` (tag `v0-lab-2026-10-04`). Use it
  as a reference for cluster, add-on, Istio and Gateway API YAML that already
  worked.

## VKS facts that bite

- The ArgoCD operator is built into the Supervisor; applying the `ArgoCD`
  resource is the whole install. Its `spec.version` must be on the mutable
  supported list (`get argocdversions argocd-supported-versions`), or the
  instance reports Failed.
- Namespace Self-Service is off: the vSphere Namespace, its storage and
  VM-class bindings are vCenter-only. `kubectl auth can-i` and server dry-run
  wrongly report that you can create them.
- Deleting a `Cluster` deletes its VMs and leaves its volumes behind as PVCs
  on the Supervisor (label `<cluster>/TKGService`). Any Cluster managed by
  ArgoCD needs `argocd.argoproj.io/sync-options: Prune=false,Delete=false`.
- Never `delete addonconfig --all` in a namespace. Platform add-ons such as
  the Antrea CNI keep AddonConfigs there too.

## Style

Comments explain *why* (VKS quirks, what breaks), not what the YAML says.
