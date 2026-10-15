# MCP marketplace on VKS (GitOps)

The pattern from Tanzu's "Enterprise MCP server marketplace" blog (Tanzu
Platform 10.3, Cloud Foundry), rebuilt on VKS without Cloud Foundry: teams
**publish** MCP servers, consumers **discover** them in a catalog, and access
is **bound** per consumer at one gateway. Running on se-pais-001; consumers are
the PAIS agents in se-pais-tp7mm.

| Step | Cloud Foundry (blog) | Here |
|---|---|---|
| Publish | `cf publish-service` | a folder `se-pais-001/mcp/servers/<name>/` (server + catalog entry); ArgoCD ApplicationSet deploys it |
| Discover | `cf marketplace` | catalog ConfigMaps (`mcp.lab/catalog=true`) |
| Grant access | `cf enable-service-access`, `cf bind-service` | a consumer key hash + rule in the gateway config, reviewed as a pull request |
| Gateway | Spring Cloud Gateway, API key | agentgateway, API key per consumer, **per-tool** rules |
| Isolation | internal route + network policy | ClusterIP Service + NetworkPolicy: only the gateway reaches servers |
| Unpublish | `cf unpublish-service` | delete the folder |

```
consumer (PAIS agent, key)  ->  agentgateway  ->  servers/lab-facts    (own code)
                                 172.19.0.28      servers/kubernetes   (kubernetes-mcp-server, read-only)
                                 keys + rules     servers/argocd       (mcp-for-argocd, read-only)
                                                  servers/catalog      (the catalog itself, as tools)
```

## Layout

```
se-pais-001/mcp/
  gateway/                     Application se-pais-001-mcp
    namespace.yaml
    agentgateway.yaml          targets (one per server) + consumers (key hash) + rules
    networkpolicy.yaml         servers (label mcp.lab/role: server) only from the gateway
  servers/<name>/              ApplicationSet se-pais-001-mcp-servers -> Application mcp-server-<name>
    server.yaml                Deployment + Service; pod label mcp.lab/role: server
    catalog.yaml               ConfigMap mcp-catalog-<name>, label mcp.lab/catalog: "true"
```

## Discover: the catalog

```sh
kubectl --context se-pais-001:se-pais-001 -n mcp get cm -l mcp.lab/catalog=true \
  -o custom-columns=SERVER:.data.name,ACCESS:.data.access,DESCRIPTION:.data.description
kubectl --context se-pais-001:se-pais-001 -n mcp get cm mcp-catalog-argocd -o jsonpath='{.data.tools}{.data.plans}'
```

| Server | Access | What it gives agents | Plans |
|---|---|---|---|
| lab-facts | read-only | SE lab VIPs and known-problem fixes (own code) | basic: check_vip, runbook_tip; full |
| kubernetes | read-only | pods, logs, events, resources in pais-db and mcp | full |
| argocd | read-only | ArgoCD applications, sync/health, resource trees, logs | status: list_applications, get_application; full |
| catalog | read-only | the marketplace itself: list_servers, get_server, find_servers (reads the catalog ConfigMaps live) | full, open to every consumer |

Agents with the catalog tools can answer "which MCP servers exist and what can
each do?" themselves: dynamic tool discovery, as in the Tanzu blog, served by
the marketplace's own catalog.

## Publish a server

1. Create `se-pais-001/mcp/servers/<name>/server.yaml`: Deployment + ClusterIP
   Service in namespace `mcp`, pod label `mcp.lab/role: server`, Streamable
   HTTP on `/mcp`, read-only where the server supports it.
2. Create `catalog.yaml` from an existing one (name, description, owner,
   source, access, gatewayTarget, endpoint, tools, plans).
3. Add a target to `gateway/agentgateway.yaml` (`name` = gatewayTarget,
   `host` = endpoint) and bump `config-version`.
4. Pull request, merge: ArgoCD creates `mcp-server-<name>`.

First-time setup, in this order: apply `supervisor/argocd-se-pais.yaml` (the
read-only ArgoCD account `mcp-reader`), create its token secret, push, then
apply `supervisor/applicationsets/se-pais-001-mcp-servers.yaml` **and**
`supervisor/applications/se-pais-001-mcp.yaml`: Applications are applied by
hand, so a path change in git does nothing until the file is applied again
(ArgoCD then refuses to auto-sync the old, now empty path: "auto-sync will wipe
out all resources").

Have the server running before the gateway lists it: agentgateway serves all
targets behind one endpoint, and a target that fails can break the session
for every consumer.

## Grant a consumer access (bind)

1. Make a key: `K=$(openssl rand -hex 32); printf %s "$K" | shasum -a 256`.
2. In `gateway/agentgateway.yaml`: add the hash under `apiKey.keys` with
   `metadata: {agent: <consumer>}`, and an `allow` rule naming the targets and
   tools of the chosen plan. Bump `config-version`. Pull request, merge.
3. Give the consumer the key: in PAIS, an MCP server registration
   `http://172.19.0.28/mcp` with credential header `Authorization: Bearer <key>`.

The consumer then sees only the tools its rule allows (tested: the others are
not even listed).

## Current consumers

| Consumer | PAIS registration | Allowed |
|---|---|---|
| ops | lab-gateway-ops | all tools of all servers |
| docs | lab-gateway-docs | labfacts: check_vip, runbook_tip; argocd: list_applications, get_application; catalog: all |
| gitops (new, key to be made) | lab-gateway-gitops | argocd: all (read-only server); catalog: all |
| concierge (new, key to be made) | lab-gateway-concierge | labfacts: runbook_tip; catalog: all |

Each consumer's calls are traced to Phoenix with its name (docs/observability.md).

## Verified (2026-10-14)

- Three Applications `mcp-server-<name>` Synced/Healthy; catalog lists argocd,
  kubernetes, lab-facts.
- argocd MCP server: 11 tools, all read-only (write tools removed by
  `MCP_READ_ONLY`); `list_applications` returns the se-pais ArgoCD's 10
  Applications with sync and health.
- In PAIS: after "Refresh server details", the ops registration lists all
  argocd tools, the docs registration only `argocd_list_applications` and
  `argocd_get_application`; both agents answer accordingly.
