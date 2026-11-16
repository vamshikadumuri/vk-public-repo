# Agent Endpoints (via Kong)

Replace `<kong-host>` and `<agent-route>` with the agent's Kong host and route prefix.

| Purpose | Method | URL |
|---|---|---|
| MCP tools | `POST` | `https://<kong-host>/<agent-route>/mcp` |
| Agent card | `GET` | `https://<kong-host>/<agent-route>/.well-known/agent-card.json` |
| Chat | `POST` | `https://<kong-host>/<agent-route>/v1/chat/completions` |

## MCP tools

The MCP endpoint takes a JSON-RPC body:

```json
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list" }
```
