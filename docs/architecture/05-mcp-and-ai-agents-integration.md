# MCP Servers and AI Agents Integration: Feasibility Study

## Executive Summary

Integrating MCP servers and AI agents into Sharly Chess presents **high-value opportunities** for:
- **Real-time rule compliance checking**
- **Anomaly detection** (suspicious results, rating manipulation)
- **Natural language tournament queries**
- **Multi-tournament coordination**
- **Intelligent pairing suggestions**

**Feasibility: HIGH** - The architecture is well-suited for MCP integration due to:
- ✅ Clean plugin system
- ✅ SQLite databases (easy to expose)
- ✅ Well-defined domain models
- ✅ Async Python (compatible with agents)

---

## 🎯 Realistic Use Cases

### Use Case 1: **AI Arbiter Assistant** (Rule Compliance)

#### Problem
FIDE chess rules are complex with 100+ edge cases:
- Illegal moves tracking
- Time control violations
- Pairing constraints (color alternation, opponent history)
- Title norm requirements (13 criteria!)
- Anti-cheating protocols

Arbiters need to verify compliance **in real-time** during tournaments.

#### Solution: MCP Server + AI Agent

```python
# MCP Server: Tournament Rules Context
class TournamentRulesMCPServer:
    """Exposes tournament data and FIDE rules to AI agents"""

    @mcp.tool()
    async def check_pairing_legality(
        self,
        tournament_id: int,
        round: int,
        white_player_id: int,
        black_player_id: int
    ) -> dict:
        """Check if a proposed pairing violates FIDE rules"""
        tournament = load_tournament(tournament_id)

        violations = []

        # Check 1: Already played before?
        if tournament.have_played(white_player_id, black_player_id):
            violations.append({
                "rule": "C.04.1",
                "severity": "critical",
                "description": "Players have already played each other"
            })

        # Check 2: Color imbalance?
        white_player = tournament.get_player(white_player_id)
        if white_player.consecutive_colors(Color.WHITE) >= 2:
            violations.append({
                "rule": "C.04.2",
                "severity": "high",
                "description": "White player has played white 2+ times consecutively"
            })

        # Check 3: Rating difference too large?
        rating_diff = abs(white_player.rating - black_player.rating)
        if rating_diff > 400:
            violations.append({
                "rule": "C.04.3",
                "severity": "medium",
                "description": f"Rating difference too large: {rating_diff}"
            })

        return {
            "legal": len(violations) == 0,
            "violations": violations
        }

    @mcp.tool()
    async def check_title_norm_progress(
        self,
        tournament_id: int,
        player_id: int,
        target_title: str
    ) -> dict:
        """Check if player is on track for title norm"""
        tournament = load_tournament(tournament_id)
        player = tournament.get_player(player_id)

        # Title norm has 13 criteria!
        checker = NormChecker(tournament, player, target_title)

        return {
            "on_track": checker.is_on_track(),
            "criteria_met": checker.criteria_met(),
            "criteria_failed": checker.criteria_failed(),
            "games_needed": checker.games_needed(),
            "performance_rating": checker.performance_rating(),
            "recommendations": checker.get_recommendations()
        }

    @mcp.resource("fide://rules/pairing")
    async def get_fide_pairing_rules(self) -> str:
        """Full text of FIDE pairing rules"""
        return load_fide_handbook_section("C.04")
```

#### AI Agent Usage

```python
from anthropic import Anthropic

client = Anthropic()

# Agent with MCP server access
response = client.messages.create(
    model="claude-opus-4",
    mcp_servers=["tournament-rules"],
    messages=[{
        "role": "user",
        "content": """
        I'm the arbiter for Tournament #123.
        Player Alice (ID 42) just claimed she's eligible for a WGM norm.
        Can you verify if she meets all criteria?
        """
    }]
)

# Agent automatically calls:
# 1. check_title_norm_progress(tournament_id=123, player_id=42, target_title="WGM")
# 2. Analyzes results against FIDE rules
# 3. Provides human-readable explanation

# Response:
"""
Based on my analysis of Tournament #123:

Alice (ID 42) is ON TRACK for a WGM norm, but hasn't achieved it yet.

✅ Criteria Met (8/13):
- Rating performance: 2425 (required: 2400+)
- Games played: 7 (required: 9 minimum)
- Opponents' average rating: 2385
- At least 3 titled opponents: Yes (4 GMs, 2 IMs)
- Federation diversity: 5 federations represented
...

❌ Criteria Not Met (2/13):
- Needs 2 more games (currently 7/9)
- Needs 1 more foreign opponent

⚠️ At Risk (3/13):
- If she loses next game, performance rating drops below 2400
- Color balance: She's played White 4 times, Black 3 times

🎯 Recommendation:
If Alice wins her next 2 games against players rated 2300+,
she will achieve the WGM norm.
"""
```

**Value:**
- Saves arbiter 30+ minutes of manual calculation
- Reduces human error
- Real-time feedback during tournament

---

### Use Case 2: **Anomaly Detection Agent** (Anti-Cheating)

#### Problem
Detecting suspicious patterns:
- Unusual rating gains
- Unexpected results against strong opponents
- Correlation with outside assistance
- Statistical impossibilities

#### Solution: Multi-Agent System

```python
# MCP Server: Tournament Analytics
class TournamentAnalyticsMCPServer:
    """Exposes statistical analysis tools"""

    @mcp.tool()
    async def get_player_performance_history(
        self,
        player_id: int,
        lookback_tournaments: int = 10
    ) -> dict:
        """Historical performance data"""
        tournaments = get_recent_tournaments(player_id, lookback_tournaments)

        return {
            "tournaments": [
                {
                    "id": t.id,
                    "date": t.date,
                    "rating_before": t.rating_before,
                    "rating_after": t.rating_after,
                    "rating_change": t.rating_change,
                    "performance": t.performance_rating,
                    "expected_score": t.expected_score,
                    "actual_score": t.actual_score,
                    "opponents_avg_rating": t.opponents_avg_rating
                }
                for t in tournaments
            ]
        }

    @mcp.tool()
    async def get_game_timing_patterns(
        self,
        tournament_id: int,
        player_id: int
    ) -> dict:
        """Analyze move timing (if time tracking enabled)"""
        games = get_player_games(tournament_id, player_id)

        return {
            "avg_move_time": calculate_avg_move_time(games),
            "critical_positions_time": analyze_critical_positions(games),
            "time_pressure_performance": analyze_time_pressure(games),
            "consistency_score": calculate_consistency(games)
        }

    @mcp.tool()
    async def compare_with_engine_moves(
        self,
        tournament_id: int,
        player_id: int,
        engine: str = "stockfish"
    ) -> dict:
        """Compare player moves with engine analysis"""
        # Note: Requires game notation (PGN)
        games = get_player_games_with_notation(tournament_id, player_id)

        return {
            "engine_correlation": calculate_correlation(games, engine),
            "top_move_percentage": calculate_top_moves(games, engine),
            "suspicious_games": identify_suspicious_patterns(games, engine)
        }
```

#### Agent Coordination (A2A)

```python
# Agent 1: Statistical Analyzer
statistical_agent = Agent(
    name="statistical-analyzer",
    mcp_servers=["tournament-analytics"],
    system_prompt="""
    You are a statistical analyzer for chess tournaments.
    Analyze player performance data and identify unusual patterns.
    Flag anything that deviates >2 standard deviations from expected.
    """
)

# Agent 2: Chess Domain Expert
chess_expert_agent = Agent(
    name="chess-expert",
    mcp_servers=["tournament-analytics", "tournament-rules"],
    system_prompt="""
    You are a chess expert and arbiter consultant.
    Interpret statistical anomalies in chess context.
    Consider: rating volatility, player style, tournament conditions.
    """
)

# Agent 3: Report Generator
report_agent = Agent(
    name="report-generator",
    system_prompt="""
    You generate clear, actionable reports for arbiters.
    Combine statistical analysis with chess expertise.
    Provide evidence and recommendations.
    """
)

# Orchestration
async def detect_anomalies(tournament_id: int):
    # Step 1: Statistical analysis
    stats_findings = await statistical_agent.run(
        f"Analyze all players in tournament {tournament_id} for anomalies"
    )

    # Step 2: Chess expert review (A2A)
    expert_analysis = await chess_expert_agent.run(
        f"Review these statistical findings:\n{stats_findings}\n"
        f"Which are genuinely suspicious vs. normal chess phenomena?"
    )

    # Step 3: Generate report
    report = await report_agent.run(
        f"Create an arbiter report from:\n"
        f"Statistical: {stats_findings}\n"
        f"Expert: {expert_analysis}"
    )

    return report
```

**Output Example:**

```
ANOMALY DETECTION REPORT - Tournament #123
Generated: 2025-02-01 14:30

🚨 HIGH PRIORITY FLAGS (2):

1. Player: John Doe (ID 456, Rating: 1800)

   Statistical Anomalies:
   - Performance rating: 2450 (+650 above personal rating)
   - Won 6/7 games against 2200+ opponents
   - Rating gain: +180 (expected: +20)
   - Deviation: 4.2 standard deviations above expected

   Chess Expert Analysis:
   - No prior tournaments above 2000 performance
   - Beat 2 IMs and 1 GM in this tournament
   - Previous best tournament: 2050 performance
   - Sudden improvement not explained by training history

   Engine Correlation:
   - Top engine move: 87% of moves (suspicious threshold: 80%)
   - Critical positions: 94% engine agreement
   - Move time: Highly consistent (~5 seconds per move)

   🎯 RECOMMENDATION:
   Request game review by Fair Play Committee.
   Consider additional monitoring for next round.

⚠️ MEDIUM PRIORITY FLAGS (3):
...

✅ NOTABLE PERFORMANCES (5):
(Players performing well within statistical expectations)
...
```

**Value:**
- Detects cheating before tournament ends
- Fair to legitimate strong performers
- Provides evidence for Fair Play committees

---

### Use Case 3: **Natural Language Tournament Queries**

#### Problem
Players and spectators ask questions like:
- "When is my next game?"
- "Who's leading the U18 category?"
- "What's the prize breakdown if I finish with 7 points?"
- "Has anyone ever achieved a GM norm in this tournament?"

Currently requires manual lookup or technical knowledge.

#### Solution: MCP Server + Conversational Agent

```python
# MCP Server: Tournament Data Access
class TournamentDataMCPServer:
    """Natural language access to tournament data"""

    @mcp.tool()
    async def get_player_next_pairing(
        self,
        tournament_id: int,
        player_name: str = None,
        player_id: int = None
    ) -> dict:
        """Find player's next pairing"""
        player = find_player(tournament_id, player_name, player_id)
        tournament = load_tournament(tournament_id)

        next_round = tournament.current_round + 1
        pairing = tournament.get_pairing(player.id, next_round)

        if not pairing:
            return {"message": "Pairings not generated yet"}

        return {
            "round": next_round,
            "board": pairing.board,
            "color": pairing.color,
            "opponent": pairing.opponent.name,
            "opponent_rating": pairing.opponent.rating,
            "scheduled_time": tournament.round_time(next_round),
            "location": pairing.location
        }

    @mcp.tool()
    async def get_standings(
        self,
        tournament_id: int,
        category: str = None,
        min_points: float = None,
        max_points: float = None
    ) -> list[dict]:
        """Get tournament standings with filters"""
        tournament = load_tournament(tournament_id)
        standings = tournament.get_standings()

        # Apply filters
        if category:
            standings = [s for s in standings if s.category == category]
        if min_points:
            standings = [s for s in standings if s.points >= min_points]
        if max_points:
            standings = [s for s in standings if s.points <= max_points]

        return [
            {
                "rank": s.rank,
                "name": s.name,
                "rating": s.rating,
                "points": s.points,
                "buchholz": s.buchholz,
                "performance": s.performance_rating
            }
            for s in standings[:20]  # Top 20
        ]

    @mcp.tool()
    async def calculate_prize_scenarios(
        self,
        tournament_id: int,
        player_id: int,
        final_points: float
    ) -> dict:
        """Calculate prize money for different final scores"""
        tournament = load_tournament(tournament_id)
        player = tournament.get_player(player_id)

        # Simulate final standings
        scenarios = tournament.simulate_prize_scenarios(
            player_id=player_id,
            final_points=final_points
        )

        return {
            "player": player.name,
            "current_points": player.points,
            "final_points": final_points,
            "estimated_rank": scenarios['rank'],
            "estimated_prize": scenarios['prize'],
            "tied_with": scenarios['tied_players'],
            "split_prize": scenarios['split_prize']
        }
```

#### Conversational Agent Interface

```python
# Embed in web interface
@get('/tournament/{id}/ask')
async def ask_tournament_question(
    id: int,
    question: str,
    mcp_client: MCPClient
) -> HTMXTemplate:
    """Natural language tournament queries"""

    response = await anthropic.messages.create(
        model="claude-3-5-sonnet-20250219",
        mcp_servers=["tournament-data"],
        messages=[{
            "role": "user",
            "content": f"Tournament #{id}: {question}"
        }]
    )

    return HTMXTemplate(
        template_name='tournament_answer.html',
        context={
            'question': question,
            'answer': response.content[0].text
        }
    )
```

**User Experience:**

```
Player: "When is my next game?"
Agent: "Your next game is Round 7, Board 12. You'll play White against
        Sarah Johnson (Rating 2150). The round starts at 15:00 in Hall A."

Player: "What are the top 5 players in the U18 category?"
Agent: "U18 Category Leaders:
        1. Alice Smith - 7.0 points (Performance: 2450)
        2. Bob Chen - 6.5 points (Performance: 2380)
        3. Charlie Davis - 6.5 points (Performance: 2340)
        4. Diana Martinez - 6.0 points (Performance: 2310)
        5. Eric Johnson - 6.0 points (Performance: 2280)"

Player: "If I win my last 2 games, will I win prize money?"
Agent: "If you win your last 2 games, you'll finish with 7.0 points.

        Estimated outcome:
        - Rank: 8th-12th place (depends on other results)
        - Prize category: 6.5-7.5 points
        - Prize pool: €500 (shared among ~10 players)
        - Your share: ~€50

        To win the top prize (€1000), you'd need 8.5+ points,
        which is no longer possible."
```

**Value:**
- Reduces arbiter workload (fewer repetitive questions)
- Better player experience
- 24/7 availability

---

### Use Case 4: **Multi-Tournament Coordination** (A2A)

#### Problem
Regional/national federations run multiple tournaments simultaneously:
- Need to coordinate schedules
- Avoid player conflicts (playing in 2 tournaments)
- Share resources (arbiters, equipment)
- Aggregate statistics

#### Solution: Agent Network

```python
# Each tournament has an agent
class TournamentAgent:
    def __init__(self, tournament_id: int):
        self.tournament_id = tournament_id
        self.mcp_server = TournamentDataMCPServer(tournament_id)

    async def check_player_availability(
        self,
        player_id: int,
        round_time: datetime
    ) -> bool:
        """Check if player is available (not double-booked)"""
        # Query other tournament agents (A2A)
        for other_tournament in federation.active_tournaments():
            if other_tournament.id == self.tournament_id:
                continue

            # A2A: Ask other tournament's agent
            is_playing = await other_tournament.agent.is_player_scheduled(
                player_id=player_id,
                time=round_time
            )

            if is_playing:
                return False

        return True

    async def coordinate_arbiter_schedule(
        self,
        required_arbiters: int,
        time_slot: datetime
    ) -> list[Arbiter]:
        """Find available arbiters across tournaments"""
        available_arbiters = []

        # A2A: Query federation arbiter pool
        all_arbiters = await federation_agent.get_all_arbiters()

        for arbiter in all_arbiters:
            # Check conflicts across all tournaments
            conflicts = await self.check_arbiter_conflicts(arbiter, time_slot)
            if not conflicts:
                available_arbiters.append(arbiter)

        return available_arbiters[:required_arbiters]
```

#### Federation Coordinator Agent

```python
class FederationCoordinatorAgent:
    """Coordinates multiple tournaments"""

    async def optimize_tournament_schedule(
        self,
        tournaments: list[Tournament],
        constraints: dict
    ) -> dict:
        """Find optimal schedule across tournaments"""

        # Gather data from all tournament agents (A2A)
        tournament_data = await asyncio.gather(*[
            t.agent.get_schedule_requirements()
            for t in tournaments
        ])

        # Run optimization
        schedule = self.schedule_optimizer.optimize(
            tournaments=tournament_data,
            constraints=constraints
        )

        # Validate with each tournament agent
        conflicts = []
        for tournament in tournaments:
            validation = await tournament.agent.validate_schedule(schedule)
            if not validation.valid:
                conflicts.append(validation.conflicts)

        return {
            "schedule": schedule,
            "conflicts": conflicts,
            "recommendations": self.generate_recommendations(conflicts)
        }
```

**Value:**
- Automated multi-tournament coordination
- Prevents double-booking players/arbiters
- Optimizes resource allocation

---

### Use Case 5: **Intelligent Pairing Suggestions**

#### Problem
Swiss pairing has multiple valid solutions:
- Algorithm finds *a* valid pairing
- But not necessarily the *best* pairing

Trade-offs:
- Rating balance vs. color balance
- Entertainment value (exciting matchups)
- Title norm opportunities
- Travel distance (for in-person)

#### Solution: AI Pairing Advisor

```python
# MCP Server: Pairing Analysis
class PairingAnalysisMCPServer:
    """Analyze pairing quality and suggest improvements"""

    @mcp.tool()
    async def evaluate_pairing_quality(
        self,
        tournament_id: int,
        round: int,
        proposed_pairings: list[dict]
    ) -> dict:
        """Evaluate quality of proposed pairings"""
        tournament = load_tournament(tournament_id)

        metrics = {
            "rating_balance": self.calc_rating_balance(proposed_pairings),
            "color_fairness": self.calc_color_fairness(proposed_pairings),
            "title_norm_impact": self.calc_title_norm_impact(
                tournament, proposed_pairings
            ),
            "entertainment_value": self.calc_entertainment_value(
                proposed_pairings
            ),
            "illegal_moves_risk": self.calc_illegal_moves_risk(
                tournament, proposed_pairings
            )
        }

        return {
            "overall_score": self.aggregate_score(metrics),
            "metrics": metrics,
            "strengths": self.identify_strengths(metrics),
            "weaknesses": self.identify_weaknesses(metrics)
        }

    @mcp.tool()
    async def suggest_pairing_improvements(
        self,
        tournament_id: int,
        round: int,
        current_pairings: list[dict]
    ) -> list[dict]:
        """Suggest alternative pairings"""
        # Generate alternative valid pairings
        alternatives = self.pairing_generator.generate_alternatives(
            tournament_id=tournament_id,
            round=round,
            base_pairings=current_pairings,
            num_alternatives=5
        )

        # Evaluate each
        evaluations = [
            await self.evaluate_pairing_quality(tournament_id, round, alt)
            for alt in alternatives
        ]

        # Rank by score
        ranked = sorted(
            zip(alternatives, evaluations),
            key=lambda x: x[1]['overall_score'],
            reverse=True
        )

        return [
            {
                "pairings": alt,
                "score": eval['overall_score'],
                "improvements": self.compare_with_current(
                    current_pairings, alt, eval
                )
            }
            for alt, eval in ranked
        ]
```

#### Agent Usage

```python
# Pairing Advisor Agent
pairing_advisor = Agent(
    name="pairing-advisor",
    mcp_servers=["pairing-analysis", "tournament-data"],
    system_prompt="""
    You are an expert chess pairing advisor.
    Analyze pairing quality and suggest improvements that balance:
    - FIDE compliance (required)
    - Rating fairness
    - Color balance
    - Entertainment value
    - Title norm opportunities
    """
)

# Usage in controller
@post('/tournament/{id}/round/{round}/generate')
async def generate_round(id: int, round: int):
    tournament = load_tournament(id)

    # Generate standard pairings
    pairings = tournament.generate_round(round)

    # Get AI suggestions (optional, user can disable)
    if tournament.settings.enable_ai_advisor:
        suggestions = await pairing_advisor.run(
            f"""
            Tournament {id}, Round {round}

            I've generated these pairings using Dutch System.
            Can you suggest improvements?

            Current pairings:
            {format_pairings(pairings)}
            """
        )

        # Show to arbiter
        return HTMXTemplate(
            'pairings_with_suggestions.html',
            context={
                'pairings': pairings,
                'ai_suggestions': suggestions
            }
        )
```

**Arbiter sees:**

```
ROUND 5 PAIRINGS - Generated with Dutch System

Current Pairings (85/100 quality score):
✅ All legal
✅ Good rating balance (avg diff: 45 points)
⚠️ Color fairness: 65% (below optimal)
⚠️ 3 players will get same color 3x in a row

AI SUGGESTIONS:

Alternative Pairing #1 (92/100 score):
Swap: Board 12 (White ↔ Black)
      Board 15 (White ↔ Black)

Improvements:
✅ Fixes all 3-in-a-row color issues
✅ Maintains rating balance
✅ Creates 2 exciting 2400+ matchups on top boards
⚠️ Slightly increases rating gap on Board 18 (+60 points)

[Accept Suggestion] [View Details] [Keep Current]
```

**Value:**
- Better pairing quality
- Learns from arbiter preferences
- Catches edge cases

---

## 🏗️ Implementation Architecture

### MCP Server Layer

```python
# src/mcp/servers/tournament_server.py
from mcp.server import Server
from mcp.types import Tool, Resource

class SharlyChessMCPServer(Server):
    """Main MCP server exposing tournament data and tools"""

    def __init__(self):
        super().__init__(name="sharly-chess")
        self.register_tools()
        self.register_resources()

    def register_tools(self):
        self.add_tool(
            Tool(
                name="get_tournament_standings",
                description="Get current standings for a tournament",
                input_schema={
                    "type": "object",
                    "properties": {
                        "tournament_id": {"type": "integer"},
                        "category": {"type": "string", "optional": True}
                    }
                }
            ),
            handler=self.handle_get_standings
        )

        # Add 20+ more tools...

    def register_resources(self):
        # Expose tournament databases as resources
        self.add_resource(
            Resource(
                uri="tournament://{tournament_id}/pairings",
                name="Tournament Pairings",
                description="All pairings for a tournament"
            )
        )

        # Add more resources...

    async def handle_get_standings(self, tournament_id: int, category: str = None):
        tournament = load_tournament(tournament_id)
        standings = tournament.get_standings()

        if category:
            standings = [s for s in standings if s.category == category]

        return {"standings": [s.to_dict() for s in standings]}
```

### Agent Integration Layer

```python
# src/ai/agents/base_agent.py
from anthropic import Anthropic

class TournamentAgent:
    """Base class for tournament AI agents"""

    def __init__(self, name: str, mcp_servers: list[str]):
        self.name = name
        self.mcp_servers = mcp_servers
        self.client = Anthropic()

    async def run(self, prompt: str, context: dict = None) -> str:
        """Execute agent task"""
        messages = [{"role": "user", "content": prompt}]

        if context:
            messages.insert(0, {
                "role": "user",
                "content": f"Context: {json.dumps(context)}"
            })

        response = await self.client.messages.create(
            model="claude-opus-4",
            mcp_servers=self.mcp_servers,
            messages=messages
        )

        return response.content[0].text

# src/ai/agents/arbiter_assistant.py
class ArbiterAssistantAgent(TournamentAgent):
    """Helps arbiters with rule compliance"""

    def __init__(self):
        super().__init__(
            name="arbiter-assistant",
            mcp_servers=["sharly-chess", "fide-rules"]
        )

    async def check_title_norm(
        self,
        tournament_id: int,
        player_id: int,
        target_title: str
    ) -> dict:
        """Check title norm eligibility"""
        prompt = f"""
        Check if player {player_id} in tournament {tournament_id}
        is eligible for a {target_title} norm.

        Use the check_title_norm_progress tool to get data,
        then analyze against FIDE rules.
        """

        result = await self.run(prompt)
        return {"analysis": result}
```

### Plugin Integration

```python
# src/plugins/ai_agents/__init__.py
from plugins.utils import Plugin

class AIAgentsPlugin(Plugin):
    """Plugin that adds AI agents to Sharly Chess"""

    @staticmethod
    def static_id() -> str:
        return 'ai_agents'

    @staticmethod
    def static_name() -> str:
        return 'AI Agents'

    def on_enable(self, event_id: str):
        """Start MCP server when plugin enabled"""
        self.mcp_server = SharlyChessMCPServer()
        self.mcp_server.start()

        # Initialize agents
        self.arbiter_assistant = ArbiterAssistantAgent()
        self.anomaly_detector = AnomalyDetectorAgent()
        self.query_agent = QueryAgent()

    def on_disable(self, event_id: str):
        """Stop MCP server"""
        self.mcp_server.stop()

# Register plugin
@hookimpl
def register_plugins():
    return [AIAgentsPlugin()]
```

### Web UI Integration

```html
<!-- src/web/templates/admin/ai_assistant.html -->
<div id="ai-assistant" class="card">
    <div class="card-header">
        <h5>🤖 AI Arbiter Assistant</h5>
    </div>
    <div class="card-body">
        <form hx-post="/admin/tournament/{{ tournament.id }}/ai/ask"
              hx-target="#ai-response">
            <textarea name="question"
                      placeholder="Ask me anything about the tournament..."
                      class="form-control"></textarea>
            <button type="submit" class="btn btn-primary mt-2">
                Ask AI
            </button>
        </form>

        <div id="ai-response" class="mt-3"></div>
    </div>
</div>

<!-- Quick actions -->
<div class="ai-quick-actions">
    <button hx-post="/admin/tournament/{{ tournament.id }}/ai/check-anomalies"
            hx-target="#anomaly-report">
        🔍 Check for Anomalies
    </button>

    <button hx-post="/admin/tournament/{{ tournament.id }}/ai/analyze-pairings"
            hx-target="#pairing-analysis">
        🎯 Analyze Pairings
    </button>

    <button hx-post="/admin/tournament/{{ tournament.id }}/ai/title-norms"
            hx-target="#norm-report">
        🏆 Check Title Norms
    </button>
</div>
```

---

## 📊 Feasibility Assessment

### Technical Feasibility: ✅ HIGH

| Aspect | Status | Notes |
|--------|--------|-------|
| **MCP Integration** | ✅ Straightforward | Python has excellent MCP support |
| **Data Access** | ✅ Easy | SQLite makes data exposure simple |
| **Agent SDK** | ✅ Compatible | Anthropic SDK works with async Python |
| **Plugin System** | ✅ Perfect fit | Existing hook system ready for agents |
| **Web Integration** | ✅ Simple | HTMX can call agent endpoints |

### Implementation Complexity

**Phase 1: MCP Server (2-3 weeks)**
- Expose basic tournament data
- 10-15 core tools
- Resource endpoints

**Phase 2: Single Agent (1-2 weeks)**
- Arbiter assistant agent
- Basic rule compliance checking

**Phase 3: Multiple Agents (3-4 weeks)**
- Anomaly detection
- Query agent
- Pairing advisor

**Phase 4: A2A Communication (2-3 weeks)**
- Multi-tournament coordination
- Agent orchestration

**Total:** ~8-12 weeks for full implementation

### Cost Considerations

**API Costs:**
- Claude Opus: $15 / 1M input tokens, $75 / 1M output tokens
- Typical query: ~5K input + 2K output = $0.225
- 1000 queries/month = $225/month
- Can use Sonnet for cheaper queries ($3/$15 per 1M tokens)

**Strategies to reduce cost:**
1. Cache common queries
2. Use Sonnet for simple tasks
3. Batch processing
4. Local model fallback for basic queries

### User Value: ✅ HIGH

| Use Case | Time Saved | Error Reduction | User Impact |
|----------|------------|-----------------|-------------|
| Rule compliance | 30 min/incident | 50% | High |
| Anomaly detection | 2 hours/tournament | 80% | Critical |
| Player queries | 5 min/query × 100 | 0% | Medium |
| Pairing optimization | 15 min/round | 20% | Medium |
| Title norm checking | 45 min/check | 90% | High |

---

## 🎯 Recommended Implementation Roadmap

### MVP (4 weeks)

**Deliverables:**
1. MCP server with 5 core tools:
   - get_standings
   - get_pairings
   - check_pairing_legality
   - get_player_stats
   - check_title_norm

2. Single AI agent: Arbiter Assistant
   - Rule compliance checking
   - Title norm verification
   - Natural language queries

3. Web UI integration:
   - "Ask AI" text box
   - Quick action buttons

**Success Criteria:**
- Arbiter can ask questions in natural language
- Title norm checks are accurate (95%+)
- Response time < 5 seconds

### Phase 2 (6 weeks)

**Deliverables:**
1. Anomaly detection agent
   - Statistical analysis
   - Automated alerts
   - Weekly reports

2. Enhanced MCP server:
   - 15+ tools
   - Historical data access
   - Real-time updates via WebSocket

3. Agent memory/context:
   - Remember tournament context
   - Learn from arbiter corrections

### Phase 3 (8 weeks)

**Deliverables:**
1. Multi-agent system:
   - Pairing advisor
   - Query agent
   - Coordination agents

2. A2A communication:
   - Multi-tournament coordination
   - Federation-level agents

3. Advanced features:
   - Proactive suggestions
   - Predictive analytics
   - Custom agent training

---

## 🚀 Getting Started

### 1. Enable MCP Server

```python
# pyproject.toml
dependencies = [
    # ... existing deps ...
    "anthropic >= 0.40.0",  # MCP support
    "mcp >= 0.9.0",         # MCP SDK
]
```

### 2. Create First MCP Server

```python
# src/mcp/servers/basic.py
from mcp.server import Server
from mcp.types import Tool

server = Server("sharly-chess-basic")

@server.tool()
async def get_tournament_info(tournament_id: int) -> dict:
    """Get basic tournament information"""
    tournament = load_tournament(tournament_id)
    return {
        "name": tournament.name,
        "rounds": tournament.rounds,
        "players": tournament.player_count,
        "current_round": tournament.current_round
    }

if __name__ == "__main__":
    server.run()
```

### 3. Test with Claude

```python
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-3-5-sonnet-20250219",
    mcp_servers=["sharly-chess-basic"],
    messages=[{
        "role": "user",
        "content": "What's the status of tournament 123?"
    }]
)

print(response.content[0].text)
```

### 4. Integrate into Web App

```python
# src/web/controllers/ai_controller.py
@post('/admin/tournament/{id}/ai/ask')
async def ask_ai(id: int, question: str):
    response = await anthropic.messages.create(
        model="claude-3-5-sonnet-20250219",
        mcp_servers=["sharly-chess"],
        messages=[{
            "role": "user",
            "content": f"Tournament {id}: {question}"
        }]
    )

    return HTMXTemplate(
        'ai_response.html',
        context={'answer': response.content[0].text}
    )
```

---

## 💡 Conclusion

**Feasibility: HIGH ✅**

The Sharly Chess architecture is **well-suited** for MCP and AI agent integration:
- Clean plugin system
- SQLite data is easy to expose
- Async Python compatible
- HTMX can call agent endpoints

**Highest Value Use Cases:**
1. **Rule compliance checking** - Saves hours, reduces errors
2. **Anomaly detection** - Critical for fair play
3. **Natural language queries** - Better UX

**Recommended Start:**
Build **Arbiter Assistant Agent** as MVP:
- Immediate value to arbiters
- Clear use case
- Manageable scope
- Proves the concept

**ROI:**
- Development: ~4 weeks for MVP
- Cost: ~$200-500/month (API costs)
- Time saved: 10+ hours/tournament
- Error reduction: 50-80%

The integration would position Sharly Chess as the **first AI-powered chess tournament management system**, differentiating it from competitors while providing genuine value to arbiters.
