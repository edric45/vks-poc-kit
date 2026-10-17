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
- **That token goes stale silently** (lab, 2026-10-16): a new registry token
  from the Broadcom Support Portal (made for AgentMinder) invalidated the one
  in the Supervisor registry. Running pods kept their images; the next new PAIS
  pod (Prometheus) failed: `failed to fetch oauth token ...
  pais-docker.packages.broadcom.com ... 401`. PAIS's pull secret
  `pais-pais-reg-creds-*` is a copy (secretgen) of
  `kube-system/image-registry-credentials`, so fix it at the source: vCenter >
  Supervisor > Configure > Container Registries, edit the entry, current token.
  **Warning:** saving that entry rolls the nodes of every VKS cluster on the
  Supervisor, all namespaces and teams (2026-10-16: ~30 new machines in 9
  namespaces, PAIS's own cluster included, so PAIS and its model endpoints are
  down until the rollout ends). Announce it first; on a shared Supervisor,
  agree a time. Adding a registry does the same: se-pais-001 started a new
  MachineSet 28 s after this registry was created (2026-10-10 08:29:37Z). The
  entry is a cluster-scoped `ClusterRegistryConfig` sharing the Secret
  `kube-system/image-registry-credentials` with the other registries; the one
  cluster on a custom ClusterClass (custom-cidr-generic) did not roll, so the
  built-in ClusterClass is what carries registries into the nodes.
  For a customer: a service account's token, not a person's.
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

**Copy the right client ID (lab):** VCF Automation also lists each
Supervisor's own login client (name `vcsa-…`, redirect
`https://<supervisor>/wcp/pinniped/callback`, not public). PAIS was first given
that ID by mistake; VCF Automation then ends at `/login/loginFailure` before any
login page. The PAIS client is the one with redirect `https://172.19.0.25/`.
With the right ID, the generic OIDC type works on VCF 9.1.0 for all-apps-3
users. (`Is Global` and the organization-aware `vcfa` provider type are 9.1.1
topics in Broadcom's docs.)

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

**Not from a Mac (lab):** the regional Harbor's certificate is valid for
3650 days; macOS rejects TLS server certificates valid for more than 825 days
even from a trusted private CA ("certificate is not standards compliant"), and
the CLI always uses the macOS verifier (SSL_CERT_FILE is ignored). Push from
Linux instead: `vcf plugin download-bundle --plugin pais` contains a
standalone `pais-linux_amd64` binary; run it on the jump host with
`DOCKER_CONFIG` (Harbor login) and `SSL_CERT_FILE` (Harbor CA) in a temp dir
under /tmp, and delete the dir afterwards.

PAIS plugin v2.1.0 is the only published version (checked 2026-10-11); the
Supervisor recommends none. It pushes fine against service 3.0.0.

## 7. Agents and MCP servers

```
PAIS agent (inside pais-api, se-pais-tp7mm)
  -> http://172.19.0.28/mcp (gateway VIP, picked by Avi)  + "Authorization: Bearer <that agent's key>"
  -> agentgateway (se-pais-001, namespace mcp): API key -> agent name -> allowed tools
  -> lab-facts (own MCP server)      kubernetes-mcp (read-only, view role in pais-db and mcp)
```

PAIS sends one fixed set of headers per **MCP server registration**, not per
agent. So the same gateway URL is registered twice, once per agent key, and
each agent is linked to its own registration. agentgateway maps the key to an
agent and filters both the tool list and the calls (tested with the v1.6.0
binary: wrong key 401; the docs agent sees only two tools).

Layers, each enforced on its own:

| Layer | Where | What it stops |
|---|---|---|
| Tool approval and linking | PAIS | an agent using a tool nobody approved or linked |
| API key + `mcpAuthorization` | agentgateway | an agent (or anyone with the URL) calling tools outside its list |
| NetworkPolicy | namespace mcp | anything in se-pais-001 reaching the MCP servers past the gateway |
| `--read-only` + RBAC `view` | kubernetes-mcp | writes and Secrets, whatever the caller |

## 8. API tokens for programs (local account)

Programs (an agent pod, Open WebUI) call the models at
`https://172.19.0.25/api/v1/compatibility/openai/v1` with `Authorization:
Bearer <API token>`. Without one the API answers 401. In PAIS 3.0 only the
**staticCredentials** provider (local accounts) issues API tokens, and only
when its `apiTokens` block is set (`kubectl explain
paisconfigurations.spec.auth.providers.staticCredentials.apiTokens`: "When
omitted, the provider supports interactive OIDC login only"). The `oidc` and
`vcfa` providers have no such setting. Our activation wrote the deprecated
`spec.auth.oidc` (step 3), so no tokens existed (checked 2026-10-16).

The local-accounts secret (`pais-static-credentials-<id>`, key
`credentials.json`) has no published format, so let the UI create it, then
add `apiTokens` by hand (Broadcom 3.0 known issue "API tokens are not turned on
when you activate Private AI Services by using the UI"; William Lam,
"Deploying Your Own AI Models with VCF PAIS 3.0", steps 5-7).

**Blocked on VCF 9.1.0 (lab, 2026-10-16):** the PAIS form here (Service
Configuration > OIDC > Edit) edits the one OIDC login only; it has no Local
User Accounts option (Broadcom's guide and Lam's post use a newer VCF
Automation). By hand it needs the `credentials.json` format, which is not
published ("See the staticCredentials documentation"). Open: ask the PAIS team
for that format. The steps below are for a UI that has the option.

```sh
K() { kubectl --context 172.17.10.2 -n se-pais-tp7mm "$@"; }
# 0. rollback copy, outside git
K get paisconfiguration default -o yaml > ~/paisconfiguration-before-tokens.yaml
```

1. VCF Automation > Build & Deploy > se-pais-tp7mm > Private AI > edit the
   configuration. Keep the OIDC provider as it is. Turn on **Local User
   Accounts**, add user `agents-svc` with a generated password (keep it in your
   vault). **Issuer Base URL** `https://172.19.0.25`: the form wants a valid
   HTTPS URL there (Lam: "a known UI quirk"). Save.
2. Check what the UI wrote: both providers under `spec.auth.providers`, the
   static one with `credentialsSecretRef`:
   ```sh
   K get paisconfiguration default -o jsonpath='{range .spec.auth.providers[*]}{.name}{" "}{.staticCredentials.credentialsSecretRef.name}{.oidc.issuerUrl}{"\n"}{end}'
   ```
3. Turn on API tokens for the static provider (30 days default, 1 year max):
   ```sh
   I=$(K get paisconfiguration default -o json | python3 -c 'import json,sys; p=json.load(sys.stdin)["spec"]["auth"]["providers"]; print([i for i,x in enumerate(p) if "staticCredentials" in x][0])')
   K patch paisconfiguration default --type json --dry-run=server -p "[{\"op\":\"add\",\"path\":\"/spec/auth/providers/$I/staticCredentials/apiTokens\",\"value\":{\"expirationDefault\":\"720h\",\"expirationMax\":\"8760h\"}}]"
   # same without --dry-run=server, then wait for READY True
   K get paisconfiguration default -w
   ```
4. Log in at `https://172.19.0.25/` as `agents-svc` (the login page now offers
   both providers); user menu (upper right) > **API Tokens** > generate. Check:
   ```sh
   read -rs "PAIS_API_KEY?PAIS API token: "; echo
   curl -sk https://172.19.0.25/api/v1/compatibility/openai/v1/models -H "Authorization: Bearer $PAIS_API_KEY" | head -c 400
   ```
   `qwen-chat` and `nomic-embed` should be listed.
5. Your own OIDC login still works (log in once as before).

Rollback: `K replace -f ~/paisconfiguration-before-tokens.yaml` (status and
resourceVersion fields may need removing first).

## 9. Gemini as a remote model

PAIS 3.0 can route a model name to a model hosted elsewhere:
`InferenceGatewayRoute` (`kubectl explain inferencegatewayroutes.spec`).
Engines: `OpenAI` (any OpenAI-compatible API, also another PAIS),
`GoogleVertexAI` (recommended for Google Cloud), `GoogleGemini` (native Gemini
API), `GoogleOpenAI` (Google's OpenAI-compatible API), `Mistral` (reserved).
The key goes in a Secret of type `pais.vmware.com/api-token-credentials`, key
`api_token`. Not on the VCF Automation "Deploy Model Endpoint" form (that one
deploys models from Harbor).

`supervisor/pais/gemini-route.yaml` uses `GoogleOpenAI` with an AI Studio
key (free tier: Google may use the data; lab only).

```sh
K() { kubectl --context 172.17.10.2 -n se-pais-tp7mm "$@"; }
read -rs "GKEY?Gemini API key: "; echo
# which models this key may use (pick a "gemini-…flash" one for agents)
curl -s https://generativelanguage.googleapis.com/v1beta/openai/models -H "Authorization: Bearer $GKEY" \
  | python3 -c 'import json,sys; [print(m["id"]) for m in json.load(sys.stdin)["data"] if "gemini" in m["id"]]'
K create secret generic gemini-api-token --type=pais.vmware.com/api-token-credentials --from-literal=api_token="$GKEY"
unset GKEY
```

Set `modelId` in the file to one of the listed ids (without a `models/`
prefix), then:

```sh
K apply -f supervisor/pais/gemini-route.yaml
K get inferencegatewayroute gemini -o jsonpath='{.status}'; echo
```

In Agent Builder, `gemini` should be offered as a model next to qwen-chat.
To test: that PAIS reaches Google (egress from the Supervisor), and that agents
can pick a route.

### Claude

`supervisor/pais/claude-route.yaml`: engine `OpenAI`, base URL
`https://api.anthropic.com` (PAIS adds `/v1`; Anthropic's OpenAI SDK compatibility layer;
the Anthropic API key is the bearer token), model `claude-haiku-5-5`. Gemini
is on hold (AI Studio keys restricted for us, 2026-10-16).

```sh
K() { kubectl --context 172.17.10.2 -n se-pais-tp7mm "$@"; }
read -rs "AKEY?Anthropic API key: "; echo
# the key works, and through the same compatibility layer PAIS will use:
curl -s https://api.anthropic.com/v1/chat/completions -H "Authorization: Bearer $AKEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"claude-haiku-5-5","max_tokens":20,"messages":[{"role":"user","content":"Say ok"}]}' | head -c 300; echo
K create secret generic anthropic-api-token --type=pais.vmware.com/api-token-credentials --from-literal=api_token="$AKEY"
unset AKEY
K apply -f supervisor/pais/claude-route.yaml
K get inferencegatewayroute claude -o jsonpath='{.status}'; echo
```

**Found 2026-10-16:** direct to Anthropic the route stays not Ready:
`ModelAvailable=False … HTTP error for /models. HTTP status: 400`. PAIS checks
`<httpBaseUrl>/v1/models` with the key as a bearer token (it adds `/v1`
itself, so httpBaseUrl has none); Anthropic lists models only with
`x-api-key`. Chat with the bearer works. Fix: `supervisor/pais/anthropic-shim.yaml`
(stdlib Python, a vSphere Pod and Service in this namespace) answers
`/v1/models` in OpenAI format from Anthropic's native list and passes
everything else through; the route points at `http://anthropic-shim:8080`
(an `http://` backend needs `tls: {}`). The shim holds no key. An OpenAI key
would not need it.

```sh
K apply -f supervisor/pais/anthropic-shim.yaml
K rollout status deploy/anthropic-shim
K apply -f supervisor/pais/claude-route.yaml
sleep 30; K get inferencegatewayroute claude -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.message}{"\n"}{end}'
K logs deploy/anthropic-shim     # GET /v1/models 200 = PAIS's check passed
```

Worked 2026-10-16 (route Ready; docs-agent-am on `claude` called check_vip
through AgentMinder). The shim also drops `temperature`: Agent Builder sends
it, Claude Haiku 5.5 refuses it ("deprecated for this model"). Removed the
same day (route, shim, key Secret) until a team API key is assigned; the
files stay, re-apply both.

### OpenAI

`supervisor/pais/openai-route.yaml`: engine `OpenAI`, base URL
`https://api.openai.com` (PAIS adds `/v1`). No shim: OpenAI answers the
`/v1/models` check with a bearer token.

```sh
K() { kubectl --context 172.17.10.2 -n se-pais-tp7mm "$@"; }
read -rs "OKEY?OpenAI API key: "; echo
curl -s https://api.openai.com/v1/models -H "Authorization: Bearer $OKEY" \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); print(sorted(m["id"] for m in d.get("data",[]) if m["id"].startswith(("gpt","o"))) or d)'
K create secret generic openai-api-token --type=pais.vmware.com/api-token-credentials --from-literal=api_token="$OKEY"
unset OKEY
# set modelId in the file, then
K apply -f supervisor/pais/openai-route.yaml
sleep 30; K get inferencegatewayroute openai -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.message}{"\n"}{end}'
```

## Status (2026-10-11)

PAIS Ready, UI reachable. Models in Harbor project se-pais-models:
qwen/qwen2.5-1.5b-instruct:q4km and nomic/nomic-embed-text-v1.5:q8. Model
endpoints qwen-chat and nomic-embed Ready (supervisor/pais/model-endpoints.yaml).
Agent Builder login works. Agents (2026-10-12, lab): ops-agent answered
"which pods run in pais-db" through kubernetes-mcp (gateway log: tools/call,
target k8s, 200); docs-agent answers VIP and runbook questions through
lab-facts and cannot see or call the Kubernetes tools or `list_vips`.
