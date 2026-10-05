# Commands: finding out what the Supervisor accepts

Reference for the whole project. Use these before writing YAML for any
resource, and whenever a version or field is in doubt. All of them are
read-only.

`K` below is short for `kubectl --context 172.17.10.2`:

```sh
K="kubectl --context 172.17.10.2"
```

## Am I logged in?

```sh
$K auth whoami
```

Logins expire after about 10 hours. `Unauthorized` means log in again.

## Writing YAML for any resource

The Supervisor stores a **schema** for every resource type it knows: a
CustomResourceDefinition (CRD), registered when the operator behind that
resource was installed. It is not a template; it lists the fields and what
they mean. It is also always correct for *this* Supervisor's version, while
documentation can lag behind.

The recipe:

```sh
# 1. Find the resource type.
$K api-resources | grep -i <word>
#    Columns (the header is hidden by grep):
#      NAME        plural name, used with get/explain (singular and KIND work too)
#      SHORTNAMES  optional shortcut, e.g. `get me` for managedentities
#      APIVERSION  goes in `apiVersion:`
#      NAMESPACED  true = the YAML needs metadata.namespace
#      KIND        goes in `kind:`

# 2. Read its fields, one level at a time. With no field, explain shows the
#    top level: apiVersion, kind, metadata, spec, status. `spec` is what you
#    write (the desired state); `status` is what the operator reports back.
$K explain <resource>
$K explain <resource>.spec
$K explain <resource>.spec.<field>

#    Or the whole field tree at once.
$K explain <resource>.spec --recursive

# 3. Write the minimal YAML, then let the Supervisor validate it without
#    creating anything.
$K apply --dry-run=server -f <file>.yaml
```

The raw schema, if `explain` is not enough:

```sh
$K get crd | grep -i <word>
$K get crd <crd-name> -o yaml
```

**The exception:** a dry run does not check the `values` of a VKS add-on
(AddonConfig). VKS validates them later, and a mistake only shows up as a
failed add-on after it is applied. See [VKS add-ons](#vks-add-ons).

### Example: ArgoCD

```sh
$K api-resources | grep -i argocd          # argocds, argocdversions, ...
$K explain argocd.spec                     # enableLoadBalancer, applicationSet, ...
$K explain argocd.spec.applicationSet      # enabled, replicas, resources, proxy
$K get crd argocds.argocd-service.vsphere.vmware.com -o yaml
$K apply --dry-run=server -f supervisor/argocd.yaml
```

## What versions are on offer

These lists **change over time**, so check them before installing or
upgrading anything.

```sh
# ArgoCD: valid values for spec.version
$K get argocdversions argocd-supported-versions -o yaml

# Kubernetes versions for clusters (READY and COMPATIBLE must be True)
$K get kubernetesreleases

# ClusterClasses (the cluster blueprints VKS provides)
$K -n vmware-system-vks-public get clusterclass

# VKS add-ons and their releases
$K -n vmware-system-vks-public get addonreleases
$K -n vmware-system-vks-public get addonreleases | grep <addon>
```

## Cluster settings (ClusterClass variables)

A Cluster's `topology.variables` (vmClass, volumes, kubernetes, node, ...)
are not in the Cluster CRD: each ClusterClass defines its own. `explain` on
`cluster` therefore stops at `variables`. Read them from the ClusterClass:

```sh
CC=builtin-generic-v3.7.0

# the variable names
$K -n vmware-system-vks-public get clusterclass $CC \
  -o jsonpath='{range .status.variables[*]}{.name}{"\n"}{end}'

# everything, with descriptions, allowed values and defaults (long)
$K -n vmware-system-vks-public get clusterclass $CC -o yaml
```

Each description says which scopes the variable supports (cluster,
controlPlane, workers) and whether changing it causes a rollout, which means
nodes are replaced with new VMs.

## What a vSphere Namespace can use

The VM classes and storage bound to the namespace in vCenter. A cluster can
only use these.

```sh
$K -n se-ns-vks get virtualmachineclass
$K -n se-ns-vks get storageclass
$K -n se-ns-vks get resourcequota        # any limits on CPU, memory, storage
```

## VKS add-ons

Each add-on release has its own settings schema, an AddonConfigDefinition.
Its name is the release name with `---` in front of `vmware`:

```sh
# release:    cert-manager.kubernetes.vmware.com.1.20.2-vmware.1-vks.1
# definition: cert-manager.kubernetes.vmware.com.1.20.2---vmware.1-vks.1
$K -n vmware-system-vks-public get addonconfigdefinitions | grep <addon>
$K -n vmware-system-vks-public get addonconfigdefinition <definition> -o yaml
```

## What is running in the namespace

```sh
$K -n se-ns-vks get argocd,pods
$K -n se-ns-vks get cluster,machines
$K -n se-ns-vks get addoninstall,addonconfig
$K -n se-ns-vks get vm,pvc
$K -n se-ns-vks get events --sort-by=.lastTimestamp | tail -20   # what just happened
```
