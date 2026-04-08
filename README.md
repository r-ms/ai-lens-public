# AI Lens

Hook-based analytics for AI coding sessions. Captures events from Claude Code,
Cursor, and Codex, queues them locally, and ships to a self-hosted server with a
web dashboard and MCP integration.

## Quick Start

### 1. Start the server

```bash
git clone https://github.com/r-ms/ai-lens-public.git
cd ai-lens-public
docker compose up -d
```

This starts three containers:

| Container | Purpose |
|-----------|---------|
| **app** | API server + web dashboard (port 3000) |
| **postgres** | PostgreSQL 16 database |
| **analyzer** | Background session analyzer (optional, needs `claude login`) |

Open the dashboard: **http://localhost:3000**

If port 3000 is already in use, set `PORT` in a `.env` file before starting:

```bash
echo "PORT=3001" > .env
docker compose up -d
npx ai-lens init --server http://localhost:3001 --yes
```

### 2. Connect your AI tools

```bash
npx ai-lens init
```

The setup wizard will:

1. Detect installed AI tools (Claude Code, Cursor, Codex)
2. Install lightweight hook scripts to `~/.ai-lens/client/`
3. Configure hooks in your tool's settings
4. Start the Codex watcher for local Codex sessions (if Codex is detected)
5. Register the MCP server for in-editor analytics (optional)

### 3. Enable session analysis (optional)

The analyzer container uses Claude Code CLI to generate AI-powered insights for
your coding sessions. Authenticate Claude inside the container using one of two
methods:

**Option A — interactive browser login:**

```bash
docker compose run --rm -it analyzer claude auth login
```

This opens a browser-based login flow. Once authenticated, credentials are stored
in a persistent Docker volume (`claude_home`) and survive container restarts.

**Option B — copy credentials from your host** (faster if Claude Code is already
authenticated on your machine):

```bash
docker run --rm \
  -v ai-lens-public_claude_home:/target \
  -v ~/.claude:/source:ro \
  alpine sh -c "
    mkdir -p /target/.claude &&
    cp /source/.credentials.json /target/.claude/ &&
    chown -R 1001:1001 /target/.claude
  "
```

The analyzer runs every hour by default (configurable via `ANALYSIS_INTERVAL`).
Without authentication, the analyzer container starts but analysis will fail —
everything else works normally.

To trigger analysis immediately (bypassing the settling window):

```bash
docker compose run --rm analyzer node /app/scripts/analyze-sessions.js --min-age 0
```

### 4. You're done

Start a coding session in Claude Code, Cursor, or Codex. Events will appear in
the dashboard within seconds.

## How It Works

```
Hook fires → capture.js → normalize → queue.jsonl → sender.js → POST /api/events → server
```

1. **Hook fires** — your AI tool triggers a hook on events (session start, prompt submit, tool use, etc.)
2. **capture.js** — reads the event from stdin, normalizes it to a unified format, appends to a local queue
3. **sender.js** — background process picks up the queue and POSTs events to the server in batches
4. **Server** — stores events in PostgreSQL, derives sessions, serves the dashboard and MCP tools

All processing happens asynchronously — hooks complete in <10ms and never block your AI tool.

## Supported Tools

| Tool | Hook mechanism |
|------|---------------|
| **Claude Code** | Hooks via `~/.claude/settings.json` |
| **Cursor** | Hooks via `~/.cursor/hooks.json` |
| **Codex** | File watcher on `~/.codex` and project-local `.codex` directories |

## Configuration

### CLI commands

```bash
npx ai-lens init       # Setup wizard — detect tools, install hooks, configure MCP
npx ai-lens status     # Run health checks and generate a diagnostic report
npx ai-lens remove     # Remove hooks, client files, and MCP config
npx ai-lens version    # Show installed version
```

### CLI options

```bash
npx ai-lens init [options]
```

| Flag | Description |
|------|-------------|
| `--server URL` | Server URL (default: `http://localhost:3000`) |
| `--yes`, `-y` | Non-interactive mode, accept all defaults |
| `--no-mcp` | Skip MCP server registration |
| `--mcp-scope SCOPE` | MCP scope: `user` (default), `local`, or `project` |
| `--projects LIST` | Comma-separated project paths to monitor (default: all) |
| `--project-hooks` | Write hooks into project directory instead of global config |
| `--use-repo-path` | Run capture.js from package path instead of copying to `~/.ai-lens/client/` |

### Environment variables

Copy `.env.example` to `.env` and adjust as needed:

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Server port |
| `POSTGRES_PASSWORD` | `ailens` | PostgreSQL password |
| `ANALYSIS_INTERVAL` | `3600` | Seconds between analysis runs |
| `OPENAI_API_KEY` | _(none)_ | OpenAI API key for embedding-based vector search (FAISS); text search works without it |

#### Optional: Auth0 SSO

For team deployments with single sign-on:

| Variable | Description |
|----------|-------------|
| `AUTH0_DOMAIN` | Auth0 tenant domain |
| `AUTH0_CLIENT_ID` | Auth0 SPA client ID |
| `AUTH0_AUDIENCE` | Auth0 API audience identifier |
| `AUTH0_ALLOWED_DOMAIN` | Restrict login to a specific email domain |
| `AUTH0_CLI_CLIENT_ID` | Auth0 Native app client ID for device code flow |

Without Auth0, the server uses git email headers for identity (personal mode).

## MCP Tools

When MCP is enabled during `npx ai-lens init`, these tools become available
inside Claude Code and Cursor:

| Tool | Description |
|------|-------------|
| `who_am_i` | Identify yourself by git email — returns your developer profile and team(s) |
| `get_overview` | Organization-wide KPIs: active developers, adoption rate, AI hours, MCP and skill distribution |
| `list_teams` | List all teams with member counts, adoption rate, and AI hours |
| `get_team` | Team detail: KPIs, members, tasks, activity trend, MCP and skill distribution |
| `get_team_analysis` | AI-generated team analysis: achievements, recurring problems, recommendations |
| `get_developer` | Developer profile: sessions, AI hours, tasks, MCP and skill usage, team comparison |
| `get_mcp_distribution` | MCP server usage across the organization |
| `get_chain` | Session chain with compact event timeline, plan mode segments, and timing |
| `get_events` | Full event data for specific event IDs |
| `get_chain_analysis` | AI-generated chain analysis: tasks, problems, tool errors, unanswered questions |
| `request_analysis` | Manually trigger analysis for a specific session chain |
| `get_token_usage` | Token usage statistics grouped by model (input/output/cache tokens) |
| `knowhow_search` | Search the team knowledge base built from session analyses |
| `knowhow_update` | Add or update a knowledge base entry |
| `export_developer_tips` | Export personalized tips as a Markdown document |
| `search` | Natural language search across sessions, tasks, and projects |

## Dashboard

The web dashboard provides:

- Organization-wide KPIs and adoption trends
- Team and developer breakdowns
- Session timelines with tool usage
- AI-generated session and team analyses
- Token usage by model
- MCP server and skill distribution
- Knowledge base and recurring problems

## Event Types

| Type | Source | Description |
|------|--------|-------------|
| `SessionStart` | All | Session opened |
| `SessionEnd` | All | Session closed |
| `UserPromptSubmit` | All | User sent a prompt |
| `PostToolUse` | All | Tool execution completed |
| `PostToolUseFailure` | All | Tool execution failed |
| `Stop` | All | Agent stopped |
| `PreCompact` | All | Context compaction triggered |
| `PlanModeStart` | Claude Code | Entered plan mode |
| `PlanModeEnd` | Claude Code | Exited plan mode |
| `SubagentStart` | All | Subagent spawned |
| `SubagentStop` | All | Subagent finished |
| `FileEdit` | Cursor | File edited |
| `ShellExecution` | Cursor | Shell command executed |
| `MCPExecution` | Cursor | MCP tool executed |
| `AgentResponse` | Cursor | Agent response (includes token usage in raw payload) |
| `AgentThought` | Cursor | Agent reasoning |

## Client Data

Stored in `~/.ai-lens/`:

| File | Purpose |
|------|---------|
| `client/` | Installed client files (capture.js, sender.js, config.js) |
| `config.json` | Server URL, auth token, project list |
| `queue.jsonl` | Pending events |
| `queue.sending.jsonl` | Events being sent (atomic rename as mutex) |
| `sender.log` | Sender activity log |
| `capture.log` | Capture drop log (normalization failures, write errors) |
| `session-paths.json` | Session-to-project path cache |

## Requirements

- Docker & Docker Compose
- Node.js 20+ (for the CLI)

## License

MIT
