# Sharly Chess AI Assistant — Architecture & Design

## 1. Vision

Enable chess arbiters and organisers to manage their tournaments through
**natural language** — either via an **embedded chatbot inside the Sharly Chess
web UI**, or via external AI clients like Claude Desktop.

The user types something like *"Generate pairings for round 3 of Open A"* and
the system figures out which operation to call, executes it, and reports back.

---

## 2. Two Access Paths, One Tool Layer

```
                    ┌──────────────────────────────────┐
                    │        Tool Functions             │
                    │   tools/events.py                 │
                    │   tools/tournaments.py            │
                    │   tools/players.py                │
                    │   tools/pairings.py               │
                    │   tools/results.py                │
                    │   ...                             │
                    │                                   │
                    │   Pure Python. No transport deps.  │
                    │   Operate on data/ + database/     │
                    └──────────┬───────────┬────────────┘
                               │           │
              ┌────────────────┘           └────────────────┐
              │                                             │
              ▼                                             ▼
┌─────────────────────────┐              ┌──────────────────────────────┐
│  Path A: MCP Server     │              │  Path B: Embedded Chatbot    │
│  (for external clients) │              │  (inside the web UI)         │
│                         │              │                              │
│  SSE transport          │              │  Chat UI in Litestar/HTMX    │
│  Claude Desktop,        │              │  ↕                           │
│  other MCP clients      │              │  Agent Loop                  │
│                         │              │  ↕                           │
│  mcp SDK registers      │              │  External LLM API            │
│  tool functions as       │              │  (OpenAI / Anthropic / …)    │
│  MCP tools              │              │  returns tool_calls →        │
│                         │              │  Agent executes them →       │
│                         │              │  sends result back to LLM    │
└─────────────────────────┘              └──────────────────────────────┘
```

**Key principle**: The tool functions in `tools/*.py` are the single source of
truth.  They know nothing about MCP, LLMs, or HTTP — they just take parameters
and operate on the Sharly Chess data layer.

---

## 3. Packaging — Directory Structure

```
src/
├── common/
├── data/                          # domain models
├── database/                      # SQLite persistence
├── plugins/                       # chess-results, FFE, …
├── web/                           # Litestar web UI (existing)
│   └── controllers/
│       └── admin/
│           └── chat_controller.py # NEW — chat UI endpoints
│
├── ai/                            # ◀ NEW — AI assistant package
│   ├── __init__.py
│   ├── tools/                     # ◀ shared tool functions (the kernel)
│   │   ├── __init__.py
│   │   ├── events.py              # list_events, get_event, create_event, …
│   │   ├── tournaments.py         # list_tournaments, get_standings, …
│   │   ├── players.py             # list_players, add_player, check_in, …
│   │   ├── pairings.py            # get_pairings, generate_pairings, …
│   │   ├── results.py             # set_result, get_standings
│   │   ├── screens.py             # list_screens, create_screen, …
│   │   ├── timers.py              # list_timers, import_timer_hours, …
│   │   └── export.py              # TRF, Chess-Results exports
│   │
│   ├── tool_registry.py           # discovers & lists all tools with schemas
│   │
│   ├── mcp/                       # ◀ Path A: MCP Server wrapper
│   │   ├── __init__.py
│   │   ├── __main__.py            # python -m ai.mcp
│   │   └── server.py              # FastMCP app, registers tools from ai.tools
│   │
│   ├── agent/                     # ◀ Path B: Embedded chatbot agent
│   │   ├── __init__.py
│   │   ├── agent.py               # agent loop: LLM ↔ tool execution
│   │   ├── llm_providers/         # pluggable LLM backends
│   │   │   ├── __init__.py
│   │   │   ├── base.py            # abstract LLMProvider interface
│   │   │   ├── openai.py          # OpenAI / Azure OpenAI
│   │   │   ├── anthropic.py       # Anthropic Claude
│   │   │   └── ollama.py          # local models (future)
│   │   └── history.py             # conversation history storage
│   │
│   └── auth.py                    # shared auth (token validation, account mapping)
│
└── utils/
```

### Why `ai/` not `mcp_server/`?

The package is larger than just MCP now — it contains the shared tools, the
agent, the LLM providers, AND the MCP server.  Naming it `ai/` reflects its
purpose: the AI assistant layer of the app.

---

## 4. The Tool Layer — `ai/tools/`

### 4.1 Design Principle

Each tool function is:
- **Pure Python** — takes simple parameters, returns simple dicts/lists
- **Self-contained** — handles its own database access
- **Transport-agnostic** — no knowledge of MCP, HTTP, or LLM APIs
- **Auth-aware** — accepts an optional `account` parameter for permission checks

```python
# ai/tools/results.py

from data.loader import EventLoader
from database.sqlite.event.event_database import EventDatabase
from utils.enum import Result

RESULT_MAP = {
    "white_wins": Result.WIN,
    "black_wins": Result.LOSS,
    "draw": Result.DRAW,
    "white_forfeits": Result.FORFEIT_LOSS,
    "black_forfeits": Result.FORFEIT_WIN,
    ...
}

def set_result(
    event_id: str,
    tournament_id: int,
    round: int,
    board_id: int,
    result: str,  # "white_wins", "black_wins", "draw", ...
) -> dict:
    """Set the result for a board in a specific round.

    Args:
        event_id: The unique event identifier (e.g. "my-event")
        tournament_id: The tournament ID within the event
        round: The round number
        board_id: The board number
        result: One of: white_wins, black_wins, draw, white_forfeits,
                black_forfeits, double_forfeit

    Returns:
        A dict with status and description of what was set.
    """
    if result not in RESULT_MAP:
        return {"error": f"Unknown result '{result}'. Valid: {list(RESULT_MAP)}"}

    event = EventLoader().load_event(event_id)
    tournament = event.tournaments_by_id.get(tournament_id)
    if not tournament:
        return {"error": f"Tournament {tournament_id} not found"}

    result_enum = RESULT_MAP[result]

    with EventDatabase(event_id, write=True) as db:
        # ... update pairing result ...
        pass

    return {
        "status": "ok",
        "message": f"Board {board_id}, Round {round}: {result} recorded.",
    }
```

### 4.2 Tool Registry

A central registry discovers all tools and generates JSON schemas for both MCP
registration and LLM function-calling:

```python
# ai/tool_registry.py
import inspect
from typing import get_type_hints
from ai.tools import events, tournaments, players, pairings, results

class ToolRegistry:
    """Registry of all available AI tools."""

    def __init__(self):
        self._tools: dict[str, callable] = {}
        self._register_module(events)
        self._register_module(tournaments)
        self._register_module(players)
        self._register_module(pairings)
        self._register_module(results)

    def _register_module(self, module):
        for name, func in inspect.getmembers(module, inspect.isfunction):
            if not name.startswith('_'):
                self._tools[name] = func

    @property
    def tools(self) -> dict[str, callable]:
        return self._tools

    def get_openai_function_schemas(self) -> list[dict]:
        """Generate OpenAI-compatible function schemas from type hints + docstrings."""
        schemas = []
        for name, func in self._tools.items():
            schemas.append(self._func_to_schema(name, func))
        return schemas

    def get_mcp_tool_definitions(self) -> list[dict]:
        """Generate MCP tool definitions."""
        ...
```

---

## 5. Path A: MCP Server — `ai/mcp/`

Thin wrapper.  Imports tools from the registry and registers them with the
`mcp` SDK.

```python
# ai/mcp/server.py
from mcp.server.fastmcp import FastMCP
from ai.tool_registry import ToolRegistry

mcp = FastMCP("sharly-chess")
registry = ToolRegistry()

# Auto-register all tools
for name, func in registry.tools.items():
    mcp.tool()(func)

# Run with: python -m ai.mcp
```

```python
# ai/mcp/__main__.py
import os, sys, argparse
from utils.scripts import init_script

def main():
    init_script()  # sets cwd to workdir (where events/ lives)

    from ai.mcp.server import mcp
    mcp.run(transport="sse")

if __name__ == "__main__":
    main()
```

### Claude Desktop config:

```json
{
  "mcpServers": {
    "sharly-chess": {
      "url": "http://localhost:8765/sse"
    }
  }
}
```

---

## 6. Path B: Embedded Chatbot — `ai/agent/`

### 6.1 The Agent Loop

This is the core of the in-app chatbot.  It implements the standard
**tool-use agent loop**:

```
User message
    │
    ▼
┌─────────────────────────────────────────┐
│  Agent Loop                             │
│                                         │
│  1. Send message + tool schemas to LLM  │
│  2. LLM responds with:                  │
│     a. Text → return to user            │
│     b. tool_call → execute tool         │
│        → append result to messages      │
│        → go to step 1                   │
└─────────────────────────────────────────┘
```

```python
# ai/agent/agent.py
from ai.tool_registry import ToolRegistry
from ai.agent.llm_providers.base import LLMProvider

class ChatAgent:
    """Orchestrates the LLM ↔ tool execution loop."""

    def __init__(self, llm: LLMProvider, event_id: str | None = None):
        self.llm = llm
        self.registry = ToolRegistry()
        self.event_id = event_id  # optional: scope to a specific event
        self.messages: list[dict] = [
            {"role": "system", "content": self._system_prompt()}
        ]

    def _system_prompt(self) -> str:
        return (
            "You are a chess tournament assistant for Sharly Chess. "
            "You help arbiters manage events, tournaments, players, "
            "pairings, and results. Use the available tools to perform "
            "operations. Always confirm destructive actions before executing."
        )

    async def chat(self, user_message: str) -> str:
        """Process a user message, possibly calling tools, return the response."""
        self.messages.append({"role": "user", "content": user_message})

        while True:
            response = await self.llm.complete(
                messages=self.messages,
                tools=self.registry.get_openai_function_schemas(),
            )

            if response.tool_calls:
                # Execute each tool call
                for tool_call in response.tool_calls:
                    func = self.registry.tools[tool_call.function_name]
                    result = func(**tool_call.arguments)

                    self.messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": json.dumps(result),
                    })
                # Loop again — LLM will see the tool results and respond
                continue

            # No tool calls — LLM gave a text response
            self.messages.append({
                "role": "assistant",
                "content": response.text,
            })
            return response.text
```

### 6.2 Pluggable LLM Providers

```python
# ai/agent/llm_providers/base.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class ToolCall:
    id: str
    function_name: str
    arguments: dict

@dataclass
class LLMResponse:
    text: str | None
    tool_calls: list[ToolCall] | None

class LLMProvider(ABC):
    """Abstract base for LLM backends."""

    @abstractmethod
    async def complete(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
    ) -> LLMResponse:
        """Send messages to the LLM, get a response (possibly with tool calls)."""
        ...
```

```python
# ai/agent/llm_providers/openai.py
import openai
from .base import LLMProvider, LLMResponse, ToolCall

class OpenAIProvider(LLMProvider):
    def __init__(self, api_key: str, model: str = "gpt-4o"):
        self.client = openai.AsyncOpenAI(api_key=api_key)
        self.model = model

    async def complete(self, messages, tools=None) -> LLMResponse:
        kwargs = {"model": self.model, "messages": messages}
        if tools:
            kwargs["tools"] = [
                {"type": "function", "function": t} for t in tools
            ]

        response = await self.client.chat.completions.create(**kwargs)
        choice = response.choices[0]

        if choice.message.tool_calls:
            return LLMResponse(
                text=None,
                tool_calls=[
                    ToolCall(
                        id=tc.id,
                        function_name=tc.function.name,
                        arguments=json.loads(tc.function.arguments),
                    )
                    for tc in choice.message.tool_calls
                ],
            )
        return LLMResponse(text=choice.message.content, tool_calls=None)
```

```python
# ai/agent/llm_providers/anthropic.py
import anthropic
from .base import LLMProvider, LLMResponse, ToolCall

class AnthropicProvider(LLMProvider):
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514"):
        self.client = anthropic.AsyncAnthropic(api_key=api_key)
        self.model = model

    async def complete(self, messages, tools=None) -> LLMResponse:
        # Convert OpenAI-style tool schemas to Anthropic format
        # The registry could also provide Anthropic-native schemas
        ...
```

### 6.3 Configuration

LLM provider settings are stored in the app config (alongside existing
settings like locale, port, etc.):

```python
# In SharlyChessConfig or a new AI config section:
ai_provider: str = "openai"        # "openai", "anthropic", "ollama"
ai_api_key: str = ""                # the API key
ai_model: str = "gpt-4o"           # model to use
ai_enabled: bool = False            # feature flag
```

Configurable via the admin UI (a new section in Settings), or via `.env`:

```env
AI_PROVIDER=openai
AI_API_KEY=sk-...
AI_MODEL=gpt-4o
AI_ENABLED=true
```

---

## 7. Web UI Chat Interface — `chat_controller.py`

### 7.1 How It Integrates

The chat interface is a **panel/drawer** in the admin UI — not a separate page.
It's always available via a floating button, similar to a support chat widget.

```
┌──────────────────────────────────────────────────────┐
│  Sharly Chess Admin UI                               │
│                                                      │
│  ┌─ Tournaments tab ──────────────────────────────┐  │
│  │  Open A  │  Open B  │  Blitz  │                │  │
│  │  ...     │  ...     │  ...    │                │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│                              ┌────────────────────┐  │
│                              │ 💬 AI Assistant     │  │
│                              │                    │  │
│                              │ You: Show me the   │  │
│                              │ standings for Open A│  │
│                              │                    │  │
│                              │ AI: Here are the   │  │
│                              │ current standings:  │  │
│                              │ 1. Carlsen (5.5)   │  │
│                              │ 2. Caruana (5.0)   │  │
│                              │ ...                │  │
│                              │                    │  │
│                              │ [Type a message…]  │  │
│                              └────────────────────┘  │
│                                              [💬]    │
└──────────────────────────────────────────────────────┘
```

### 7.2 Controller

```python
# web/controllers/admin/chat_controller.py

class ChatAdminController(BaseEventAdminController):
    """Handles the embedded AI chat interface."""

    @post("/chat/{event_uniq_id:str}")
    async def send_message(self, request: HTMXRequest, message: str) -> Template:
        """Process a chat message via the AI agent."""
        web_context = AdminWebContext(request)
        event = web_context.get_admin_event()

        agent = get_or_create_agent(request, event)
        response = await agent.chat(message)

        return HTMXTemplate(
            template_name="admin/chat/message.html",
            context={"response": response},
        )
```

The chat uses **HTMX** (already used throughout the app) for seamless
message streaming without full page reloads.

---

## 8. Tool Catalog

### 8.1 Events

| Tool                | Parameters                          | Returns                        | Write? |
|---------------------|-------------------------------------|--------------------------------|--------|
| `list_events`       | `status?: passed\|current\|coming`  | List of event summaries        | No     |
| `get_event`         | `event_id: str`                     | Full event details             | No     |
| `create_event`      | `name, start_date, stop_date, ...`  | Created event ID               | Yes    |
| `update_event`      | `event_id, fields...`               | Confirmation                   | Yes    |
| `delete_event`      | `event_id`                          | Confirmation                   | Yes    |

### 8.2 Tournaments

| Tool                    | Parameters                              | Returns                    | Write? |
|-------------------------|-----------------------------------------|----------------------------|--------|
| `list_tournaments`      | `event_id`                              | List of tournament details | No     |
| `get_tournament`        | `event_id, tournament_id`               | Full tournament info       | No     |
| `get_standings`         | `event_id, tournament_id, round?`       | Ranked player list         | No     |
| `create_tournament`     | `event_id, name, rounds, rating, ...`   | Created tournament         | Yes    |
| `update_tournament`     | `event_id, tournament_id, fields...`    | Confirmation               | Yes    |
| `delete_tournament`     | `event_id, tournament_id`               | Confirmation               | Yes    |

### 8.3 Players

| Tool                    | Parameters                                   | Returns                | Write? |
|-------------------------|----------------------------------------------|------------------------|--------|
| `list_players`          | `event_id, tournament_id?, filter?`          | Player list            | No     |
| `search_player`         | `name?, fide_id?, national_id?`              | Player info from source| No     |
| `add_player`            | `event_id, tournament_id, fide_id or data..`  | Added player           | Yes    |
| `remove_player`         | `event_id, tournament_id, player_id`         | Confirmation           | Yes    |
| `check_in_player`       | `event_id, tournament_id, player_id`         | Confirmation           | Yes    |
| `check_out_player`      | `event_id, tournament_id, player_id`         | Confirmation           | Yes    |
| `withdraw_player`       | `event_id, tournament_id, player_id`         | Confirmation           | Yes    |

### 8.4 Pairings

| Tool                     | Parameters                                 | Returns              | Write? |
|--------------------------|--------------------------------------------|----------------------|--------|
| `get_pairings`           | `event_id, tournament_id, round`           | Board pairings       | No     |
| `generate_pairings`      | `event_id, tournament_id`                  | Generated pairings   | Yes    |
| `unpair_round`           | `event_id, tournament_id, round`           | Confirmation         | Yes    |

### 8.5 Results

| Tool                    | Parameters                                          | Returns       | Write? |
|-------------------------|------------------------------------------------------|---------------|--------|
| `set_result`            | `event_id, tournament_id, round, board_id, result`  | Confirmation  | Yes    |
| `set_result_by_player`  | `event_id, tournament_id, round, player_name, result`| Confirmation | Yes    |

### 8.6 Screens, Timers, Export

| Tool                    | Parameters                              | Returns            | Write? |
|-------------------------|-----------------------------------------|--------------------|--------|
| `list_screens`          | `event_id`                              | Screen list        | No     |
| `list_timers`           | `event_id`                              | Timer list         | No     |
| `import_timer_hours`    | `event_id, timer_id, tournament_id`     | Confirmation       | Yes    |
| `export_trf`            | `event_id, tournament_id`               | TRF file content   | No     |
| `upload_chess_results`  | `event_id, tournament_id`               | Upload status      | Yes    |

---

## 9. MCP Resources (Read-Only)

In addition to tools, the MCP server exposes **MCP Resources** for contextual
data that AI clients can read passively:

| Resource URI                              | Description                           |
|-------------------------------------------|---------------------------------------|
| `event://{id}/summary`                    | Event name, dates, tournament count   |
| `event://{id}/tournament/{tid}/standings` | Current standings as formatted text   |
| `event://{id}/tournament/{tid}/schedule`  | Round schedule with datetimes         |

---

## 10. Authentication & Safety

### 10.1 Authentication Layers

```
Layer 1: Transport security
    ├── MCP: Token in Authorization header
    └── Embedded chat: Inherits web session (already logged in)

Layer 2: Action permissions
    └── Map caller to a Sharly Chess Account
    └── Reuse existing AuthAction system (guards.py)

Layer 3: Confirmation for destructive actions
    └── Tools that write return a confirmation prompt
    └── The AI asks the user before executing
```

### 10.2 Mapping to Existing Auth

```python
# ai/auth.py
class AIAuth:
    """Authentication for AI tool access."""

    @staticmethod
    def from_mcp_token(token: str) -> Account:
        """Resolve an MCP bearer token to an account."""
        ...

    @staticmethod
    def from_web_session(request: HTMXRequest) -> Account:
        """Resolve the logged-in web session to an account."""
        # For the embedded chatbot — user is already authenticated
        return RequestUtils.get_client(request).account
```

For the embedded chatbot, the user is already logged in via the web UI — no
additional authentication needed.  The chatbot inherits the same permissions as
the logged-in user.

---

## 11. Data Flow Examples

### Example A: Embedded Chatbot — "Show standings for Open A"

```
1. User types in chat panel: "Show standings for Open A"

2. chat_controller.py receives the HTMX POST

3. ChatAgent.chat() is called:
   a. Sends [system_prompt, user_msg] + tool schemas to OpenAI API
   b. OpenAI responds: tool_call → get_standings(event_id, tournament_id)
   c. Agent executes get_standings() from ai/tools/tournaments.py
   d. Appends tool result to messages, calls OpenAI again
   e. OpenAI responds with formatted text

4. Controller returns the AI response as an HTMX partial

5. User sees the standings in the chat panel
```

### Example B: Claude Desktop via MCP — "Set board 5 to white wins"

```
1. User says in Claude Desktop: "Set board 5 to white wins"

2. Claude calls MCP tool: set_result(…)

3. MCP Server (ai/mcp/server.py) delegates to ai/tools/results.py

4. Tool executes, returns confirmation

5. Claude shows: "✅ Board 5, Round 2: White wins recorded."
```

---

## 12. Concurrency & Database Safety

Sharly Chess uses **SQLite with file-level locking**.  Both paths (MCP and
embedded chatbot) must respect the same patterns:

- **Read operations**: Use `EventDatabase(uniq_id)` (no `write=True`)
- **Write operations**: Use `EventDatabase(uniq_id, write=True)` in a `with`
  block — this acquires the write lock
- **No long-lived connections**: Open/close per tool call
- **Tournament dirty tracking**: Pass `check_dirty_tournaments=True` (default)
  so the web UI knows to refresh

---

## 13. Implementation Phases

### Phase 1: Tool Layer + MCP Server (Read-Only)

**Goal**: External clients (Claude Desktop) can query tournament state.

- [ ] Create `ai/` package structure
- [ ] `ai/tools/events.py` — `list_events`, `get_event`
- [ ] `ai/tools/tournaments.py` — `list_tournaments`, `get_tournament`,
  `get_standings`
- [ ] `ai/tools/players.py` — `list_players`
- [ ] `ai/tools/pairings.py` — `get_pairings`
- [ ] `ai/tool_registry.py` — auto-discovery + schema generation
- [ ] `ai/mcp/server.py` — FastMCP wrapper with SSE
- [ ] `ai/mcp/__main__.py` — standalone entry point
- [ ] Add `mcp` SDK to `pyproject.toml` optional dependencies
- [ ] Basic tests for tool functions

### Phase 2: Write Operations + Auth

**Goal**: Full CRUD via MCP, with permission checks.

- [ ] `ai/auth.py` — token validation + account mapping
- [ ] `ai/tools/results.py` — `set_result`
- [ ] `ai/tools/pairings.py` — `generate_pairings`, `unpair_round`
- [ ] `ai/tools/players.py` — `add_player`, `check_in_player`,
  `withdraw_player`
- [ ] Permission checks using existing `AuthAction`
- [ ] Safety confirmations for destructive operations

### Phase 3: Embedded Chatbot

**Goal**: In-app chat panel using an external LLM.

- [ ] `ai/agent/agent.py` — agent loop (LLM ↔ tool execution)
- [ ] `ai/agent/llm_providers/base.py` — abstract provider interface
- [ ] `ai/agent/llm_providers/openai.py` — OpenAI provider
- [ ] `ai/agent/llm_providers/anthropic.py` — Anthropic provider
- [ ] `web/controllers/admin/chat_controller.py` — chat endpoints
- [ ] Chat UI templates (HTMX chat panel)
- [ ] AI config section in admin settings (API key, model, enable/disable)
- [ ] Conversation history storage

### Phase 4: Polish & Extend

**Goal**: Full feature parity, more providers, production readiness.

- [ ] `ai/tools/screens.py` — screen management
- [ ] `ai/tools/timers.py` — timer management
- [ ] `ai/tools/export.py` — TRF, Chess-Results exports
- [ ] `ai/agent/llm_providers/ollama.py` — local model support
- [ ] Streaming responses in chat UI
- [ ] i18n: multilingual system prompts
- [ ] Rate limiting / cost controls
- [ ] MCP resources (event summaries, standings)

### Phase 5: Embedded MCP (Optional)

**Goal**: MCP server runs inside the main Sharly Chess process.

- [ ] Mount SSE endpoint on the existing Litestar server (`/mcp/sse`)
- [ ] Share the event loop with `ServerEngine`
- [ ] Single process deployment

---

## 14. Dependencies

```toml
[project.optional-dependencies]
ai = [
    "mcp[cli] >= 1.0.0",         # Anthropic MCP SDK with SSE transport
    "openai >= 1.0.0",            # OpenAI API client
    "anthropic >= 0.30.0",        # Anthropic API client (optional)
]
```

Making it optional keeps the main app lightweight — users who don't use the AI
features don't need to install LLM client libraries.

---

## 15. Testing Strategy

```
tests/
├── test_ai/
│   ├── test_tools/
│   │   ├── test_events.py         # unit tests for tool functions
│   │   ├── test_tournaments.py
│   │   ├── test_pairings.py
│   │   └── test_results.py
│   ├── test_tool_registry.py      # schema generation tests
│   ├── test_agent.py              # agent loop with mock LLM
│   └── test_mcp_integration.py    # end-to-end SSE with mcp SDK client
```

Tool functions are **pure Python** (take parameters, return dicts) — easy to
unit test with a test database.  The agent loop can be tested with a mock LLM
provider.  Integration tests verify the full MCP SSE round-trip.
