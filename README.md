# AI Lens

Hook-based analytics for AI coding sessions. Captures events from Claude Code
and Cursor, queues them locally, and ships to a self-hosted server with a web
dashboard and MCP integration.

## Quick Start

### 1. Start the server

Clone this repo and run:

    git clone https://github.com/r-ms/ai-lens-public.git
    cd ai-lens-public
    docker compose up -d

This starts three containers:

| Container | Purpose |
|-----------|---------|
| **app** | API server + web dashboard (port 3000) |
| **postgres** | PostgreSQL 16 database |
| **analyzer** | Background session analyzer (optional, needs `claude login`) |

Open the dashboard: **http://localhost:3000**

### 2. Connect your AI tools

    npx ai-lens init

The setup wizard will:

1. Detect installed AI tools (Claude Code, Cursor, or both)
2. Install lightweight hook scripts to `~/.ai-lens/client/`
3. Configure hooks in your tool's settings
4. Register the MCP server for in-editor analytics (optional)

### 3. Enable session analysis (optional)

The analyzer container uses Claude Code CLI to generate AI-powered insights for
your coding sessions. To enable it, authenticate Claude inside the container:

    docker exec -it ai-lens-analyzer-1 claude login

This opens a browser-based login flow. Once authenticated, credentials are stored
in a persistent Docker volume (`claude_home`) and survive container restarts.

The analyzer runs every hour by default (configurable via `ANALYSIS_INTERVAL`).
Without authentication, the analyzer container starts but analysis will fail —
everything else works normally.

### 4. You're done

Start a coding session in Claude Code or Cursor. Events will appear in the
dashboard within seconds.

## Configuration

### CLI options

    npx ai-lens init [options]

| Flag | Description |
|------|-------------|
| `--server URL` | Server URL (default: `http://localhost:3000`) |
| `--yes`, `-y` | Non-interactive mode, accept all defaults |
| `--no-mcp` | Skip MCP server registration |
| `--mcp-scope SCOPE` | MCP scope: `user` (default), `local`, or `project` |
| `--projects LIST` | Comma-separated project paths to monitor (default: all) |

### Other commands

    npx ai-lens remove     # Remove hooks, client files, and MCP config
    npx ai-lens status     # Run health checks and generate a diagnostic report
    npx ai-lens version    # Show installed version

### Environment variables

Copy `.env.example` to `.env` and adjust as needed:

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Server port |
| `POSTGRES_PASSWORD` | `ailens` | PostgreSQL password |
| `ANALYSIS_INTERVAL` | `3600` | Seconds between analysis runs |
| `AI_LENS_ADMIN_SECRET` | _(none)_ | Admin secret for auth token management |
| `AUTH0_DOMAIN` | _(none)_ | Auth0 tenant domain |
| `AUTH0_CLIENT_ID` | _(none)_ | Auth0 SPA client ID |
| `AUTH0_AUDIENCE` | _(none)_ | Auth0 API audience identifier |
| `AUTH0_ALLOWED_DOMAIN` | _(none)_ | Restrict login to a specific email domain |
| `AUTH0_CLI_CLIENT_ID` | _(none)_ | Auth0 Native app client ID for device code flow |
| `MCP_SERVER_URL` | `http://localhost:3000` | Public server URL for MCP OAuth callbacks |
| `OPENAI_API_KEY` | _(none)_ | OpenAI API key for session analysis |

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
| `search` | Natural language search across sessions, tasks, and projects |

## How It Works

    Hook fires → capture.js → normalize → queue.jsonl → sender.js → POST /api/events → server

1. **Hook fires** — your AI tool triggers a hook on events (session start, prompt submit, tool use, etc.)
2. **capture.js** — reads the event from stdin, normalizes it to a unified format, appends to a local queue
3. **sender.js** — background process picks up the queue and POSTs events to the server in batches
4. **Server** — stores events in PostgreSQL, derives sessions, serves the dashboard and MCP tools

All processing happens asynchronously — hooks complete in <10ms and never block your AI tool.

## Supported Tools

- **Claude Code** — hooks via `~/.claude/settings.json`
- **Cursor** — hooks via `~/.cursor/hooks.json`

## Requirements

- Docker & Docker Compose
- Node.js 20+ (for the CLI)

## License

MIT
