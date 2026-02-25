# MCP Server — Phase 1: Read-Only Tools

## Context

Sharly Chess needs a way to expose tournament management data to external AI clients (Claude Desktop).
Phase 1 builds the `ai/` package with a shared tool layer and an SSE MCP server — read-only tools only.
All tools are pure Python functions operating directly on the existing data layer.

---

## Directory Structure to Create

```
src/ai/
├── __init__.py
├── tools/
│   ├── __init__.py
│   ├── events.py        # list_events, get_event
│   ├── tournaments.py   # list_tournaments, get_tournament, get_standings
│   ├── players.py       # list_players
│   └── pairings.py      # get_pairings
└── mcp/
    ├── __init__.py
    ├── __main__.py      # entry point: python -m ai.mcp
    └── server.py        # FastMCP app with SSE
```

---

## Tool Functions

### `src/ai/tools/events.py`

Uses:
- `EventLoader.get_events_metadata(status)` → `list[EventMetadata]` from `src/data/loader.py`
- `EventLoader().load_event(uniq_id)` → `Event` from `src/data/loader.py`
- `BaseStoredEvent` fields: `uniq_id`, `name`, `start_date`, `stop_date`, `location`, `federation`, `public`
- `EventMetadata` extra fields: `tournament_count`, `player_count`

```python
def list_events(status: str | None = None) -> list[dict]:
    """List events. status: 'current', 'coming', 'passed', or None for all."""

def get_event(event_id: str) -> dict:
    """Get full details for a single event including tournament list."""
```

### `src/ai/tools/tournaments.py`

Uses:
- `EventLoader().load_event(event_id).sorted_tournaments` → `list[Tournament]`
- `Tournament`: `id`, `name`, `rounds`, `current_round`, `started`, `finished`, `location`, `start_date`, `stop_date`, `has_schedule`
- `tournament.compute_tournament_player_ranks(after_round=tournament.current_round)` → `dict[int, TournamentPlayer]`
- `TournamentPlayer`: `full_name`, `points`, `rank`, `pairing_number`, `rating`

```python
def list_tournaments(event_id: str) -> list[dict]:
    """List all tournaments in an event."""

def get_tournament(event_id: str, tournament_id: int) -> dict:
    """Get details for a specific tournament."""

def get_standings(event_id: str, tournament_id: int, after_round: int | None = None) -> list[dict]:
    """Get current standings. after_round defaults to current round."""
```

### `src/ai/tools/players.py`

Uses:
- `EventLoader().load_event(event_id).sorted_players` → `list[Player]`
- `Player`: `id`, `full_name`, `last_name`, `first_name`, `fide_id`, `federation`, `title`, `rating`
- For tournament-scoped: `tournament.tournament_players`
- `TournamentPlayer`: above + `pairing_number`, `check_in_status`, `points`

```python
def list_players(event_id: str, tournament_id: int | None = None) -> list[dict]:
    """List players. If tournament_id given, scoped to that tournament with pairing info."""
```

### `src/ai/tools/pairings.py`

Uses:
- `tournament.get_round_boards(round_)` → `list[Board]`
- `Board`: `number`, `result`, `white_tournament_player.full_name`, `black_tournament_player` (nullable for bye)
- `Result` enum: use `.name` for string representation

```python
def get_pairings(event_id: str, tournament_id: int, round_: int) -> list[dict]:
    """Get board pairings for a specific round."""
```

---

## MCP Server

### `src/ai/mcp/server.py`

```python
from mcp.server.fastmcp import FastMCP
from ai.tools import events, tournaments, players, pairings

mcp = FastMCP('sharly-chess')

mcp.tool()(events.list_events)
mcp.tool()(events.get_event)
mcp.tool()(tournaments.list_tournaments)
mcp.tool()(tournaments.get_tournament)
mcp.tool()(tournaments.get_standings)
mcp.tool()(players.list_players)
mcp.tool()(pairings.get_pairings)
```

### `src/ai/mcp/__main__.py`

```python
def main():
    from utils.scripts import init_script
    init_script()  # sets cwd to workdir — must be before other project imports

    from ai.mcp.server import mcp
    mcp.run(transport='sse', host='127.0.0.1', port=8765)

if __name__ == '__main__':
    main()
```

---

## pyproject.toml

Add optional dependency group:

```toml
ai = [
    "mcp[cli] >= 1.0.0",
]
```

---

## Error Handling Pattern

- Each tool returns a `dict` or `list[dict]` on success.
- On failure: `{"error": "message"}`.
- Wrap `EventLoader().load_event()` in try/except for missing events.
- Use `event.tournaments_by_id.get(tournament_id)` and return error dict if `None`.

---

## Critical Files (read-only references)

| File | Purpose |
|------|---------|
| `src/data/loader.py` | `EventLoader` — load events and metadata |
| `src/data/event.py` | `Event.sorted_tournaments`, `Event.sorted_players`, `Event.tournaments_by_id` |
| `src/data/event_metadata.py` | `EventMetadata` fields |
| `src/data/tournament.py` | `Tournament`, `compute_tournament_player_ranks()`, `get_round_boards()` |
| `src/data/player.py` | `Player`, `TournamentPlayer` |
| `src/data/board.py` | `Board.result`, `Board.white/black_tournament_player` |
| `src/utils/scripts.py` | `init_script()` — cwd setup before imports |
| `pyproject.toml` | Add `ai` optional dep group |

---

## Verification

1. Install MCP dep: `pip install -e ".[ai]"`
2. Run the server from the `src/` directory:
   ```bash
   python -m ai.mcp --path /path/to/events/dir
   ```
3. Configure Claude Desktop (`~/.config/Claude/claude_desktop_config.json`):
   ```json
   {
     "mcpServers": {
       "sharly-chess": {
         "url": "http://127.0.0.1:8765/sse"
       }
     }
   }
   ```
4. Test each tool in Claude Desktop:
   - *"List all events"* → `list_events`
   - *"Show tournaments for event X"* → `list_tournaments`
   - *"Show standings for tournament Y"* → `get_standings`
   - *"List players in tournament Y"* → `list_players`
   - *"Show pairings for round 3 of tournament Y"* → `get_pairings`
