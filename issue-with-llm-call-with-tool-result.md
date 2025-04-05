## 🐛 Bug: Tool result returned as stringified JSON instead of valid LLM `content`

### Summary

The MCP tool server correctly returns structured `tools/call` responses that conform to the [MCP 2024-11-05 spec](https://spec.modelcontextprotocol.io/specification/2024-11-05/server/tools/), but the LLM (e.g. LM Studio, Claude) receives a **stringified JSON array** in the `content` field of the `"tool"` message — which causes parsing errors and rendering artifacts.

Using Version: 0.9.9

<img width="883" alt="Image" src="https://github.com/user-attachments/assets/d3f57854-7cb9-403e-acbb-251bae979973" />

---

### ✅ What works

1. **Manual test of the server response is valid JSON-RPC**:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "create a simple encyclopedia of the first 25 legendary Pokémon including their types..."
      }
    ],
    "isError": false
  }
}
```

2. **Claude Desktop** handles the response with some minor display artifact (shows `{}` but still renders the text), suggesting it digs into `result.content` or does fallback parsing.

---

### ❌ What fails

In LM Studio logs, we observe:

```json
{
  "role": "tool",
  "content": "[{"type":"text","text":"Quasar Alpha: This Mystery..."}]",
  "name": "clipboard--ClipboardPaste",
  "tool_call_id": "313278275"
}
```

Which leads to:

```
[Server Error] Invalid 'content': 'content' field must be a string or an array of objects.
```

This violates the expected OpenAI-compatible format:

```json
"content": [ { "type": "text", "text": "..." } ]
```

---

### 🔍 Root cause

It appears the MCP **client** (or the 5ire consumer layer) is passing `result.content` as a **stringified JSON**, rather than an actual array or string.

This likely happens when:
- The tool returns `{ content: [ { type: "text", ... } ], isError: false }`
- The MCP client or 5ire layer does `JSON.stringify(result.content)` or passes it directly without inspecting the type.

---

### 🤔 Who’s responsible?

| Layer             | Responsibility                                               | Status    |
|------------------|---------------------------------------------------------------|-----------|
| ✅ MCP Tool       | Returns valid `tools/call` result conforming to spec          | ✅ Good    |
| 🤷 MCP Client SDK | Possibly returns `result` as-is (with `isError`, etc.)        | Needs fix or wrapper |
| 🤷 5ire Client     | Might pass raw `result` (with `content`) to LLM API as-is     | Needs clarification |

---

### 🛠️ Next Steps

- Determine whether `@modelcontextprotocol/sdk`'s `callTool()` should extract `result.content`, or if it’s the consumer’s (e.g. 5ire’s) responsibility.
- In either case, ensure that the final message sent to the LLM contains:
  - A **string**, or
  - An **array of structured message objects** (e.g. `[{"type": "text", "text": "..."}]`)

---

### 📦 Proposed Fixes (to be tracked separately)

1. **Fix in 5ire client**: Extract `result.content` from tool response before passing it as `content` in the tool message to the LLM.
2. **Optional improvement in SDK**: Add helper to `callTool()` to safely unwrap top-level `content` if present.
