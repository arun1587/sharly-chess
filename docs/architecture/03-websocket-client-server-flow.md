# WebSocket Client-Server Flow: Complete Guide

## Overview

This document provides a detailed explanation of WebSocket architecture in Sharly Chess, clarifying the roles of clients, servers, publishers, and consumers.

---

## Architecture Summary

### Server = Message Broker (Publisher)
The **web server** acts as the **message broker/publisher**. It sends messages to ALL connected clients.

### Clients = Message Subscribers (Consumers)
The **browsers** (admin screens, display screens, player phones) are **consumers**. They listen for messages and react.

---

## Complete WebSocket Flow

### 1️⃣ Client Connects (Browser → Server)

#### Frontend Template

**File:** `src/web/templates/common/base.html:71`

```html
<!-- Line 22: Load HTMX WebSocket extension -->
<script src="/static/lib/htmx/ws.js"></script>

<!-- Line 71: body tag in admin base template -->
<div id="main-wrapper"
     hx-ext="ws"
     ws-connect="/ws">
  <!-- HTMX automatically opens WebSocket to /ws -->
</div>
```

#### Backend WebSocket Handler

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

#### What Happens

1. **Browser connects** → Creates WebSocket connection to `ws://localhost:8000/ws`
2. **Server subscribes** this connection to the `'ws'` channel
3. **Server keeps connection open**, waiting to forward messages
4. Connection stays alive until browser closes or server shuts down

---

### 2️⃣ Server Publishes Message (Server → All Clients)

#### Scenario: Arbiter Enters a Result

**File:** `src/web/controllers/admin/pairings_admin_controller.py:1315-1335`

```python
@post('/tournament/{tournament_id}/board/{board_id}/result')
async def save_result(
    self,
    tournament_id: int,
    board_id: int,
    result: str,
    channels: ChannelsPlugin  # ← Injected by Litestar
):
    # 1. Save result to database
    board = load_board(board_id)
    board.result = result
    board.save_to_database()

    # 2. Publish WebSocket event to ALL connected clients
    self.publish_new_user_results(
        channels=channels,
        event_uniq_id='event-123',
        tournament_id=tournament_id,
        round_=3
    )

def publish_new_user_results(cls, channels: ChannelsPlugin, ...):
    # Publish to the 'ws' channel
    channels.publish(
        {
            'event': 'new-user-results|event-123|5|3',  # ← Event name
            'data': '',
        },
        ['ws'],  # ← Channel name
    )
    # → This message is sent to ALL clients subscribed to 'ws'
```

#### Internal Flow

```
1. channels.publish() sends message to in-memory channel
   ↓
2. ChannelsPlugin broadcasts to all subscribers
   ↓
3. Each ws_handler (one per connected client) receives message
   ↓
4. ws_handler forwards message to its WebSocket connection
   ↓
5. Browser receives message via WebSocket
```

---

### 3️⃣ Clients Receive & React (Browser Receives Message)

#### Frontend Template

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
  <table>
    <tr><td>Board 5</td><td>White vs Black</td><td>1-0</td></tr>
  </table>
</div>
```

#### HTMX Automatic Behavior

1. **Listens** for WebSocket message with event name: `new-user-results|event-123|5|3`
2. When received, **triggers HTTP GET** to `/admin/tournament/5/round/3/pairings/header`
3. Server returns **fresh HTML**
4. HTMX **swaps** the HTML into `#pairings-table-header`

#### Result
Table updates without page reload! 🎉

---

## Visual Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    WEB SERVER (Litestar)                        │
│                                                                 │
│  1. WebSocket Handler (/ws)                                    │
│     ┌─────────────────────────────────────────┐                │
│     │ @websocket_stream('/ws')                │                │
│     │ async def ws_handler():                 │                │
│     │   async with channels.subscribe(['ws']): │               │
│     │     async for event in subscriber:      │                │
│     │       yield event  # → Send to client   │                │
│     └─────────────────────────────────────────┘                │
│                                                                 │
│  2. HTTP Controller                                            │
│     ┌─────────────────────────────────────────┐                │
│     │ @post('/board/{id}/result')             │                │
│     │ async def save_result():                │                │
│     │   board.result = result                 │                │
│     │   board.save()                          │                │
│     │   channels.publish(                     │                │
│     │     {'event': 'new-user-results'},      │                │
│     │     ['ws']                               │                │
│     │   )                                      │                │
│     └─────────────────────────────────────────┘                │
│                                                                 │
│  3. In-Memory Channel (PubSub)                                 │
│     ┌─────────────────────────────────────────┐                │
│     │ MemoryChannelsBackend                   │                │
│     │ • Subscribers: [ws_handler_1,           │                │
│     │                 ws_handler_2, ...]      │                │
│     │ • Broadcasts to all subscribers         │                │
│     └─────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────────┘
                          │
                          │  WebSocket Protocol
                          │  (ws://localhost:8000/ws)
                          ▼
    ┌─────────────────────┴───────────────────────┐
    │                                             │
    ▼                                             ▼
┌──────────────────────┐                ┌──────────────────────┐
│  CLIENT 1            │                │  CLIENT 2            │
│  (Admin Screen)      │                │  (Display Screen)    │
│                      │                │                      │
│  WebSocket Client    │                │  WebSocket Client    │
│  • Connected to /ws  │                │  • Connected to /ws  │
│                      │                │                      │
│  Receives Message:   │                │  Receives Message:   │
│  {                   │                │  {                   │
│    event: "new-user- │                │    event: "new-user- │
│           results",  │                │           results",  │
│    data: ""          │                │    data: ""          │
│  }                   │                │  }                   │
│         ↓            │                │         ↓            │
│  HTMX Matches:       │                │  HTMX Matches:       │
│  hx-trigger="ws:     │                │  hx-trigger="ws:     │
│  new-user-results"   │                │  new-user-results"   │
│         ↓            │                │         ↓            │
│  Sends HTTP Request: │                │  Sends HTTP Request: │
│  GET /admin/pairings │                │  GET /user/standings │
│         ↓            │                │         ↓            │
│  Server Returns HTML │                │  Server Returns HTML │
│         ↓            │                │         ↓            │
│  HTMX Swaps HTML     │                │  HTMX Swaps HTML     │
│         ↓            │                │         ↓            │
│  UI Updates! ✓       │                │  UI Updates! ✓       │
└──────────────────────┘                └──────────────────────┘
```

---

## Detailed Example: Check-in Flow

### Scenario

- **Player Alice** checks in on her phone
- **Admin screen** needs to update the check-in count instantly
- **Display screen** showing player list needs to update

### Complete Step-by-Step

#### Step 1: Player Action (Client → Server)

```
Alice opens her phone browser
↓
Navigates to: http://192.168.1.5:8000/user/tournament/5
↓
Clicks "I'm Here" button
↓
HTMX sends: POST /user/tournament/5/checkin
             Body: {player_id: 42}
```

#### Step 2: Server Processes Request

**File:** `src/web/controllers/admin/player_admin_controller.py:1762`

```python
@post('/tournament/{tournament_id}/checkin')
async def checkin(
    tournament_id: int,
    player_id: int,
    channels: ChannelsPlugin
):
    # 1. Load player from database
    player = load_player(player_id)

    # 2. Update check-in status
    player.checked_in = True
    player.checkin_time = datetime.now()

    # 3. Save to database
    with SQLiteDatabase(db_path, write=True) as db:
        db.execute(
            "UPDATE tournament_player SET checked_in = 1, "
            "checkin_time = ? WHERE id = ?",
            (player.checkin_time, player_id)
        )

    # 4. Broadcast WebSocket event to ALL clients
    self.publish_new_checkin(channels, 'event-123', player)
```

#### Step 3: Publishing to Channel

```python
def publish_new_checkin(
    cls,
    channels: ChannelsPlugin,
    event_uniq_id: str,
    player: Player
):
    # Broadcast to all connected clients
    channels.publish(
        {
            'event': f'new-checkins|{event_uniq_id}',
            'data': '',
        },
        ['ws'],  # Channel name
    )

    # Also broadcast tournament-specific event
    channels.publish(
        {
            'event': f'new-checkins|{event_uniq_id}|{player.tournament.id}|{player.tournament.current_round}',
            'data': '',
        },
        ['ws'],
    )
```

#### Step 4: Message Distribution

```
In-Memory Channel Broker:
  Receives: {'event': 'new-checkins|event-123'}
  ↓
  Has 3 active subscriptions:
    - ws_handler for Admin Screen (connection #1)
    - ws_handler for Display Screen (connection #2)
    - ws_handler for Alice's Phone (connection #3)
  ↓
  Forwards message to all 3 connections simultaneously
```

#### Step 5: Admin Screen Receives (Client Reacts)

**HTML:**
```html
<!-- admin/players/tab.html -->
<div id="checkin-stats"
     hx-get="/admin/tournament/5/checkin-stats"
     hx-target="#checkin-stats"
     hx-swap="outerHTML"
     hx-trigger="ws:new-checkins|event-123 from:body">

  <!-- Current content -->
  <div class="alert alert-info">
    Checked in: 45 / 100 players
  </div>
</div>
```

**What happens:**
```
1. Browser WebSocket receives message:
   {event: "new-checkins|event-123"}

2. HTMX detects match with:
   hx-trigger="ws:new-checkins|event-123 from:body"

3. HTMX triggers HTTP GET:
   GET /admin/tournament/5/checkin-stats

4. Server responds with fresh HTML:
   <div class="alert alert-info">
     Checked in: 46 / 100 players
   </div>

5. HTMX swaps HTML into #checkin-stats

6. Admin sees updated count: 46 / 100
```

#### Step 6: Display Screen Receives (Client Reacts)

**HTML:**
```html
<!-- display/player-list.html -->
<div id="player-list"
     hx-get="/display/tournament/5/players"
     hx-target="#player-list"
     hx-swap="outerHTML"
     hx-trigger="ws:new-checkins|event-123 from:body">

  <!-- Current player list -->
  <ul>
    <li>Alice Johnson - ⏳ Waiting</li>
    <li>Bob Smith - ✓ Checked in</li>
    ...
  </ul>
</div>
```

**What happens:**
```
1. Display screen WebSocket receives same message
2. HTMX matches trigger
3. Fetches fresh player list
4. Server returns updated HTML:
   <ul>
     <li>Alice Johnson - ✓ Checked in</li>  ← Updated!
     <li>Bob Smith - ✓ Checked in</li>
     ...
   </ul>
5. Display screen updates to show Alice checked in
```

---

## Roles Summary

### Who is the Server?

**Component:** Web Server (Litestar running on localhost:8000)

**Roles:**
1. **WebSocket Server**: Maintains persistent connections to browsers
2. **Message Publisher**: Sends messages when data changes
3. **HTTP Server**: Responds to data requests from clients

**File:** `src/web/controllers/index_controller.py:161`

```python
@websocket_stream('/ws')
async def ws_handler(self, channels: ChannelsPlugin):
    # This is the WebSocket server
    # One instance runs per connected client
    async with channels.start_subscription(['ws']) as subscriber:
        async for event in subscriber.iter_events():
            yield event  # Send to connected browser
```

### Who is the Client?

**Component:** Browsers (Desktop app, phones, tablets)

**Types:**
1. **Admin Browser**: Arbiter's interface (full control)
2. **Display Browsers**: Projector screens showing pairings/standings
3. **Player Browsers**: Players' phones for viewing pairings and checking in

**Connection:** Each browser opens a WebSocket connection to `ws://localhost:8000/ws`

### Who is the Publisher?

**Component:** Server-side controllers

**File:** `src/web/controllers/admin/pairings_admin_controller.py:1322`

```python
@post('/tournament/{id}/board/{board_id}/result')
async def save_result(channels: ChannelsPlugin):
    # This controller is the publisher
    channels.publish(
        {'event': 'new-user-results|...'},
        ['ws']
    )
```

**When do they publish?**
- After saving a game result
- After generating pairings
- After player check-in
- After tournament settings change
- After prize recalculation

### Who is the Consumer?

**Component:** Browsers (same as clients)

**How they consume:**
1. Receive WebSocket message
2. Match event name with `hx-trigger` attribute
3. Fetch fresh data from server
4. Update their UI

---

## Message Flow Summary

```
PUBLISH SIDE (Server):
┌─────────────────────────────────────┐
│  1. User Action (HTTP Request)      │
│     POST /board/5/result            │
└─────────────────┬───────────────────┘
                  ↓
┌─────────────────────────────────────┐
│  2. Controller Processes            │
│     • Update database               │
│     • channels.publish()            │
└─────────────────┬───────────────────┘
                  ↓
┌─────────────────────────────────────┐
│  3. Channels Plugin (Message Broker)│
│     • Broadcasts to all subscribers │
└─────────────────┬───────────────────┘
                  ↓
┌─────────────────────────────────────┐
│  4. WebSocket Handlers              │
│     • One per connected client      │
│     • Forwards message to browser   │
└─────────────────┬───────────────────┘
                  ↓
        ┌─────────┴─────────┐
        │  WebSocket Wire   │
        │  (Network)        │
        └─────────┬─────────┘
                  ↓
CONSUME SIDE (Browsers):
┌─────────────────────────────────────┐
│  5. Browser WebSocket Client        │
│     • Receives message              │
└─────────────────┬───────────────────┘
                  ↓
┌─────────────────────────────────────┐
│  6. HTMX Event Handler              │
│     • Matches hx-trigger            │
│     • Sends HTTP GET                │
└─────────────────┬───────────────────┘
                  ↓
┌─────────────────────────────────────┐
│  7. Server Responds with HTML       │
└─────────────────┬───────────────────┘
                  ↓
┌─────────────────────────────────────┐
│  8. HTMX Swaps HTML                 │
│     • UI updates                    │
└─────────────────────────────────────┘
```

---

## Key Insights

### 1. Publish-Subscribe Pattern

**Pattern:** PubSub (Publish-Subscribe)

**Characteristics:**
- **One publisher** (server) → **Many subscribers** (browsers)
- **Decoupled**: Server doesn't know which clients exist
- **Broadcast**: One message reaches everyone instantly
- **Asynchronous**: Clients react independently

### 2. Server is Both Sender and Receiver

```
Server as WebSocket Server:
  • Receives WebSocket connections from clients
  • Keeps connections alive
  • Forwards messages to connected clients

Server as Message Publisher:
  • Publishes events when data changes
  • Doesn't know or care who's listening
  • Messages go through in-memory channel broker
```

### 3. Clients Only Consume (Never Publish)

```
Browser Capabilities:
  ✓ Connect to WebSocket
  ✓ Receive messages
  ✓ React to messages (fetch fresh HTML)
  ✗ Cannot publish messages to other clients
  ✗ Cannot broadcast events

All state changes go through server.
```

### 4. Event Naming Convention

**Pattern:** `event-type|event-id|tournament-id|round`

**Examples:**
```
new-user-results|event-123|5|3
  ↑              ↑         ↑ ↑
  Event type     Event ID  │ Round
                           Tournament ID

new-checkins|event-123
  ↑          ↑
  Event type Event ID (all tournaments)
```

**Why?**
- Clients can filter events by tournament/round
- Fine-grained control over which elements update
- Multiple tournaments can run simultaneously

---

## Performance Characteristics

### Comparison: Polling vs WebSockets

**Scenario:** 50 connected clients, 1 update per minute

#### Polling (5-second interval)
```
Requests per minute: 50 clients × 12 polls/minute = 600 requests
Latency: 0-5 seconds (average 2.5 seconds)
Server load: High (constant requests)
Bandwidth: High (repeated responses)
```

#### WebSockets
```
Messages per minute: 1 broadcast = 50 deliveries
Latency: <100ms (network only)
Server load: Low (one message)
Bandwidth: Low (one message, minimal overhead)
```

**WebSockets are 600× more efficient!**

---

## Troubleshooting

### Client Not Receiving Messages

**Check:**
1. WebSocket connection established?
   ```javascript
   // In browser console:
   console.log(htmx.config.webSocketClient);
   ```

2. Event name matches exactly?
   ```html
   hx-trigger="ws:new-user-results|event-123|5|3"
   <!-- Must match published event name exactly -->
   ```

3. Server publishing to correct channel?
   ```python
   channels.publish({...}, ['ws'])  # Must be 'ws'
   ```

### All Clients Disconnecting

**Possible causes:**
- Server restart (in-memory channel loses state)
- Network issue
- Server crash

**Solution:**
- Clients auto-reconnect (HTMX handles this)
- Consider persistent channel backend (Redis)

---

## Conclusion

**Who is Who:**

| Role | Component | Responsibility |
|------|-----------|----------------|
| **Server** | Litestar Web Server | Maintains WebSocket connections |
| **Publisher** | HTTP Controllers | Publishes events when data changes |
| **Broker** | ChannelsPlugin (in-memory) | Routes messages to subscribers |
| **Clients** | Browsers | Receive messages, update UI |
| **Consumers** | Browsers (same as clients) | React to events by fetching fresh data |

**Flow:**
1. Client connects → Server maintains WebSocket
2. User action → Server publishes event
3. Broker broadcasts → All clients receive
4. Clients react → Fetch fresh HTML → Update UI

This architecture enables **real-time collaboration** across multiple devices with minimal latency and server load.
