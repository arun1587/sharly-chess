# MCP Server Feasibility Study for SharlyChess

## Executive Summary

This document evaluates the feasibility of creating an MCP (Model Context Protocol) server for SharlyChess to expose its functionality as tools that can be orchestrated by AI agents in natural language.

**Conclusion: Highly Feasible.** SharlyChess's existing architecture (clean separation of database, domain logic, and controllers) makes it well-suited for MCP server integration. Approximately 85-90% of arbiter-facing functionality can be meaningfully exposed as MCP tools.

---

## 1. Can Existing Functionalities Be Exposed as MCP Tools?

### Answer: Yes — Architecture is Favorable

SharlyChess already has a clean separation between its web controllers (HTTP/HTMX layer) and its business logic/database layer. The MCP server would wrap the same domain objects and database operations that the web controllers use today.

### Workflow-to-Tool Mapping

| Workflow | Candidate Tools | Feasibility |
|---|---|---|
| **Event management** | `create_event`, `load_event`, `list_events`, `update_event` | Straightforward — wraps `EventDatabase` + `Event` |
| **Tournament management** | `create_tournament`, `update_tournament`, `list_tournaments` | Straightforward — wraps `TournamentAdminController` logic |
| **Player registration** | `add_player`, `search_player`, `register_player_in_tournament`, `check_in_player` | Straightforward — wraps player DB + data sources |
| **Pairings** | `generate_pairings`, `get_pairings_for_round`, `add_manual_pairing`, `assign_bye` | Wraps BBP engine + pairing logic |
| **Results** | `enter_result`, `get_board`, `list_boards_for_round` | Wraps board/result logic |
| **Rankings** | `get_standings`, `get_tiebreak_details` | Wraps ranking computation |
| **Screens/Display** | `create_screen`, `update_screen`, `list_screens` | Wraps screen controller |
| **Timers** | `start_timer`, `pause_timer`, `get_timer_status` | Wraps timer logic |
| **Prizes** | `configure_prizes`, `get_prize_distribution` | Wraps prize module |
| **Import/Export** | `import_trf`, `export_trf`, `import_from_chessevent` | Wraps I/O layer |
| **Documents** | `generate_pairing_sheet`, `generate_ranking_document` | Wraps print_documents |
| **Accounts** | `list_accounts`, `create_account`, `assign_role` | Wraps account/permission layer |

**Estimated coverage: ~85-90% of arbiter-facing functionality** can be meaningfully exposed.

---

## 2. What Will Be Missing or Challenging?

### Hard to Expose

#### Real-time Screen Display
- Screens use WebSocket push + HTML rendering. An MCP tool can *configure* screens, but the visual output is inherently a browser experience.
- The agent could trigger updates but cannot replace the live display component.
- **Mitigation:** Tools provide configuration and status; the web UI remains the canonical display channel.

#### Print/PDF Documents
- Generating a PDF is possible, but delivering it to the user through MCP is awkward (binary data).
- You'd need to write the file to disk and return a filesystem path.
- **Mitigation:** Tools return file paths; the user can open locally or integrate with system file manager.

#### File-based Event Loading
- Events are `.sce` SQLite files. The MCP server needs access to the same filesystem.
- This works naturally in stdio mode (same machine), but limits remote usage.
- **Mitigation:** Acceptable for desktop use; requires clarification in documentation.

#### HTMX Interactive Flows
- Some admin workflows have multi-step UI interactions (modals, confirmations, drag-and-drop pairing adjustments).
- These would need to be decomposed into discrete tool calls.
- **Mitigation:** Design tools as individual, atomic operations; the agent orchestrates the sequence.

#### Plugin-specific GUIs
- FFE database browser, ChessEvent import wizard, ChessResults upload status — these have rich UI.
- Would need to be simplified to tool-based interactions.
- **Mitigation:** Plugins that require complex UX can opt out; basic I/O (import/export) remains available.

### Missing Capabilities an Agent Would Need

#### Validation Feedback
- The web UI shows inline validation errors. MCP tools need to return clear error messages so the LLM can self-correct.
- **Solution:** Implement structured error responses with `isError: true` and detailed messages.

#### State Awareness
- An agent needs context about the current event state (which round is active, which tournaments exist, etc.).
- This suggests adding **MCP Resources** alongside tools (e.g., `event://current/status`, `tournament://{id}/state`).
- **Solution:** Define read-only resources for querying state.

#### Confirmation for Destructive Actions
- Pairing generation, result publication, and round closure are difficult to undo.
- The MCP server should flag these with `annotations.destructiveHint: true` so the host prompts the user.
- **Solution:** Mark tools appropriately; rely on host to confirm before execution.

---

## 3. Ideal Architecture & Deployment

### Recommended Architecture

```
┌──────────────────────────────────────────────────┐
│  MCP Host (Claude Desktop / Claude Code / IDE)   │
│                                                   │
│  User: "Pair round 3 of the Open tournament"     │
│         ↓                                         │
│  LLM decides → calls tool: generate_pairings     │
│         ↓                                         │
│  MCP Client ──────stdio──→ MCP Server process    │
└────────────────────────┬──────────────────────────┘
                         │ stdin/stdout (JSON-RPC)
                         ↓
┌──────────────────────────────────────────────────┐
│       sharly-chess-mcp-server (new module)       │
│                                                   │
│  FastMCP("sharly-chess")                          │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ Event    │  │Tournament│  │ Pairing  │ ...    │
│  │ Tools    │  │ Tools    │  │ Tools    │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│       └──────────────┼──────────────┘             │
│                      ↓                            │
│         Existing SharlyChess Core                 │
│    (Event, Tournament, Player, Board, ...)        │
│                      ↓                            │
│           EventDatabase (SQLite)                  │
│            ↓ reads/writes ↓                       │
│           .sce event files                        │
└──────────────────────────────────────────────────┘
```

### Key Design Decisions

#### 1. Transport: **stdio**
SharlyChess is a local desktop app. stdio is the natural fit:
- **No network surface** to secure
- **No port conflicts** with the existing Litestar web server
- **Clean process lifecycle** (host starts/stops the server)
- **Zero latency overhead**

#### 2. Thin Wrapper Pattern
The MCP server imports SharlyChess's existing Python modules directly (same virtualenv). Each `@mcp.tool()` function is a thin adapter that:
- Accepts validated parameters (auto-generated JSON Schema from type hints)
- Calls into existing domain logic (`Event`, `Tournament`, `EventDatabase`, etc.)
- Returns structured text results

**Example:**
```python
from mcp.server.fastmcp import FastMCP
from src.database.sqlite.event.event_database import EventDatabase
from src.data.tournament import Tournament

mcp = FastMCP("sharly-chess")

@mcp.tool()
async def tournament_list(event_path: str) -> str:
    """List all tournaments in an event.

    Args:
        event_path: Path to the event .sce file
    """
    db = EventDatabase(event_path)
    await db.connect()
    tournaments = await db.get_tournaments()
    await db.close()

    return json.dumps([
        {"id": t.uniq_id, "name": t.name, "round_count": t.round_count}
        for t in tournaments
    ])
```

#### 3. Lifespan for DB Connections
Use FastMCP's lifespan context manager to open/close the event database:

```python
from contextlib import asynccontextmanager
from dataclasses import dataclass

@dataclass
class AppContext:
    db: EventDatabase

@asynccontextmanager
async def app_lifespan(server: FastMCP):
    db = EventDatabase(event_path)
    await db.connect()
    try:
        yield AppContext(db=db)
    finally:
        await db.close()

mcp = FastMCP("sharly-chess", lifespan=app_lifespan)
```

#### 4. MCP Resources for State
Expose read-only state as resources so the agent can orient itself without making unnecessary calls:
- `event://current` — current event summary
- `tournament://{id}/status` — round state, player count, pairing status
- `tournament://{id}/standings` — current rankings
- `round://{id}/boards` — all boards in a round

```python
@mcp.resource("tournament://{tournament_id}/status")
async def tournament_status(tournament_id: str) -> str:
    """Get the status of a tournament."""
    # ... fetch and return status
```

#### 5. Tool Grouping
Organize tools by domain with clear, consistent naming:

- **Event:** `event_create`, `event_load`, `event_list`, `event_update`
- **Tournament:** `tournament_create`, `tournament_list`, `tournament_update`, `tournament_import`
- **Players:** `player_add`, `player_search`, `player_register`, `player_check_in`
- **Rounds:** `round_pair`, `round_enter_result`, `round_get_boards`, `round_close`
- **Standings:** `standings_get`, `standings_export`, `tiebreak_get_details`
- **Screens:** `screen_create`, `screen_update`, `screen_list`
- **Timers:** `timer_start`, `timer_pause`, `timer_status`
- **Prizes:** `prize_configure`, `prize_get_distribution`
- **Documents:** `document_generate_pairings`, `document_generate_ranking`
- **Import/Export:** `tournament_import_trf`, `tournament_export_trf`
- **Accounts:** `account_list`, `account_create`, `role_assign`

---

### Deployment Options

#### Option A: stdio via Claude Desktop (Standalone)
**How it works:** User adds the server to `~/.config/Claude/claude_desktop_config.json` pointing to the MCP server Python script. Claude Desktop spawns it as a subprocess.

**Config example:**
```json
{
  "mcpServers": {
    "sharly-chess": {
      "command": "uv",
      "args": ["--directory", "/path/to/sharly-chess", "run", "src/mcp_server.py"],
      "env": {
        "EVENT_PATH": "/path/to/event.sce"
      }
    }
  }
}
```

**Best for:** Arbiters using Claude Desktop (separate from SharlyChess UI)

**Pros:**
- Decoupled from SharlyChess; can use different Python environments
- User installs separately if they want it

**Cons:**
- Requires manual setup
- ENV variable management is awkward

---

#### Option B: stdio via Claude Code (Terminal-based)
**How it works:** Same as Option A but configured in a `.mcp.json` file or via `/remember` commands in Claude Code.

**Best for:** Developer/power-user arbiters who want CLI access to SharlyChess automation

**Pros:**
- Natural fit with development workflows
- Can be scripted and versioned in the repo

**Cons:**
- Requires Claude Code knowledge
- Not suitable for end-user arbiters

---

#### Option C: Bundled with SharlyChess (Recommended)
**How it works:** The MCP server ships as part of the SharlyChess distribution (new entry point). SharlyChess's settings page lets you "Enable MCP Server" and shows the config snippet to paste into Claude Desktop.

**Directory structure:**
```
src/
├── mcp_server.py (new entry point)
├── mcp/ (new module)
│   ├── __init__.py
│   ├── server.py (FastMCP server definition)
│   ├── tools/ (organized by domain)
│   │   ├── event_tools.py
│   │   ├── tournament_tools.py
│   │   ├── player_tools.py
│   │   ├── round_tools.py
│   │   ├── standings_tools.py
│   │   └── ...
│   └── resources.py (read-only state)
└── ...
```

**Entry point (`src/mcp_server.py`):**
```python
#!/usr/bin/env python3
"""Run the SharlyChess MCP server."""
import asyncio
import sys
from pathlib import Path

from src.mcp.server import create_mcp_server

async def main():
    # Get event path from CLI arg or ENV
    event_path = sys.argv[1] if len(sys.argv) > 1 else None

    if not event_path:
        print("Usage: python -m src.mcp_server <event_path>", file=sys.stderr)
        sys.exit(1)

    server = create_mcp_server(event_path)
    await server.run(transport="stdio")

if __name__ == "__main__":
    asyncio.run(main())
```

**Settings UI integration:** SharlyChess can add a UI panel (Admin → MCP Integration) that shows:
- "Enable MCP Server" checkbox
- Path to MCP server entry point
- Auto-generated config snippet for Claude Desktop
- "Copy to Clipboard" button

**Best for:** End-user arbiters who want seamless integration with SharlyChess

**Pros:**
- Cleanest UX — no separate install
- Shared virtualenv, same Python environment
- SharlyChess can manage lifecycle (enable/disable in settings)
- Natural place to document

**Cons:**
- Adds another subsystem to the SharlyChess codebase
- Requires updating PyInstaller bundling

---

#### Option D: Streamable HTTP (Future Cloud Use)
**How it works:** If SharlyChess ever goes multi-machine (e.g., arbiter tablet + server PC), the MCP server could run as an HTTP endpoint alongside Litestar.

**When:** Only if/when SharlyChess supports remote arbiters

**Challenges:**
- Requires authentication
- Origin validation needed
- More complex deployment
- DNS rebinding protection

**Not recommended for current local-only use.**

---

### Recommendation: **Option C (Bundled)**

**Rationale:**
1. SharlyChess is already packaged as a PyInstaller desktop app — bundling is natural
2. Arbiters expect everything to work "out of the box"
3. MCP server is just another entry point into the same codebase
4. Settings UI can provide one-click Claude Desktop integration
5. Future enhancement: when multi-device support arrives, can upgrade to HTTP transport

**Implementation timeline:**
- Phase 1: Create MCP server module with tools (1-2 weeks)
- Phase 2: Add settings UI integration (1 week)
- Phase 3: Update PyInstaller config + documentation (1 week)

---

## 4. Technical Requirements

### Python Dependencies
Add to `pyproject.toml`:
```toml
[project.optional-dependencies]
mcp = ["mcp[cli]>=1.2.0"]
```

Users who want MCP support install with: `pip install sharly-chess[mcp]`

### Code Organization
```
src/
├── mcp_server.py           # CLI entry point
├── mcp/
│   ├── __init__.py
│   ├── server.py           # FastMCP initialization
│   ├── context.py          # AppContext dataclass
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── event.py        # Event tools
│   │   ├── tournament.py    # Tournament tools
│   │   ├── player.py        # Player tools
│   │   ├── round.py         # Round/pairing tools
│   │   ├── standings.py     # Ranking tools
│   │   ├── results.py       # Result entry tools
│   │   ├── screen.py        # Display tools
│   │   ├── prize.py         # Prize tools
│   │   ├── document.py      # Print document tools
│   │   └── import_export.py # I/O tools
│   ├── resources.py         # Read-only state resources
│   └── errors.py            # Custom error types
└── ...
```

### Logging
Use `logging` module exclusively (not `print()`):
```python
import logging
logger = logging.getLogger("sharly_chess.mcp")

@mcp.tool()
async def some_tool():
    logger.info("Tool invoked")
    # Never use: print("something") -- corrupts JSON-RPC stream
```

---

## 5. Security Considerations

### stdio Transport Safety
- **Good:** No network exposure, no authentication needed (local machine only)
- **Ensure:** Never use `print()` except to stderr (for logging)
- **Validate:** All tool inputs at the MCP layer (schemas auto-validate, but check business logic)

### Data Access
- **Current:** MCP server has same access to databases as web server
- **Future:** If remote usage added, implement per-user access control
- **Sensitive data:** Be mindful of returning player ratings, federation IDs, etc. — the agent output may be logged

### File Access
- **Event files:** MCP server needs read/write access to `.sce` files on the local filesystem
- **Paths:** Validate that tools don't allow directory traversal attacks
- **Example:** Whitelist allowed event paths; don't allow arbitrary filesystem access

---

## 6. Success Criteria

| Criterion | Metric |
|---|---|
| **Tool coverage** | ≥80% of documented arbiter workflows have corresponding tools |
| **Error handling** | All business logic errors return structured `isError: true` responses |
| **State discovery** | Agent can query current event/tournament state via resources |
| **User experience** | Arbiter can complete multi-step workflows (e.g., create tournament → pair round → enter results) without returning to web UI |
| **Documentation** | MCP server setup, tool descriptions, and example workflows documented for end users |
| **Testing** | MCP server can be tested with MCP Inspector; unit tests cover all tools |

---

## 7. Next Steps

1. **Prototype** (1 week):
   - Create basic MCP server with 5-10 key tools (event/tournament CRUD)
   - Test with MCP Inspector and Claude

2. **Expand tools** (2 weeks):
   - Add all tools from the workflow mapping above
   - Implement resources for state discovery
   - Handle error cases gracefully

3. **Integration** (1 week):
   - Add settings UI for MCP enablement
   - Update PyInstaller bundling
   - Document for users

4. **User testing** (ongoing):
   - Have arbiter beta-test the agent workflow
   - Refine tool descriptions based on feedback

---

## Conclusion

**Creating an MCP server for SharlyChess is highly feasible and would be valuable for power-user arbiters and AI-assisted workflows.**

The existing architecture is well-suited to MCP integration. The stdio transport is ideal for a local desktop application. By wrapping the existing domain logic and database layer, the MCP server would provide natural orchestration of tournament workflows while keeping the web UI as the primary display layer.

**Recommendation: Proceed with Option C (bundled) implementation.**
