# Tracing agents and MCP calls

What a customer asks: "how do I see what my agents do, and which tools they
call?" Two tracers feed one backend:

```
 PAIS agent run (se-pais-tp7mm)                       agentgateway (se-pais-001, mcp)
   spans: agent -> LLM call -> tool call   --OTLP-->  span per MCP request, tagged:
   (OpenInference: prompts, tools, tokens)     |      agent, MCP server, tool
                                               v                 |
                                     Phoenix (se-pais-001, observability)  <--+
                                     one project "lab-agents": traces, tool calls, latency
```

Labels: **lab** = seen in our lab; **documented** = product docs or source;
**to test** = not yet seen working.

| Piece | Status |
|---|---|
| PAIS 3.0 traces: `PAISConfiguration spec.observability.llmTraces` (`protocol` grpc or http/protobuf, `endpoint`, `projectName`, `headersSecretRef`) | documented (API schema); server dry-run passes. Traces arriving: **to test** |
| PAIS 3.0 metrics and monitoring dashboards: `spec.observability.prometheusRuntime` | documented (API schema); **to test** |
| agentgateway v1.6.0 spans with agent, server and tool tags, continuing an incoming `traceparent` | **lab** (local run of the v1.6.0 binary, 2026-10-15) |
| One trace from the PAIS agent through the gateway | **to test**: only if PAIS sends `traceparent` to MCP servers |
| Phoenix 20.21.4 on VKS | manifests validated (server dry-run, pod passes "restricted"); running: **to test** |

## The agents

One job each, each with its own gateway key, so every trace shows which
agent did what. The first two exist; the other two are new.

| Agent | Job | MCP access (agentgateway.yaml) |
|---|---|---|
| **ops-agent** | Kubernetes troubleshooting on se-pais-001 | every tool of every server |
| **docs-agent** | lab facts: VIPs, runbook tips | labfacts `check_vip`, `runbook_tip`; argocd `list_applications`, `get_application` |
| **gitops-agent** (new) | what's deployed where, sync and health, from ArgoCD | every argocd tool (the server runs read-only) |
| **concierge-agent** (new) | the MCP marketplace: which servers exist, what each offers, how to get access | catalog (open to all) + labfacts `runbook_tip` |

Instructions to paste in Agent Builder (model: qwen-chat):

**gitops-agent**
> You are the GitOps assistant for the lab. Use the ArgoCD tools to answer
> questions about applications: what is deployed to which cluster, its sync
> and health status, when it last synced and what differs from git. You are
> read-only: never suggest syncing, deleting or changing anything. Answer
> with the application name, destination cluster, sync status and health.

**concierge-agent**
> You are the concierge of the lab's MCP marketplace. Use the catalog tools
> to tell users which MCP servers exist, what each one offers, and which
> access plan grants which tools. For a known lab problem, use runbook_tip.
> You cannot answer questions about the cluster or ArgoCD yourself: say which
> agent can (ops-agent or gitops-agent).

Test questions, and what each should show:

| Ask | Agent | Expected tool calls |
|---|---|---|
| "Which ArgoCD applications are out of sync or degraded?" | gitops-agent | argocd `list_applications`, maybe `get_application` |
| "Is the MCP gateway application healthy, and when did it last sync?" | gitops-agent | argocd `get_application` |
| "Which MCP servers can I use, and what does the argocd one offer?" | concierge-agent | catalog `list_servers`, `get_server` |
| "How do I fix the PAIS database CA problem?" | concierge-agent | labfacts `runbook_tip` |
| "Which pods run in pais-db?" | concierge-agent | **none**: it should send you to ops-agent (and if it tried, the gateway would refuse) |

## Setting it up (in order)

1. **Commit and push** (ArgoCD reads git).
2. **Phoenix:**
   ```sh
   kubectl --context 172.17.10.2 apply -f supervisor/applications/se-pais-001-observability.yaml
   kubectl --context se-pais-001:se-pais-001 -n observability get pods,pvc,svc -w
   ```
   Note the `phoenix-lb` EXTERNAL-IP (Avi picks it from 172.19.x). Check it for
   stale NSX routes as for any new VIP (`docs/pais.md`). UI: `http://<VIP>:6006`.
3. **Gateway tracing and the two new agents:** the push changed
   `se-pais-001/mcp/gateway/agentgateway.yaml`; ArgoCD (`se-pais-001-mcp`)
   rolls the gateway (config-version 4). The new agents' key hashes are
   placeholders that match nothing, so make their keys first:
   ```sh
   GITOPS_KEY=$(openssl rand -hex 32);    printf %s "$GITOPS_KEY"    | shasum -a 256
   CONCIERGE_KEY=$(openssl rand -hex 32); printf %s "$CONCIERGE_KEY" | shasum -a 256
   ```
   Put the two hashes into `agentgateway.yaml` (the two `0000…` lines),
   commit, push. Keep the keys themselves out of git.
4. **PAIS traces:** put the Phoenix VIP into
   `supervisor/pais/observability-traces.patch.yaml`, then:
   ```sh
   kubectl --context 172.17.10.2 -n se-pais-tp7mm patch paisconfiguration default \
     --type merge --patch-file supervisor/pais/observability-traces.patch.yaml
   kubectl --context 172.17.10.2 -n se-pais-tp7mm get paisconfiguration default -w   # READY True again
   ```
   Optional, for the monitoring tab: the same with
   `observability-metrics.patch.yaml`.
5. **PAIS registrations and agents:** in the PAIS UI, register the gateway
   URL `http://172.19.0.28/mcp` twice more, as **lab-gateway-gitops** and
   **lab-gateway-concierge**, each with credential header `Authorization: Bearer
   <that agent's key>`. Refresh the tools, approve them, create the two
   agents with the instructions above, and link each to its registration.
6. **Ask the test questions**, then open Phoenix, project **lab-agents**.

## What to look at in Phoenix

- **Traces**: one row per agent run (from PAIS) and one per MCP request (from
  the gateway). Open one: the span tree shows the LLM call(s), the tool
  call(s) and their arguments and results.
- **The gateway spans** carry `agent` (ops, docs, gitops, concierge),
  `mcp.target` (the server), `mcp.tool` and `mcp.method.name`. Filter on
  `agent` to see one agent's calls, or on status to see refused calls.
- **Joined or separate?** If a gateway span sits *inside* the PAIS agent's
  trace, PAIS passes `traceparent` to MCP servers and the trace is end to
  end. If they appear as separate traces, PAIS doesn't, and the link is by
  time and agent name. Either answer is worth having for the customer.

## If traces don't arrive

```sh
# Phoenix up and listening?
kubectl --context se-pais-001:se-pais-001 -n observability logs deploy/phoenix --tail=30
# gateway exporting? ("Export.Failed" = it can't reach Phoenix)
kubectl --context se-pais-001:se-pais-001 -n mcp logs deploy/agentgateway --tail=50 | grep -i -E 'otlp|export|trace'
# what PAIS deployed for llmTraces (its collector or exporter config)
kubectl --context 172.17.10.2 -n se-pais-tp7mm get pods,configmaps | grep -i -E 'otel|collector|telemetry|trace'
```

PAIS fallback: `protocol: http/protobuf`, `endpoint:
http://<VIP>:6006/v1/traces`.

## Next: agent-to-agent through the gateway (phase 2)

The customer's case: agent A calls agent B, with an MCP endpoint and the
gateway in between. The plan, not built yet:

1. **A token PAIS accepts for a service.** PAIS 3.0 can add an auth provider
   with `staticCredentials` (local accounts with **API tokens**), next to
   the VCF Automation login (`kubectl explain
   paisconfigurations.spec.auth.providers`). It changes PAIS login settings,
   so it needs a careful test.
2. **An "agents" MCP server**, like lab-facts: tools such as
   `ask_gitops_agent(question)` that call the gitops agent through the PAIS
   API with that token.
3. **A triage agent** in PAIS whose only tool is the agents server, through
   the gateway, with a rule that only the triage agent may call it.

Result: agent-to-agent over MCP, an identity per agent at the gateway, and
every hop in Phoenix. `oidcCustom` (another 3.0 auth provider, which
validates tokens from any identity provider with CEL rules) is where a real
agent identity provider, such as AgentMinder, would plug into PAIS.
