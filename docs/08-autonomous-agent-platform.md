# Autonomous Agent Platform
## A Discord-Native, Self-Improving, Multi-Client AI System

This document designs the full architecture for a system where you describe what you want in Discord, the right agents execute it using the correct hierarchical context for the client/project, the system improves itself over time, and you can monitor and intervene at any point.

This is the synthesis of everything in this research repo applied to a single cohesive platform.

---

## Table of Contents

1. [What This System Is](#1-what-this-system-is)
2. [Is Anyone Doing This? Industry Evidence](#2-is-anyone-doing-this-industry-evidence)
3. [End-to-End Walkthrough](#3-end-to-end-walkthrough)
4. [Full System Architecture](#4-full-system-architecture)
5. [Layer 1: Interface & Intake](#5-layer-1-interface--intake)
6. [Layer 2: Task Intelligence](#6-layer-2-task-intelligence)
7. [Layer 3: Orchestration & Execution](#7-layer-3-orchestration--execution)
8. [Layer 4: Hierarchical Context](#8-layer-4-hierarchical-context)
9. [Layer 5: Memory & Knowledge](#9-layer-5-memory--knowledge)
10. [Layer 6: Self-Improvement Engine](#10-layer-6-self-improvement-engine)
11. [Layer 7: Observability & Control](#11-layer-7-observability--control)
12. [Layer 8: Safety & Guardrails](#12-layer-8-safety--guardrails)
13. [Complete Technology Stack](#13-complete-technology-stack)
14. [Data Flow: Request to Completion](#14-data-flow-request-to-completion)
15. [Self-Improvement Loop Detail](#15-self-improvement-loop-detail)
16. [Human Oversight Patterns](#16-human-oversight-patterns)
17. [Deployment Architecture](#17-deployment-architecture)
18. [Implementation Roadmap](#18-implementation-roadmap)
19. [What Makes This Different](#19-what-makes-this-different)

---

## 1. What This System Is

You are describing a **Personal AI Operating System** — a platform that:

| Capability | What It Means |
|------------|---------------|
| **Conversational intake** | Describe tasks in Discord; no forms, no dashboards required |
| **Hierarchical context** | Automatically applies the right skills + client context per task |
| **Parallel autonomous execution** | Multiple specialist agents work simultaneously on complex tasks |
| **Strict multi-client isolation** | Client A’s data is architecturally invisible to Client B’s agents |
| **Self-improving** | The system gets better at your work patterns over time, without manual updates |
| **Human oversight** | You can monitor, pause, approve, reject, or redirect any task at any time |
| **Observable** | Full trace of every decision, tool call, and cost is available on demand |

This is not a chatbot. It is a **team of AI agents** that works like a skilled development team: you assign work, it executes using established standards, asks when uncertain, and continuously raises the quality bar.

---

## 2. Is Anyone Doing This? Industry Evidence

### Closest Existing Systems

**Devin (Cognition AI)** — https://cognition.ai
- The most capable autonomous software agent commercially available
- Slack integration for task intake
- Long-horizon multi-step execution (hours of work per task)
- **Missing:** no multi-client isolation, no hierarchical context, no self-improvement from your patterns, no extensibility beyond code

**OpenHands (formerly OpenDevin)** — https://github.com/All-Hands-AI/OpenHands
- Open-source equivalent of Devin (20k+ stars)
- Sandboxed execution, multi-model, extensible
- **Missing:** same gaps as Devin; designed for single-user, single-project

**Sweep** — https://github.com/sweepai/sweep
- GitHub bot: create an issue in natural language, Sweep writes the PR
- Has a `sweep.yaml` rules file (closest thing to a global skills layer)
- **Missing:** no multi-client, no self-improvement, code-only

**Cursor Background Agents** — https://cursor.com
- Agents that run in the background on your codebase
- Project-level context via `.cursorrules`
- **Missing:** single project, no client isolation, no Discord UX, no memory

**Linear AI / Notion AI / GitHub Copilot Workspace**
- Narrow task-specific agents embedded in tools
- **Missing:** all of the above, plus siloed to their own product

### What Enterprises Are Building (Privately)

From published case studies and research:
- **Law firms:** Agents with client matter files (isolated per client), legal best practices (global), jurisdiction rules (domain). Lawyers describe tasks in Slack.
- **Insurance companies:** Policy context per insurer (isolated), regulatory guidelines (global), product rules (domain). Claims agents execute with full audit trail.
- **Software agencies:** Exactly your use case. Client codebases isolated, shared coding standards, Discord/Slack intake. Most are 6-18 months into building this.

### The Gap

No publicly available system combines **all five** of:
1. Conversational intake (Discord/Slack)
2. Multi-client hierarchical context with strict isolation
3. Parallel multi-agent execution
4. True self-improvement from usage patterns
5. Full human observability and control

This is the system you’re describing. It is buildable today with production-ready tools.

---

## 3. End-to-End Walkthrough

Before the architecture, here’s what the full experience looks like:

```
You type in Discord:
“For ACME Corp, build an endpoint to export user data as CSV. 
Needs to be GDPR compliant. Low priority.”

════════════════════════════════════════════════

Bot instantly replies:
“⏳ Understood. Creating task for ACME Corp.
   Context loaded: FastAPI + PostgreSQL stack, GDPR profile active
   Assigning to: Code Agent + Security Review Agent
   Thread created → #task-acme-csv-export”

════════════════════════════════════════════════

In the thread, 4 minutes later:
“✅ Task Complete — ACME CSV Export

   What was built:
   • GET /api/v1/users/export endpoint (FastAPI)
   • Exports only requesting user’s own data (GDPR Article 20)
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
   skill. Want me to propose it for review? [Yes] [No]”

════════════════════════════════════════════════

You react with ✅. The system:
   • Drafts a "GDPR Data Export" domain skill
   • Adds it to staging (10% of future tasks)
   • Runs eval for 1 week
   • Reports: "GDPR skill improved compliance review pass rate by 23%”
   • Promotes to global domain skill
   • All future agents across all clients now benefit from this
```

---

## 4. Full System Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                  INTERFACE LAYER                                       │
│   Discord Bot   │   Slack Bot   │   REST API   │   Web UI          │
└───────────────────────────────────┬───────────────────────────────────┘
                                  │
┌────────────────────────────────▼───────────────────────────────────┐
│               TASK INTELLIGENCE LAYER                               │
│  Task Parser • Intent Classifier • Client/Scope Resolver            │
│  Ambiguity Detector • Priority Scorer • Tag Extractor               │
│  Task State Machine (PostgreSQL)                                     │
└────────────────────────────────┬───────────────────────────────────┘
                                  │
┌────────────────────────────────▼───────────────────────────────────┐
│               ORCHESTRATION LAYER                                   │
│  Supervisor Agent (LangGraph)                                        │
│  • Task decomposition • Agent routing • Result synthesis            │
│  • HITL interrupt nodes • Checkpoint/resume                         │
│              │ dispatches via Redis Streams                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Code Agents   │  │ Research Agt  │  │ Review Agents │  │
│  │ (Ray Pool)    │  │ (Ray Pool)    │  │ (Ray Pool)    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└────────────────────────────────┬───────────────────────────────────┘
                                  │
           ┌──────────────────┬───────────────────┐
           │                  │                   │
┌──────────▼────────┐ ┌─────▼──────┐ ┌──────────▼────────┐
│ CONTEXT LAYER   │ │ MEMORY  │ │ SELF-IMPROVE    │
│ Skills Registry │ │  LAYER  │ │ ENGINE          │
│ Directus+Qdrant │ │Mem0+Zep│ │ DSPy+RAGAS      │
└──────────────────┘ └────────────┘ └──────────────────┘
           │                  │                   │
           └──────────────────┼───────────────────┘
                                  │
┌────────────────────────────────▼───────────────────────────────────┐
│  OBSERVABILITY + SAFETY LAYER                                        │
│  Langfuse • AgentOps • Grafana • LiteLLM Proxy                      │
│  Guardrails • Sandbox • Audit Logs • HITL Gates                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## 5. Layer 1: Interface & Intake

### Discord Bot (Primary Interface)

The Discord bot is the human’s primary interaction surface. It does far more than relay messages:

```python
# discord_bot.py
import discord
from discord.ext import commands

bot = commands.Bot(command_prefix="/")

@bot.event
async def on_message(message):
    """Intercept natural language task descriptions."""
    if message.author.bot:
        return
    if bot.user.mentioned_in(message) or is_task_channel(message.channel):
        # Send to Task Intelligence Layer
        task = await task_parser.parse(message.content, author=message.author.id)
        thread = await message.create_thread(name=f"Task: {task.short_title}")
        await submit_task(task, discord_thread=thread)

@bot.slash_command(name="status")
async def status(ctx, task_id: str = None):
    """Check status of running tasks."""
    if task_id:
        info = await task_store.get(task_id)
        await ctx.respond(format_task_status(info))
    else:
        active = await task_store.get_active(user=ctx.author.id)
        await ctx.respond(format_active_tasks(active))

@bot.slash_command(name="pause")
async def pause(ctx, task_id: str):
    """Pause a running agent task."""
    await orchestrator.pause(task_id)
    await ctx.respond(f"⏸️ Task {task_id} paused. Use /resume to continue.")

@bot.slash_command(name="approve")
async def approve(ctx, task_id: str):
    """Approve a pending HITL decision."""
    await hitl_gate.approve(task_id, approver=ctx.author.id)
    await ctx.respond(f"✅ Approved. Resuming task {task_id}.")

@bot.slash_command(name="skills")
async def skills(ctx, action: str, skill_slug: str = None):
    """View or manage the skills registry."""
    # /skills list, /skills show react-conventions, /skills edit ...
    ...

@bot.slash_command(name="context")
async def context(ctx, client_id: str):
    """View context loaded for a specific client."""
    ctx_summary = await context_loader.summarize(client_id)
    await ctx.respond(ctx_summary)
```

### Discord UX Patterns

| Pattern | How It Works |
|---------|-------------|
| **Thread per task** | Every task gets its own Discord thread for clean updates |
| **Reaction-based approval** | ✅ = approve, ❌ = reject, ⏸️ = pause (no commands needed) |
| **Progress updates** | Agent posts “step complete” messages in thread as work progresses |
| **Mention-based HITL** | `@ryan` tag in thread when human decision needed |
| **Daily digest** | Morning summary of overnight tasks, cost, quality scores |
| **Alert channel** | Separate #ai-alerts channel for anomalies, budget alerts, errors |

### Task Channels Pattern

```
Discord Server: Virtek Labs AI
├── #task-intake        ← Drop tasks here
├── #ai-alerts          ← Anomalies, HITL requests, budget warnings
├── #daily-digest       ← Automated morning summary
├── #skill-proposals    ← New/updated skills awaiting approval
└── Threads (auto)      ← #task-acme-csv-export, #task-globex-api-v2 ...
```

---

## 6. Layer 2: Task Intelligence

This layer transforms natural language into a structured task the system can execute.

### Task Parser

```python
from pydantic import BaseModel
from typing import Optional

class ParsedTask(BaseModel):
    # What
    intent: str              # 'build_feature', 'fix_bug', 'research', 'review', 'analyze'
    description: str         # cleaned task description
    short_title: str         # 5-word summary for thread naming
    
    # Who/Where
    client_id: Optional[str] # 'client-acme' — None = internal task
    project_id: Optional[str] # 'main-app', 'api-v2'
    
    # How
    tags: list[str]          # ['python', 'fastapi', 'gdpr'] — drives context loading
    agent_types_needed: list[str]  # ['code', 'security-review', 'test']
    requires_human_approval: bool  # True for high-stakes actions
    
    # Priority
    priority: str            # 'low', 'medium', 'high', 'urgent'
    estimated_complexity: str  # 'trivial', 'small', 'medium', 'large'
    
    # Ambiguity
    clarifying_questions: list[str]  # Non-empty if intent is unclear
    confidence: float        # 0-1, how confident the parser is

async def parse_task(message: str, author_id: str, context: dict) -> ParsedTask:
    """
    Uses an LLM to parse natural language into a structured task.
    Also uses conversation history and known clients to resolve ambiguity.
    """
    known_clients = await db.get_client_names()  # ['ACME Corp', 'Globex Inc', ...]
    past_tasks = await mem0.search(message, user_id=author_id, limit=3)
    
    prompt = TASK_PARSER_PROMPT.format(
        message=message,
        known_clients=known_clients,
        past_context=past_tasks,
        current_projects=await db.get_active_projects()
    )
    
    result = await llm.structured_output(prompt, output_schema=ParsedTask)
    
    # If confidence < 0.7 or clarifying questions exist, ask before proceeding
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
                                          COMPLETED      REVISED─► EXECUTING
                                          
    (Any state can transition to PAUSED via /pause command)
    (FAILED state triggers Discord alert + retry logic)
```

```sql
CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    trace_id        UUID NOT NULL,        -- Links to Langfuse trace
    discord_thread  VARCHAR(100),         -- Discord thread ID for updates
    author_id       VARCHAR(100),         -- Discord user ID
    client_id       VARCHAR(100),         -- Which client this task is for
    project_id      VARCHAR(100),
    intent          VARCHAR(50),
    description     TEXT,
    tags            TEXT[],
    status          VARCHAR(30) DEFAULT 'received',
    priority        VARCHAR(20) DEFAULT 'medium',
    plan            JSONB,                -- Supervisor's decomposition
    result          JSONB,                -- Final output
    cost_usd        DECIMAL(10,4),        -- Total LLM cost
    quality_score   FLOAT,               -- Post-task evaluation score
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    completed_at    TIMESTAMPTZ
);
```

---

## 7. Layer 3: Orchestration & Execution

### Supervisor Agent (LangGraph)

The Supervisor receives a parsed task and manages the full execution:

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver

class TaskState(TypedDict):
    task: ParsedTask
    context: AgentContext       # Loaded hierarchical context
    plan: list[SubTask]         # Decomposed steps
    results: dict[str, Any]    # Results from each sub-agent
    awaiting_human: bool        # HITL flag
    final_output: str

def build_supervisor_graph():
    graph = StateGraph(TaskState)
    
    graph.add_node("load_context", load_hierarchical_context)
    graph.add_node("plan", decompose_task_into_subtasks)
    graph.add_node("execute_parallel", execute_agents_in_parallel)
    graph.add_node("hitl_check", human_in_the_loop_gate)
    graph.add_node("synthesize", combine_agent_results)
    graph.add_node("evaluate", score_task_output)
    graph.add_node("notify", send_discord_update)
    graph.add_node("extract_learnings", propose_skill_updates)
    
    graph.set_entry_point("load_context")
    graph.add_edge("load_context", "plan")
    graph.add_edge("plan", "execute_parallel")
    graph.add_conditional_edges(
        "execute_parallel",
        lambda s: "hitl_check" if s["awaiting_human"] else "synthesize"
    )
    graph.add_edge("hitl_check", "execute_parallel")  # Resume after approval
    graph.add_edge("synthesize", "evaluate")
    graph.add_edge("evaluate", "notify")
    graph.add_edge("notify", "extract_learnings")
    graph.add_edge("extract_learnings", END)
    
    # Checkpoint to PostgreSQL — tasks survive service restarts
    checkpointer = PostgresSaver(conn_string=DATABASE_URL)
    return graph.compile(checkpointer=checkpointer, interrupt_before=["hitl_check"])
```

### Parallel Agent Execution (Ray)

```python
import ray
from ray.util.queue import Queue

@ray.remote
class SpecialistAgentActor:
    """
    A stateful Ray actor representing one specialist agent.
    Each actor maintains its own LLM client, memory connection,
    and tool access list (scoped to its allowed capabilities).
    """
    def __init__(self, agent_type: str, allowed_tools: list[str]):
        self.agent_type = agent_type
        self.allowed_tools = allowed_tools
        self.llm = build_litellm_client()    # Routes through LiteLLM proxy
        self.tracer = build_langfuse_tracer()  # Every call is traced
        
    async def execute(
        self, 
        subtask: SubTask,
        context: AgentContext,
        client_id: str
    ) -> SubTaskResult:
        with self.tracer.trace(subtask.id, client_id=client_id):
            # Build system prompt from context
            system = context.to_system_prompt()
            
            # Execute with ReAct loop (Thought → Action → Observation)
            result = await self.react_loop(
                task=subtask.description,
                system=system,
                tools=self.get_scoped_tools(client_id)  # Client-scoped tool access
            )
            
            # Store episodic memory for this client
            await mem0.add(
                messages=[{"role": "assistant", "content": result.summary}],
                user_id=f"client:{client_id}"
            )
            
            return result

# Pool of agents, always warm
code_agents = [SpecialistAgentActor.remote("code", CODE_TOOLS) for _ in range(5)]
review_agents = [SpecialistAgentActor.remote("security-review", READONLY_TOOLS) for _ in range(3)]
research_agents = [SpecialistAgentActor.remote("research", SEARCH_TOOLS) for _ in range(5)]

# Execute subtasks in parallel across the pool
async def execute_agents_in_parallel(state: TaskState) -> TaskState:
    futures = []
    for subtask in state["plan"]:
        agent = get_available_agent(subtask.agent_type)
        future = agent.execute.remote(subtask, state["context"], state["task"].client_id)
        futures.append((subtask.id, future))
    
    results = {}
    for subtask_id, future in futures:
        results[subtask_id] = await future
    
    return {**state, "results": results}
```

---

## 8. Layer 4: Hierarchical Context

Fully described in [doc 07](07-hierarchical-context-architecture.md). Summary of how it integrates here:

```python
async def load_hierarchical_context(state: TaskState) -> TaskState:
    """
    Called at the start of every task.
    Composes global + domain + client context into a single object
    that every agent in this task will use.
    """
    ctx = await context_loader.load(
        agent_type=state["task"].intent,
        task_tags=state["task"].tags,
        client_id=state["task"].client_id
    )
    return {**state, "context": ctx}
```

Key guarantee: **the context loader is the only code that touches client data.** Agents never query the database directly for context — they receive a pre-composed, pre-validated context object. This means cross-client contamination cannot happen through an agent bug.

---

## 9. Layer 5: Memory & Knowledge

### Memory Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  MEMORY TYPE      TOOL         SCOPE        LIFETIME         │
├────────────────────────────────────────────────────────────────┤
│  Semantic/RAG     Qdrant       Per-client   Permanent        │
│  • Code patterns, decisions, architecture docs per client   │
├────────────────────────────────────────────────────────────────┤
│  Episodic         Mem0         Per-client   Permanent        │
│  • Past decisions, feedback, preferences extracted from work │
├────────────────────────────────────────────────────────────────┤
│  Temporal/Graph   Graphiti     Per-client   Permanent        │
│  • How entities, decisions, and relationships evolved over time│
├────────────────────────────────────────────────────────────────┤
│  User/Personal    Mem0         Per-user     Permanent        │
│  • Your preferences, communication style, working patterns   │
├────────────────────────────────────────────────────────────────┤
│  Working Memory   LangGraph    Per-task     Single session   │
│  • Current task state, intermediate results, tool outputs    │
└────────────────────────────────────────────────────────────────┘
```

### Memory as Accumulating Intelligence

This is key: **every task makes the system smarter about that client.** An agent working on Client ACME in month 6 knows:
- All architectural decisions made in months 1-5
- Which patterns were tried and discarded
- The client’s evolving preferences as their team changed
- Which solutions triggered feedback or revisions

This happens automatically through Mem0’s extraction on task completion:

```python
async def store_task_learnings(task: Task, result: TaskResult):
    # Mem0 automatically extracts facts, preferences, decisions
    await mem0.add(
        messages=[
            {"role": "user", "content": task.description},
            {"role": "assistant", "content": result.summary}
        ],
        user_id=f"client:{task.client_id}",
        metadata={"task_id": str(task.id), "tags": task.tags}
    )
    # Result: "Client ACME prefers rate limiting on all export endpoints"
    # This fact is now searchable for all future ACME tasks
```

---

## 10. Layer 6: Self-Improvement Engine

This is the layer that makes the system **autonomously get better over time** — not just at individual tasks, but at the systemic level.

### Three Improvement Loops

```
LOOP 1: IMMEDIATE (after every task)
  Task completes → Evaluator scores output → Score stored in Langfuse
  Low score → Reflexion: agent stores "what went wrong" in its memory
  High score → Extractor: "what made this good?" → propose skill update

LOOP 2: NIGHTLY (scheduled)
  Collect last 24h of traces + scores
  Identify skills with declining effectiveness (score trending down)
  DSPy generates improved prompt variants for those skills
  Automated A/B test in staging
  Best variant → Discord #skill-proposals for your approval

LOOP 3: PERIODIC (weekly/on-demand)
  Pattern mining across all tasks
  Identify emerging best practices that repeat across clients
  Anonymize (strip client-specific details)
  Propose as new global or domain skill
  Human reviews + approves via Discord reaction
```

### Evaluator Agent

```python
class EvaluatorAgent:
    """
    Runs automatically after every task completion.
    Produces a quality score that drives the improvement loops.
    """
    
    CRITERIA = [
        "goal_achievement",    # Did it do what was asked?
        "standard_compliance", # Did it follow the loaded skills/standards?
        "completeness",        # Was the output complete or did it cut corners?
        "correctness",         # Is the output factually/technically correct?
        "efficiency",          # Was it unnecessarily verbose or redundant?
    ]
    
    async def evaluate(self, task: Task, result: TaskResult) -> EvalReport:
        scores = {}
        for criterion in self.CRITERIA:
            score = await self.llm_judge(
                criterion=criterion,
                task=task.description,
                context_applied=task.context_summary,
                output=result.content,
                rubric=EVAL_RUBRICS[criterion]
            )
            scores[criterion] = score
        
        overall = sum(scores.values()) / len(scores)
        
        # Store in Langfuse for trend analysis
        await langfuse.score(task.trace_id, name="quality", value=overall)
        
        # Trigger improvement loop if score is notable
        if overall > 0.90:
            await self.flag_for_extraction(task, result, scores)
        elif overall < 0.65:
            await self.flag_for_review(task, result, scores)
        
        return EvalReport(scores=scores, overall=overall)
```

### DSPy Optimization Loop

```python
import dspy
from dspy.teleprompt import MIPROv2

class SkillOptimizer:
    """
    Nightly job: find underperforming skills and optimize their prompts.
    """
    
    async def run_nightly_optimization(self):
        # 1. Find skills with declining scores
        declining = await langfuse.query(
            metric="skill_effectiveness",
            window="7d",
            threshold=0.75,
            trend="declining"
        )
        
        for skill in declining:
            # 2. Get training examples from Langfuse traces
            examples = await self.build_training_set(skill)
            
            # 3. DSPy optimizes the skill instructions
            program = SkillProgram(skill.instructions)
            optimizer = MIPROv2(metric=quality_metric)
            optimized = optimizer.compile(program, trainset=examples)
            
            # 4. Test against holdout set
            holdout_score = self.evaluate_on_holdout(optimized, skill)
            current_score = skill.eval_score
            
            # 5. If better, propose update
            if holdout_score > current_score + 0.05:  # 5% improvement threshold
                await self.propose_skill_update(
                    skill=skill,
                    new_instructions=optimized.instructions,
                    improvement=holdout_score - current_score,
                    evidence=examples[:3]  # Show examples to human reviewer
                )
    
    async def propose_skill_update(self, skill, new_instructions, improvement, evidence):
        """Send proposal to Discord #skill-proposals channel."""
        embed = discord.Embed(
            title=f"💡 Skill Update Proposal: {skill.name}",
            description=f"Projected improvement: +{improvement:.0%}",
            color=0x00ff00
        )
        embed.add_field(name="Current instructions (excerpt)", value=skill.instructions[:300])
        embed.add_field(name="Proposed instructions (excerpt)", value=new_instructions[:300])
        embed.add_field(name="Evidence", value=format_examples(evidence))
        
        msg = await proposals_channel.send(embed=embed)
        await msg.add_reaction("✅")  # Approve
        await msg.add_reaction("❌")  # Reject
        await msg.add_reaction("✏️")  # Request edits
        
        # Wait for reaction (24h timeout → auto-approve if high confidence)
        ...
```

---

## 11. Layer 7: Observability & Control

### What You Can See

**Via Discord (always available):**
```
/status                     → All active tasks and their current step
/status <task-id>           → Full task detail + current agent + cost so far
/trace <task-id>            → Link to Langfuse trace for deep debugging
/costs today                → Today’s LLM spend breakdown by client/project
/costs client-acme          → ACME’s total spend this month
/skills list                → All active skills with effectiveness scores
/context client-acme        → What context ACME agents are currently using
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
- LLM cost per hour/day
- Error rates by agent type
- Self-improvement loop status

### Control Commands

```
/pause <task-id>            → Pause mid-execution (agent saves state)
/resume <task-id>           → Resume from exact checkpoint
/cancel <task-id>           → Cancel task, agents clean up
/redirect <task-id> <msg>   → Change task direction mid-execution
/approve <task-id>          → Approve HITL gate
/reject <task-id> <reason>  → Reject with feedback
/rollback skill <slug> <v>  → Revert skill to previous version immediately
/budget client-acme $500    → Set monthly budget cap for a client
/disable agent code         → Emergency: disable all code agents globally
```

### Real-Time Notification Strategy

```python
class NotificationRouter:
    """
    Determines what deserves to interrupt you vs. what should be silent.
    """
    
    INTERRUPT_EVENTS = [
        "hitl_approval_needed",     # Always notify
        "task_failed",              # Always notify  
        "budget_80pct_reached",     # Warn before hard limit
        "skill_anomaly_detected",   # Quality dropped suddenly
        "security_guardrail_hit",   # Safety system triggered
    ]
    
    THREAD_UPDATE_EVENTS = [
        "task_step_complete",       # Progress update in task thread
        "task_complete",            # Final result in task thread
        "skill_proposal_ready",     # Post to #skill-proposals
    ]
    
    SILENT_EVENTS = [
        "task_started",             # Don’t ping for every start
        "agent_retried",            # Internal; don’t clutter
        "cache_hit",                # Cost saving; show in digest
        "skill_score_updated",      # Batch into daily digest
    ]
    
    DAILY_DIGEST_EVENTS = [
        "daily_cost_summary",
        "skill_performance_summary",
        "tasks_completed_count",
        "self_improvement_summary",
    ]
```

---

## 12. Layer 8: Safety & Guardrails

### Tiered Authorization for Agent Actions

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

All code generated by agents runs in E2B ephemeral sandboxes:

```python
from e2b_code_interpreter import Sandbox

async def execute_code_safely(code: str, client_id: str) -> ExecutionResult:
    async with Sandbox() as sandbox:
        # Fresh VM per execution — destroyed after task
        # No access to host filesystem or network by default
        result = await sandbox.run_code(code)
        
        if result.error:
            # Trigger Reflexion: store "this code pattern caused an error"
            await reflexion.store_failure(
                code=code, 
                error=result.error,
                client_id=client_id
            )
        
        return result
    # VM destroyed here — no persistence, no escape
```

---

## 13. Complete Technology Stack

### Core Services

| Service | Technology | Purpose |
|---------|------------|----------|
| **Discord Bot** | discord.py | Primary human interface |
| **Task Intelligence** | FastAPI + Claude Sonnet | Parse + classify tasks |
| **Orchestrator** | LangGraph + PostgreSQL | Stateful task execution |
| **Agent Workers** | Ray Actors + Ray Serve | Parallel specialist agents |
| **Context Loader** | FastAPI (internal) | Compose hierarchical context |
| **Skills Registry** | Directus + PostgreSQL | Store/manage skills with UI |
| **Self-Improvement** | Custom + DSPy | Nightly optimization loop |
| **API Gateway** | LiteLLM Proxy | Model routing + budget mgmt |
| **Notification Router** | Custom + Redis pub/sub | Route events to Discord |

### Databases

| Database | Use Case | Why |
|----------|----------|---------|
| **PostgreSQL** | Task state, skills registry, audit logs, client context, LangGraph checkpoints | Relational integrity, RLS for isolation, battle-tested |
| **Redis** | Task queues (Streams), pub/sub for real-time notifications, context caching | Sub-millisecond, native queue primitives |
| **Qdrant** | Vector search per client namespace | Filtering + vectors, production performance, payload isolation |
| **Neo4j** | Temporal knowledge graph (via Graphiti) | Relationship-aware queries, entity evolution over time |
| **S3 / GCS** | Artifact storage (code outputs, reports, generated files) | Cheap, durable, CDN-accessible |

### AI & Agent Tools

| Tool | Role |
|------|------|
| **LangGraph** | Supervisor workflow, stateful execution, HITL interrupts |
| **Ray** | Parallel agent workers, autoscaling pool |
| **Mem0** | Episodic memory per client/user (auto-extraction) |
| **Graphiti / Zep** | Temporal knowledge graph |
| **DSPy** | Automated prompt optimization (nightly loop) |
| **RAGAS** | RAG pipeline quality scoring |
| **DeepEval** | CI/CD quality gates on skill changes |
| **E2B** | Sandboxed code execution |
| **NeMo Guardrails** | Dialogue control + topic rails |
| **LLM Guard** | Prompt injection detection |
| **Langfuse** | Full observability: traces, evals, prompts |
| **AgentOps** | Agent session replay and debugging |
| **Grafana** | System-level dashboards |
| **Prometheus** | Metrics collection |
| **Appsmith** | Admin UI for context management, HITL approvals |

### LLM Model Strategy (via LiteLLM Proxy)

| Task Type | Model | Reason |
|-----------|-------|--------|
| Task parsing | Claude Haiku 3.5 | Fast, cheap, accurate enough |
| Research/Analysis | Claude Sonnet 4.5 | Best reasoning-per-dollar |
| Code generation | GPT-4o or Claude Sonnet | Strongest at code |
| Code review | Claude Sonnet | Best at nuanced critique |
| Evaluation/scoring | GPT-4o-mini | High volume, needs consistency |
| Optimization (DSPy) | Claude Opus / GPT-4o | Highest capability for meta-tasks |
| Embeddings | text-embedding-3-small | OpenAI, fast, cheap, 1536 dim |

---

## 14. Data Flow: Request to Completion

```
1. YOU type in Discord:
   “For ACME: fix the login bug where tokens expire too early”

2. DISCORD BOT receives message
   → Creates Discord thread #task-acme-login-bug
   → Sends to Task Intelligence Service

3. TASK PARSER (LLM call, ~200ms)
   Input: raw message + known clients + ACME history from Mem0
   Output: ParsedTask {
     client_id: "client-acme",
     intent: "fix_bug",
     tags: ["auth", "jwt", "python"],
     agents_needed: ["code", "test"],
     complexity: "small"
   }

4. TASK STORE
   → Creates task record in PostgreSQL
   → Assigns trace_id → Langfuse trace opened

5. CONTEXT LOADER
   → Global skills: dev-best-practices, security-baseline
   → Domain skills: python-conventions, jwt-patterns, auth-security
   → Client context: ACME’s FastAPI stack, JWT library=python-jose,
                     token TTL currently 3600s, ACME’s test conventions
   → Episodic: "ACME had a similar issue in March; used refresh token pattern"
   → Returns: AgentContext object (isolated to ACME)

6. SUPERVISOR AGENT (LangGraph) plans:
   Step 1: Code Agent → find the token expiry logic
   Step 2: Code Agent → implement fix with correct TTL configuration
   Step 3: Test Agent (parallel with Step 2) → write regression test
   Step 4: Review Agent → security review of auth change

7. SPECIALIST AGENTS execute (Ray, parallel where possible)
   Code Agent:
     - Reads ACME codebase context from Qdrant
     - Finds auth.py token generation
     - Implements configurable TTL with env var
     - Commits to feature branch
   Test Agent:
     - Writes pytest tests for new TTL behavior
     - Runs in E2B sandbox, all tests pass
   Review Agent:
     - Checks against ACME’s security skill
     - Flags: "confirm refresh token rotation also updated"
     → HITL triggered: @ryan in Discord thread

8. YOU approve via Discord reaction ✅

9. EXECUTION RESUMES
   Code Agent updates refresh token logic too
   Review Agent re-checks: ✅ all clear

10. SUPERVISOR synthesizes:
    → Opens PR in ACME’s GitHub
    → Posts result in Discord thread

11. EVALUATOR scores the output (async)
    goal_achievement: 0.95
    standard_compliance: 0.92
    overall: 0.93
    → Stored in Langfuse

12. MEMORY updates (async)
    Mem0 extracts: "ACME: JWT fix pattern used env-var TTL config"
    Graphiti updates: JWT token → [fixed] → uses env-based TTL

13. EXTRACTOR checks (async)
    Notes: "refresh token rotation was almost missed — could be in auth skill"
    → Posts to #skill-proposals: "Propose adding refresh token check to auth-security skill"

Total elapsed time: ~3-6 minutes
LLM cost: ~$0.08-0.15
```

---

## 15. Self-Improvement Loop Detail

```
┌────────────────────────────────────────────────────────────────┐
│  HOW THE SYSTEM GETS SMARTER WITHOUT YOU DOING ANYTHING              │
├────────────────────────────────────────────────────────────────┤
│                                                                      │
│  WHAT IMPROVES          HOW                    TRIGGER              │
│  ─────────────────────────────────────────────────────────────  │
│  Skills / best practices DSPy prompt optimizer  Nightly cron        │
│  Client memory          Mem0 extraction         After every task    │
│  Knowledge graph        Graphiti updates         Real-time           │
│  Skill library growth   Extractor agent          High-quality tasks  │
│  Agent reflexion        Reflexion pattern        After failures      │
│  Task routing           Score feedback → retrain Weekly              │
│  Model selection        Cost/quality A/B         Monthly             │
└────────────────────────────────────────────────────────────────┘

Year-over-year effect:
  Month 1:  Agents follow your written standards
  Month 3:  Agents know each client’s patterns from accumulated memory
  Month 6:  Skills have been auto-optimized 50+ times from real data
  Month 12: System handles 80% of routine tasks with minimal intervention
            The remaining 20% are genuinely novel problems
```

---

## 16. Human Oversight Patterns

### Oversight Dial (Autonomy vs. Control)

You can tune how much autonomy the system has at any level:

```yaml
# config/autonomy.yaml
global_defaults:
  code_commits: auto          # Auto-commit to feature branches
  pull_requests: auto         # Auto-open PRs, notify in thread
  merges: require_approval    # Always ask before merging
  deploys: require_approval   # Always ask before production deploys
  external_comms: require_approval  # Emails, Slack to clients — always ask

client_overrides:
  client-acme:
    pull_requests: require_approval  # ACME wants to review every PR
    
  client-trusted-startup:
    merges: auto              # Startup trusts us, wants fast iteration

skill_updates:
  auto_promote_threshold: 0.92   # Auto-promote if score > 92%
  require_approval_below: 0.92   # Else, ask you
  rollback_on_regression: true   # Auto-rollback if quality drops post-update
```

### Escalation Ladder

```
Level 1: Silent execution          → Task completes, update in thread
Level 2: Notify                    → Post update, no action needed
Level 3: Soft HITL                 → "I’m about to do X. OK? (auto-approves in 10min)"
Level 4: Hard HITL                 → "Waiting for your approval to proceed"
Level 5: Emergency stop            → Task halted, all agents for this task paused
Level 6: Global halt               → /disable all — everything stops
```

---

## 17. Deployment Architecture

### Docker Compose (Development / Small Scale)

```yaml
# docker-compose.yml
services:
  discord-bot:       # discord.py bot
  task-intelligence: # FastAPI task parser
  orchestrator:      # LangGraph supervisor
  ray-head:          # Ray cluster head node
  ray-worker:        # Ray agent workers (scale: 2-5 replicas)
  context-loader:    # FastAPI context service
  self-improvement:  # DSPy optimizer + evaluator
  litellm:           # LLM proxy
  postgres:          # Main database
  redis:             # Queues + cache
  qdrant:            # Vector search
  langfuse:          # Observability
  directus:          # Skills registry UI
  grafana:           # Metrics dashboard
  prometheus:        # Metrics collection
```

### Kubernetes (Production)

```yaml
# High-level: each service becomes a Deployment or StatefulSet
# Key scaling rules:

HorizontalPodAutoscaler:
  ray-worker:
    metric: redis_task_queue_depth
    min_replicas: 2
    max_replicas: 20
    scale_up_threshold: 10  # tasks in queue
    
  context-loader:
    metric: http_requests_per_second
    min_replicas: 2
    max_replicas: 8

# Resource isolation per client via Kubernetes namespaces
Namespaces:
  - platform-core     (orchestrator, discord-bot, litellm)
  - client-acme       (ACME-specific workers, scoped secrets)
  - client-globex     (Globex-specific workers, scoped secrets)
  - platform-data     (postgres, redis, qdrant, langfuse)
  - platform-ops      (grafana, prometheus, alertmanager)
```

---

## 18. Implementation Roadmap

### Phase 1 — Foundations (Weeks 1-3): The Loop Closes
*Goal: First message in Discord results in an agent completing a task and reporting back.*

```
☐ Discord bot: message intake + thread creation
☐ Task parser: LLM-based intent + client extraction
☐ LangGraph supervisor: basic plan + execute + notify
☐ 2 specialist agents: Research + Code (single instance each)
☐ Skills registry: Directus + 5 global skills populated
☐ Context loader: global + client context composition
☐ LiteLLM proxy: model routing + basic cost tracking
☐ Langfuse: basic tracing active
☐ 1 client onboarded end-to-end
═ Milestone: You send a Discord message, an agent does the work, reports in thread.
```

### Phase 2 — Memory & Parallelism (Weeks 4-6)
*Goal: Multiple clients, parallel agents, memory accumulating.*

```
☐ Mem0 integrated: client memory extracted after every task
☐ Qdrant: namespaced vector search per client
☐ Ray: pool of 5 parallel code agents + 3 research agents
☐ Redis Streams: proper async task queue
☐ HITL gates: Discord reaction-based approval flow
☐ 3+ clients onboarded with isolated contexts
☐ /status, /pause, /resume, /approve slash commands
═ Milestone: Two client tasks run simultaneously without context mixing.
```

### Phase 3 — Safety & Observability (Weeks 7-9)
*Goal: System you can trust with real client work.*

```
☐ NeMo Guardrails + LLM Guard on all agent I/O
☐ E2B sandbox for all code execution
☐ Tiered action authorization (Tier 0-3)
☐ PostgreSQL Row-Level Security for client isolation
☐ Grafana dashboard: active agents, queue depth, costs, errors
☐ AgentOps: agent session replay for debugging
☐ Daily digest to Discord: costs, tasks completed, quality scores
═ Milestone: First real client task executed in production.
```

### Phase 4 — Self-Improvement (Weeks 10-13)
*Goal: System that gets better without manual effort.*

```
☐ Evaluator agent: post-task quality scoring
☐ Extractor agent: pattern mining from high-quality completions
☐ #skill-proposals Discord channel + reaction-based approval
☐ DSPy nightly optimization job
☐ Skill versioning + one-command rollback
☐ Graphiti / Zep: temporal knowledge graph per client
☐ Reflexion: agents store failure modes in memory
═ Milestone: First skill auto-improved from real task data.
```

### Phase 5 — Scale & Harden (Weeks 14-18)
*Goal: Production-hardened, Kubernetes-deployed, fully autonomous daily operation.*

```
☐ Kubernetes deployment with HPA
☐ Autonomy config: per-client tunable control levels
☐ Advanced model routing: cost/quality optimizer
☐ Semantic caching: reduce costs 20-40%
☐ Appsmith admin panel for context management
☐ RAGAS + DeepEval in CI/CD for skill changes
☐ Load test: 50 concurrent tasks
═ Milestone: System runs 8 hours unattended; morning Discord digest is the only interaction needed.
```

---

## 19. What Makes This Different

| System | Multi-Client Isolation | Hierarchical Context | Self-Improving | Discord Native | Open/Controllable |
|--------|----------------------|---------------------|----------------|----------------|-------------------|
| **This system** | ✅ Architectural | ✅ 4-tier | ✅ DSPy+Reflexion | ✅ Primary UI | ✅ Full control |
| Devin | ❌ | ❌ | ❌ | Slack only | ❌ Black box |
| OpenHands | ❌ | ❌ | ❌ | ❌ | ✅ Open source |
| Cursor Agents | ❌ Single project | ❌ File only | ❌ | ❌ | Partial |
| Sweep | ❌ | ❌ Rules file | ❌ | ❌ | ✅ Open source |
| GitHub Copilot | ❌ | Partial (org rules) | ❌ | ❌ | ❌ |

**The compound advantage:** Each improvement builds on the last. After 12 months of real use:
- Skills have been auto-optimized dozens of times from your actual patterns
- Every client’s context is rich with accumulated memory
- Agents know which approaches fail for which clients and avoid them
- Your good work patterns have been extracted into reusable skills automatically
- The system handles ~80% of routine tasks with minimal oversight

This is not achievable by any existing off-the-shelf product. It requires intentional architecture — which is exactly what this system is designed for.

---

*Last updated: April 2026 | [← Hierarchical Context](07-hierarchical-context-architecture.md) | [↑ Back to README](../README.md)*
