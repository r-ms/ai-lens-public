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
| **postgres** | PostgreSQL 16 with pgvector (semantic search) |
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
3. Configure hooks in each tool's settings (Codex hooks are pre-trusted in
   `~/.codex/config.toml` so they work in `codex exec` too)
4. Register the MCP server for in-editor analytics (optional)
5. Offer to import your existing Claude Code / Cursor history

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

The analyzer runs every hour by default (configurable via `ANALYSIS_INTERVAL`)
and only analyzes "settled" session chains — ones with no activity for 5+ hours.
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
Hook fires → capture.js → redact → normalize → pending/ spool → sender.js → POST /api/events → server
```

1. **Hook fires** — your AI tool triggers a hook on events (session start, prompt submit, tool use, etc.)
2. **capture.js** — reads the event from stdin, redacts secrets, normalizes it to a unified format, writes it to a local spool
3. **sender.js** — background process picks up the spool and POSTs events to the server in batches
4. **Server** — stores events in PostgreSQL, derives sessions, serves the dashboard and MCP tools

All processing happens asynchronously — hooks complete in <10ms and never block
your AI tool. Secrets (API keys, tokens, passwords, JWTs, connection strings)
are redacted client-side before anything leaves your machine, and again
server-side on ingest.

## Supported Tools

| Tool | Hook mechanism |
|------|---------------|
| **Claude Code** | Hooks via `~/.claude/settings.json` |
| **Cursor** | Hooks via `~/.cursor/hooks.json` |
| **Codex** | Native hooks via `<project>/.codex/hooks.json`, pre-trusted in `~/.codex/config.toml` |

## Configuration

### CLI commands

```bash
npx ai-lens init                  # Setup wizard — detect tools, install hooks, configure MCP
npx ai-lens status                # Run health checks and generate a diagnostic report
npx ai-lens remove                # Remove hooks, client files, and MCP config
npx ai-lens version               # Show installed version

# Historical import (idempotent — safe to re-run)
npx ai-lens import claude-code --days 30              # Import Claude Code history
npx ai-lens import cursor --days 30                   # Import Cursor history (IDE + CLI agent)
npx ai-lens import claude-code --from 2026-06-01 --to 2026-06-07   # Precise window

# Self-service data management
npx ai-lens list-sessions --days 30                   # List your own sessions
npx ai-lens find-session <query>                      # Find a session by id / project / source
npx ai-lens delete-sessions <id…> --yes               # Delete your own sessions (dry-run without --yes)
npx ai-lens delete-sessions --from D --to D --yes     # …or a whole date range
```

### CLI options

```bash
npx ai-lens init [options]
```

| Flag | Description |
|------|-------------|
| `--server URL` | Server URL (default: `http://localhost:3000`) |
| `--yes`, `-y` | Non-interactive mode, accept all defaults (on a fresh install, tracking defaults to the current git repo — not all projects) |
| `--projects LIST` | Comma-separated project paths to monitor |
| `--import` | Import existing Claude Code / Cursor history after setup |
| `--no-mcp` | Skip MCP server registration |
| `--mcp-only` | (Re)register only the MCP server — skip hooks, auth, and import |
| `--mcp-scope SCOPE` | MCP scope: `user` (default), `local`, or `project` |
| `--project-hooks` | Write hooks into the project directory instead of global config |
| `--use-repo-path` | Run capture.js from the package path instead of copying to `~/.ai-lens/client/` |

### Environment variables

Copy `.env.example` to `.env` and adjust as needed:

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Server port |
| `POSTGRES_PASSWORD` | `ailens` | PostgreSQL password |
| `ANALYSIS_INTERVAL` | `3600` | Seconds between analysis runs |
| `OPENAI_API_KEY` | _(none)_ | OpenAI API key for semantic vector search (pgvector); text search works without it |
| `TEAMS_CONFIG` | _(none)_ | JSON team definitions: `{"teams":[{"id":"backend","name":"Backend","members":["a@x.com"]}]}` |

#### Optional: Auth0 SSO

For team deployments with single sign-on:

| Variable | Description |
|----------|-------------|
| `AUTH0_DOMAIN` | Auth0 tenant domain |
| `AUTH0_CLIENT_ID` | Auth0 SPA client ID |
| `AUTH0_AUDIENCE` | Auth0 API audience identifier |
| `AUTH0_ALLOWED_DOMAIN` | Restrict login to a specific email domain |
| `AUTH0_CLI_CLIENT_ID` | Auth0 Native app client ID for device code flow |
| `AI_LENS_ADMIN_SECRET` | Admin secret for token management endpoints |
| `MCP_SERVER_URL` | Public URL of the server (for MCP OAuth callbacks) |
| `CORS_ALLOWED_ORIGINS` | Comma-separated allowed CORS origins |

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
| `find_developer` | Find a developer by name substring or email |
| `get_mcp_distribution` | MCP server usage across the organization |
| `get_chain` | Session chain with compact event timeline, plan mode segments, files touched, and timing |
| `get_events` | Full event data for specific event IDs |
| `get_chain_analysis` | AI-generated chain analysis: tasks, problems, tool errors, unanswered questions |
| `request_analysis` | Manually trigger analysis for a specific session chain |
| `get_token_usage` | Token usage statistics grouped by model (input/output/cache tokens) |
| `get_recommendations` | Rule-based usage signals (context bloat, autocompact recurrence, MCP overhead) |
| `search` | Natural language search across sessions — semantic + full-text + instant substring, with skill/MCP/file filters |
| `grep_events` | Substring search over raw event content |
| `knowhow_search` | Search the team knowledge base built from session analyses |
| `knowhow_update` | Add or update a knowledge base entry |
| `export_developer_tips` | Export personalized tips as a Markdown document |

## Dashboard

The web dashboard provides:

- Organization-wide KPIs and adoption trends
- Team and developer breakdowns
- Session timelines with tool usage
- AI-generated session and team analyses
- Token usage by model
- MCP server and skill distribution
- Knowledge base and recurring problems
- Semantic + full-text search across all sessions

## Event Types

Core lifecycle events, captured from all tools:

| Type | Source | Description |
|------|--------|-------------|
| `SessionStart` / `SessionEnd` | All | Session opened / closed |
| `UserPromptSubmit` | All | User sent a prompt |
| `PostToolUse` / `PostToolUseFailure` | All | Tool execution completed / failed |
| `Stop` | All | Agent stopped |
| `TokenUsage` | All | Per-API-call token usage (input/output/cache) |
| `SubagentStart` / `SubagentStop` | All | Subagent spawned / finished |
| `PreCompact` / `PostCompact` | All | Context compaction |
| `PlanModeStart` / `PlanModeEnd` | Claude Code | Entered / exited plan mode |
| `AssistantText` / `AssistantThinking` | Claude Code, Codex | Assistant response / reasoning text |
| `PermissionRequest`, `Notification`, `TaskCreated`, `TaskCompleted` | Claude Code | Observability events |
| `FileEdit`, `ShellExecution`, `MCPExecution` | Cursor | Per-tool execution detail |
| `AgentResponse`, `AgentThought` | Cursor | Agent response / reasoning |

## Client Data

Stored in `~/.ai-lens/`:

| Path | Purpose |
|------|---------|
| `client/` | Installed client files (capture.js, sender.js, config.js, redact.js) |
| `config.json` | Server URL, auth token, project list |
| `pending/` | Event spool — one file per pending batch |
| `sending/` | Batches currently being sent (atomic rename as mutex) |
| `sender.log` | Sender activity log |
| `capture.log` | Capture drop log (normalization failures, write errors) |
| `session-paths/` | Session-to-project path cache |
| `init.log` | Setup wizard log |

## Upgrading

```bash
docker compose pull && docker compose up -d
```

Database migrations run automatically when `app` starts.

> **Upgrading from an older version of this kit** (postgres was
> `postgres:16-alpine`): the database image is now `pgvector/pgvector:pg16` —
> the server requires the pgvector extension. The base image change (Alpine →
> Debian) means the existing data volume is not directly compatible. Dump with
> the old image, then restore into the new one:
>
> ```bash
> docker compose exec postgres pg_dump -U ailens ailens > backup.sql
> docker compose down
> docker volume rm ai-lens-public_pgdata
> docker compose up -d postgres
> docker compose exec -T postgres psql -U ailens ailens < backup.sql
> docker compose up -d
> ```

CLI updates reach developers automatically on the next `npx ai-lens@latest` run.

## Requirements

- Docker & Docker Compose
- Node.js 22.5+ (for the CLI; `import cursor` uses the built-in `node:sqlite`)

## License

MIT
