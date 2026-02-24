# MCP Integration

> The [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) enables AI
> assistants to interact with Linera applications through GraphQL. This allows
> tools like Claude Desktop and other MCP-compatible clients to query on-chain
> state and execute mutations on a running Linera application.

## Architecture

An MCP integration for Linera applications has four components:

1. **Linera Application** — deployed on-chain, exposing a GraphQL API via its
   service binary.
2. **Node Service** — `linera service --port 8080` exposes the GraphQL endpoint
   for the application.
3. **Apollo MCP Server** — bridges between MCP (stdio transport) and the GraphQL
   endpoint.
4. **MCP Client** — Claude Desktop or any MCP-compatible AI assistant.

```text
MCP Client  <-->  Apollo MCP Server  <-->  Linera Node Service  <-->  On-Chain App
  (stdio)           (bridge)                  (HTTP/GraphQL)           (Wasm)
```

## Prerequisites

- A Linera application deployed and running (see
  [Deploying the Application](../backend/deploy.md))
- A node service exposing the application's GraphQL endpoint (see
  [Node Service](../core_concepts/node_service.md))
- An MCP-compatible client (e.g. Claude Desktop)

## Setting Up the Apollo MCP Server

The [Apollo MCP Server](https://github.com/apollographql/apollo-mcp-server)
translates between MCP tool calls and GraphQL operations. Build it from source:

```bash
git clone https://github.com/apollographql/apollo-mcp-server.git
cd apollo-mcp-server
cargo build --release
# Binary at: ./target/release/apollo-mcp-server
```

## Creating a GraphQL Schema File

The Apollo MCP Server needs a schema file describing your application's GraphQL
API. You can obtain this by introspecting the running service or writing it
manually.

**Option A: Introspect from the running service**

With `linera service` running, query the application's endpoint:

```bash
CHAIN_ID="<your-chain-id>"
APP_ID="<your-app-id>"

curl "http://localhost:8080/chains/$CHAIN_ID/applications/$APP_ID" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { types { name kind fields { name type { name kind ofType { name } } } } } }"}'
```

Convert the introspection result into an SDL schema file (e.g. `app.graphql`).

**Option B: Write the schema manually**

If your application is a simple counter, the schema might look like:

```graphql
type Query {
  counter: Int!
}

type Mutation {
  increment(value: Int!): [Int!]!
}
```

For more complex applications, refer to your service's `QueryRoot` and
`MutationRoot` implementations.

## Configuring an MCP Client

### Claude Desktop

Edit the Claude Desktop configuration file:

- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

Add an entry under `mcpServers`:

```json
{
  "mcpServers": {
    "linera-app": {
      "command": "/absolute/path/to/apollo-mcp-server",
      "args": [
        "--schema", "/absolute/path/to/app.graphql",
        "--endpoint", "http://localhost:8080/chains/<CHAIN_ID>/applications/<APP_ID>",
        "--allow-mutations", "all",
        "--introspection"
      ]
    }
  }
}
```

**Important**:
- All paths must be absolute.
- Replace `<CHAIN_ID>` and `<APP_ID>` with the actual values from
  `linera wallet show`.
- `--allow-mutations all` enables write operations (mutations schedule
  operations on the application).
- `--introspection` enables runtime schema discovery.
- Use the default `stdio` transport — it is more reliable than HTTP for
  desktop MCP clients.

After saving the configuration, restart the MCP client completely.

### YAML Configuration

The Apollo MCP Server also accepts a YAML configuration file, which can be
easier to manage:

```yaml
schema: /absolute/path/to/app.graphql
endpoint: http://localhost:8080/chains/<CHAIN_ID>/applications/<APP_ID>
allowMutations: all
introspection: true
```

Then reference it in the client configuration:

```json
{
  "mcpServers": {
    "linera-app": {
      "command": "/absolute/path/to/apollo-mcp-server",
      "args": ["--config", "/absolute/path/to/mcp-config.yaml"]
    }
  }
}
```

## Multiple Applications

A single MCP client can connect to multiple Linera applications by adding
separate server entries. Each application needs its own schema file and endpoint:

```json
{
  "mcpServers": {
    "linera-counter": {
      "command": "/path/to/apollo-mcp-server",
      "args": [
        "--schema", "/path/to/counter.graphql",
        "--endpoint", "http://localhost:8080/chains/<CHAIN1>/applications/<APP1>",
        "--allow-mutations", "all"
      ]
    },
    "linera-fungible": {
      "command": "/path/to/apollo-mcp-server",
      "args": [
        "--schema", "/path/to/fungible.graphql",
        "--endpoint", "http://localhost:8080/chains/<CHAIN2>/applications/<APP2>",
        "--allow-mutations", "all"
      ]
    }
  }
}
```

## Troubleshooting

### MCP server won't connect

1. Verify the node service is running: `curl http://localhost:8080/`
2. Test the application endpoint directly:
   ```bash
   curl "http://localhost:8080/chains/<CHAIN_ID>/applications/<APP_ID>" \
     -H "Content-Type: application/json" \
     -d '{"query": "{ __typename }"}'
   ```
3. Ensure all paths in the client configuration are absolute.
4. Restart the MCP client after any configuration change.

### Mutations not working

1. Confirm `--allow-mutations all` is in the server arguments.
2. Ensure the wallet has sufficient funds for transaction fees.
3. Verify mutation names in the schema match the application's `MutationRoot`
   exactly.

### Schema mismatch errors

1. Re-export the schema from the running service if the application was
   redeployed.
2. Field names are case-sensitive and must match exactly.
3. Ensure types align (e.g. `Int` vs `Int!`, `String` vs `ID`).
