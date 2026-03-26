# Making an API Available to a Claude Skill

There are three distinct contexts where "skill" applies in the Claude ecosystem, and each has a different integration pattern. The cleanest universal approach is **MCP-first** — wrap your API as an MCP server once, and it becomes consumable across all contexts.

---

## Context 1: SKILL.md Files (This Environment)

Skills here are instruction documents that guide Claude on how to use tools already available in the container environment (`bash_tool`, `create_file`, etc.). There is no plugin system — you make an API accessible by documenting how to call it and ensuring credentials and tooling are present.

### Strategy 1: Document direct HTTP calls via curl or httpx

In your `SKILL.md`, describe the API contract and instruct Claude to call it using shell commands:

```markdown
## API Access

Base URL: `$MY_API_BASE_URL`  
Auth header: `Authorization: Bearer $MY_API_KEY`

### List resources
```bash
curl -s -X GET "$MY_API_BASE_URL/v1/resources" \
  -H "Authorization: Bearer $MY_API_KEY" \
  -H "Content-Type: application/json"
```

### Create a resource
```bash
curl -s -X POST "$MY_API_BASE_URL/v1/resources" \
  -H "Authorization: Bearer $MY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "example", "type": "widget"}'
```

Always check the HTTP status code. On non-2xx responses, surface the error body to the user.
```

### Strategy 2: Pre-install an SDK or CLI

If the API has a Python or Node SDK, install it in the container and reference it in the skill:

```markdown
## Setup

Ensure the SDK is installed before calling the API:

```bash
pip install my-api-sdk --break-system-packages
```

## Usage

```python
import os
from my_api_sdk import Client

client = Client(
    api_key=os.environ["MY_API_KEY"],
    base_url=os.environ.get("MY_API_BASE_URL", "https://api.example.com")
)

resources = client.resources.list()
print(resources)
```
```

### Environment Variables

Set the following env vars in the container before Claude runs the skill:

| Variable | Description | Example |
|---|---|---|
| `MY_API_KEY` | API secret key | `sk-abc123...` |
| `MY_API_BASE_URL` | Base URL for the API | `https://api.example.com` |
| `MY_API_TIMEOUT` | Request timeout in seconds | `30` |

---

## Context 2: Claude Code (Agentic / CLAUDE.md)

The primary mechanism is **MCP (Model Context Protocol)**. You expose your API as an MCP server, register it with Claude Code, and document its use in `CLAUDE.md`.

### Step 1: Build an MCP Server

Using the official `@modelcontextprotocol/sdk` package:

```javascript
// mcp-server/index.js
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const API_BASE_URL = process.env.MY_API_BASE_URL ?? "https://api.example.com";
const API_KEY = process.env.MY_API_KEY;

if (!API_KEY) {
  console.error("MY_API_KEY environment variable is required");
  process.exit(1);
}

const server = new McpServer({
  name: "my-api",
  version: "1.0.0",
});

// Tool: List resources
server.tool(
  "list_resources",
  "Retrieve all resources from the API",
  {
    limit: z.number().optional().describe("Max results to return (default 20)"),
    offset: z.number().optional().describe("Pagination offset"),
  },
  async ({ limit = 20, offset = 0 }) => {
    const url = new URL(`${API_BASE_URL}/v1/resources`);
    url.searchParams.set("limit", String(limit));
    url.searchParams.set("offset", String(offset));

    const res = await fetch(url.toString(), {
      headers: {
        Authorization: `Bearer ${API_KEY}`,
        "Content-Type": "application/json",
      },
    });

    if (!res.ok) {
      const err = await res.text();
      return { content: [{ type: "text", text: `Error ${res.status}: ${err}` }], isError: true };
    }

    const data = await res.json();
    return { content: [{ type: "text", text: JSON.stringify(data, null, 2) }] };
  }
);

// Tool: Create a resource
server.tool(
  "create_resource",
  "Create a new resource via the API",
  {
    name: z.string().describe("Name of the resource"),
    type: z.enum(["widget", "gadget", "component"]).describe("Resource type"),
    metadata: z.record(z.string()).optional().describe("Optional key-value metadata"),
  },
  async ({ name, type, metadata = {} }) => {
    const res = await fetch(`${API_BASE_URL}/v1/resources`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ name, type, metadata }),
    });

    if (!res.ok) {
      const err = await res.text();
      return { content: [{ type: "text", text: `Error ${res.status}: ${err}` }], isError: true };
    }

    const data = await res.json();
    return { content: [{ type: "text", text: JSON.stringify(data, null, 2) }] };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Step 2: Register in `.claude/settings.json`

```json
{
  "mcpServers": {
    "my-api": {
      "command": "node",
      "args": ["./mcp-server/index.js"],
      "env": {
        "MY_API_KEY": "${MY_API_KEY}",
        "MY_API_BASE_URL": "https://api.example.com",
        "MY_API_TIMEOUT": "30"
      }
    }
  }
}
```

> **Note:** `${MY_API_KEY}` instructs Claude Code to inherit the variable from the shell environment at startup — never hardcode secrets in this file.

### Step 3: Document in `CLAUDE.md`

```markdown
## Available Tools

### my-api (MCP)
Wraps the Example API. Use these tools when the user needs to manage resources:

- `list_resources` — fetch a paginated list of resources; use `limit`/`offset` for pagination
- `create_resource` — create a new resource; `type` must be one of: widget, gadget, component

Prefer `list_resources` before `create_resource` to check for duplicates.
Error responses include an HTTP status code — surface these to the user verbatim.
```

---

## Context 3: Artifacts (Claude-in-Claude)

When calling the Anthropic API from inside an artifact, pass `mcp_servers` in the request body. The inner Claude instance will be able to invoke your API's MCP tools directly.

### Full Working Example

```javascript
// React artifact — AI-powered resource manager
import { useState } from "react";

const ANTHROPIC_API_URL = "https://api.anthropic.com/v1/messages";
const MCP_SERVER_URL = "https://your-mcp-server.example.com/sse"; // must be SSE transport for URL-based MCP

async function askClaude(userMessage, conversationHistory = []) {
  const messages = [
    ...conversationHistory,
    { role: "user", content: userMessage },
  ];

  const response = await fetch(ANTHROPIC_API_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      // Note: API key injection is handled by the Claude.ai runtime —
      // do not include an Authorization header here.
    },
    body: JSON.stringify({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1000,
      system: `You are a helpful assistant that manages API resources.
When the user asks to list or create resources, use the available MCP tools.
Always confirm the result of tool calls before responding.`,
      messages,
      mcp_servers: [
        {
          type: "url",
          url: MCP_SERVER_URL,
          name: "my-api",
        },
      ],
    }),
  });

  if (!response.ok) {
    const err = await response.json();
    throw new Error(err.error?.message ?? `HTTP ${response.status}`);
  }

  const data = await response.json();

  // Extract all text blocks; tool_use and mcp_tool_result blocks are handled
  // internally by the model — only the final text reply surfaces here.
  const textBlocks = data.content
    .filter((block) => block.type === "text")
    .map((block) => block.text)
    .join("\n");

  // Return updated history so callers can maintain multi-turn context
  const updatedHistory = [
    ...messages,
    { role: "assistant", content: data.content },
  ];

  return { reply: textBlocks, history: updatedHistory };
}

export default function ResourceManager() {
  const [input, setInput] = useState("");
  const [history, setHistory] = useState([]);
  const [messages, setMessages] = useState([]);
  const [loading, setLoading] = useState(false);

  const handleSend = async () => {
    if (!input.trim()) return;
    const userText = input.trim();
    setInput("");
    setLoading(true);
    setMessages((prev) => [...prev, { role: "user", text: userText }]);

    try {
      const { reply, history: newHistory } = await askClaude(userText, history);
      setHistory(newHistory);
      setMessages((prev) => [...prev, { role: "assistant", text: reply }]);
    } catch (err) {
      setMessages((prev) => [...prev, { role: "error", text: err.message }]);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div style={{ padding: "1rem", fontFamily: "sans-serif", maxWidth: 600 }}>
      <h2>Resource Manager</h2>
      <div style={{ minHeight: 200, border: "1px solid #ccc", borderRadius: 8, padding: 12, marginBottom: 12 }}>
        {messages.map((m, i) => (
          <div key={i} style={{ marginBottom: 8, color: m.role === "error" ? "red" : "inherit" }}>
            <strong>{m.role === "user" ? "You" : m.role === "error" ? "Error" : "Claude"}:</strong> {m.text}
          </div>
        ))}
        {loading && <div><em>Thinking...</em></div>}
      </div>
      <div style={{ display: "flex", gap: 8 }}>
        <input
          style={{ flex: 1, padding: "8px 12px", borderRadius: 6, border: "1px solid #ccc" }}
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && handleSend()}
          placeholder="List resources, create a widget..."
        />
        <button
          onClick={handleSend}
          disabled={loading}
          style={{ padding: "8px 16px", borderRadius: 6, background: "#5A4FCF", color: "white", border: "none", cursor: "pointer" }}
        >
          Send
        </button>
      </div>
    </div>
  );
}
```

### Handling MCP Tool Responses (Advanced)

If you need to inspect tool results directly rather than letting Claude summarize them, filter by block type:

```javascript
const data = await response.json();

// Final natural language reply from Claude
const textReply = data.content
  .filter((b) => b.type === "text")
  .map((b) => b.text)
  .join("\n");

// Raw data returned by the MCP tool (before Claude processed it)
const toolResults = data.content
  .filter((b) => b.type === "mcp_tool_result")
  .flatMap((b) => b.content ?? [])
  .filter((c) => c.type === "text")
  .map((c) => {
    try { return JSON.parse(c.text); }
    catch { return c.text; }
  });

// Which tools were invoked and with what arguments
const toolCalls = data.content
  .filter((b) => b.type === "mcp_tool_use")
  .map((b) => ({ tool: b.name, input: b.input }));
```

---

## Summary: Choosing the Right Pattern

| Context | Mechanism | Auth Pattern | When to Use |
|---|---|---|---|
| SKILL.md | `bash_tool` + curl/SDK | Env vars in container | Simple HTTP calls, no server to host |
| Claude Code | MCP via `stdio` transport | Env vars in `.claude/settings.json` | Full agentic workflows, local dev |
| Artifacts | MCP via `url` (SSE) transport | Hosted MCP server | Deployed apps, end-user facing |

**Rule of thumb:** If you control the deployment environment, go MCP. If you're scripting one-off automations inside the SKILL.md container, curl + env vars is sufficient. MCP via `url` transport is the only option for artifacts since they run in a browser sandbox with no filesystem access.
