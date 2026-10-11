# Private AI Services (PAIS) 3.0 in se-pais-tp7mm

How PAIS was installed and activated in this lab, and the problems that had to
be fixed on the way. The UI and VCF Automation steps are not in git; this page
is their record.

Labels: **documented** (Broadcom docs), **lab** (seen working here),
**unverified**.

## Layout

```
Supervisor service   PAIS 3.0.0, namespace svc-pais-*        (vCenter, Supervisor Management > Services)
se-pais-tp7mm        PAIS instance: API, ingress, workers as vSphere Pods
                     UI on 172.19.0.25
                     PAIS-managed cluster pais-b64af190 (model endpoints), API on 172.19.0.26
se-pais-001          PostgreSQL + pgvector for PAIS (se-pais-001/pais-db/, 172.19.0.24)
regional Harbor      172.17.10.8, model gallery (a public project)
```

PAIS is activated per vSphere Namespace and builds its own VKS cluster there.
It is never installed inside a workload cluster such as se-pais-001.

## 1. Install the Supervisor Service

Download **pais-3.0.0-external.yml** from the Broadcom portal (VMware Private
AI Services 3.0.0). Not `pais-3.0.0.yml`: that is the depot edition, whose
bundle points at `depot.kube-system.svc`, which this lab's depot does not hold.
Validation then fails at Signature Verification ("Image ... does not exist").

- Add a Supervisor container registry for `pais-docker.packages.broadcom.com`
  with a Broadcom token. It does not need to be the default registry.
- Both files carry version 3.0.0. To replace one with the other, deactivate
  and delete the whole service in vCenter first; otherwise the old package
  survives on the Supervisor. Check with:

```sh
kubectl --context 172.17.10.2 -n vmware-system-supervisor-services get package pais.vmware.com.3.0.0 \
  -o jsonpath='{.spec.template.spec.fetch[0].imgpkgBundle.image}{"\n"}'
```

## 2. Prerequisites in the namespace

| What | Value here | Why |
|---|---|---|
| Storage limit | 1000Gi | PAIS cluster, model VMs, se-pais-001 |
| CPU / memory **reservation** | namespace 8000 MHz / 12000 MB | vSphere Pods reserve their requests; with 0 reservation every PAIS pod with a request is refused ("parent resource pool is insufficient") |
| Org region quota reservation | all-apps-3: 10000 MHz / 16 GB | the namespace can only reserve what the org has; default was 0 |
| Database | se-pais-001/pais-db (PostgreSQL 17, pgvector 0.8.7) | required for Agent Builder and indexing (**documented**) |

## 3. Login (OIDC)

PAIS cannot use vSphere / VCF Single Sign-On (**documented**, "not supported").
Here VCF Automation is the provider:

1. Provider portal (`https://vcfa.mgmt.vks.lab/provider`) > OIDC Services >
   Relying Parties: redirect URI `https://172.19.0.25/` (trailing slash
   required), Is Public on, Requires PKCE on. Copy the client ID.
2. In the PAIS form: issuer `https://vcfa.mgmt.vks.lab/oidc`, the client ID,
   and the management vCenter's CA (VCF Automation's certificate is signed by
   it).

The UI writes the deprecated `spec.auth.oidc` field (a warning condition only).

## 4. Activate

VCF Automation > Build & Deploy > se-pais-tp7mm > Private AI.

| Field | Value |
|---|---|
| Control plane | best-effort-large, vsan-default-storage-policy |
| Database | 172.19.0.24:5432, db `pais`, user `pais`, SSL mode **Allow** (the lab DB has no TLS) |

**PAIS 3.0 bug (lab):** the DB setup job mounts key `ca.crt` from the
UI-made secret `self-managed-db-secret-*` even when no certificate is given,
so its pods end in ProviderFailed ("references non-existent secret key:
ca.crt"). Add any PEM under that key; with SSL mode Allow it is not used:

```sh
kubectl --context 172.17.10.2 -n se-pais-tp7mm patch secret <self-managed-db-secret-...> \
  --type merge -p "{\"data\":{\"ca.crt\":\"$(base64 < some-ca.pem | tr -d '\n')\"}}"
```

The job backs off (10 min and more between tries); it picks the key up on its
next try.

## 5. The PAIS cluster's node image

PAIS 3.0 pins VKr v1.34.9 on Ubuntu 24.04 (builtin-generic-v3.5.0). The
subscribed content library "vSphere Kubernetes Services" downloads images on
demand from Broadcom and **pins the publisher's TLS certificate**. Broadcom
renewed it on 2026-09-19, so new downloads failed with "cannot authenticate SSL
certificate for host wp-content.broadcom.com".

Fix: vCenter > Content Libraries > the library > Edit Settings, accept the new
certificate, then synchronize the item. This recurs at every Broadcom renewal
(next by 2027-03-26).

## 6. Model gallery

The regional Harbor (172.17.10.8) works without DNS: its token realm and
certificate use the IP. A public project means model endpoints need no pull
secret (`spec.model.pullSecrets` is optional).

- Add Harbor's CA (`https://172.17.10.8/api/v2.0/systeminfo/getcert`) in PAIS
  > CA Trust Bundles. **Any change to the CA bundles replaces the PAIS
  cluster's nodes** (about 5 minutes, PAIS NotReady meanwhile), so add CAs
  before deploying models.
- CPU endpoints use llama.cpp and GGUF models (**documented**). Push with the
  PAIS CLI; it has no login flag and reads `~/.docker/config.json`:

```sh
cd <folder with one .gguf>
vcf pais models push -s 172.17.10.8/<project> -m qwen/qwen2.5-1.5b-instruct -t q4km -x .cache
vcf pais models list -s 172.17.10.8/<project>
```

The VCF CLI 9.1 bundle ships PAIS plugin v2.1.0 against service 3.0.0
(**unverified** whether that matters).

## Status (2026-10-11)

PAIS Ready, UI reachable. Models Qwen2.5-1.5B-Instruct (q4_k_m) and
nomic-embed-text-v1.5 (Q8_0) downloaded; push and model endpoints pending.
