# MCP — Model Context Protocol Reference
> Attach when setting up MCP servers, integrating external tools with AI assistants, or evaluating MCP server security. Covers architecture, configuration, and key servers worth using.

---

## 1. What MCP Is

MCP (Model Context Protocol) is a standard that allows AI assistants to interact with external tools and data sources through a consistent interface. Think of it as a plugin system for AI — instead of pasting API docs into a prompt, you connect the AI directly to a live data source or tool.

**Analogy:** MCP is to AI tools what USB is to hardware — a standard port that lets any compliant tool connect to any compliant AI.

**Developed by:** Anthropic. Now adopted by OpenAI, Google, and major IDE vendors.

---

## 2. Three Types of MCP Capabilities

| Type | Controlled by | What it does | Example |
|------|--------------|--------------|---------|
| **Tools** | Model | AI decides when to call these | "Search the web", "Run a query" |
| **Resources** | Application | App makes data available for context | Current file, project files |
| **Prompts** | User | User-invocable templates | Pre-built workflows |

**Most MCP servers you will use expose Tools** — the AI calls them automatically when relevant.

---

## 3. Configuring MCP in Claude Code

```json
// .claude/mcp.json (project-level, committed)
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/directory"]
    }
  }
}
```

```json
// ~/.claude/mcp.json (global, user-level — not committed)
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."
      }
    }
  }
}
```

**Rule:** Project-level MCP config (`.claude/mcp.json`) can be committed — it defines what tools the AI uses in this project. Global config (`~/.claude/mcp.json`) should not be committed — it contains personal tokens.

---

## 4. MCP in Cursor

```json
// .cursor/mcp.json (project-level)
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    }
  }
}
```

Or configure via: Cursor Settings → MCP → Add Server

---

## 5. Useful MCP Servers

### Context7 — Live Documentation

Fetches current, version-specific documentation for any library and injects it into AI context. Eliminates hallucinated API signatures from outdated training data.

```json
{
  "command": "npx",
  "args": ["-y", "@upstash/context7-mcp@latest"]
}
```

**Usage in prompt:**
```
Use context7 to get the latest Next.js 15 documentation for the App Router, 
then help me set up a middleware for authentication.
```

**When to use:** Any time you're working with a library that has changed recently or where AI consistently gives outdated syntax.

### Filesystem

Gives the AI controlled access to read/write files outside the current project directory.

```json
{
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/you/allowed-folder"]
}
```

### GitHub

Lets the AI read issues, PRs, and code from GitHub repositories.

```json
{
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-github"],
  "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..." }
}
```

### Brave Search / Web Search

Gives the AI ability to search the web for current information.

### PostgreSQL / SQLite

Lets the AI query your database directly (read-only mode recommended).

**Finding more:** `github.com/punkpeye/awesome-mcp-servers` — curated list of verified MCP servers.

---

## 6. llms.txt — AI-Friendly Documentation

Some projects publish `llms.txt` at their root — a simplified, AI-optimized version of their documentation. This is the AI equivalent of `robots.txt`.

Format defined by Jeremy Howard (fast.ai).

**Where to look:**
- `https://[library-domain]/llms.txt` (e.g., `https://docs.astro.build/llms.txt`)
- `https://[library-domain]/llms-full.txt` — extended version with full API reference

**How to use in prompts:**
```
Fetch https://docs.astro.build/llms.txt and use it as context 
to help me add a content collection to my Astro project.
```

**Context7 does this automatically** — you don't need to manually fetch llms.txt if you have Context7 MCP configured.

---

## 7. Security Checklist for MCP

MCP servers run on your machine with your credentials. Evaluate each server before installing:

- [ ] **Source code is public and readable** — avoid servers that are closed source
- [ ] **Verified publisher** — check if the package is from a known organization
- [ ] **Minimal permissions** — the server should request only what it needs
- [ ] **No outbound network calls for data you didn't intend to send** — review what data the server sends
- [ ] **Token scopes are minimal** — GitHub token should have only the scopes needed (read:repo, not full admin)
- [ ] **Secrets go in global config, not project config** — tokens in `~/.claude/mcp.json`, not committed `.claude/mcp.json`
- [ ] **User consent for sensitive operations** — servers that modify files, send emails, or call APIs should confirm before acting

**Principle of least privilege:** Give each MCP server only the access it actually needs. A documentation server doesn't need filesystem write access.

**Principle of minimal tool sets:** More tools available to an agent does not mean better performance — it means more confusion. When an engineer cannot immediately say which tool applies to a situation, neither can the LLM. Each additional tool is a branch the model must reason about before acting.

Practical limit: expose **5–7 tools maximum** per agent or MCP context. If you have more, group them into separate agents with focused roles (a "search agent" with 3 search tools vs a "write agent" with 3 write tools) rather than giving one agent 10 tools.

---

## 8. Building a Simple MCP Server (TypeScript)

If you need a custom integration, MCP servers are straightforward to build:

```bash
npm create cloudflare@latest -- my-mcp-server --template=mcp
# or build locally:
npm install @modelcontextprotocol/sdk
```

```typescript
// Minimal MCP server with one tool
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import { z } from 'zod'

const server = new McpServer({ name: 'my-server', version: '1.0.0' })

server.tool(
  'get_project_status',
  'Returns the current status of the project',
  { includeMetrics: z.boolean().optional() },
  async ({ includeMetrics }) => {
    const status = await fetchProjectStatus()
    return {
      content: [{
        type: 'text',
        text: JSON.stringify(status, null, 2),
      }]
    }
  }
)

const transport = new StdioServerTransport()
await server.connect(transport)
```

**Use cases for custom MCP servers:**
- Query your internal APIs or databases
- Integrate with proprietary tools (Jira, Confluence, internal dashboards)
- Create project-specific workflows (deploy, release notes generation)

---

## 9. MCP AI Instructions for CLAUDE.md

```markdown
## MCP Configuration

### Configured servers (project-level)
- context7: fetches live library documentation — use for any library API questions
- [list other servers]

### How to use context7
Add "use context7" to your prompt when you need current documentation:
"Use context7 to get Next.js 15 App Router docs, then..."

### Security
- MCP server tokens are in ~/.claude/mcp.json (not committed)
- Project MCP config in .claude/mcp.json contains no secrets
```
