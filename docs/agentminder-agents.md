# Agents with an AgentMinder identity

Part B of AgentMinder (install: `docs/agentminder.md`). Two ways an agent gets
an AgentMinder identity here:

- **A. A PAIS agent, with AgentMinder as its MCP gateway** (the demo). The
  agent is built in PAIS Agent Builder and uses the PAIS model; its tool calls
  go through AgentMinder's AI Gateway, which knows which agent is calling,
  allows or refuses each tool by policy, and records every call.
- **B. A custom agent with Broadcom's SDK** (am-docs-agent, a Job): the full
  model, with short-lived tokens and missions.

```
A.  PAIS agent docs-agent-am (Agent Builder, qwen-chat)
      MCP registration "lab-facts-agentminder": URL = AgentMinder route,
      header Authorization: Bearer <pais-docs-agent's AgentMinder token>
        -> AgentMinder AI Gateway (se-pais-001, ns agentminder)
             token -> which agent; policy -> this tool allowed?; audit record
        -> lab-facts (ns mcp)
```

Why A uses a long-lived token: a PAIS 3.0 MCP registration is a URL plus fixed
headers, nothing else (PAIS API `McpServerCreateRequest`: `url`, `headers[{name,
value, is_secret}]`, `verify_tls`; no OAuth). PAIS cannot fetch or renew
AgentMinder's short-lived tokens, so we mint one with a long lifetime and
paste it in. Missions are off for the same reason (their credential expires
too). This is the honest limit to state to a customer; production would put
a broker (the SDK's sidecar mode) between PAIS and AgentMinder.

Status: written 2026-10-16 from the SDK guide (agentminder-sdk 4.1.1) and
Broadcom's 4.1 docs; **not run yet**. Console labels are quoted from the docs;
where the console differs, follow the console.

## Pieces

| Object | Where | Value |
|---|---|---|
| AI Resource Server | AgentMinder console | `lab-facts`, MCP Server Endpoint `http://lab-facts.mcp.svc.cluster.local:8000/mcp` |
| intents | on the resource server | `urn:iam:agent:intent:check_vip`, `urn:iam:agent:intent:get_runbook_tip` |
| tool bindings | on the resource server | `check_vip` -> check_vip intent, `runbook_tip` -> get_runbook_tip intent |
| AI Agent (A) | AgentMinder console | `pais-docs-agent`: Autonomous Only, Require Missions off |
| authorization policy | AgentMinder console | target lab-facts; Client App IN pais-docs-agent; Grant both intents |
| network policy | git | `se-pais-001/mcp/gateway/networkpolicy.yaml` `lab-facts-from-agentminder` (ArgoCD app se-pais-001-mcp) |
| MCP registration + agent (A) | PAIS UI | `lab-facts-agentminder`, agent `docs-agent-am` |
| am-docs-agent (B) | git | `se-pais-001/agents/am-docs-agent.yaml` |

## 1. Network policy (git)

lab-facts accepts traffic only from a gateway. The new policy adds
AgentMinder's gateway pod (`app: ssp-aigateway` in namespace agentminder),
port 8000 only. Push; ArgoCD (se-pais-001-mcp) applies it:

```sh
k() { kubectl --context se-pais-001:se-pais-001 "$@"; }
k -n mcp get networkpolicy lab-facts-from-agentminder
```

## 2. AgentMinder console: the resource server

1. **AI Resource Servers > + Add AI Resource Server**. Name `lab-facts`;
   **MCP Server Endpoint** `http://lab-facts.mcp.svc.cluster.local:8000/mcp`.
   The gateway runs in the same cluster, so it reaches lab-facts by its
   Service name.
2. Note the **App ID**. The route of the built-in gateway (Gateway Type
   **AI Gateway-as-a-Service**, the default) is the tenant URL plus the App ID,
   base64url without padding (SDK guide §9.2):
   ```sh
   APPID=<App ID>
   ROUTE="https://agentminder.172-19-0-31.sslip.io/default/aigateway/v1/mcp/$(printf %s "$APPID" | base64 | tr '+/' '-_' | tr -d '=')"
   echo $ROUTE
   ```
3. Tab **AI Resource Server**: **Primary Audience** = `$ROUTE` (the token's
   `aud` must match it). Keep **Secure All MCP Resources By Default**,
   **Enforce Authentication** and **Enforce Authorization** on.
4. **Agent Tool Bindings > Import Tools from MCP Server** (or **Add Agent Tool
   Binding** twice). Bind `check_vip` to `urn:iam:agent:intent:check_vip` and
   `runbook_tip` to `urn:iam:agent:intent:get_runbook_tip` (create the intents
   with **Manage Intents > + Add**). **Public** off for both.

## 3. AgentMinder console: the agent identity and its policy

1. **AI Agents > + Add AI Agent**: `pais-docs-agent`. Tab **AI Agent**:
   Identity Mode **Autonomous Only**, Default Risk **Normal**, **Require
   Missions** off, **Use All Allowed Intents as Default** on. **Save AI
   Agent**.
2. Tab **OAuth Client**: copy client ID and secret (vault, not chat). Look for
   the **access token lifetime** there and raise it for the lab (e.g. 7 days):
   it is how long the token in PAIS works before you mint a new one.
3. **Policies > Create Policy**: Policy Type **Authorization Policy**, Target
   Application lab-facts; rule Principal > Client App IN pais-docs-agent, Effect
   **Grant**, Privileges both intents. (Both for now, so PAIS sees both tools;
   step 7 takes one away.)

## 4. Mint the token (terminal)

The same request the SDK makes (`client_credentials`, `resource` = the route,
scope `urn:iam:myscopes` = every intent policy grants). The token goes to the
clipboard, not the screen; its claims are printed to check it:

```sh
H=https://agentminder.172-19-0-31.sslip.io
read -r "ACID?pais-docs-agent client ID: "; read -rs "ASEC?pais-docs-agent client secret: "; echo
printf %s "$ASEC" | curl -sk $H/default/oauth2/v1/token -d grant_type=client_credentials \
  -d client_id="$ACID" --data-urlencode resource="$ROUTE" --data-urlencode scope=urn:iam:myscopes \
  --data-urlencode client_secret@- | python3 -c '
import base64, json, subprocess, sys
d = json.load(sys.stdin); t = d.get("access_token")
if not t: sys.exit("ERROR " + json.dumps(d))
c = json.loads(base64.urlsafe_b64decode(t.split(".")[1] + "=="))
print({k: c.get(k) for k in ("azp", "sub", "aud", "scope", "exp")}, "expires_in", d.get("expires_in"))
subprocess.run(["pbcopy"], input=t.encode())'
unset ASEC
```

Expect `azp` = `sub` = the client ID, `aud` = the route, both intents in
`scope`. Test the route before PAIS does (paste the token at the prompt):

```sh
read -rs "AMT?token: "; echo
curl -sk "$ROUTE" -H "Authorization: Bearer $AMT" -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl","version":"0"}}}' -D - | head -20
unset AMT
```

`200` with a `Mcp-Session-Id` header: the gateway accepts the token and
reaches lab-facts.

## 5. PAIS UI: register AgentMinder as the MCP server

Register a new MCP server (where the existing `lab-gateway-*` registrations
are):

- Name `lab-facts-agentminder`, URL = `$ROUTE`, transport Streamable HTTP
- Header `Authorization` = `Bearer <token>` (paste from the clipboard), secret
- Verify TLS **off**: AgentMinder's certificate is from the lab CA, which PAIS
  doesn't trust (PAIS has a CA Trust Bundles tab, the clean fix)
- **Test connection**, refresh the tools, approve `check_vip` and `runbook_tip`

If the test fails with a name or connection error: PAIS runs on the Supervisor
and must resolve `agentminder.172-19-0-31.sslip.io` (public DNS) and reach
172.19.0.31.

## 6. PAIS UI: the agent

Create agent `docs-agent-am`, model qwen-chat, instructions as for docs-agent
(docs/observability.md), tools from `lab-facts-agentminder` only. Ask:

- "Is 172.19.0.31 in use?" -> `check_vip`, allowed
- "How do I fix the PAIS database CA problem?" -> `runbook_tip`, allowed

Both calls appear in AgentMinder under pais-docs-agent.

## 7. The demo: take a tool away without touching PAIS

In AgentMinder, edit the policy: remove `…:get_runbook_tip` from the grant (or
add a rule Effect **Deny** for it). Ask the runbook question again: PAIS still
offers the tool, AgentMinder refuses the call (the gateway asks the policy
engine on every call, so the change applies at once; allow about a minute for
the policy engine to load it). The refusal is in the audit view with the agent
and the tool.

The story: the platform team changes what an agent may do centrally, per
tool, and every attempt is recorded, whatever the agent's builder configured.

## If it fails

| Symptom | Look at |
|---|---|
| token request `invalid_client` | client ID/secret; tenant `default` in the URL |
| token without the intents in `scope` | the policy (step 3.3) and the bindings (2.4) |
| curl test 401 | `aud` = route? Primary Audience (2.3) |
| curl test 404 | route URL (App ID encoding), Gateway Type |
| curl test 502/503 | gateway can't reach lab-facts: network policy (step 1); `k -n agentminder logs deploy/ssp-ssp-aigateway --all-containers --tail=50` |
| PAIS test connection fails | DNS/route from the Supervisor to the sslip.io name; Verify TLS off |
| PAIS worked, then 401 | the token expired: step 4 again, update the header in PAIS |

## B. A custom agent with the SDK (am-docs-agent)

The agent PAIS can't be (yet): short-lived tokens fetched by the SDK, a
mission per run, `tools/list` filtered to what the mission allows. No LLM:
fixed tool calls, so the output shows identity and policy only.

Extra console objects:

1. **Orchestrator**: an ordinary OAuth application, confidential,
   client_credentials, scope `urn:iam:t.aiagentorchestrationclient` (creates
   missions; never calls tools).
2. **AI Agent** `am-docs-agent`: Autonomous Only, **Require Missions** on
   (JWT), mission type `LabLookup` (+ Add), Authorization Surface: both
   lab-facts intents ticked for LabLookup.
3. A policy granting am-docs-agent `…:check_vip` only. The mission gets
   policy ∩ mission type = check_vip, so runbook_tip is refused.

Cluster (the SDK wheel is proprietary and the repo public, so it stays out
of git):

```sh
k create namespace agents
tar -xzf ~/Downloads/AgentMinder-sdk-4.1.1.tar.gz -C ~/Downloads
k -n agents create configmap agentminder-sdk \
  --from-file=$HOME/Downloads/agentminder-sdk-4.1.1/agentminder-0.1.0-py3-none-any.whl
k -n agents create configmap lab-ca \
  --from-literal=ca.crt="$(k -n cert-manager get secret selfsigned-root-ca -o jsonpath='{.data.ca\.crt}' | base64 -d)"
read -r "OCID?Orchestrator client ID: "; read -rs "OSEC?Orchestrator client secret: "; echo
read -r "ACID?am-docs-agent client ID: "; read -rs "ASEC?am-docs-agent client secret: "; echo
k -n agents create secret generic am-docs-agent \
  --from-literal=orchestratorClientId="$OCID" --from-literal=orchestratorClientSecret="$OSEC" \
  --from-literal=agentClientId="$ACID" --from-literal=agentClientSecret="$ASEC"
unset OSEC ASEC
```

In `se-pais-001/agents/am-docs-agent.yaml` replace `REPLACE_GATEWAY_ROUTE`
and `REPLACE_AUDIENCE` with `$ROUTE`, then:

```sh
k apply -f se-pais-001/agents/am-docs-agent.yaml
k -n agents logs -f job/am-docs-agent
```

Expected: the mission's intents = check_vip only; `tools visible: ['check_vip']`;
check_vip ALLOWED with `azp` = `sub` = am-docs-agent; runbook_tip REFUSED
(`IntentNotAuthorizedError` or `PolicyDeniedError`, with a txn ID to find in
AgentMinder's logs). If it fails: `MissionError` = mission type/intents on the
agent; `invalid_scope` on mission create = the orchestrator's scope;
`SSLCertVerificationError` = ConfigMap lab-ca; `pip` errors = no PyPI egress.

## Next

1. A broker between PAIS and AgentMinder (the SDK's sidecar mode): short-lived
   tokens and missions for PAIS agents, no AgentMinder secret in PAIS.
2. **Delegated**: a user signs in to AgentMinder; the agent acts for them
   (`sub` = user, `azp` = agent); then agent-to-agent with the `act` chain.
3. **Workload identity** for B: no client secret in the pod (projected
   ServiceAccount token, SDK guide §9.4).
