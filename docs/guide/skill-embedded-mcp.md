# Writing a Skill-Embedded MCP Server

Skill-embedded MCPs are MCP servers that ship alongside a skill. They spin up on-demand when the skill is loaded and are scoped to the session. No global context bloat.

## Two Ways to Declare

You have two options. Pick one (if both exist, `mcp.json` wins).

### Option A: YAML Frontmatter in SKILL.md

Declare MCP servers under the `mcp:` key in the frontmatter:

```yaml
---
name: my-db-skill
description: Database query skill with embedded SQLite MCP
mcp:
  sqlite:
    command: uvx
    args:
      - mcp-server-sqlite
      - --db-path
      - ./data.db
  memory:
    command: npx
    args: [-y, "@anthropic-ai/mcp-server-memory"]
---

Use the `skill_mcp` tool to query the database:
- skill_mcp(mcp_name="sqlite", tool_name="query", arguments='{"sql": "SELECT * FROM users"}')
```

### Option B: mcp.json File

Place an `mcp.json` file next to your `SKILL.md`:

```
my-skill/
  SKILL.md
  mcp.json     <-- MCP config lives here
```

Two formats are accepted:

**Wrapped format** (matches `.mcp.json` convention):

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

**Direct format** (servers at top level, detected by presence of `command` field):

```json
{
  "sqlite": {
    "command": "uvx",
    "args": ["mcp-server-sqlite"]
  }
}
```

## Supported Config Fields

Each server entry follows this shape (from `ClaudeCodeMcpServer`):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `command` | string | for stdio | The binary to spawn |
| `args` | string[] | no | Command arguments |
| `env` | Record\<string, string\> | no | Environment variables. Supports `${VAR}` expansion |
| `url` | string | for HTTP | Remote MCP server URL |
| `headers` | Record\<string, string\> | no | HTTP headers (supports `${VAR}` expansion) |
| `type` | `"stdio"` \| `"http"` \| `"sse"` | no | Explicit transport. Inferred if omitted |
| `oauth` | `{clientId?, scopes?}` | no | OAuth 2.0 + PKCE for remote servers |
| `disabled` | boolean | no | Skip this server |

## Connection Type Detection

If you omit `type`, the manager infers transport:

1. `type: "http"` or `type: "sse"` set explicitly -> HTTP
2. `type: "stdio"` set explicitly -> stdio
3. `url` present -> HTTP
4. `command` present -> stdio
5. Neither -> error at connection time

## How It Works at Runtime

```
1. Agent calls skill("my-db-skill")
   -> Skill loaded, mcpConfig extracted from frontmatter/mcp.json

2. Agent calls skill_mcp(mcp_name="sqlite", tool_name="query", arguments='...')
   -> skill_mcp tool scans all loaded skills for matching mcp_name
   -> SkillMcpManager.getOrCreateClient() connects (or reuses existing connection)
   -> For stdio: spawns process via StdioClientTransport
   -> For HTTP: connects via StreamableHTTPClientTransport
   -> Executes the MCP operation (tool/resource/prompt)

3. Connection stays alive for 5 minutes idle, then auto-cleaned up
4. Session disconnect kills all connections for that session
```

## Three Operation Types

The `skill_mcp` tool supports three MCP primitives. Exactly one must be specified per call:

```
# Call a tool
skill_mcp(mcp_name="sqlite", tool_name="query", arguments='{"sql": "SELECT 1"}')

# Read a resource
skill_mcp(mcp_name="memory", resource_name="memory://notes")

# Get a prompt
skill_mcp(mcp_name="helper", prompt_name="summarize", arguments='{"text": "..."}')
```

Optional `grep` parameter filters output lines by regex:

```
skill_mcp(mcp_name="sqlite", tool_name="query", arguments='...', grep="error|warning")
```

## Complete Example: Stdio Skill

```
my-sql-skill/
  SKILL.md
```

````yaml
---
name: sql-assistant
description: Query databases using natural language
mcp:
  sqlite:
    command: uvx
    args:
      - mcp-server-sqlite
      - --db-path
      - ./project.db
---

# SQL Assistant

You have access to a SQLite database via the `sqlite` MCP server.

## Available Operations

- **query**: Execute SQL queries
- **list_tables**: Show all tables
- **describe_table**: Show table schema

## Usage

Use `skill_mcp` to interact:

```
skill_mcp(mcp_name="sqlite", tool_name="list_tables", arguments='{}')
skill_mcp(mcp_name="sqlite", tool_name="query", arguments='{"sql": "SELECT * FROM users LIMIT 10"}')
```
````

## Complete Example: HTTP Skill with Auth

```yaml
---
name: api-connector
description: Connect to a remote API service
mcp:
  api-server:
    url: https://mcp.example.com/mcp
    headers:
      Authorization: Bearer ${API_KEY}
    env:
      API_KEY: ${API_KEY}
---

# API Connector

Connects to the remote API. The `${API_KEY}` environment variable must be set.
```

## Complete Example: Builtin Skill (Programmatic)

If you're writing a builtin skill in TypeScript (under `src/features/builtin-skills/skills/`):

```typescript
import type { BuiltinSkill } from "../types"

export const mySkill: BuiltinSkill = {
  name: "my-tool",
  description: "Does the thing",
  template: `# My Tool\n\nInstructions here.`,
  mcpConfig: {
    "my-server": {
      command: "npx",
      args: ["@my-org/mcp-server@latest"],
    },
  },
}
```

## Key Details

- **Priority**: `mcp.json` always beats YAML frontmatter. Don't use both.
- **Env expansion**: `${VAR}` in `env`, `headers`, `url`, and `args` fields gets expanded from process environment at connection time.
- **Retry**: Operations auto-retry up to 3 times on connection failures, with automatic reconnection.
- **OAuth**: HTTP servers can use OAuth 2.0 + PKCE via the `oauth` field. Step-up auth is handled automatically.
- **Idle cleanup**: Connections idle for 5 minutes are automatically closed.
- **Session scoping**: Each connection is keyed by `sessionID:skillName:serverName`. Different sessions get independent connections.

## Skill Directory Locations

Skills (and their embedded MCPs) are discovered from 4 scopes, highest priority first:

1. **Project**: `.opencode/skills/my-skill/SKILL.md`
2. **OpenCode config**: `~/.config/opencode/skills/my-skill/SKILL.md`
3. **User**: `~/.config/opencode/oh-my-opencode/skills/my-skill/SKILL.md`
4. **Builtin**: Compiled into the plugin

Same-named skill at a higher scope overrides the lower one entirely (including its MCP config).
