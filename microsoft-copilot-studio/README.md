# Microsoft Copilot Studio + Neo4j

Microsoft Copilot Studio is a low-code platform for building, testing, and publishing agents across Microsoft 365 and other channels.
Neo4j provides the graph database and knowledge layer that grounds those agents in connected enterprise data: relationships, hierarchies, multi-hop paths, and graph-shaped facts.

## Why Graph

Copilot Studio agents often need to answer relationship questions: which companies compete in the same industry, who runs them, what articles mention them, and how organizations are connected. Neo4j keeps those relationships queryable. With the Neo4j MCP server, Copilot Studio can call graph tools such as `get-schema` and `read-cypher` instead of relying only on flattened document retrieval.

## Architecture

```mermaid
flowchart LR
    user["User"] --> agent["Copilot Studio agent<br/>(model + tools)"]
    agent -->|MCP tool| mcp["Neo4j MCP server<br/>Azure Container Apps"]
    mcp --> neo4j[("Neo4j Aura<br/>or self-managed")]
```

The MCP endpoint is shared infrastructure. Deploy it once from [`../microsoft-foundry/infra`](../microsoft-foundry/infra/), then attach the same server from Copilot Studio, Microsoft Foundry, Microsoft Agent Framework, or any other MCP client.

## Quick Start

Deploy the shared Neo4j MCP server:

```bash
cd ../microsoft-foundry/infra
./deploy.sh
./test-mcp.sh "$(azd env get-value mcpEndpoint)"
```

The deployment writes `../microsoft-foundry/.env`. Use `NEO4J_MCP_ENDPOINT` as the Copilot Studio MCP server URL.

For the default public `companies` demo graph, the MCP authentication header is:

```text
Authorization: Basic Y29tcGFuaWVzOmNvbXBhbmllcw==
```

Generate the value yourself:

```bash
printf '%s:%s' companies companies | base64
```

For a real Neo4j database, replace the demo credentials with your own Basic auth value, or use `Bearer <token>` if your Neo4j deployment is configured for SSO or OIDC.

## Copilot Studio Walkthrough

### 1. Create a blank agent

Open [Copilot Studio](https://copilotstudio.microsoft.com) and go to **Agents**. Select **Create blank agent**.

<img src="images/copilot-studio-01-agents-page.png" alt="Copilot Studio Agents page with the Create blank agent button" width="960">

Name the agent `neo4j-mcp`, then create it.

<img src="images/copilot-studio-02-name-agent.png" alt="Name your agent modal with neo4j-mcp entered" width="720">

After the agent is created, Copilot Studio opens the agent workspace with the test panel available.

<img src="images/copilot-studio-03-agent-test-panel.png" alt="Newly created agent workspace with the test panel open" width="960">

Open the **Overview** tab. Choose the model for the agent and add graph-grounded instructions.

<img src="images/copilot-studio-04-agent-instructions.png" alt="Provisioned agent overview with model selector and instructions" width="960">

Example instructions:

```text
Role: investment research analyst. Source of truth: a Neo4j knowledge graph
reached only through the get-schema and read-cypher tools (read-only). Be
thorough and data-driven — cross-reference company data with news,
relationships, and people.

## Workflows

Company research: profile the company → fetch peers in its industry →
fetch its relationships and people → fetch news mentions → synthesise.

Industry analysis: list industries → companies in the chosen category →
cross-org relationships across the leaders → industry news → synthesise.

News-driven: articles by date or mentions → profile each mentioned company
→ relationships across them → synthesise.

Always project `id` properties (e.g. `o.id AS company_id`) so follow-up
questions can build on them.

## Output

Cite every company_id and article_id. Use tables when comparing multiple
entities, bullet lists for attributes of a single entity. Connect the dots
— highlight patterns, anomalies, network position, sentiment trends.

## Grounding

Call get-schema once per conversation with get-schema({
  "properties": {}
})). 
You MUST call read-cypher before any
factual claim about a company, person, industry, location, or article.
get-schema alone is not data. Answer only from read-cypher rows. Never use
prior knowledge. If read-cypher returns nothing, reply "the graph doesn't
contain that". Use modern Cypher (`WHERE x IS NOT NULL`).
```

### 2. Open the Tools tab

Go to the agent's **Tools** tab. For a new agent, Copilot Studio shows the empty tools state. Select **Add a tool**.

<img src="images/copilot-studio-05-empty-tools.png" alt="Tools tab showing Create your first tool and Add a tool" width="960">

### 3. Select Model Context Protocol

In the **Add tool** catalog, select **Model Context Protocol**. You can use the category filter or the MCP tile in the create-new row.

<img src="images/copilot-studio-06-select-mcp.png" alt="Add tool catalog with Model Context Protocol selected" width="960">

### 4. Create the MCP server

Fill in the MCP server form with the shared Neo4j MCP endpoint from `../microsoft-foundry/.env`.

<img src="images/copilot-studio-07-create-mcp-server.png" alt="Add a Model Context Protocol server form" width="640">

Use these values:

| Field | Value |
| --- | --- |
| **Server name** | `neo4j-mcp-01` |
| **Server description** | `Neo4j MCP server for graph-powered retrieval, Cypher queries, and connected data exploration` |
| **Server URL** | `NEO4J_MCP_ENDPOINT` from `../microsoft-foundry/.env` |
| **Authentication** | **API key** |
| **Type** | **Header** |
| **Header name** | `Authorization` |
| **Header value** | `Basic <base64(username:password)>` |

Create the MCP server.

### 5. Create or pick the connection

If no connection exists yet, open the connection dropdown and select **Create new connection**.

<img src="images/copilot-studio-08-create-new-connection.png" alt="Add tool screen showing no connections available and Create new connection" width="760">

Pick the `neo4j-mcp-01` connection and submit it.

<img src="images/copilot-studio-09-pick-connection.png" alt="Create or pick a connection screen with neo4j-mcp-01 selected" width="760">

Once the connection is selected and healthy, select **Add and configure**.

<img src="images/copilot-studio-10-add-and-configure.png" alt="Add tool screen with neo4j-mcp-01 connected" width="760">

### 6. Verify the MCP tool

The configured tool should be enabled and connected to `neo4j-mcp-01`. The detail page shows the server, connection, and the agent that can use it.

<img src="images/copilot-studio-11-tool-details.png" alt="Configured MCP tool detail page with connected status" width="960">

After setup, Copilot Studio should show the Neo4j MCP server with a connected status. The expected tools are:

- `get-schema`
- `read-cypher`

### 7. Test the agent

Open **Test your agent** and try:

```text
Find three companies that compete in the same industry as Microsoft.
```

The agent should call `get-schema`, then `read-cypher`, and return graph-grounded peer companies from the Neo4j `companies` database.

Copilot Studio may ask you to verify the connection before the first tool call succeeds.

<img src="images/copilot-studio-12-connection-required.png" alt="Test panel asking to open connection manager before retrying" width="960">

Open the connection manager and find `neo4j-mcp-01`. If the status is **Not Connected**, select **Connect**.

<img src="images/copilot-studio-13-manage-connections.png" alt="Manage your connections page with neo4j-mcp-01 not connected" width="960">

Enter the required API key value for the `Authorization` header, then create the connection.

<img src="images/copilot-studio-14-connect-api-key.png" alt="Connect to neo4j-mcp-01 prompt with API key field" width="760">

Return to the agent test panel and retry the same prompt.

<img src="images/copilot-studio-15-successful-test.png" alt="Successful test run showing get-schema and read-cypher tool calls" width="960">


## References

- [Microsoft Copilot Studio](https://www.microsoft.com/microsoft-copilot/microsoft-copilot-studio)
- [Neo4j MCP server](https://github.com/neo4j/mcp)
- [Neo4j MCP configuration](https://neo4j.com/docs/mcp/current/configuration/)
- [Shared Azure MCP deployment](../microsoft-foundry/infra/)
