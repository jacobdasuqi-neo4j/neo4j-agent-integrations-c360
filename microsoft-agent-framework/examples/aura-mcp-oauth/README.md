# Neo4j Aura-hosted MCP over OAuth (DCR) — Agent Framework

Neo4j Aura ships a built-in MCP endpoint per instance
(`https://<INSTANCE_ID>.mcp-instances.neo4j.io/mcp`). It's protected by **OAuth
2.0 with Dynamic Client Registration (DCR)** — the client registers itself at
runtime, so there's no client ID or secret to paste anywhere.

That's why this path lives in Agent Framework rather than the Foundry portal: the
portal's MCP tool form requires a **static Client ID**, which a DCR-only server
doesn't issue. Here the [MCP SDK's](https://pypi.org/project/mcp/)
`OAuthClientProvider` performs the whole handshake and Agent Framework consumes
the authenticated session.

```
MCP 401 → resource metadata → authorization server
   → DCR self-registration → browser sign-in + consent → bearer token
      → MCPStreamableHTTPTool(http_client=…) → Agent
```

## Bring your own Aura instance

The sign-in is a real Neo4j Aura account (email / SSO), so the agent connects to
**your** instance — the same model as the
[Copilot Studio Aura option](../../../microsoft-copilot-studio/). Starting
fresh? [Create a free Aura instance](https://neo4j.com/docs/aura/getting-started/create-instance/),
choose the built-in **Movies** sample dataset, and
[enable its MCP endpoint](https://neo4j.com/docs/mcp/current/mcp-for-aura/). (The
public `companies` demo graph isn't reachable this way — it has no OAuth.)

## Run

The script uses [uv](https://docs.astral.sh/uv/) with inline dependencies — no
virtualenv to manage.

```bash
# Source the shared Foundry env (project endpoint, deployment, tenant) written
# by microsoft-foundry/infra/deploy.sh — the leading `.` loads it into this shell.
. ../../../microsoft-foundry/.env

# Your Aura instance's MCP endpoint
export NEO4J_AURA_MCP_URL="https://<INSTANCE_ID>.mcp-instances.neo4j.io/mcp"

uv run aura_mcp_oauth_agent.py
```

The **first run opens your browser** to sign in to Aura and consent. The token
is cached at `~/.neo4j-aura-mcp-oauth.json` (mode `600`), so later runs are
non-interactive. Delete that file to force re-consent.

> Because the initial sign-in is interactive, this is a **local / developer**
> pattern — it isn't suitable for the unattended Foundry hosted-agent runtime,
> which would need a non-interactive credential instead.

Ask your own question and the agent calls `get-schema` then `read-cypher`
against your graph:

```bash
QUESTION="Which actors have worked with the most directors?" uv run aura_mcp_oauth_agent.py
```

## How it differs from the other examples

| | Neo4j access | Auth |
| --- | --- | --- |
| [`../multi-agent`](../multi-agent/) | `neo4j` driver, direct Cypher | database user/password |
| [`../foundry-hosted`](../foundry-hosted/) | same, packaged as a hosted agent | database user/password |
| **this** | Aura-hosted **MCP** (`get-schema`, `read-cypher`) | **OAuth 2.0 DCR**, no credentials in code |

The chat model still comes from the shared Foundry deployment
([`microsoft-foundry/infra`](../../../microsoft-foundry/infra/)); only the graph
connection changes.
