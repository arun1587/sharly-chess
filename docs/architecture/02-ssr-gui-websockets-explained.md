# Server-Side Rendering, GUI Wrapper, and WebSockets Explained

## Overview

This document clarifies three key architectural components of Sharly Chess:
1. **Server-Side Rendering (SSR)** with HTMX
2. **GUI Wrapper** purpose and implementation
3. **WebSocket** usage for real-time updates

---

## 1. Is it Server-Side Rendering? ✅ YES

### Technology: HTMX (Not React/Vue)

The server renders **HTML fragments** and sends them to the browser. HTMX intercepts user interactions and swaps HTML without page reloads.

### Example from Template

**File:** `src/web/templates/admin/common/base.html:9`

```html
<body hx-ext="ws" ws-connect="/ws">
  <!-- HTMX connects WebSocket automatically -->
  <button hx-post="/admin/tournament/123/round/3/generate"
          hx-target="#pairings-table">
    Generate Round 3
  </button>

  <div id="pairings-table">
    <!-- Server sends HTML here, not JSON -->
  </div>
</body>
```

### Request/Response Flow

**Traditional SPA (React/Vue):**
```
Browser → Server: GET /api/pairings
Server → Browser: {"pairings": [...]}  # JSON
Browser: Renders JSON into HTML using JavaScript
```

**Sharly Chess (HTMX):**
```
Browser → Server: GET /admin/pairings
Server → Browser: <table><tr>...</tr></table>  # HTML
Browser: Swaps HTML directly (no JavaScript rendering)
```

### Complete Flow Example

1. **User clicks "Generate Round 3"**
2. **HTMX sends POST to server**
   ```http
   POST /admin/tournament/123/round/3/generate
   ```
3. **Server renders HTML** (via Jinja2 template)
   ```python
   @post('/tournament/{id}/round/{round}/generate')
   async def generate_round(id: int, round: int) -> HTMXTemplate:
       tournament = load_tournament(id)
       tournament.generate_round(round)

       # Render HTML template
       return HTMXTemplate(
           template_name='pairings_table.html',
           context={'tournament': tournament, 'round': round}
       )
   ```
4. **HTMX swaps the HTML into `#pairings-table`**
5. **No JavaScript framework needed!**

### Why HTMX for This Project?

✅ **Document-centric UI** (tables, forms, lists)
✅ **No build process** (no npm, webpack, babel)
✅ **Fast time-to-interactive**
✅ **Server-side validation** is easier
✅ **Simpler mental model** for non-frontend developers

---

## 2. Why the GUI Wrapper?

### Problem: Web Apps Need a Server

Without the GUI wrapper, users would need to:
```bash
$ python -m sharly-chess
Starting web server on http://localhost:8000
# Then manually open browser to localhost:8000
```

This is **not user-friendly** for chess arbiters who aren't developers.

### Solution: Desktop App (Toga)

The GUI wrapper makes it a **one-click desktop application**:

**File:** `src/gui/server_gui_toga.py`

```python
class SharlyChessServerToga(toga.App):
    def startup(self):
        # 1. Create window with embedded WebView
        self.web_view = toga.WebView()

        # 2. Start web server in background thread
        self.server_thread = Thread(target=self._run_server)
        self.server_thread.start()

        # 3. Show server logs in the app
        self.log_display = toga.DetailedList()

        # 4. Generate QR code for mobile access
        self.qr_code = generate_qr_code(f"http://{ip}:8000")

        # 5. Load web app in embedded browser
        self.web_view.url = "http://localhost:8000"
```

### What the GUI Provides

#### A. Embedded Browser
- User doesn't need to know about "localhost:8000"
- Just double-click the app icon

#### B. Server Log Display
```
┌─────────────────────────────────────┐
│  Sharly Chess                    [x]│
├─────────────────────────────────────┤
│  Web View (embedded browser)        │
│  ┌───────────────────────────────┐  │
│  │                               │  │
│  │  [Tournament Management UI]   │  │
│  │                               │  │
│  └───────────────────────────────┘  │
├─────────────────────────────────────┤
│  Server Logs:                       │
│  ✓ Server started on port 8000      │
│  ✓ Database loaded successfully     │
│  ✓ Round 3 generated (50 boards)    │
└─────────────────────────────────────┘
```

#### C. QR Code for Mobile Access
```
┌─────────────────────┐
│ Scan to access from │
│ your phone:         │
│                     │
│  ███▀▀▀███▀▀███▀██  │
│  ██▀▄██▄▀▄█▀▄█▀▀██  │
│  ██▀███▀▄▀█▀███▀▀██  │
│                     │
│ http://192.168.1.5  │
│      :8000          │
└─────────────────────┘
```

Players can scan the QR code and check in from their phones!

#### D. Cross-Platform Packaging
- **Windows:** `.exe` file
- **macOS:** `.app` bundle
- **Linux:** `.AppImage` or `.deb`

### Entry Point Logic

**File:** `src/sharly_chess.py:250-377`

```python
if not TEST_ENV and not (DEVEL_ENV and args.cli):
    # Production mode: Launch GUI
    app = SharlyChessServerToga(debug=debug, port=port)
    app.main_loop()  # Blocks until user closes window
else:
    # Development/CLI mode: Just run web server
    se = ServerEngine(debug=debug, port=port)
    asyncio.run(se.serve())
```

### Benefits of GUI Wrapper

| Benefit | Description |
|---------|-------------|
| **User-Friendly** | No terminal commands needed |
| **Portable** | Single .exe/.app/.dmg file |
| **Integrated Logs** | See server output without opening terminal |
| **QR Code** | Easy mobile access for players |
| **Cross-Platform** | Windows, Mac, Linux |
| **Professional** | Looks like a "real" desktop app |

---

## 3. Where are WebSockets Used?

### Use Case: Real-Time Tournament Updates

WebSockets enable **real-time updates** across multiple screens/devices **without polling**.

### Scenario: Multi-Screen Tournament Display

**Setup:**
- **Admin screen**: Arbiter entering results
- **Display Screen 1**: Shows pairings on projector
- **Display Screen 2**: Shows standings on another screen
- **Player phones**: 50+ players viewing their pairings

**Total:** 52+ connected devices

### Without WebSockets (Polling)
```
❌ Each screen polls every 5 seconds: "Any updates?"
   → 52 requests every 5 seconds = 624 requests/minute!
   → High server load
   → Delay in updates (up to 5 seconds)
   → Wasted bandwidth
```

### With WebSockets
```
✅ Admin updates result → Server broadcasts once to all 52 clients
   → 1 message reaches everyone instantly
   → Near-zero latency
   → Minimal server load
   → Efficient bandwidth usage
```

---

## WebSocket Architecture

### Server Setup

**File:** `src/web/channels.py`

```python
from litestar.channels import ChannelsPlugin
from litestar.channels.backends.memory import MemoryChannelsBackend

channels_plugin = ChannelsPlugin(
    backend=MemoryChannelsBackend(),  # In-memory pub/sub
    arbitrary_channels_allowed=True,
)
```

### WebSocket Connection Handler

**File:** `src/web/controllers/index_controller.py:161-170`

```python
@websocket_stream('/ws')
async def ws_handler(self, channels: ChannelsPlugin) -> AsyncGenerator[dict, None]:
    # When a browser connects to ws://localhost:8000/ws
    # This function starts running

    # Subscribe to the 'ws' channel
    async with channels.start_subscription(['ws']) as subscriber:
        # Keep listening for events forever
        async for raw_event in subscriber.iter_events():
            event = json.loads(raw_event) if isinstance(raw_event, (bytes, str)) else raw_event
            # Send event to THIS connected client
            yield event  # ← Sends message to browser
```

### Publishing Updates

**File:** `src/web/controllers/admin/pairings_admin_controller.py:1322`

```python
@post('/tournament/{tournament_id}/board/{board_id}/result')
async def save_result(
    self,
    tournament_id: int,
    board_id: int,
    result: str,
    channels: ChannelsPlugin
):
    # 1. Save result to database
    board = load_board(board_id)
    board.result = result
    board.save_to_database()

    # 2. Publish WebSocket event to ALL connected clients
    channels.publish(
        {
            'event': 'new-user-results|event-123|5|3',
            'data': '',
        },
        ['ws'],  # ← Channel name
    )
    # → This message is sent to ALL clients subscribed to 'ws'
```

### Client-Side (HTMX)

**File:** `src/web/templates/admin/pairings/tab.html`

```html
<!-- Pairings table header -->
<div id="pairings-table-header"
     hx-get="/admin/tournament/5/round/3/pairings/header"
     hx-target="#pairings-table-header"
     hx-swap="outerHTML"
     hx-trigger="ws:new-user-results|event-123|5|3 from:body">
     <!--         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                  Listen for WebSocket event with this name -->

  <!-- Current content -->
  <table>...</table>
</div>
```

### What HTMX Does Automatically

1. **Connects** to WebSocket at `/ws` (via `ws-connect="/ws"` on body)
2. **Listens** for message with event: `new-user-results|event-123|5|3`
3. When received, **triggers HTTP GET** to `/admin/tournament/5/round/3/pairings/header`
4. Server returns **fresh HTML**
5. HTMX **swaps** the HTML into `#pairings-table-header`

**Result:** Table updates without page reload! 🎉

---

## Complete Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    WEB SERVER (Litestar)                    │
│                                                             │
│  1. WebSocket Handler (/ws)                                │
│     • Subscribes clients to 'ws' channel                   │
│     • Forwards messages to connected clients               │
│                                                             │
│  2. HTTP Controller                                        │
│     • Receives POST: "Save result for Board 5"             │
│     • Updates database                                     │
│     • Publishes: channels.publish({...}, ['ws'])           │
│                                                             │
│  3. In-Memory Channel (PubSub)                             │
│     • Distributes messages to all subscribers              │
└─────────────────────────────────────────────────────────────┘
                          │
                          │  WebSocket messages
                          ▼
    ┌─────────────────────┴───────────────────────┐
    │                                             │
    ▼                                             ▼
┌─────────────────┐                    ┌─────────────────┐
│  CLIENT 1       │                    │  CLIENT 2       │
│  (Admin Screen) │                    │  (Display)      │
│                 │                    │                 │
│  Browser JS:    │                    │  Browser JS:    │
│  • Receives:    │                    │  • Receives:    │
│    {event:      │                    │    {event:      │
│     "new-user-  │                    │     "new-user-  │
│     results"}   │                    │     results"}   │
│                 │                    │                 │
│  HTMX:          │                    │  HTMX:          │
│  • Matches      │                    │  • Matches      │
│    hx-trigger   │                    │    hx-trigger   │
│  • Fetches new  │                    │  • Fetches new  │
│    HTML         │                    │    HTML         │
│  • Updates DOM  │                    │  • Updates DOM  │
└─────────────────┘                    └─────────────────┘
```

---

## Real-World Example: Check-in Flow

### Scenario
Player Alice checks in on her phone. Admin screen needs to update the check-in count **instantly**.

### Step-by-Step

#### 1. Player Phone (Client → Server)
```
Alice clicks "I'm here" button
↓
POST /user/tournament/5/checkin
```

#### 2. Server Processes & Publishes
```python
# player_admin_controller.py:1762
@post('/tournament/{id}/checkin')
async def checkin(player_id: int, channels: ChannelsPlugin):
    # Save check-in to database
    player.checked_in = True
    player.save()

    # Broadcast to all clients
    channels.publish(
        {'event': 'new-checkins|event-123|5|3', 'data': ''},
        ['ws']
    )
```

#### 3. Admin Screen HTML
```html
<!-- admin/players/tab.html -->
<div id="checkin-count"
     hx-get="/admin/tournament/5/checkins"
     hx-trigger="ws:new-checkins|event-123|5|3 from:body">
  Checked in: 45/100
</div>
```

#### 4. Admin Screen Updates
```
WebSocket receives: {event: "new-checkins|event-123|5|3"}
↓
HTMX triggers: GET /admin/tournament/5/checkins
↓
Server returns: <div>Checked in: 46/100</div>
↓
HTMX swaps HTML
↓
Admin sees: "Checked in: 46/100" (updated instantly!)
```

### Timing
- **Without WebSockets:** Up to 5 seconds delay (polling interval)
- **With WebSockets:** ~100ms delay (network latency only)

---

## Summary

### Server-Side Rendering (HTMX)
✅ Server renders HTML, not JSON
✅ No build process needed
✅ Simple for document-centric UIs
✅ Perfect for chess tournament management

### GUI Wrapper (Toga)
✅ Makes web app feel like desktop app
✅ One-click install for non-technical users
✅ Embedded browser + server logs
✅ QR code for mobile access

### WebSockets
✅ Real-time updates across all devices
✅ Efficient (broadcast once, reach everyone)
✅ Low latency (< 100ms)
✅ Essential for multi-screen tournaments

---

## Technology Choices Summary

| Component | Technology | Why? |
|-----------|-----------|------|
| **Rendering** | Server-Side (HTMX) | Document-centric UI, no build process |
| **Frontend Interactivity** | HTMX | Minimal JavaScript, server controls everything |
| **Backend** | Python + Litestar | Async, type-safe, domain logic friendly |
| **Database** | SQLite | Offline-first, file-based, ACID |
| **Real-time** | WebSockets | Broadcast updates to multiple screens |
| **Desktop Wrapper** | Toga | Cross-platform, embedded browser |

This architecture is **perfectly suited** for a chess tournament management application that needs to work offline, support multiple displays, and be accessible to non-technical arbiters.
