# Agent Endpoints (via Kong)

Replace `<kong-host>` and `<agent-route>` with the agent's Kong host and route prefix.

| Purpose | Method | URL |
|---|---|---|
| MCP tools | `POST` | `https://<kong-host>/<agent-route>/mcp` |
| Agent card | `GET` | `https://<kong-host>/<agent-route>/.well-known/agent-card.json` |
| Chat | `POST` | `https://<kong-host>/<agent-route>/v1/chat/completions` |

> Example values below (agent names, tools, IDs) are illustrative.

---

## 1. MCP tools

### Request

Headers: `Content-Type: application/json`, `Accept: application/json, text/event-stream`
(plus `Mcp-Session-Id` if the server returned one from `initialize`)

```json
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list" }
```

### Response

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "search_policy_docs",
        "description": "Search internal policy documents and return matching passages.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": { "type": "string", "description": "Search text" },
            "top_k": { "type": "integer", "default": 5 }
          },
          "required": ["query"]
        },
        "annotations": { "readOnlyHint": true }
      },
      {
        "name": "create_ticket",
        "description": "Create a support ticket.",
        "inputSchema": {
          "type": "object",
          "properties": {
            "title": { "type": "string" },
            "priority": { "type": "string", "enum": ["low", "medium", "high"] }
          },
          "required": ["title"]
        },
        "annotations": { "readOnlyHint": false, "destructiveHint": false }
      }
    ]
  }
}
```

If the server answers as a stream, the same JSON arrives wrapped in SSE:

```
event: message
data: {"jsonrpc":"2.0","id":1,"result":{"tools":[...]}}
```

---

## 2. Agent card

### Request

```
GET https://<kong-host>/<agent-route>/.well-known/agent-card.json
```

### Response (A2A v1.0 shape)

```json
{
  "name": "Policy Q&A Agent",
  "description": "Answers questions on internal policies and raises tickets.",
  "version": "1.2.0",
  "supportedInterfaces": [
    {
      "url": "https://<kong-host>/<agent-route>/a2a",
      "protocolBinding": "JSONRPC",
      "protocolVersion": "1.0"
    }
  ],
  "provider": { "organization": "Example Org" },
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "extendedAgentCard": false
  },
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["text/plain", "application/json"],
  "skills": [
    {
      "id": "policy-lookup",
      "name": "Policy lookup",
      "description": "Find and summarise the relevant policy section.",
      "tags": ["policy", "search"],
      "examples": ["What is the travel reimbursement limit?"]
    }
  ]
}
```

Older (v0.3) agents put `url`, `preferredTransport` and `protocolVersion: "0.3.0"` at the top level instead of `supportedInterfaces`. Cards may also carry `securitySchemes` describing auth.

---

## 3. Chat (OpenAI-compatible)

### Request

```json
{
  "model": "policy-qa-agent",
  "messages": [
    { "role": "user", "content": "What is the travel reimbursement limit?" }
  ],
  "stream": false
}
```

### Response

```json
{
  "id": "chatcmpl-7f3a9c",
  "object": "chat.completion",
  "created": 1763273400,
  "model": "policy-qa-agent",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Per the travel policy, the reimbursement limit is ..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": { "prompt_tokens": 42, "completion_tokens": 58, "total_tokens": 100 }
}
```

### Streaming response (`"stream": true`)

```
data: {"id":"chatcmpl-7f3a9c","object":"chat.completion.chunk","created":1763273400,"model":"policy-qa-agent","choices":[{"index":0,"delta":{"role":"assistant","content":"Per the"},"finish_reason":null}]}

data: {"id":"chatcmpl-7f3a9c","object":"chat.completion.chunk","created":1763273400,"model":"policy-qa-agent","choices":[{"index":0,"delta":{"content":" travel policy..."},"finish_reason":null}]}

data: {"id":"chatcmpl-7f3a9c","object":"chat.completion.chunk","created":1763273400,"model":"policy-qa-agent","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```
