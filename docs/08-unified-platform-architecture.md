# OpenClaw: Unified Platform Architecture
## Self-Improving Multi-Client AI Operating System

This is the authoritative architecture document for the OpenClaw platform. It consolidates the original autonomous agent platform design (formerly doc 08) and the expert-reviewed enhanced design (formerly doc 09) into a single cohesive reference. It also introduces the **AI Agency Platform** — the meta-use case where this system is used to design and deliver custom AI systems for clients.

> **Supersedes:** `08-autonomous-agent-platform.md` and `09-enhanced-system-design.md`

---

## Table of Contents

1. [What This System Is](#1-what-this-system-is)
2. [System Relationship: NanoClaw & OpenClaw](#2-system-relationship-nanoclaw--openclaw)
3. [Is Anyone Doing This? Industry Evidence](#3-is-anyone-doing-this-industry-evidence)
4. [End-to-End Walkthrough](#4-end-to-end-walkthrough)
5. [Full System Architecture](#5-full-system-architecture)
6. [Layer 1: Interface & Intake](#6-layer-1-interface--intake)
7. [Layer 2: Task Intelligence & LLM Routing](#7-layer-2-task-intelligence--llm-routing)
8. [Layer 3: Orchestration & Execution](#8-layer-3-orchestration--execution)
9. [Layer 4: Hierarchical Context](#9-layer-4-hierarchical-context)
10. [Layer 5: Memory & Knowledge](#10-layer-5-memory--knowledge)
11. [Layer 6: Self-Improvement Engine](#11-layer-6-self-improvement-engine)
12. [Layer 7: Observability & Control](#12-layer-7-observability--control)
13. [Layer 8: Safety & Guardrails](#13-layer-8-safety--guardrails)
14. [AI Agency Platform: Building AI Systems for Clients](#14-ai-agency-platform-building-ai-systems-for-clients)
15. [MCP-First Architecture](#15-mcp-first-architecture)
16. [Complete Technology Stack](#16-complete-technology-stack)
17. [Data Flow: Request to Completion](#17-data-flow-request-to-completion)
18. [Self-Improvement Loop Detail](#18-self-improvement-loop-detail)
19. [Human Oversight Patterns](#19-human-oversight-patterns)
20. [Deployment Architecture](#20-deployment-architecture)
21. [Implementation Roadmap](#21-implementation-roadmap)
22. [What Makes This Different](#22-what-makes-this-different)

---

## 1. What This System Is

OpenClaw is a **Personal AI Operating System** — a backend platform that:

| Capability | What It Means |
|------------|---------------|
| **Conversational intake** | Describe tasks in Discord; no forms, no dashboards required |
| **Hierarchical context** | Automatically applies the right skills + client context per task |
| **Parallel autonomous execution** | Multiple specialist agents work simultaneously on complex tasks |
| **Strict multi-client isolation** | Client A's data is architecturally invisible to Client B's agents |
| **Self-improving** | The system gets better at your work patterns over time, without manual updates |
| **Human oversight** | You can monitor, pause, approve, reject, or redirect any task at any time |
| **Observable** | Full trace of every decision, tool call, and cost is available on demand |
| **AI agency capable** | Can design and deliver complete AI systems for clients using its own accumulated research |

This is not a chatbot. It is a **team of AI agents** that works like a skilled development team: you assign work, it executes using established standards, asks when uncertain, and continuously raises the quality bar.

### Cost Architecture Philosophy

The system uses two LLM cost models intentionally:

| Mode | Technology | When Used | Why |
|------|-----------|-----------|-----|
| **Subscription flat-rate** | Claude Code Agent SDK | Complex tasks: planning, code gen, research, evaluation, optimization | ~$200/month/seat regardless of token volume |
| **Pay-per-token API** | Anthropic API (Haiku) | High-volume cheap ops: parsing, classification, embedding, extraction | Haiku at ~$0.001/1K tokens keeps these negligible |

This hybrid approach keeps costs predictable for the work that matters while minimizing spend on routine operations.

---

## 2. System Relationship: NanoClaw & OpenClaw

**NanoClaw is not replaced. OpenClaw is the brain behind it.**

```
Discord User
     │
     ▼
┌─────────────────────────────────────────┐
│              NanoClaw                   │
│         (Discord Bot — existing)        │
│                                         │
│  • Handles simple commands instantly    │
│  • Manages Discord UX (threads, etc.)  │
│  • Contains the Task Bridge component  │
└─────────────────┬───────────────────────┘
                  │ complex tasks only
                  ▼
┌─────────────────────────────────────────┐
│              OpenClaw                   │
│     (AI Backend — this system)          │
│                                         │
│  • Hierarchical context engine          │
│  • Parallel agent execution             │
│  • Memory & self-improvement            │
│  • MCP server ecosystem                 │
└─────────────────────────────────────────┘
```

### The Task Bridge

The Task Bridge is a component inside NanoClaw that classifies every incoming message and routes it appropriately:

```python
class OpenClawBridge:
    """
    Lives inside NanoClaw. Classifies messages and delegates
    complex tasks to the OpenClaw backend via REST API.
    """
    async def handle_message(self, discord_message, group_context):
        complexity = await self.task_classifier.assess(discord_message.content)

        if complexity.is_simple:
            return None  # NanoClaw handles it — no change to existing behavior

        # Complex task: delegate to OpenClaw
        task_id = await self.openclaw_client.submit(
            content=discord_message.content,
            author_id=str(discord_message.author.id),
            group_config=group_context,
            discord_thread_id=discord_message.channel.id,
            correlation_id=str(discord_message.id)  # Trace root
        )
        return task_id  # NanoClaw posts "⏳ Task submitted..." and monitors
```

### What Stays Simple (NanoClaw)

- One-shot factual questions
- Discord server management commands
- Status checks, balance queries, simple lookups
- Commands that resolve in < 3 seconds with a single LLM call

### What Goes to OpenClaw

- Multi-step tasks ("build a feature", "research and summarize", "review all PRs")
- Any task requiring client context (tagged to a specific client)
- Tasks requiring tool use, code execution, or external API calls
- Tasks that benefit from parallel agent execution

---

## 3. Is Anyone Doing This? Industry Evidence

### Closest Existing Systems

**Devin (Cognition AI)** — https://cognition.ai
- The most capable autonomous software agent commercially available
- Slack integration for task intake; long-horizon multi-step execution
- **Missing:** no multi-client isolation, no hierarchical context, no self-improvement from your patterns

**OpenHands (formerly OpenDevin)** — https://github.com/All-Hands-AI/OpenHands
- Open-source equivalent of Devin (20k+ stars)
- **Missing:** single-user, single-project; no persistent skills architecture

**Sweep** — https://github.com/sweepai/sweep
- GitHub bot: natural language issue → PR
- Has a `sweep.yaml` rules file (closest thing to a global skills layer)
- **Missing:** no multi-client, no self-improvement, code-only

**Cursor Background Agents** — https://cursor.com
- Project-level context via `.cursorrules`
- **Missing:** single project, no client isolation, no Discord UX, no memory accumulation

### The Gap

No publicly available system combines **all five** of:
1. Conversational intake (Discord/Slack)
2. Multi-client hierarchical context with strict isolation
3. Parallel multi-agent execution
4. True self-improvement from usage patterns
5. Full human observability and control

This is the system you're building.

---

## 4. End-to-End Walkthrough

```
You type in Discord:
"For ACME Corp, build an endpoint to export user data as CSV.
Needs to be GDPR compliant. Low priority."

════════════════════════════════════════════════

NanoClaw Task Bridge detects complexity → routes to OpenClaw

Bot instantly replies:
"⏳ Understood. Creating task for ACME Corp.
   Context loaded: FastAPI + PostgreSQL stack, GDPR profile active
   Assigning to: Code Agent + Security Review Agent
   Thread created → #task-acme-csv-export"

════════════════════════════════════════════════

In the thread, 4 minutes later:
"✅ Task Complete — ACME CSV Export

   What was built:
   • GET /api/v1/users/export endpoint (FastAPI)
   • Exports only requesting user's own data (GDPR Article 20)
   • Rate limited: 1 export per user per hour
   • Audit log entry on every export
   • Unit tests: 96% coverage
   • PR opened: github.com/acme/backend/pull/312

   Applied from your standards:
   • Input validation (Global: security best practices)
   • Parameterized queries (Global: SQL safety)
   • FastAPI dependency injection pattern (ACME convention)
   • GDPR data minimization (ACME: compliance profile)

   💡 System noticed: GDPR export pattern could be added as a reusable
   skill. Want me to propose it for review? [Yes] [No]"

════════════════════════════════════════════════

You react with ✅. The system:
   • Drafts a "GDPR Data Export" domain skill
   • Adds it to staging (10% of future tasks)
   • Runs eval for 1 week
   • Reports: "GDPR skill improved compliance review pass rate by 23%"
   • Promotes to global domain skill
   • All future agents across all clients now benefit from this
```

---

## 5. Full System Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                     INTERFACE LAYER                                    │
│   NanoClaw (Discord Bot) │  Task Bridge  │  REST API  │  Web UI       │
└───────────────────────────────┬───────────────────────────────────────┘
                                │ complex tasks
┌───────────────────────────────▼───────────────────────────────────────┐
│                  TASK INTELLIGENCE LAYER                               │
│  LLM Router • Task Parser • Intent Classifier • Client Resolver       │
│  Ambiguity Detector • Priority Scorer • Tag Extractor                 │
│  Claude Code SDK (complex) ←→ Direct API/Haiku (parsing/classify)    │
│  Task State Machine (PostgreSQL + Celery + Redis)                     │
└───────────────────────────────┬───────────────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────────┐
│                   ORCHESTRATION LAYER                                  │
│  Claude Code Agent SDK (Supervisor)                                    │
│  • Task decomposition • Agent routing • Result synthesis              │
│  • HITL interrupt nodes • Checkpoint/resume                           │
│              │ asyncio WorkerPool (replaces Ray)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Code Agents  │  │Research Agts │  │Review Agents │               │
│  │ (async pool) │  │ (async pool) │  │ (async pool) │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
└───────────────────────────────┬───────────────────────────────────────┘
                                │
         ┌──────────────────────┼───────────────────────┐
         │                      │                        │
┌────────▼────────┐   ┌─────────▼──────┐   ┌───────────▼────────┐
│  MCP CONTEXT    │   │  MCP MEMORY    │   │  MCP SELF-IMPROVE  │
│  SERVER         │   │  SERVER        │   │  ENGINE            │
│  Skills Registry│   │  pgvector      │   │  DSPy + RAGAS      │
│  Client Ctx     │   │  Graphiti      │   │  Modal (serverless) │
│  4-Tier Model   │   │  PostgreSQL    │   │  Prefect jobs      │
└─────────────────┘   └────────────────┘   └────────────────────┘
         │                      │                        │
         └──────────────────────┼───────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────────┐
│          OBSERVABILITY + SAFETY + COST CONTROL LAYER                  │
│  Langfuse • Prometheus • Grafana • Correlation IDs (end-to-end)       │
│  Layered Guardrails (Presidio→LLM Guard→Llama Guard 3→Custom)        │
│  E2B Sandbox • Audit Logs • HITL Gates • Per-Client Budget Manager   │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 6. Layer 1: Interface & Intake

### NanoClaw Discord Bot

The Discord bot (NanoClaw) remains the primary human interaction surface. The Task Bridge component routes complex work to OpenClaw.

```python
# discord_bot.py — NanoClaw with Task Bridge integration
import discord
from discord.ext import commands

bot = commands.Bot(command_prefix="/")
bridge = OpenClawBridge(openclaw_url=OPENCLAW_API_URL)

@bot.event
async def on_message(message):
    if message.author.bot:
        return
    if bot.user.mentioned_in(message) or is_task_channel(message.channel):
        task_id = await bridge.handle_message(message, get_group_config(message))
        if task_id:
            thread = await message.create_thread(name=f"Task: {await get_short_title(message.content)}")
            await thread.send(f"⏳ Working on it... (task `{task_id}`)")

@bot.slash_command(name="status")
async def status(ctx, task_id: str = None):
    if task_id:
        info = await bridge.get_task_status(task_id)
        await ctx.respond(format_task_status(info))
    else:
        active = await bridge.get_active_tasks(user=ctx.author.id)
        await ctx.respond(format_active_tasks(active))

@bot.slash_command(name="pause")
async def pause(ctx, task_id: str):
    await bridge.pause_task(task_id)
    await ctx.respond(f"⏸️ Task {task_id} paused. Use /resume to continue.")

@bot.slash_command(name="approve")
async def approve(ctx, task_id: str):
    await bridge.approve_hitl(task_id, approver=ctx.author.id)
    await ctx.respond(f"✅ Approved. Resuming task {task_id}.")
```

### Discord Channel Structure

```
Discord Server: Virtek Labs AI
├── #task-intake        ← Drop tasks here (or @mention the bot anywhere)
├── #ai-alerts          ← Anomalies, HITL requests, budget warnings
├── #daily-digest       ← Automated morning summary
├── #skill-proposals    ← New/updated skills awaiting approval
└── Threads (auto)      ← #task-acme-csv-export, #task-globex-api-v2 ...
```

### Discord UX Patterns

| Pattern | How It Works |
|---------|-------------|
| **Thread per task** | Every OpenClaw task gets its own Discord thread for clean updates |
| **Reaction-based approval** | ✅ = approve, ❌ = reject, ⏸️ = pause (no commands needed) |
| **Progress updates** | Agent posts step-complete messages in thread as work progresses |
| **Mention-based HITL** | `@ryan` tag in thread when human decision needed |
| **Daily digest** | Morning summary of overnight tasks, cost, quality scores |
| **Alert channel** | Separate #ai-alerts for anomalies, budget alerts, errors |

---

## 7. Layer 2: Task Intelligence & LLM Routing

### LLM Router — Subscription vs. API

The router decides which LLM execution mode to use before any work begins:

```python
class LLMRouter:
    """
    Routes tasks to the right LLM mode based on task type.
    Subscription (Claude Code SDK): flat-rate, no token anxiety.
    Direct API (Haiku): cheap per-call for high-volume simple operations.
    """
    CLAUDE_CODE_SDK_TASKS = {
        "planning", "code_generation", "research",
        "evaluation", "optimization", "architecture_design",
        "security_review", "complex_analysis"
    }
    DIRECT_API_TASKS = {
        "task_parsing", "embedding", "classification",
        "simple_extraction", "intent_detection", "pii_check"
    }

    async def route(self, task_type: str) -> LLMExecutor:
        if task_type in self.CLAUDE_CODE_SDK_TASKS:
            return ClaudeCodeSDKExecutor()   # Flat-rate subscription
        elif task_type in self.DIRECT_API_TASKS:
            return HaikuAPIExecutor()         # Pay-per-token, cheap
        else:
            return ClaudeCodeSDKExecutor()    # Default to subscription for unknowns
```

### Task Parser

```python
from pydantic import BaseModel
from typing import Optional

class ParsedTask(BaseModel):
    # What
    intent: str              # 'build_feature', 'fix_bug', 'research', 'review', 'analyze'
    description: str         # cleaned task description
    short_title: str         # 5-word summary for thread naming
    task_type: str           # maps to LLMRouter — drives cost decision

    # Who/Where
    client_id: Optional[str] # 'client-acme' — None = internal task
    project_id: Optional[str]

    # How
    tags: list[str]                    # ['python', 'fastapi', 'gdpr']
    agent_types_needed: list[str]      # ['code', 'security-review', 'test']
    requires_human_approval: bool

    # Priority & Complexity
    priority: str              # 'low', 'medium', 'high', 'urgent'
    estimated_complexity: str  # 'trivial', 'small', 'medium', 'large'

    # Ambiguity
    clarifying_questions: list[str]
    confidence: float          # 0-1

    # Cost control
    estimated_cost_usd: float  # Pre-flight estimate
    idempotency_key: str       # Prevents double-execution on retry

async def parse_task(message: str, author_id: str) -> ParsedTask:
    """Uses Haiku (direct API) for fast, cheap task parsing."""
    known_clients = await db.get_client_names()
    past_tasks = await memory.search(message, user_id=author_id, limit=3)

    result = await haiku_client.structured_output(
        TASK_PARSER_PROMPT.format(
            message=message,
            known_clients=known_clients,
            past_context=past_tasks
        ),
        output_schema=ParsedTask
    )

    # Budget pre-check before queuing
    if result.client_id:
        await budget_manager.check_or_raise(result.client_id, result.estimated_cost_usd)

    if result.confidence < 0.7 or result.clarifying_questions:
        return await ask_for_clarification(result, author_id)

    return result
```

### Task State Machine

```
RECEIVED ──► PARSING ──► PLANNING ──► EXECUTING ──► REVIEWING
                                                          │
                                        ┌────────────────▼
                                        AWAITING_APPROVAL
                                              │ approve    │ reject
                                        COMPLETED      REVISED ──► EXECUTING

(Any state → PAUSED via /pause)
(FAILED → Discord alert + Celery retry with exponential backoff)
(COMPLETED → async: evaluate, extract learnings, update memory)
```

---

## 8. Layer 3: Orchestration & Execution

### Claude Code Agent SDK as Supervisor

The supervisor uses the Claude Code Agent SDK — flat-rate subscription billing, not per-token:

```python
from claude_agent_sdk import ClaudeAgent, AgentSession
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver

class TaskState(TypedDict):
    task: ParsedTask
    context: TaskContext        # Loaded hierarchical context
    plan: list[SubTask]         # Decomposed steps
    results: dict[str, Any]    # Results from each sub-agent
    awaiting_human: bool
    final_output: str

def build_supervisor_graph():
    graph = StateGraph(TaskState)

    graph.add_node("load_context",       load_hierarchical_context)
    graph.add_node("plan",               decompose_task_with_claude_code)
    graph.add_node("execute_parallel",   execute_agents_in_parallel)
    graph.add_node("hitl_check",         human_in_the_loop_gate)
    graph.add_node("synthesize",         combine_agent_results)
    graph.add_node("evaluate",           score_task_output)
    graph.add_node("notify",             send_discord_update)
    graph.add_node("extract_learnings",  propose_skill_updates)

    graph.set_entry_point("load_context")
    graph.add_edge("load_context", "plan")
    graph.add_edge("plan", "execute_parallel")
    graph.add_conditional_edges(
        "execute_parallel",
        lambda s: "hitl_check" if s["awaiting_human"] else "synthesize"
    )
    graph.add_edge("hitl_check", "execute_parallel")
    graph.add_edge("synthesize", "evaluate")
    graph.add_edge("evaluate", "notify")
    graph.add_edge("notify", "extract_learnings")
    graph.add_edge("extract_learnings", END)

    checkpointer = PostgresSaver(conn_string=DATABASE_URL)
    return graph.compile(checkpointer=checkpointer, interrupt_before=["hitl_check"])
```

### asyncio WorkerPool (Replaces Ray)

LLM calls are I/O-bound, not CPU-bound. `asyncio` handles 95% of parallel agent needs with zero operational overhead:

```python
import asyncio
from pydantic import BaseModel

class AgentHandoff(BaseModel):
    """Typed handoff protocol between agents."""
    completed_steps: list[str]
    current_state_summary: str
    relevant_context: dict
    remaining_objectives: list[str]
    discovered_constraints: list[str]
    questions_for_next_agent: list[str]

class AgentResult(BaseModel):
    """Structured output from every agent."""
    status: str           # 'completed', 'failed', 'needs_hitl'
    confidence: float
    output: Any
    citations: list[str]
    token_usage: dict
    errors: list[str]
    handoff: Optional[AgentHandoff]

class WorkerPool:
    """asyncio-based agent pool. Replaces Ray for I/O-bound LLM work."""

    def __init__(self, concurrency: int = 10):
        self.semaphore = asyncio.Semaphore(concurrency)
        self.active_tasks: dict[str, asyncio.Task] = {}

    async def execute(self, subtask: SubTask, context: TaskContext) -> AgentResult:
        async with self.semaphore:
            agent = self._get_agent(subtask.agent_type)
            return await agent.run(subtask, context)

    async def execute_parallel(self, subtasks: list[SubTask], context: TaskContext) -> dict:
        """Execute all subtasks concurrently. Partial failures don't kill the batch."""
        coros = [self.execute(st, context) for st in subtasks]
        results = await asyncio.gather(*coros, return_exceptions=True)
        # return_exceptions=True: partial context beats total task failure
        return {
            st.id: r if not isinstance(r, Exception) else AgentResult(
                status="failed", errors=[str(r)], confidence=0.0,
                output=None, citations=[], token_usage={}, handoff=None
            )
            for st, r in zip(subtasks, results)
        }

    def _get_agent(self, agent_type: str) -> SpecialistAgent:
        return AGENT_REGISTRY[agent_type]

pool = WorkerPool(concurrency=10)  # ~3-5 per Claude Code seat; scale seats as needed
```

### Circuit Breakers on External Services

```python
from circuitbreaker import circuit

class ResilientContextLoader:
    """Context loading with circuit breakers. Partial context beats task failure."""

    @circuit(failure_threshold=5, recovery_timeout=30)
    async def load_client_context(self, client_id: str) -> dict:
        return await mcp_context_server.get_client_context(client_id)

    async def load_context(self, task: ParsedTask) -> TaskContext:
        results = await asyncio.gather(
            self.load_global_context(task),
            self.load_domain_context(task),
            self.load_client_context(task.client_id),
            self.load_vector_context(task.description),
            return_exceptions=True
        )
        return TaskContext.from_partial_results(results)
        # If client context circuit trips: task runs on global+domain only
        # Better than: task fails entirely
```

---

## 9. Layer 4: Hierarchical Context

Fully detailed in [doc 07](07-hierarchical-context-architecture.md). This layer implements the 4-tier context model as an MCP server.

### MCP Context Server

Every backend capability is exposed as an MCP server. Agents connect to only the servers they need:

```python
from mcp.server import MCPServer

class ContextMCPServer(MCPServer):
    """MCP server exposing the 4-tier context system to Claude Code agents."""

    @mcp_tool("get_client_context")
    async def get_client_context(self, client_id: str) -> ClientContext:
        if not self.session.can_access_client(client_id):
            raise PermissionError(f"Session not scoped to {client_id}")
        return await self.context_store.get_client(client_id)

    @mcp_tool("get_global_skills")
    async def get_global_skills(self, tags: list[str] = None) -> list[Skill]:
        return await self.context_store.get_global_skills(tags=tags)

    @mcp_tool("get_domain_skills")
    async def get_domain_skills(self, domain_tags: list[str]) -> list[Skill]:
        return await self.context_store.get_domain_skills(tags=domain_tags)

    @mcp_tool("update_client_context")
    async def update_client_context(self, client_id: str, updates: dict) -> bool:
        await self.context_store.update_client(client_id, updates)
        await cache.invalidate(f"client_ctx:{client_id}")
        return True
```

### Context Loading at Task Start

```python
async def load_hierarchical_context(state: TaskState) -> TaskState:
    """
    Called at the start of every task.
    The context loader is the only code that touches client data.
    Agents never query the database directly — cross-client contamination
    cannot happen through an agent bug.
    """
    ctx = await resilient_context_loader.load_context(state["task"])
    return {**state, "context": ctx}
```

### Rolling Summarization (Context Window Protection)

```python
async def manage_context_window(messages: list, max_tokens: int = 100_000) -> list:
    """
    At 60% context window: compress earliest 30% of history to summary.
    Prevents hard context limit failures on long-running tasks.
    """
    current_tokens = count_tokens(messages)
    if current_tokens > max_tokens * 0.6:
        split_idx = len(messages) // 3
        early_messages = messages[:split_idx]
        summary = await haiku_client.summarize(early_messages)
        return [{"role": "system", "content": f"[Earlier context summary]: {summary}"}] + messages[split_idx:]
    return messages
```

### Trust Level Tagging

All retrieved data is tagged with a trust level to prevent prompt injection through retrieved content:

| Source | Trust Level | Effect |
|--------|-------------|--------|
| system_generated | 1.0 | Full trust; injected directly |
| agent_generated | 0.8 | High trust; lightly validated |
| retrieved_semantic | 0.6 | Medium trust; prefixed with "[Retrieved]" |
| user_provided | 0.3 | Low trust; wrapped in XML tags, validated |

---

## 10. Layer 5: Memory & Knowledge

### Memory Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  MEMORY TYPE      TOOL              SCOPE        LIFETIME      │
├────────────────────────────────────────────────────────────────┤
│  Semantic/RAG     Qdrant            Per-client   Permanent     │
│  • Code patterns, decisions, architecture docs per client      │
├────────────────────────────────────────────────────────────────┤
│  Episodic         PostgreSQL+       Per-client   Permanent     │
│                   pgvector          Per-user                   │
│  • Past decisions, feedback, preferences from real work        │
├────────────────────────────────────────────────────────────────┤
│  Temporal/Graph   Graphiti on       Per-client   Permanent     │
│                   PostgreSQL                                   │
│  • How entities, decisions, relationships evolved over time    │
├────────────────────────────────────────────────────────────────┤
│  Working Memory   LangGraph State   Per-task     Single session│
│  • Current task state, intermediate results, tool outputs      │
└────────────────────────────────────────────────────────────────┘
```

> **Stack change from earlier design:** Mem0 Cloud and Neo4j are removed. Episodic memory uses PostgreSQL + pgvector (already in stack). Graphiti runs on the same PostgreSQL instance. This eliminates two external dependencies and keeps all data in one place with RLS enforcing isolation automatically.

### Memory as Accumulating Intelligence

Every task makes the system smarter about that client. An agent working on Client ACME in month 6 automatically knows:
- All architectural decisions made in months 1–5
- Which patterns were tried and discarded
- The client's evolving preferences as their team changed
- Which solutions triggered feedback or revisions

```python
async def store_task_learnings(task: ParsedTask, result: AgentResult):
    """Runs async after every task completion."""
    # Store episodic memory
    await pgvector_store.add(
        content=f"Task: {task.description}\nResult: {result.output}",
        client_id=task.client_id,
        metadata={"task_id": task.id, "tags": task.tags, "quality": result.confidence}
    )
    # Update temporal graph
    await graphiti.update(
        client_id=task.client_id,
        entities=result.extracted_entities,
        relationships=result.extracted_relationships,
        timestamp=datetime.utcnow()
    )
```

### Temporal Memory Consolidator (Nightly)

```python
class TemporalMemoryConsolidator:
    """
    Nightly Prefect job. Prevents context explosion over months of use.
    Detects contradictions, compresses clusters, applies recency decay.
    """
    async def consolidate(self, client_id: str):
        recent = await pgvector_store.get_recent(client_id, days=7)
        historical = await pgvector_store.get_older(client_id, days=7)

        # Find contradictions (e.g., "uses Python 2.7" vs "uses Python 3.12")
        contradictions = await self.find_contradictions(recent, historical)
        for old, new in contradictions:
            await self.deprecate_memory(old, superseded_by=new)

        # Compress episodic sequences into semantic summaries
        clusters = await self.cluster_related_memories(recent + historical)
        for cluster in clusters:
            if len(cluster) > 5:
                summary = await self.summarize_cluster(cluster)
                await self.replace_cluster_with_summary(cluster, summary)

        # Apply recency decay weights to retrieval scoring
        await self.apply_decay_weights(client_id)
```

### Skill Conflict Resolution

```python
class SkillConflictResolver:
    """
    When skills at different tiers conflict, this determines which wins.
    Behavioral skills: more specific wins (task > client > domain > global).
    Safety skills: more restrictive wins (global > domain > client > task).
    """
    BEHAVIORAL_PRECEDENCE = ["task", "client", "domain", "global"]
    SAFETY_PRECEDENCE = ["global", "domain", "client", "task"]

    def resolve(self, skill_a: Skill, skill_b: Skill) -> Skill:
        if skill_a.is_safety or skill_b.is_safety:
            # Safety: global overrides everything — security cannot be overridden
            precedence = self.SAFETY_PRECEDENCE
        else:
            # Behavioral: most specific context wins
            precedence = self.BEHAVIORAL_PRECEDENCE

        for tier in precedence:
            if skill_a.tier == tier:
                winner = skill_a
            elif skill_b.tier == tier:
                winner = skill_b
            else:
                continue
            await self.log_conflict(skill_a, skill_b, winner)
            return winner
```

---

## 11. Layer 6: Self-Improvement Engine

This is the layer that makes the system autonomously improve over time.

### Three Improvement Loops

```
LOOP 1: IMMEDIATE (after every task)
  Task completes → Evaluator scores output → Score stored in Langfuse
  Low score  → Agent stores "what went wrong" in memory (Reflexion)
  High score → Extractor: "what made this good?" → propose skill update

LOOP 2: NIGHTLY (Prefect scheduled job)
  Collect last 24h of traces + scores
  Identify skills with declining effectiveness (trending down)
  DSPy MIPROv2 generates improved prompt variants (runs on Modal — serverless)
  Automated eval on holdout set
  Best variant → Discord #skill-proposals for your approval

LOOP 3: WEEKLY (Prefect scheduled job)
  Pattern mining across all tasks
  Identify repeating patterns across clients (anonymized)
  Propose as new global or domain skill
  Human reviews + approves via Discord reaction
```

### Evaluator Agent

```python
class EvaluatorAgent:
    """Runs async after every task. Drives all three improvement loops."""

    CRITERIA = [
        "goal_achievement",     # Did it do what was asked?
        "standard_compliance",  # Did it follow the loaded skills?
        "completeness",         # Complete or cut corners?
        "correctness",          # Factually/technically correct?
        "efficiency",           # Unnecessarily verbose?
    ]

    async def evaluate(self, task: ParsedTask, result: AgentResult) -> EvalReport:
        scores = {}
        for criterion in self.CRITERIA:
            score = await self.llm_judge(
                criterion=criterion,
                task=task.description,
                context_applied=task.context_summary,
                output=result.output,
                rubric=EVAL_RUBRICS[criterion]
            )
            scores[criterion] = score

        overall = sum(scores.values()) / len(scores)
        await langfuse.score(task.trace_id, name="quality", value=overall)

        # Per-skill metric update for Grafana
        for skill_slug in task.context.loaded_skill_slugs:
            await metrics.update_skill_score(skill_slug, overall)

        if overall > 0.90:
            await self.flag_for_extraction(task, result, scores)
        elif overall < 0.65:
            await self.flag_for_review(task, result, scores)

        return EvalReport(scores=scores, overall=overall)
```

### DSPy Optimization (Modal Serverless)

```python
import dspy
from dspy.teleprompt import MIPROv2
import modal

@modal.function(gpu="any", timeout=3600)
async def run_dspy_optimization(skill_id: str, training_examples: list):
    """
    Runs on Modal (serverless). Only pay when this actually runs.
    Bursty, infrequent — perfect for serverless.
    """
    skill = await db.get_skill(skill_id)
    program = SkillProgram(skill.instructions)
    optimizer = MIPROv2(metric=quality_metric)
    optimized = optimizer.compile(program, trainset=training_examples)

    holdout_score = evaluate_on_holdout(optimized, skill)
    if holdout_score > skill.eval_score + 0.05:
        return {
            "skill_id": skill_id,
            "new_instructions": optimized.instructions,
            "improvement": holdout_score - skill.eval_score
        }
    return None

class SkillOptimizer:
    """Nightly Prefect job orchestrator."""

    async def run_nightly_optimization(self):
        declining = await langfuse.query(
            metric="skill_effectiveness",
            window="7d",
            threshold=0.75,
            trend="declining"
        )
        for skill in declining:
            examples = await self.build_training_set(skill)
            result = await run_dspy_optimization.remote(skill.id, examples)
            if result:
                await self.propose_skill_update(skill, result)

    async def propose_skill_update(self, skill, result):
        """Send proposal to Discord #skill-proposals channel."""
        embed = discord.Embed(
            title=f"💡 Skill Update Proposal: {skill.name}",
            description=f"Projected improvement: +{result['improvement']:.0%}",
            color=0x00ff00
        )
        msg = await proposals_channel.send(embed=embed)
        await msg.add_reaction("✅")  # Approve
        await msg.add_reaction("❌")  # Reject
        await msg.add_reaction("✏️")  # Request edits
```

### Canary Deployments for Skill Changes

New skill versions are promoted safely:

```
5% of tasks → new skill version → measure quality delta
  If quality improves: 25% → 50% → 100%
  If quality regresses: immediate auto-rollback, Discord alert
```

---

## 12. Layer 7: Observability & Control

**Observability is deployed in Week 1, not Week 7.** You cannot improve what you cannot see.

### Correlation IDs — End-to-End Tracing

Every operation shares a single correlation chain:
```
Discord message ID
  → task_id (PostgreSQL)
    → agent_session_id (LangGraph checkpoint)
      → LLM_call_id (Langfuse trace)
        → result_id (stored output)
```

This chain flows through every log, metric, and trace automatically.

### What You Can See

**Via Discord (always available):**
```
/status                     → All active tasks and current step
/status <task-id>           → Full task detail + agent + cost so far
/trace <task-id>            → Link to Langfuse trace for deep debugging
/costs today                → Today's LLM spend breakdown by client
/costs client-acme          → ACME's total spend this month
/skills list                → All active skills with effectiveness scores
/context client-acme        → What context ACME agents are using
/history client-acme 10     → Last 10 tasks for ACME
```

**Via Langfuse Dashboard:**
- Full token-level trace of every agent call
- Which skills were loaded for each task
- Latency breakdown per tool call
- Quality scores over time
- Prompt version history with score comparison

**Via Grafana Dashboard:**
- Active agents (real-time)
- Task queue depth
- LLM cost per hour/day (per client attribution)
- Error rates by agent type
- Self-improvement loop status
- Per-skill effectiveness trends

### Control Commands

```
/pause <task-id>            → Pause mid-execution (agent saves checkpoint)
/resume <task-id>           → Resume from exact checkpoint
/cancel <task-id>           → Cancel; agents clean up
/redirect <task-id> <msg>   → Change task direction mid-execution
/approve <task-id>          → Approve HITL gate
/reject <task-id> <reason>  → Reject with feedback
/rollback skill <slug> <v>  → Revert skill to previous version immediately
/budget client-acme $500    → Set monthly budget cap for client
/disable agent code         → Emergency: disable all code agents globally
```

### Per-Client Cost Attribution

```python
class BudgetManager:
    """Pre-flight cost check before any task is queued."""

    async def check_or_raise(self, client_id: str, estimated_cost: float):
        budget = await db.get_client_budget(client_id)
        spent = await db.get_client_spend_this_month(client_id)

        if spent + estimated_cost > budget.monthly_cap:
            raise BudgetExceededError(f"Client {client_id} at monthly cap")

        if (spent + estimated_cost) / budget.monthly_cap > 0.80:
            await notify.discord_alert(
                f"⚠️ {client_id} at 80% of monthly budget (${spent:.2f} / ${budget.monthly_cap:.2f})"
            )
```

### Notification Strategy

| Event | Channel | Interrupts You? |
|-------|---------|-----------------|
| HITL approval needed | @mention in task thread | Yes |
| Task failed | #ai-alerts | Yes |
| Budget 80% reached | #ai-alerts | Yes |
| Skill anomaly detected | #ai-alerts | Yes |
| Task step complete | Task thread | No |
| Task complete | Task thread | No |
| Skill proposal ready | #skill-proposals | No |
| Task started | Silent | No |
| Agent retried | Silent | No |
| Daily cost/quality summary | #daily-digest | No |

---

## 13. Layer 8: Safety & Guardrails

### Layered Guardrails (Replaces NeMo)

NeMo Guardrails is removed from this stack. It was designed for Llama, not Claude, and has poor compatibility. The replacement is a defense-in-depth stack:

```
INPUT PATH:
  User message
    → Presidio (PII detection + redaction)
    → LLM Guard (prompt injection detection)
    → Llama Guard 3 (safety classification)
    → Custom business rules
    → Agent execution

OUTPUT PATH:
  Agent output
    → LLM Guard (output injection check)
    → Llama Guard 3 (safety re-check)
    → Presidio (PII in output — redact before delivery)
    → Custom business rules
    → User
```

```python
class LayeredGuardrails:
    async def check_input(self, text: str, client_id: str) -> GuardrailResult:
        # Stage 1: PII (fastest, run first)
        pii_result = await presidio.analyze(text)
        if pii_result.has_pii:
            text = presidio.anonymize(text, pii_result)

        # Stage 2: Injection detection
        injection_result = await llm_guard.scan_prompt(text)
        if injection_result.risk_score > 0.8:
            return GuardrailResult(blocked=True, reason="prompt_injection")

        # Stage 3: Safety classification
        safety_result = await llama_guard.classify(text)
        if safety_result.unsafe:
            return GuardrailResult(blocked=True, reason=safety_result.category)

        # Stage 4: Client-specific business rules
        rules_result = await custom_rules.check(text, client_id)
        if rules_result.blocked:
            return GuardrailResult(blocked=True, reason=rules_result.reason)

        return GuardrailResult(blocked=False, sanitized_text=text)
```

### Tiered Action Authorization

```python
AGENT_ACTION_TIERS = {
    # Tier 0: Auto-execute, silent
    "tier_0": [
        "read_file", "web_search", "read_github",
        "read_database", "semantic_search"
    ],

    # Tier 1: Auto-execute, log and notify in thread
    "tier_1": [
        "write_file", "create_branch", "commit_code",
        "write_draft_content"
    ],

    # Tier 2: Execute but notify immediately
    "tier_2": [
        "open_pull_request", "write_database",
        "send_internal_message", "create_jira_ticket"
    ],

    # Tier 3: ALWAYS require explicit human approval
    "tier_3": [
        "merge_pull_request", "deploy_to_production",
        "send_external_email", "delete_data",
        "modify_billing", "access_credentials"
    ]
}
```

### Sandboxed Code Execution

```python
from e2b_code_interpreter import Sandbox

async def execute_code_safely(code: str, client_id: str) -> ExecutionResult:
    async with Sandbox() as sandbox:
        # Fresh ephemeral VM per execution — destroyed after task
        # No access to host filesystem or network by default
        result = await sandbox.run_code(code)
        if result.error:
            await memory.store_failure(
                code=code,
                error=result.error,
                client_id=client_id
            )
        return result
    # VM destroyed here — no persistence, no escape
```

### Idempotent Task Execution

```python
async def execute_task_idempotent(task: ParsedTask) -> AgentResult:
    """Prevent double-execution on Celery retries."""
    existing = await db.get_result_by_idempotency_key(task.idempotency_key)
    if existing:
        return existing  # Return cached result, don't re-execute

    # Store partial results by step number for resume-on-retry
    result = await execute_with_checkpointing(task)
    await db.store_result(task.idempotency_key, result)
    return result
```

---

## 14. AI Agency Platform: Building AI Systems for Clients

This is the **meta-use case** of the platform: using OpenClaw itself to design and build custom AI systems for clients. When a client asks "Can you build us an AI solution for X?", this is the workflow.

### What This Enables

The platform has accumulated:
- **Global skills:** AI system design patterns, LLM selection criteria, agent architecture blueprints, evaluation frameworks, deployment playbooks
- **Research history:** Findings from the research in docs 01–07 of this very repo, continuously updated
- **Prior client AI projects:** Architecture decisions, what worked, what failed, lessons learned
- **Domain skills:** Specific patterns for industries you've served (fintech, legal, e-commerce, etc.)

When you take on a new AI project for a client, you don't start from scratch. The system composes all of this into a working advantage.

### End-to-End: ACME Corp Email Routing

```
Client (ACME Corp) says:
"Our support team is drowning. We get 2,000 emails/day.
Can you build us something to automatically route them
to the right department and draft initial responses?"

────────────────────────────────────────────────────────

You type in Discord:
"ACME Corp wants an AI email routing and response system.
They're on Gmail, their CRM is Salesforce, Python backend.
~2000 emails/day. Start with a proposal."

════════════════════════════════════════════════════════

OpenClaw routes to the AI Agency workflow.

System Architect Agent activates with:
  • Global skills: AI system design, LLM routing architectures,
    evaluation frameworks, production deployment patterns
  • Domain skills: email classification, CRM integrations,
    human-in-the-loop patterns for customer-facing AI
  • Client context: ACME's Gmail + Salesforce + Python stack,
    prior ACME work (GDPR CSV export pattern from doc 4-walkthrough)
  • Research history: Classification benchmarks from doc 01,
    guardrails patterns from doc 03

════════════════════════════════════════════════════════

15 minutes later, in #task-acme-email-ai:

"✅ Proposal Ready — ACME Email AI System

  Recommended Architecture:
  ├── Gmail API webhook → intake service (FastAPI)
  ├── Classification agent (Claude Haiku — cheap, fast)
  │     Departments: billing, technical, sales, general
  │     Confidence threshold: 0.85 (below → human queue)
  ├── Response draft agent (Claude Sonnet — quality matters)
  │     Uses ACME tone guide + department-specific templates
  │     Human review before send (configurable)
  ├── Salesforce integration for case creation
  └── Dashboard: routing accuracy, response quality, volume metrics

  Why this stack:
  • Haiku for classification: ~$12/month at 2K emails/day
  • Sonnet for drafts: ~$180/month (only on 85%+ confidence emails)
  • Total LLM cost: ~$200/month (vs. $8K/month for a FTE)
  • 3-week build estimate

  Evidence from prior work:
  • Used similar classification pattern for Globex ticketing (94% accuracy)
  • GDPR-compliant data handling from your ACME profile
  • FastAPI + Salesforce integration pattern already in your skills library

  Next steps: [Approve Proposal] [Request Changes] [Schedule Deep Dive]"
```

### AI Agency Workflow

```python
class SystemArchitectAgent:
    """
    Specialist agent for designing AI systems for clients.
    Activated when task intent = 'design_ai_system' or 'build_ai_solution'.
    """
    AGENT_TYPE = "system_architect"
    TOOLS = ["research_ai_patterns", "estimate_costs", "generate_architecture",
             "check_prior_projects", "draft_proposal"]

    async def run(self, task: ParsedTask, context: TaskContext) -> AgentResult:
        # 1. Load all accumulated AI design knowledge
        ai_patterns = await mcp_memory.search(
            "AI system architecture patterns email classification routing",
            include_global=True,  # All prior research
            include_domain="ai-systems"
        )

        # 2. Load prior similar projects for evidence
        similar_projects = await mcp_memory.search_similar_projects(
            task.description,
            client_id=task.client_id,
            domain="ai-systems"
        )

        # 3. Generate architecture proposal
        proposal = await self.generate_proposal(
            client_context=context.client,
            task=task,
            patterns=ai_patterns,
            prior_evidence=similar_projects
        )

        return AgentResult(
            status="completed",
            output=proposal,
            confidence=0.88,
            citations=[p.source for p in ai_patterns[:5]]
        )

class AISystemDeliveryAgent:
    """
    Executes the approved AI system build plan.
    Follows the same architecture it proposed.
    """
    AGENT_TYPE = "ai_delivery"

    async def run(self, approved_proposal: Proposal, task: ParsedTask) -> AgentResult:
        # Decompose proposal into implementation tasks
        subtasks = await self.decompose_proposal(approved_proposal)

        # Execute: scaffold code, write tests, configure deployments
        results = await pool.execute_parallel(subtasks, task.context)

        # Store learnings back into client context
        await mcp_memory.update_client_context(
            task.client_id,
            {
                "ai_systems_built": [approved_proposal.system_name],
                "proven_patterns": approved_proposal.key_patterns,
                "delivery_learnings": results.learnings
            }
        )

        # Promote reusable patterns to domain skills
        for pattern in results.reusable_patterns:
            await skill_proposer.propose(
                pattern=pattern,
                evidence=[task.id],
                domain="ai-systems"
            )

        return AgentResult(status="completed", output=results.deliverables, confidence=0.91)
```

### Client Handoff HITL Checkpoint

Before delivering any AI system to a client, a mandatory review gate:

```python
class ClientDeliveryHITL:
    """
    Mandatory human checkpoint before client-facing AI delivery.
    Tier 3 action — always requires approval.
    """
    REQUIRED_CHECKS = [
        "architecture_review_complete",
        "security_audit_passed",
        "cost_estimate_approved",
        "client_expectations_aligned",
        "rollback_plan_documented"
    ]

    async def request_approval(self, delivery: AISystemDelivery) -> bool:
        checklist = "\n".join([f"- [ ] {c}" for c in self.REQUIRED_CHECKS])
        embed = discord.Embed(
            title=f"🚀 Client Delivery Approval: {delivery.client_id}",
            description=f"**{delivery.system_name}** is ready for client handoff.",
            color=0xffa500
        )
        embed.add_field(name="Required Checks", value=checklist)
        embed.add_field(name="Estimated Value", value=delivery.business_case)
        msg = await hitl_channel.send(f"@{OWNER_DISCORD_ID}", embed=embed)
        return await wait_for_approval(msg, timeout_hours=24)
```

### The Compounding Knowledge Effect

Each AI system you build for a client makes the next one faster and better:

```
Project 1: Email routing for ACME Corp
  → Learns: Gmail webhook patterns, classification thresholds, Salesforce API
  → Stored: ACME client context + "email-routing" domain skill

Project 2: Ticket classification for Globex
  → Loads: email-routing skill as starting point
  → Improves: multi-label classification, confidence calibration
  → Stored: Globex context + updated "email-classification" domain skill

Project 3: Document routing for Legal Corp
  → Loads: email-classification skill + legal domain context
  → Now has: 6 months of classification patterns across 2 prior clients
  → Estimates: 70% faster build, 15% higher accuracy out of the box

Month 12:
  → "ai-classification" domain skill has been DSPy-optimized 40+ times
  → Pattern library covers: email, tickets, documents, forms, support chat
  → New client onboarding time: 2 weeks → 4 days
  → Proposal accuracy: 65% → 91% (measured by client approval rate)
```

---

## 15. MCP-First Architecture

Every backend capability is a standalone MCP server. Claude Code agents connect to only what they need for their task.

```
MCP Server Ecosystem:
├── mcp-context      ← 4-tier context: global skills, domain skills, client context
├── mcp-memory       ← pgvector episodic memory + Graphiti temporal graph
├── mcp-skills       ← Skills registry CRUD + versioning + conflict resolution
├── mcp-guardrails   ← Input/output safety pipeline (Presidio + LLM Guard + Llama Guard 3)
├── mcp-hitl         ← Human-in-the-loop request/response + Discord notifications
├── mcp-exec         ← E2B sandbox code execution
├── mcp-db           ← Scoped PostgreSQL access (RLS enforced per-client)
└── mcp-github       ← GitHub operations scoped to client's org
```

### Agent-to-MCP Connection Pattern

```python
async def configure_agent_session(task: ParsedTask) -> AgentSession:
    """
    Each agent session connects to only the MCP servers it needs.
    Principle of least privilege: a research agent gets no write access.
    """
    base_servers = ["mcp-context", "mcp-memory", "mcp-guardrails"]

    task_servers = {
        "code_generation": ["mcp-exec", "mcp-github", "mcp-db"],
        "research":        ["mcp-memory"],
        "security_review": ["mcp-github"],
        "ai_design":       ["mcp-memory", "mcp-skills"],
        "hitl_decision":   ["mcp-hitl"],
    }.get(task.task_type, [])

    return await ClaudeCodeAgent.create_session(
        mcp_servers=base_servers + task_servers,
        client_scope=task.client_id,
        correlation_id=task.id
    )
```

---

## 16. Complete Technology Stack

### Authoritative Stack (2026)

| Layer | Component | Technology | Notes |
|-------|-----------|-----------|-------|
| **Interface** | Discord Bot | discord.py (NanoClaw) | Existing; Task Bridge added |
| **Interface** | Task Bridge | Custom FastAPI | Routes simple vs. complex |
| **Task Intelligence** | LLM Router | Custom | SDK vs API routing |
| **Task Intelligence** | Task Parser | Haiku API | Fast, cheap, structured output |
| **Task Intelligence** | Task Queue | Celery + Redis | Durable, retry semantics, DLQ |
| **Orchestration** | Supervisor | Claude Code SDK + LangGraph | Flat-rate subscription |
| **Orchestration** | Agent Workers | asyncio WorkerPool | I/O-bound; no Ray needed |
| **Context** | Context Server | FastAPI + MCP | 4-tier model, circuit breakers |
| **Context** | Skills Registry | PostgreSQL + Directus | CRUD UI, versioning, RLS |
| **Memory** | Episodic Memory | PostgreSQL + pgvector | Replaces Mem0 |
| **Memory** | Temporal Graph | Graphiti on PostgreSQL | Replaces Neo4j + Zep Cloud |
| **Memory** | Vector Search | Qdrant | Per-client namespaces |
| **Self-Improvement** | Optimization | DSPy MIPROv2 on Modal | Serverless; pay per run |
| **Self-Improvement** | Scheduling | Prefect | Nightly/weekly jobs |
| **Self-Improvement** | Evaluation | DeepEval + RAGAS | Post-task + CI/CD |
| **Observability** | Tracing | Langfuse | Deployed Week 1 |
| **Observability** | Metrics | Prometheus + Grafana | Deployed Week 1 |
| **Safety** | PII | Presidio | Input + output paths |
| **Safety** | Injection | LLM Guard | Input + output paths |
| **Safety** | Safety Classification | Llama Guard 3 | Replaces NeMo |
| **Safety** | Sandbox | E2B | Per-execution VM isolation |
| **Databases** | Primary | PostgreSQL 16 | Tasks, skills, audit, checkpoints |
| **Databases** | Cache/Queue | Redis 7 | Celery broker + context cache |
| **Databases** | Vector | Qdrant | Semantic search |

### Removed from Earlier Design (And Why)

| Removed | Replaced By | Reason |
|---------|-------------|--------|
| Ray Actors | asyncio WorkerPool | LLM calls are I/O-bound; asyncio handles this with zero ops overhead |
| Mem0 | PostgreSQL + pgvector | Eliminates external dependency; RLS applies automatically |
| Neo4j | PostgreSQL (Graphiti) | Single database; Graphiti is open-source on any PG backend |
| Zep Cloud | Graphiti on PostgreSQL | Same capability, no external vendor dependency |
| NeMo Guardrails | Layered stack (Presidio+LLMGuard+LlamaGuard3) | NeMo designed for Llama; poor Claude compatibility |
| AgentOps | Langfuse | Langfuse covers traces + evals; one less tool |
| Redis Streams (primary) | Celery + Redis | Better retry semantics, dead letter queue, less custom consumer code |

### LLM Model Strategy

| Task Type | Model | Mode | Reason |
|-----------|-------|------|--------|
| Task parsing | Claude Haiku 3.5 | Direct API | Fast, cheap, accurate enough |
| Complex reasoning | Claude (via SDK) | Subscription | Flat-rate; no token anxiety |
| Code generation | Claude (via SDK) | Subscription | Best at code; subscription absorbs cost |
| Research/analysis | Claude (via SDK) | Subscription | Long context; subscription optimal |
| Evaluation/scoring | Haiku API | Direct API | High volume; Haiku is accurate for scoring |
| Embeddings | text-embedding-3-small | OpenAI API | Fast, cheap, 1536-dim |
| DSPy optimization | Claude Opus (via SDK) | Subscription | Meta-task; highest capability needed |

---

## 17. Data Flow: Request to Completion

```
1. YOU type in Discord:
   "For ACME: fix the login bug where tokens expire too early"

2. NANOCLAW — Task Bridge assesses complexity
   → Classified as complex (auth + code change + security implications)
   → Submitted to OpenClaw REST API
   → Discord thread created: #task-acme-login-bug
   → Correlation ID: discord_msg_id → task_id (chained through entire flow)

3. TASK PARSER (Haiku API, ~150ms, ~$0.001)
   Output: ParsedTask {
     client_id: "client-acme",
     intent: "fix_bug",
     task_type: "code_generation",
     tags: ["auth", "jwt", "python"],
     agents_needed: ["code", "test"],
     idempotency_key: hash(message + author + timestamp)
   }

4. BUDGET CHECK
   → ACME monthly spend: $180 / $500 cap ✅
   → Estimated task cost: $0.12 (subscription flat-rate; internal accounting only)

5. CONTEXT LOADER (circuit-breaker protected, partial OK)
   → Global skills: dev-best-practices, security-baseline
   → Domain skills: python-conventions, jwt-patterns, auth-security
   → Client context: ACME FastAPI stack, JWT lib=python-jose, TTL=3600s
   → pgvector search: "ACME had refresh token issue in March; used env-var pattern"
   → All loaded in parallel via asyncio.gather(return_exceptions=True)

6. SUPERVISOR (Claude Code SDK — subscription)
   Plan:
     Step 1: Code Agent → find token expiry logic
     Step 2: Code Agent → implement fix with env-var TTL config
     Step 3: Test Agent (parallel with Step 2) → regression test
     Step 4: Review Agent → security review

7. SPECIALIST AGENTS execute (asyncio WorkerPool, parallel)
   Code Agent:
     → Reads ACME codebase context from Qdrant
     → Finds auth.py; implements configurable TTL via env var
     → Commits to feature branch
   Test Agent:
     → Writes pytest for new TTL behavior
     → Runs in E2B sandbox; all tests pass
   Review Agent:
     → Checks against auth-security skill
     → Flags: "confirm refresh token rotation also updated"
     → HITL triggered: @ryan in Discord thread

8. YOU approve via ✅ reaction

9. EXECUTION RESUMES
   Code Agent updates refresh token rotation
   Review Agent re-checks: ✅ clear

10. SUPERVISOR synthesizes
    → Opens PR in ACME's GitHub
    → Posts result in Discord thread

11. EVALUATOR (async, non-blocking)
    overall_score: 0.93
    Stored in Langfuse with full correlation chain

12. MEMORY UPDATE (async)
    pgvector: "ACME: JWT fix used env-var TTL + refresh rotation"
    Graphiti: JWT token → [fixed] → env-based TTL + rotation

13. EXTRACTOR (async)
    Notes: "refresh rotation almost missed — auth-security skill gap"
    → Proposes update to auth-security skill in #skill-proposals

Total elapsed: ~4-6 minutes
Effective LLM cost: ~$0.08-0.15 (direct API calls only; SDK is subscription)
```

---

## 18. Self-Improvement Loop Detail

```
┌────────────────────────────────────────────────────────────────┐
│  HOW THE SYSTEM GETS SMARTER WITHOUT YOU DOING ANYTHING        │
├────────────────────────────────────────────────────────────────┤
│  WHAT IMPROVES          HOW                    TRIGGER         │
│  ─────────────────────────────────────────────────────────     │
│  Skills / best practices DSPy prompt optimizer  Nightly cron   │
│  Client memory          pgvector extraction     After every task│
│  Knowledge graph        Graphiti updates         Real-time      │
│  Skill library growth   Extractor agent          High-quality   │
│  Agent reflexion        Memory + failure store   After failures │
│  Task routing           Score feedback           Weekly         │
│  AI design patterns     Agency Platform learnings Post-delivery │
└────────────────────────────────────────────────────────────────┘

Year-over-year effect:
  Month 1:  Agents follow your written standards
  Month 3:  Agents know each client's patterns from accumulated memory
  Month 6:  Skills have been auto-optimized 50+ times from real data
  Month 12: System handles 80% of routine tasks with minimal intervention
            AI agency projects: 3x faster, 2x higher quality than month 1
            The remaining 20% are genuinely novel problems
```

---

## 19. Human Oversight Patterns

### Autonomy Configuration

```yaml
# config/autonomy.yaml
global_defaults:
  code_commits: auto          # Auto-commit to feature branches
  pull_requests: auto         # Auto-open PRs, notify in thread
  merges: require_approval    # Always ask before merging
  deploys: require_approval   # Always ask before production deploys
  external_comms: require_approval  # Emails to clients — always ask
  ai_system_delivery: require_approval  # Client AI deliveries — always ask

client_overrides:
  client-acme:
    pull_requests: require_approval  # ACME wants to review every PR

  client-trusted-startup:
    merges: auto              # Startup trusts us, wants fast iteration

skill_updates:
  auto_promote_threshold: 0.92   # Auto-promote if score > 92%
  require_approval_below: 0.92   # Else, propose in #skill-proposals
  rollback_on_regression: true   # Auto-rollback if quality drops post-update
  canary_traffic_pct: 5          # Start canary at 5% before full rollout
```

### Escalation Ladder

```
Level 1: Silent execution          → Task completes, update in thread
Level 2: Notify                    → Post update, no action needed
Level 3: Soft HITL                 → "I'm about to do X. OK? (auto-approves in 10min)"
Level 4: Hard HITL                 → "Waiting for your approval to proceed"
Level 5: Emergency stop            → Task halted, all agents for this task paused
Level 6: Global halt               → /disable all — everything stops
```

---

## 20. Deployment Architecture

### Docker Compose (Development / Small Scale)

```yaml
services:
  nanoclaw:          # discord.py bot (NanoClaw) with Task Bridge
  task-intelligence: # FastAPI: task parser + LLM router
  orchestrator:      # LangGraph supervisor
  worker:            # Celery workers (asyncio pool)
  context-server:    # MCP context server (FastAPI)
  memory-server:     # MCP memory server (pgvector + Graphiti)
  self-improvement:  # Evaluator + Extractor (Prefect managed)
  postgres:          # Primary database (tasks, skills, audit, pgvector)
  redis:             # Celery broker + context cache
  qdrant:            # Vector search
  langfuse:          # Observability (Week 1)
  prometheus:        # Metrics (Week 1)
  grafana:           # Dashboards (Week 1)
  directus:          # Skills registry UI
```

### Kubernetes (Production)

```yaml
HorizontalPodAutoscaler:
  worker:
    metric: celery_queue_depth
    min_replicas: 2
    max_replicas: 20
    scale_up_threshold: 15  # tasks in queue

  context-server:
    metric: http_requests_per_second
    min_replicas: 2
    max_replicas: 8

# Resource isolation per client
Namespaces:
  - platform-core     (orchestrator, nanoclaw, task-intelligence)
  - platform-mcp      (context-server, memory-server, mcp servers)
  - platform-data     (postgres, redis, qdrant, langfuse)
  - platform-ops      (grafana, prometheus, alertmanager)
```

---

## 21. Implementation Roadmap

### Phase 1 — Foundations (Weeks 1-3): The Loop Closes
*Goal: First Discord message results in an agent completing a task and reporting back. Observability live from day 1.*

```
☐ Langfuse + Prometheus + Grafana deployed (Week 1 — before agents)
☐ NanoClaw Task Bridge: simple/complex routing
☐ Task parser (Haiku API) with LLM Router
☐ LangGraph supervisor: plan + execute + notify
☐ asyncio WorkerPool: 2 specialist agents (Research + Code)
☐ Skills registry: Directus + PostgreSQL + 5 global skills
☐ MCP context server: global + client context composition
☐ Celery + Redis task queue (replaces Redis Streams)
☐ Per-client budget manager
☐ 1 client onboarded end-to-end with full correlation IDs
═ Milestone: Discord message → agent executes → reports in thread, full trace visible in Langfuse.
```

### Phase 2 — Memory & Parallelism (Weeks 4-6)
*Goal: Multiple clients, parallel agents, memory accumulating.*

```
☐ pgvector memory: client episodic memory extracted after every task
☐ Qdrant: namespaced vector search per client
☐ asyncio WorkerPool expanded: 5 parallel code + 3 research agents
☐ HITL gates: Discord reaction-based approval flow
☐ 3+ clients onboarded with isolated RLS-enforced contexts
☐ /status, /pause, /resume, /approve, /costs slash commands
☐ Trust-level tagging on all retrieved context
☐ Idempotent task execution with Celery retry semantics
═ Milestone: Two client tasks run simultaneously without context mixing. Memory search returns relevant history from prior tasks.
```

### Phase 3 — Safety & Guardrails (Weeks 7-9)
*Goal: System you can trust with real client work.*

```
☐ Layered guardrails: Presidio → LLM Guard → Llama Guard 3 → custom rules
☐ E2B sandbox for all code execution
☐ Tiered action authorization (Tier 0-3)
☐ PostgreSQL RLS fully enforced across all client tables
☐ Rolling context summarization (60% window → compress 30%)
☐ Circuit breakers on all external service calls
☐ Daily digest to Discord: costs, tasks, quality scores
☐ Canary deployment infrastructure for skill changes
═ Milestone: First real client task executed in production. Security audit baseline established.
```

### Phase 4 — Self-Improvement (Weeks 10-13)
*Goal: System that gets meaningfully better without manual effort.*

```
☐ Evaluator agent: post-task quality scoring (DeepEval + RAGAS)
☐ Extractor agent: pattern mining from high-quality completions
☐ #skill-proposals Discord channel + reaction-based approval workflow
☐ DSPy on Modal: nightly optimization job (Prefect-scheduled)
☐ Skill versioning + one-command rollback
☐ Graphiti temporal graph per client (Prefect-managed consolidation)
☐ Memory consolidator: nightly contradiction detection + compression
☐ Skill conflict resolver: behavioral vs safety precedence rules
═ Milestone: First skill auto-improved from real task data. Canary deployed and promoted.
```

### Phase 5 — AI Agency Platform (Weeks 14-16)
*Goal: System capable of designing and delivering AI systems for clients.*

```
☐ System Architect agent type (AI design + proposal generation)
☐ AI System Delivery agent type (builds what Architect proposed)
☐ AI-systems domain skill library (classification, routing, retrieval, generation patterns)
☐ Client Delivery HITL checkpoint (mandatory before client handoff)
☐ Proposal generation workflow with cost estimation
☐ Post-delivery learnings stored to domain skills
☐ First client AI system designed and delivered using the platform
═ Milestone: Platform used to deliver a complete AI system to a client. Patterns stored as domain skills.
```

### Phase 6 — Scale & Harden (Weeks 17-18)
*Goal: Production-hardened, Kubernetes-deployed, fully autonomous daily operation.*

```
☐ Kubernetes deployment with HPA
☐ Autonomy config: per-client tunable control levels
☐ Semantic caching: reduce costs 20-40%
☐ Load test: 50 concurrent tasks
☐ RAGAS + DeepEval in CI/CD for all skill changes
☐ Appsmith admin panel for context + HITL management
═ Milestone: System runs 8 hours unattended. Morning Discord digest is the primary interaction.
```

---

## 22. What Makes This Different

| System | Multi-Client Isolation | Hierarchical Context | Self-Improving | Discord Native | AI Agency Capable |
|--------|----------------------|---------------------|----------------|----------------|-------------------|
| **OpenClaw** | ✅ Architectural (RLS) | ✅ 4-tier + MCP | ✅ DSPy+Reflexion+Canary | ✅ Primary UI | ✅ Core use case |
| Devin | ❌ | ❌ | ❌ | Slack only | ❌ Black box |
| OpenHands | ❌ | ❌ | ❌ | ❌ | ❌ |
| Cursor Agents | ❌ Single project | File only | ❌ | ❌ | ❌ |
| Sweep | ❌ | Rules file only | ❌ | ❌ | ❌ |

### The Compound Advantage

Each improvement builds on the last. After 12 months of real use:

- Skills have been auto-optimized dozens of times from your actual work patterns
- Every client's context is rich with accumulated memory and temporal history
- Agents know which approaches fail for which clients and avoid them
- Your best patterns have been extracted into reusable skills automatically
- The AI agency practice has a growing pattern library from every delivered project
- Proposal accuracy has improved from measurement → reflection → optimization
- The system handles ~80% of routine tasks with minimal oversight

This is not achievable by any existing off-the-shelf product. It requires intentional architecture designed for compounding returns — which is exactly what this system is.

---

*Last updated: April 2026 | Supersedes: `08-autonomous-agent-platform.md` and `09-enhanced-system-design.md` | [← Hierarchical Context](07-hierarchical-context-architecture.md) | [↑ Back to README](../README.md)*
