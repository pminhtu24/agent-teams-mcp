<div align="center">

# Agent Teams MCP

MCP server that implements Claude Code's [agent teams](https://code.claude.ai/docs/agent-teams) protocol for any MCP client.

</div>

## Overview

Claude Code ships with agent teams, but it's a vendor-locked implementation. This project decouples the protocol from the tool, offering an MCP server that any client can consume. Same team orchestration, no vendor lock-in.

## Features

- **Team Management**: Create/delete teams with persistent configuration
- **Agent Spawning**: Spawn teammates in tmux panes or windows
- **Inter-Agent Messaging**: JSON inbox system with file locking
- **Task Coordination**: Shared task lists with dependencies and ownership
- **Multi-Backend**: Support for Claude Code and OpenCode backends
- **Push Notifications**: Real-time notifications for OpenCode via HTTP API

## Requirements

- Python 3.12+
- [tmux](https://github.com/tmux/tmux)
- At least one coding agent on PATH:
  - [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (`claude`)
  - [OpenCode](https://opencode.ai) (`opencode`)
- OpenCode teammates require `OPENCODE_SERVER_URL` and the MCP server connected in that instance

## Installation

### For OpenCode

Add to `~/.config/opencode/opencode.json`:

```json
{
  "mcp": {
    "agent-teams": {
      "type": "local",
      "command": [
        "uvx",
        "--from",
        "git+https://github.com/pminhtu24/agent-teams-mcp",
        "agent-teams"
      ],
      "enabled": true
    }
  }
}
```

### From Source

```bash
git clone https://github.com/pminhtu24/agent-teams-mcp.git
cd agent-teams-mcp
uv sync
uv run python -m agent_teams.server
```

## Configuration

| Environment Variable    | Description                                             | Default                 |
| ----------------------- | ------------------------------------------------------- | ----------------------- |
| `CLAUDE_TEAMS_BACKENDS` | Comma-separated enabled backends (`claude`, `opencode`) | Auto-detect from client |
| `OPENCODE_SERVER_URL`   | OpenCode HTTP API URL (required for opencode teammates) | _(unset)_               |
| `USE_TMUX_WINDOWS`      | Spawn teammates in tmux windows instead of panes        | _(unset)_               |

### Backend Configuration

```json
{
  "mcpServers": {
    "agent-teams": {
      "command": "uvx",
      "args": [
        "--from",
        "git+https://github.com/pminhtu24/agent-teams-mcp",
        "agent-teams"
      ],
      "env": {
        "AGENT_TEAMS_BACKENDS": "opencode",
        "OPENCODE_SERVER_URL": "http://localhost:4096"
      }
    }
  }
}
```

## MCP Tools

| Tool                        | Description                                          |
| --------------------------- | ---------------------------------------------------- |
| `team_create`               | Create a new agent team (one per session)            |
| `team_delete`               | Delete team and all data (fails if teammates active) |
| `spawn_teammate`            | Spawn a teammate in tmux                             |
| `send_message`              | Send DMs, broadcasts, shutdown/plan responses        |
| `read_inbox`                | Read messages from an agent's inbox                  |
| `read_config`               | Read team config and member list                     |
| `task_create`               | Create a task (auto-incrementing ID)                 |
| `task_update`               | Update task status, owner, dependencies              |
| `task_list`                 | List all tasks                                       |
| `task_get`                  | Get full task details                                |
| `check_teammate`            | Check teammate status, unread messages, tmux output  |
| `force_kill_teammate`       | Kill a teammate's tmux pane/window                   |
| `process_shutdown_approved` | Remove teammate after graceful shutdown              |

## Architecture

### Spawning

Teammates launch in tmux panes (default) or windows (`USE_TMUX_WINDOWS`). Each gets a unique agent ID and color.

### Messaging

JSON inboxes at `~/.claude/teams/<team>/inboxes/`. Lead messages anyone; teammates message only lead.

```
~/.claude/
├── teams/<team>/
│   ├── config.json
│   └── inboxes/
│       ├── team-lead.json
│       ├── worker-1.json
│       └── .lock
└── tasks/<team>/
    ├── 1.json
    ├── 2.json
    └── .lock
```

### Concurrency

Atomic writes via `tempfile` + `os.replace`. Cross-platform file locks via `filelock`.

### Backend Differences

| Feature            | Claude Code                         | OpenCode                      |
| ------------------ | ----------------------------------- | ----------------------------- |
| Push notifications | Poll only                           | HTTP API                      |
| Agent CLI flags    | `--agent-id`, `--agent-color`, etc. | Wrapper prompt with MCP tools |
| Session management | N/A                                 | HTTP sessions                 |

## Development

### Setup

```bash
uv sync
source .venv/bin/activate
```

### Testing

```bash
uv run pytest                    # Run all tests
uv run pytest tests/test_tasks.py -v  # Run specific file
uv run pytest -k "test_name" -v       # Run matching pattern
```

### Project Structure

```
src/agent_teams/
├── server.py           # MCP entrypoint, all tools
├── models.py           # Pydantic models
├── teams.py            # Team CRUD
├── tasks.py            # Task management with dependencies
├── messaging.py        # Inbox system
├── spawner.py          # tmux spawning
├── opencode_client.py  # OpenCode HTTP client
└── _filelock.py        # Cross-platform file locking
```

## Communication Model

```
                 Team-lead
                /    |    \
               /     |     \
         Agent A  Agent B  Agent C
            ↕        ↕        ↕
      (message lead only)
```

Agents can only send direct messages to team-lead. For agent-to-agent communication, relay through lead.

## License

[MIT](./LICENSE)
