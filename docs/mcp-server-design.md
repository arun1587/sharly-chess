# Sharly Chess — MCP Server: Implementation & Design Document

> **Audience:** Coding agents / developers implementing the MCP server.
> **Goal:** Expose Sharly Chess tournament data and operations as MCP tools that AI assistants (e.g. Claude Desktop) can call, packaged alongside the existing application.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Packaging & Project Structure](#2-packaging--project-structure)
3. [Service Layer](#3-service-layer)
4. [Tool Catalogue](#4-tool-catalogue)
5. [Authentication (AuthN)](#5-authentication-authn)
6. [Authorization (AuthZ)](#6-authorization-authz)
7. [Safety & Guardrails](#7-safety--guardrails)
8. [Configuration](#8-configuration)
9. [Error Handling](#9-error-handling)
10. [Audit Logging](#10-audit-logging)
11. [Testing](#11-testing)
12. [Claude Desktop Integration](#12-claude-desktop-integration)

---

## 1. Overview

The MCP (Model Context Protocol) server is a **thin tool-dispatch layer** that sits alongside the existing Litestar web application. It reuses the existing service/data layer directly — no business logic lives in the MCP server itself.

```
Claude Desktop / AI Agent
        │  (stdio or SSE transport)
        ▼
┌──────────────────────┐
│   MCP Server         │  src/mcp/server.py
│   Tool definitions   │
│   AuthN / AuthZ      │
└──────────┬───────────┘
           │ calls (same process or subprocess)
           ▼
┌──────────────────────┐
│   Service Layer      │  src/data/event.py, tournament.py, …
│   (existing code)    │
└──────────┬───────────┘
           ▼
┌──────────────────────┐
│   SQLite Databases   │  aiosqlite / event_database.py
└──────────────────────┘
```

**Key constraints:**
- The MCP server **must not** bypass the existing access-control system.
- Write operations are **opt-in** and must be explicitly enabled per service-account token.
- The MCP server is a separate **entry point**, not a separate package.

---

## 2. Packaging & Project Structure

### 2.1 New directories

```
src/
└── mcp/
    ├── __init__.py
    ├── server.py          # Entry point — builds MCP app, registers tools
    ├── tools/
    │   ├── __init__.py
    │   ├── events.py      # read tools: list_events, get_event
    │   ├── tournaments.py # read tools: list_tournaments, get_tournament, get_pairings, get_standings
    │   ├── players.py     # read tools: list_players, get_player
    │   └── results.py     # write tool: set_result  (guarded)
    ├── auth.py            # token validation, service-account → Client mapping
    ├── audit.py           # append-only audit log
    └── config.py          # loads mcp_config.toml
```

### 2.2 pyproject.toml changes

Add the MCP entry point and the `mcp` dependency:

```toml
[project.scripts]
# existing
sharly-chess = "sharly_chess:main"
# new
sharly-chess-mcp = "mcp.server:main"

[project.dependencies]
# … existing deps …
mcp = ">=1.0"               # Anthropic MCP Python SDK

[project.optional-dependencies]
mcp = ["mcp>=1.0"]          # installable separately if desired
```

### 2.3 Runtime relationship

- The MCP server **can** share the same SQLite files as the running web app because SQLite WAL mode supports concurrent readers.
- The web app does **not** need to be running for the MCP server to work — it accesses the databases directly through the same `EventDatabase` / `ConfigDatabase` classes.
- Version parity is automatic: one `pip install` delivers both entry points.

---

## 3. Service Layer

### 3.1 Reuse existing classes — do not duplicate

| Need | Existing class | Location |
|---|---|---|
| Load event list | `EventLoader` | `src/data/event.py` |
| Load single event | `EventLoader.get_event()` | `src/data/event.py` |
| Tournament data (pairings, standings) | `Tournament` | `src/data/tournament.py` |
| Player data | `TournamentPlayer` | `src/data/tournament.py` |
| Set game result | `Tournament.set_result()` | `src/data/tournament.py` |
| Access-level model | `Client`, `AuthAction` | `src/data/access_levels/` |
| Database connection | `EventDatabase` | `src/database/sqlite/event/event_database.py` |
| Config | `SharlyChessConfig` | `src/common/sharly_chess_config.py` |

### 3.2 Constructing a `Client` for MCP calls

The existing `Client` class (used by web guards) must be instantiated for every MCP tool call, mapping the incoming service-account token to an access level:

```python
# src/mcp/auth.py

from data.access_levels.client import Client
from data.access_levels.access_levels import AccessLevel

def client_from_token(token: str, service_accounts: dict) -> Client:
    """
    Resolve a bearer token to a Client with the correct access level.
    Raises McpAuthError if the token is unknown or revoked.
    """
    account_cfg = service_accounts.get(token)
    if account_cfg is None:
        raise McpAuthError("Unknown token")
    return Client.build_service_account(
        access_level=AccessLevel[account_cfg["access_level"]],
        name=account_cfg["name"],
    )
```

`Client.build_service_account()` is a **new factory method** to add to `Client` (a few lines). It sets `is_admin` and the permission set the same way the localhost shortcut does, but driven by config rather than IP address.

---

## 4. Tool Catalogue

All tools follow the same pattern:
1. Validate and deserialise arguments.
2. Resolve the caller's `Client` from the token passed in MCP context.
3. Check the required `AuthAction` using existing guard logic.
4. Call the service layer.
5. Return a serialised result (plain dict / list — no HTML).

### 4.1 Read tools (no side-effects)

#### `list_events`
- **Auth required:** `AuthAction.VIEW_EVENTS` (or public, depending on event visibility)
- **Arguments:** `include_private: bool = false`
- **Returns:** `[{id, name, start_date, status}]`

#### `get_event`
- **Auth required:** `AuthAction.VIEW_EVENTS`
- **Arguments:** `event_id: str`
- **Returns:** full event metadata dict

#### `list_tournaments`
- **Auth required:** `AuthAction.VIEW_EVENTS`
- **Arguments:** `event_id: str`
- **Returns:** `[{id, name, rounds_total, rounds_played, status}]`

#### `get_tournament`
- **Auth required:** `AuthAction.VIEW_EVENTS`
- **Arguments:** `event_id: str`, `tournament_id: str`
- **Returns:** tournament metadata + current standings summary

#### `get_pairings`
- **Auth required:** `AuthAction.VIEW_EVENTS`
- **Arguments:** `event_id: str`, `tournament_id: str`, `round: int | None` (default = current round)
- **Returns:** `[{board, white_player, black_player, result}]`

#### `get_standings`
- **Auth required:** `AuthAction.VIEW_EVENTS`
- **Arguments:** `event_id: str`, `tournament_id: str`
- **Returns:** `[{rank, player, score, tie_breaks}]`

#### `list_players`
- **Auth required:** `AuthAction.VIEW_PLAYERS`
- **Arguments:** `event_id: str`, `tournament_id: str`, `search: str | None`
- **Returns:** `[{id, last_name, first_name, rating, status}]`

#### `get_player`
- **Auth required:** `AuthAction.VIEW_PLAYERS`
- **Arguments:** `event_id: str`, `tournament_id: str`, `player_id: str`
- **Returns:** player detail dict (PII fields filtered based on access level)

### 4.2 Write tools (side-effects — opt-in)

Write tools are **only registered** if the service account has `"allow_writes": true` in `mcp_config.toml`.

#### `set_result`
- **Auth required:** `AuthAction.SET_RESULTS`
- **Arguments:** `event_id`, `tournament_id`, `round`, `board`, `result: "1-0" | "0-1" | "=" | "*"`
- **Guardrails:** see Section 7
- **Returns:** `{ok: true, new_standings_preview: [...]}` (preview of standings change)

---

## 5. Authentication (AuthN)

### 5.1 Transport

The MCP server runs over **stdio** (default for Claude Desktop) or optionally SSE. For stdio, no network auth is needed — the caller is the local OS process. For SSE (remote), tokens are passed as an HTTP `Authorization: Bearer <token>` header.

### 5.2 Service-account tokens

- Tokens are **random 32-byte hex strings** (`secrets.token_hex(32)`).
- Stored only in `mcp_config.toml` (never in the application database).
- The config file must be **0600** permissions (the startup check enforces this).

### 5.3 Token passing in MCP context

The MCP Python SDK provides a `request_context` object. Tokens should be passed via:
- **stdio mode:** an environment variable `SHARLY_MCP_TOKEN` set in the Claude Desktop config.
- **SSE mode:** the `Authorization` header, read from `request_context.headers`.

```python
# src/mcp/server.py  (stdio mode example)
import os
TOKEN = os.environ.get("SHARLY_MCP_TOKEN", "")
```

---

## 6. Authorization (AuthZ)

### 6.1 Mapping tokens to access levels

Each token in `mcp_config.toml` declares an access level name that **must** match one of the existing `AccessLevel` enum values:

```toml
[[service_accounts]]
name        = "read-only-bot"
token       = "abcdef..."
access_level = "OBSERVER"       # maps to AccessLevel.OBSERVER
allow_writes = false

[[service_accounts]]
name        = "results-entry-bot"
token       = "123456..."
access_level = "RESULTS_ENTRY"  # maps to AccessLevel.RESULTS_ENTRY
allow_writes = true
```

### 6.2 Per-call AuthAction check

Every tool handler calls a shared helper before touching any data:

```python
# src/mcp/auth.py

def require_action(client: Client, action: AuthAction, event=None) -> None:
    """
    Mirror of the existing ActionGuard logic.
    Raises McpPermissionError if the client cannot perform the action.
    """
    if not client.can_perform(action, event=event):
        raise McpPermissionError(f"Action {action.name} not allowed for this token")
```

This is intentionally identical to how `ActionGuard` works in `src/web/guards.py` — the same permission matrix is enforced.

### 6.3 PII filtering

`get_player` and `list_players` must strip fields (e.g. birth date, FIDE ID, email) unless the client holds `AuthAction.VIEW_PRIVATE_PLAYER_DATA`. Apply this at the serialisation step, not in the service layer.

---

## 7. Safety & Guardrails

### 7.1 Write operations are disabled by default

The `set_result` tool (and any future write tools) are **not registered** unless `allow_writes = true` is present in the service-account config **and** the token actually holds the required `AuthAction`.

### 7.2 Input validation

All tool arguments must be validated before reaching the service layer:

| Check | Rule |
|---|---|
| `event_id` | Must match `^[a-zA-Z0-9_-]{1,64}$` |
| `tournament_id` | Same pattern |
| `player_id` | Same pattern |
| `round` | Integer, 1 ≤ round ≤ max rounds |
| `result` | Enum: `"1-0"`, `"0-1"`, `"="`, `"*"` only |

Fail fast with a descriptive `McpValidationError` — never pass raw user strings to SQL or filesystem paths.

### 7.3 Rate limiting

A simple in-process token-bucket limiter per service-account token:

```python
MAX_CALLS_PER_MINUTE = 60   # reads
MAX_WRITES_PER_MINUTE = 10  # writes
```

Exceeding the limit raises `McpRateLimitError` (maps to MCP error code `-32029`).

### 7.4 Confirmation for destructive write tools

For `set_result` when overwriting an **already-set** result, the tool returns an intermediate response asking for confirmation rather than applying the change immediately:

```json
{
  "requires_confirmation": true,
  "message": "Round 3, Board 2 already has result '1-0'. Overwrite with '='?",
  "confirmation_token": "<one-time-use UUID>"
}
```

The agent must call `set_result` again with `confirmation_token` set to proceed. The token expires after 60 seconds.

### 7.5 Read-only SQLite connection for read tools

Open all databases used by read tools with `uri=True` and `?mode=ro` to prevent accidental writes at the driver level:

```python
conn = await aiosqlite.connect(f"file:{db_path}?mode=ro", uri=True)
```

Write tools use the normal read-write connection — same as the web app.

### 7.6 No shell execution, no file system writes

The MCP server must not:
- Execute subprocesses (`subprocess`, `os.system`)
- Write to the filesystem (except through the existing `EventDatabase` write path)
- Expose internal Python tracebacks to the caller (log them server-side only)

---

## 8. Configuration

### 8.1 `mcp_config.toml` (gitignored, user-created)

```toml
[server]
transport       = "stdio"          # or "sse"
sse_host        = "127.0.0.1"      # only used if transport = "sse"
sse_port        = 8765             # only used if transport = "sse"
log_level       = "INFO"
audit_log_path  = "~/.sharly-chess/mcp_audit.log"

[[service_accounts]]
name            = "claude-desktop"
token           = "REPLACE_WITH_secrets.token_hex(32)"
access_level    = "OBSERVER"
allow_writes    = false

# Example second account with write access:
# [[service_accounts]]
# name            = "results-bot"
# token           = "REPLACE_WITH_ANOTHER_TOKEN"
# access_level    = "RESULTS_ENTRY"
# allow_writes    = true
```

### 8.2 Config loading (`src/mcp/config.py`)

```python
import tomllib, pathlib, stat

def load_mcp_config(path: str) -> McpConfig:
    p = pathlib.Path(path).expanduser()
    # Enforce restrictive permissions on non-Windows
    if hasattr(stat, "S_IRWXG"):
        mode = p.stat().st_mode & 0o777
        if mode & 0o077:
            raise McpConfigError(
                f"{p} is readable by group/others (mode {oct(mode)}). "
                "Run: chmod 600 mcp_config.toml"
            )
    with p.open("rb") as f:
        return McpConfig.from_dict(tomllib.load(f))
```

### 8.3 CLI flag

```
sharly-chess-mcp --config /path/to/mcp_config.toml
```

`--config` defaults to `~/.sharly-chess/mcp_config.toml` if omitted.

---

## 9. Error Handling

Define a small exception hierarchy in `src/mcp/errors.py`:

```python
class McpBaseError(Exception):
    mcp_code: int = -32000

class McpAuthError(McpBaseError):
    mcp_code = -32001   # Unknown / revoked token

class McpPermissionError(McpBaseError):
    mcp_code = -32003   # Token lacks required AuthAction

class McpValidationError(McpBaseError):
    mcp_code = -32602   # Invalid params (standard JSON-RPC)

class McpRateLimitError(McpBaseError):
    mcp_code = -32029   # Too many requests

class McpNotFoundError(McpBaseError):
    mcp_code = -32004   # Event/tournament/player not found
```

A top-level error handler in `server.py` catches these and converts them to proper MCP error responses. Internal Python exceptions are caught, logged server-side with full traceback, and returned to the caller as a generic `McpBaseError` with no stack trace.

---

## 10. Audit Logging

Every tool call must be appended to an append-only audit log **before** the response is returned — including failed calls.

### 10.1 Log format (JSONL)

```json
{
  "ts":        "2025-09-01T14:32:10.123Z",
  "account":   "claude-desktop",
  "tool":      "set_result",
  "args":      {"event_id": "open2025", "tournament_id": "open", "round": 3, "board": 2, "result": "="},
  "outcome":   "ok",
  "error":     null,
  "duration_ms": 42
}
```

For write tools, also log the **before/after state**:

```json
{
  ...
  "before": {"result": "1-0"},
  "after":  {"result": "="}
}
```

### 10.2 Implementation

```python
# src/mcp/audit.py
import json, pathlib
from datetime import datetime, timezone

class AuditLog:
    def __init__(self, path: str):
        self._path = pathlib.Path(path).expanduser()
        self._path.parent.mkdir(parents=True, exist_ok=True)

    def record(self, entry: dict) -> None:
        line = json.dumps(entry, default=str) + "\n"
        with self._path.open("a", encoding="utf-8") as f:
            f.write(line)
```

---

## 11. Testing

### 11.1 Unit tests (`tests/unit/mcp/`)

- `test_auth.py` — token validation, unknown token, revoked token
- `test_authz.py` — each `AuthAction` allowed/denied per access level
- `test_tools_read.py` — each read tool with mock service layer
- `test_tools_write.py` — `set_result` happy path, confirmation flow, overwrite guard
- `test_rate_limit.py` — token bucket behaviour
- `test_config.py` — config loading, permission check, missing file

### 11.2 Integration tests (`tests/e2e/mcp/`)

Use `mcp.client.stdio` from the MCP Python SDK to spin up the real MCP server against the test SQLite database (same fixture as existing e2e tests):

```python
async def test_list_events(mcp_client):
    result = await mcp_client.call_tool("list_events", {})
    assert isinstance(result, list)
    assert result[0]["name"] == "Test Event"
```

### 11.3 Fixture additions in `conftest.py`

```python
@pytest.fixture
async def mcp_client(backend_server, tmp_path):
    config = write_test_mcp_config(tmp_path)
    async with mcp.client.stdio.stdio_client(["sharly-chess-mcp", "--config", str(config)]) as client:
        yield client
```

---

## 12. Claude Desktop Integration

Once installed (`pip install sharly-chess`), users add this to their Claude Desktop `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "sharly-chess": {
      "command": "sharly-chess-mcp",
      "args": ["--config", "/home/user/.sharly-chess/mcp_config.toml"],
      "env": {
        "SHARLY_MCP_TOKEN": "YOUR_TOKEN_HERE"
      }
    }
  }
}
```

> **Security note:** Tokens in `env` are visible to any process that can read the Claude Desktop config. For higher-security deployments use the SSE transport with mTLS instead.

---

## Appendix A — Files to Create

| File | Purpose |
|---|---|
| `src/mcp/__init__.py` | Package marker |
| `src/mcp/server.py` | Entry point, tool registration, startup |
| `src/mcp/config.py` | Config loading and validation |
| `src/mcp/auth.py` | Token → Client resolution, `require_action()` |
| `src/mcp/audit.py` | Append-only JSONL audit log |
| `src/mcp/errors.py` | Exception hierarchy |
| `src/mcp/tools/events.py` | `list_events`, `get_event` |
| `src/mcp/tools/tournaments.py` | `list_tournaments`, `get_tournament`, `get_pairings`, `get_standings` |
| `src/mcp/tools/players.py` | `list_players`, `get_player` |
| `src/mcp/tools/results.py` | `set_result` (write, opt-in) |
| `tests/unit/mcp/` | Unit test suite |
| `tests/e2e/mcp/` | Integration test suite |
| `mcp_config.toml.example` | Template config (committed) |
| `.gitignore` addition | `mcp_config.toml` (never commit real tokens) |

## Appendix B — Files to Modify

| File | Change |
|---|---|
| `pyproject.toml` | Add `mcp` dependency, add `sharly-chess-mcp` script entry point |
| `src/data/access_levels/client.py` | Add `Client.build_service_account()` factory method |
| `.gitignore` | Add `mcp_config.toml` |
| `conftest.py` | Add `mcp_client` fixture |

---

*End of document.*
