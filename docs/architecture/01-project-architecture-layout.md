# Sharly Chess - Project Architecture & Layout

## 🏗️ High-Level Architecture

Sharly Chess follows a **layered architecture** with a **plugin-based extensibility model**:

```
┌─────────────────────────────────────────────────────────────┐
│                     GUI Layer (Toga)                        │
│              Desktop app with embedded browser              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Web Server Layer (Litestar)                │
│            HTMX-based web app + WebSocket                   │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │  Admin Routes    │  │   User Routes    │                │
│  │  (Arbiters)      │  │   (Players)      │                │
│  └──────────────────┘  └──────────────────┘                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Business Logic Layer                     │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐    │
│  │Tournament│  │  Player  │  │ Pairings │  │  Prize  │    │
│  │  Engine  │  │ Manager  │  │  Engine  │  │ Manager │    │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Plugin System Layer                      │
│  ┌─────────┐ ┌─────────┐ ┌────────────┐ ┌──────────────┐  │
│  │   FFE   │ │  FIDE   │ │ChessEvent │ │Chess-Results │  │
│  │ Plugin  │ │ Plugin  │ │  Plugin   │ │   Plugin     │  │
│  └─────────┘ └─────────┘ └────────────┘ └──────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Data Access Layer                         │
│           SQLite Databases + SQL Server (optional)          │
│  ┌──────────────┐  ┌────────────┐  ┌──────────────┐        │
│  │  Config DB   │  │  Event DB  │  │ External DBs │        │
│  │   (.scc)     │  │   (.sce)   │  │ (FIDE, FFE)  │        │
│  └──────────────┘  └────────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 Directory Structure

```
sharly-chess/
│
├── src/                          # Main source code
│   ├── gui/                      # Desktop GUI (Toga framework)
│   │   └── server_gui_toga.py    # Main GUI entry point
│   │
│   ├── web/                      # Web server & HTTP layer
│   │   ├── server_engine.py      # Litestar/Uvicorn server
│   │   ├── settings.py           # Web app configuration
│   │   ├── controllers/          # HTTP request handlers
│   │   │   ├── admin/            # Arbiter/admin endpoints
│   │   │   │   ├── index_admin_controller.py
│   │   │   │   ├── player_admin_controller.py
│   │   │   │   ├── pairings_admin_controller.py
│   │   │   │   ├── prize_admin_controller.py
│   │   │   │   └── ...
│   │   │   └── user/             # Player-facing endpoints
│   │   ├── templates/            # Jinja2 HTML templates
│   │   ├── static/               # CSS, JS, images
│   │   ├── guards.py             # Authorization guards
│   │   ├── channels.py           # WebSocket channels
│   │   └── session.py            # Session management
│   │
│   ├── data/                     # Domain models & business logic
│   │   ├── event.py              # Event (multi-tournament)
│   │   ├── tournament.py         # Tournament entity
│   │   ├── player.py             # Player & TournamentPlayer
│   │   ├── board.py              # Board (single pairing)
│   │   ├── pairing.py            # Pairing management
│   │   ├── account.py            # User accounts
│   │   ├── timer.py              # Round timers
│   │   ├── screen.py             # Display screens
│   │   ├── pairings/             # Pairing algorithms
│   │   │   ├── engines.py        # Swiss, Round-Robin engines
│   │   │   ├── systems.py        # Pairing system types
│   │   │   ├── checkers.py       # Pairing validation
│   │   │   └── generators.py     # Pairing generation
│   │   ├── tie_breaks/           # Tie-break calculations
│   │   ├── prize/                # Prize distribution
│   │   ├── input_output/         # Import/export logic
│   │   ├── access_levels/        # User permissions
│   │   └── columns/              # Data grid columns
│   │
│   ├── database/                 # Data persistence layer
│   │   ├── sqlite/               # SQLite implementation
│   │   │   ├── sqlite_database.py      # DB context manager
│   │   │   ├── config/                 # Config DB
│   │   │   │   ├── config_database.py
│   │   │   │   └── migrations/
│   │   │   └── event/                  # Event DB
│   │   │       ├── event_database.py
│   │   │       └── migrations/
│   │   └── sql_server/           # SQL Server support
│   │
│   ├── plugins/                  # Plugin system
│   │   ├── manager.py            # Plugin manager
│   │   ├── hookspec.py           # Hook specifications
│   │   ├── utils.py              # Plugin base classes
│   │   ├── ffe/                  # French federation
│   │   ├── chessevent/           # ChessEvent integration
│   │   ├── chess_results/        # Chess-Results.com
│   │   ├── handicap_games/       # Handicap system
│   │   └── pairing_acceleration/ # BBP acceleration
│   │
│   ├── common/                   # Shared utilities & config
│   │   ├── sharly_chess_config.py  # App config (Singleton)
│   │   ├── singleton.py            # Metaclass singleton
│   │   ├── engine.py               # Base engine class
│   │   ├── logger.py               # Logging utilities
│   │   ├── network.py              # Network monitoring
│   │   ├── i18n/                   # Internationalization
│   │   └── licence_templates/      # Tournament templates
│   │
│   ├── utils/                    # Generic utilities
│   │   ├── entity.py             # Base entity classes
│   │   ├── option.py             # Options pattern
│   │   ├── types.py              # Type definitions
│   │   ├── enum.py               # Enum utilities
│   │   └── __init__.py           # Utility functions
│   │
│   └── antivirus/                # File scanning
│
├── docs/                         # Documentation
│   ├── technical-appendices/
│   │   ├── databases.md          # DB schema docs
│   │   ├── dev-setup.md          # Dev environment
│   │   └── files.md              # File structure
│   └── contributing/
│
├── events/                       # Runtime data directory
│   ├── .scc                      # Config database
│   └── *.sce                     # Event databases
│
├── tmp/                          # Temporary files
│   ├── fide.db                   # FIDE player cache
│   └── ffe/ffe.db                # FFE player cache
│
├── tests/                        # Test suite
├── pyproject.toml                # Project metadata & deps
└── README.md
```

---

## 🎯 Key Architectural Layers

### 1. **Presentation Layer** (`src/gui/` + `src/web/`)

**GUI (Desktop App):**
- **Framework:** Toga (cross-platform Python GUI)
- **Purpose:** Wraps the web server in a desktop application
- **Entry Point:** `src/gui/server_gui_toga.py`
- **Features:**
  - Embedded browser view
  - Server log display
  - QR code for mobile access
  - System tray integration

**Web Server:**
- **Framework:** Litestar (async ASGI framework) + Uvicorn
- **Architecture:** HTMX-based (server-rendered HTML with AJAX)
- **Entry Point:** `src/web/server_engine.py`
- **Components:**
  - **Controllers:** Handle HTTP requests
  - **Guards:** Authorization checks
  - **Templates:** Jinja2 HTML templates
  - **Channels:** WebSocket for real-time updates
  - **Session:** Client state management

**Controller Structure:**
```
controllers/
├── admin/              # Arbiter interface (full access)
│   ├── index_admin_controller.py     # Tournament list
│   ├── player_admin_controller.py    # Player management
│   ├── pairings_admin_controller.py  # Pairing generation
│   └── prize_admin_controller.py     # Prize distribution
└── user/               # Player interface (limited access)
    ├── board_controller.py           # View pairings
    └── result_controller.py          # Submit results
```

---

### 2. **Business Logic Layer** (`src/data/`)

Core domain models and tournament logic:

**Key Entities:**
- **Event** (`event.py`): Container for multiple tournaments
- **Tournament** (`tournament.py`): Single tournament with rounds
- **Player** (`player.py`): Global player + tournament-specific data
- **Board** (`board.py`): Single pairing in a round
- **Pairing** (`pairing.py`): Player's pairing history
- **Account** (`account.py`): User authentication

**Pairing Engine** (`data/pairings/`):
```python
# Pairing system hierarchy
PairingSystem (ABC)
├── SwissSystem
│   ├── StandardSwiss
│   ├── AcceleratedSwiss
│   └── Monrad
├── RoundRobinSystem
│   ├── SimpleRoundRobin
│   └── DoubleRoundRobin
└── CustomSystem

# Generation flow:
PairingEngine.generate()
  → PairingGenerator.generate_pairings()
  → PairingChecker.validate()
  → Tournament.save_pairings()
```

**Tie-Break System** (`data/tie_breaks/`):
- Buchholz, Sonneborn-Berger, Performance Rating, etc.
- Extensible through `TieBreak` base class

**Prize Distribution** (`data/prize/`):
- Prize categories and criteria
- Automated prize calculation

---

### 3. **Plugin System** (`src/plugins/`)

**Architecture:** Hook-based using `apluggy` (async fork of pluggy)

**Core Components:**
- **PluginManager** (`manager.py`): Manages plugin lifecycle
- **HookSpec** (`hookspec.py`): Defines extension points
- **Plugin Base Classes** (`utils.py`)

**Available Hooks:**
```python
# Data source hooks
@hookspec
def insert_data_sources(self, data_sources: list[DataSource]):
    """Add federation-specific player databases"""

# Player augmentation
@hookspec
async def augment_player_after_search(self, stored_player: StoredPlayer):
    """Enrich player data from external APIs"""

# Custom rating systems
@hookspec
def get_player_rating(self, tournament_rating: TournamentRating) -> PlayerRating:
    """Provide federation-specific ratings"""

# UI customization
@hookspec
def get_extra_statistics_sections(self, tournament: Tournament) -> list:
    """Add custom statistics to reports"""
```

**Built-in Plugins:**
- **FFE Plugin** (`plugins/ffe/`): French Chess Federation integration
- **ChessEvent Plugin** (`plugins/chessevent/`): ChessEvent import
- **Chess-Results Plugin** (`plugins/chess_results/`): Upload results
- **Handicap Games** (`plugins/handicap_games/`): Custom scoring
- **Pairing Acceleration** (`plugins/pairing_acceleration/`): BBP

---

### 4. **Data Access Layer** (`src/database/`)

**SQLite Implementation:**

**Context Manager Pattern:**
```python
@dataclass
class SQLiteDatabase:
    file: Path
    write: bool = False

    def __enter__(self) -> Self:
        # Open connection, begin transaction

    def __exit__(self, exc_type, exc_value, tb):
        # Commit or rollback

# Usage:
with SQLiteDatabase(event_path, write=True) as db:
    db.cursor.execute("INSERT INTO ...")
    # Auto-commits on success, rolls back on exception
```

**Database Files:**
- **Config DB** (`.scc`): Application settings, plugins, metadata
- **Event DB** (`.sce`): Tournament data, players, pairings, results
- **External DBs**: FIDE ratings, FFE data (cached)

**Migration System:**
- Version-based migrations
- Applied automatically on DB open
- Located in `migrations/` folders

---

### 5. **Utilities & Common** (`src/utils/`, `src/common/`)

**Utils:**
- **Entity Framework** (`entity.py`): Base classes with IDs
- **Options Pattern** (`option.py`): Type-safe configuration
- **Enums** (`enum.py`): Result, Gender, Title, etc.
- **Types** (`types.py`): Federation, Rating types

**Common:**
- **Configuration** (`sharly_chess_config.py`): Singleton config
- **Logging** (`logger.py`): Colored console output
- **i18n** (`i18n/`): Multi-language support
- **Network Monitor** (`network.py`): Connectivity checks

---

## 🔄 Data Flow

### Example: Generating Round Pairings

```
1. User clicks "Generate Round 3" in browser
   ↓
2. HTMX sends POST to /admin/tournament/{id}/round/3/generate
   ↓
3. pairings_admin_controller.generate_round()
   ├── Loads Tournament from database
   ├── Calls PairingEngine.generate()
   │   ├── Reads BBP history
   │   ├── Calculates opponent matrix
   │   ├── Runs Swiss algorithm
   │   └── Validates pairings
   ├── Saves boards to database
   └── Returns HTML fragment
   ↓
4. HTMX swaps HTML into page
   ↓
5. WebSocket broadcasts "pairings_updated" event
   ↓
6. Display screens refresh automatically
```

### Example: Player Self Check-in

```
1. Player scans QR code on phone
   ↓
2. Opens /user/tournament/{id}/checkin
   ↓
3. Submits player ID
   ↓
4. user/board_controller.checkin()
   ├── Validates player exists
   ├── Updates check-in status in DB
   └── Returns success message
   ↓
5. WebSocket notifies admin screen
   ↓
6. Admin sees real-time check-in count update
```

---

## 🚀 Application Entry Points

### Desktop Application
```python
# src/gui/server_gui_toga.py
def main():
    return SharlyChessGUI()

# Starts Toga app which:
# 1. Creates event loop
# 2. Starts ServerEngine in background thread
# 3. Opens embedded browser to localhost:8000
```

### CLI Mode
```bash
# Direct server start (no GUI)
python -m src.web.server_engine

# Or using installed package
sharly-chess --no-gui
```

### Web Server Startup Sequence
```python
# src/web/server_engine.py
ServerEngine.__init__()
  → Load configuration
  → Initialize plugin system
  → Configure Litestar app
  → Setup routes, middleware, guards

ServerEngine.serve()
  → Start uvicorn server
  → Launch browser (optional)
  → Handle signals (SIGINT, SIGTERM)
```

---

## 🔑 Key Design Patterns

### 1. **Singleton Pattern** (Metaclass)
- Configuration management (`common/singleton.py`)
- Ensures single app config instance

### 2. **Context Manager Pattern**
- Database transactions (`database/sqlite/sqlite_database.py`)
- Automatic commit/rollback

### 3. **Plugin Pattern** (Hook-based)
- Federation-specific extensions (`plugins/`)
- Decoupled extensibility

### 4. **Guard Pattern**
- Authorization (`web/guards.py`)
- Declarative access control

### 5. **Repository Pattern**
- Database abstraction
- Separation of data access from business logic

### 6. **Abstract Base Classes**
- Entity hierarchy (`utils/entity.py`)
- Enforced contracts

### 7. **Weak References**
- Prevents circular dependencies (`data/pairing.py`)
- Memory-efficient object graphs

### 8. **Session Variable Pattern**
- Type-safe session state (`web/session.py`)
- Scoped to event/tournament

---

## 🛠️ Technology Stack

| Layer | Technology |
|-------|-----------|
| **GUI** | Toga 0.5.2 |
| **Web Framework** | Litestar 2.16 (ASGI) |
| **Server** | Uvicorn 0.34 |
| **Templates** | Jinja2 3.1 |
| **Frontend** | HTMX (server-rendered) |
| **Database** | SQLite 3 (aiosqlite) |
| **Plugin System** | apluggy 1.1 |
| **Type Checking** | mypy + Python 3.13 type hints |
| **i18n** | Babel 2.16 |
| **Testing** | pytest + pytest-playwright |

---

## 📊 Database Schema Overview

**Config DB (`.scc`):**
- `info`: App version, settings
- `plugin`: Enabled plugins per event
- `local_source_database`: External data sources
- `player_category_set`: Custom categories
- `metadata`: Key-value storage

**Event DB (`.sce`):**
- `tournament`: Tournament metadata
- `player`: Global player registry
- `tournament_player`: Player in specific tournament
- `board`: Pairings (white, black, result)
- `pairing`: Player's pairing history
- `screen`, `screen_set`: Display management
- `timer`, `timer_hour`: Time controls
- `prize_*`: Prize categories and distribution
- `account`: User authentication

---

## 🎓 Learning Path for Developers

**To understand the codebase:**

1. **Start with domain models** (`src/data/`)
   - Read `tournament.py`, `player.py`, `board.py`
   - Understand core entities and relationships

2. **Explore pairing engine** (`src/data/pairings/`)
   - See how Swiss system works
   - Study `engines.py`

3. **Follow web request flow**
   - Pick a controller (e.g., `pairings_admin_controller.py`)
   - Trace request → business logic → database → response

4. **Study plugin system** (`src/plugins/`)
   - Read `hookspec.py` for extension points
   - Examine `ffe/` plugin as example

5. **Review database layer** (`src/database/sqlite/`)
   - Understand context manager pattern
   - Look at migration system

---

## 🔍 Quick Reference

**Common Tasks:**

| Task | File Location |
|------|--------------|
| Add new HTTP route | `src/web/controllers/` |
| Add plugin hook | `src/plugins/hookspec.py` |
| Modify pairing algorithm | `src/data/pairings/engines.py` |
| Change database schema | `src/database/sqlite/*/migrations/` |
| Add configuration option | `src/common/sharly_chess_config.py` |
| Add tie-break method | `src/data/tie_breaks/` |
| Add access control | `src/data/access_levels/` |
| Customize UI template | `src/web/templates/` |

---

This architecture balances **flexibility** (plugin system), **maintainability** (clear layers), and **performance** (async I/O, SQLite) for a desktop chess tournament management application.
