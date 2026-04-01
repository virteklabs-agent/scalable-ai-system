# Hierarchical Context Architecture
## Reusable AI Skills Across Projects with Isolated Client Contexts

This document addresses one of the most practical challenges in scaling AI systems: **how do you make global best practices universally available to all agents, while keeping client/project-specific context completely isolated?**

This pattern is sometimes called **Hierarchical Context Management**, **Scoped Agent Memory**, or a **Multi-Tenant Skills Architecture**.

---

## Table of Contents

1. [The Core Problem](#1-the-core-problem)
2. [Is This Being Done? (Industry Evidence)](#2-is-this-being-done-industry-evidence)
3. [The Four-Tier Context Model](#3-the-four-tier-context-model)
4. [Reference Architecture](#4-reference-architecture)
5. [Implementation: Skills Registry](#5-implementation-skills-registry)
6. [Implementation: Client Context Isolation](#6-implementation-client-context-isolation)
7. [Implementation: Agent Context Loading](#7-implementation-agent-context-loading)
8. [Preventing Context Contamination](#8-preventing-context-contamination)
9. [Scalability Improvements](#9-scalability-improvements)
10. [Productivity Patterns Beyond This System](#10-productivity-patterns-beyond-this-system)
11. [Recommended Stack for This Pattern](#11-recommended-stack-for-this-pattern)
12. [Quick Start Implementation](#12-quick-start-implementation)

---

## 1. The Core Problem

Imagine you have:
- **Global skills** you want every agent to always know: developer best practices, security standards, code review guidelines, communication templates
- **Domain skills** that apply in certain tech contexts: React conventions, Python idioms, DevOps patterns
- **Client A context**: their codebase, stack preferences, business rules, past decisions
- **Client B context**: completely different — their API contracts, naming conventions, deploy targets
- **Task context**: the specific thing being worked on right now

The challenge: Agent working on Client A's React app should know **React best practices (global) + Client A's context**, but must **never** see Client B's data. And you shouldn't have to copy your global best practices into every client project.

```
What we want:

  Agent on Client A task
  ├── ✅ Global: Developer best practices
  ├── ✅ Domain: React conventions
  ├── ✅ Client A: Their stack, history, preferences
  └── ❌ Client B: Completely invisible

  Agent on Client B task
  ├── ✅ Global: Developer best practices
  ├── ✅ Domain: Python conventions
  ├── ✅ Client B: Their stack, history, preferences
  └── ❌ Client A: Completely invisible
```

---

## 2. Is This Being Done? (Industry Evidence)

**Yes — and it's one of the fastest-growing patterns in production AI.**

### What companies are doing:

**Anthropic (Claude's own system)**  
Claude Code uses a `CLAUDE.md` file pattern — a markdown file in each project repo that injects project-specific context into every Claude session. Global preferences live in `~/.claude/CLAUDE.md`, project context in `./CLAUDE.md`. This is the same hierarchical override pattern described here, just in a file system form.

**GitHub Copilot Workspace**  
Uses repository-level context files (`.github/copilot-instructions.md`) for project-specific instructions that all Copilot agents inherit. Microsoft extended this with enterprise-level instructions that propagate to all repos in an org — exactly the global → project hierarchy.

**Cursor IDE**  
Implements `.cursorrules` (project-level) and global rules in settings. Rules compose: global + project-level rules are merged, with project rules taking precedence on conflicts.

**Agency Swarm (open source)**  
Built the explicit concept of an "Agency Manifesto" — shared instructions visible to all agents in a crew — plus agent-specific instructions that only that agent sees. Direct implementation of this pattern.

**Letta (MemGPT)**  
Introduced "shared memory blocks" — a memory object multiple agents can read/write simultaneously. Perfect for a global skills block shared across all agents, with separate per-client blocks only accessible to agents in that client's context.

**LangGraph Long-Term Memory Store**  
Uses namespaced memory: `("skills", "global")`, `("client", "client-a")`, `("client", "client-b")` — agents query only the namespaces they have access to.

**Enterprise RAG systems (Elastic, Pinecone docs)**  
Multi-tenant RAG is a documented pattern: one vector database, multiple isolated namespaces/collections per tenant, a shared "public" namespace all tenants can query. Pinecone calls this the "Namespace" pattern; Qdrant uses payload filters + named collections.

---

## 3. The Four-Tier Context Model

```
┌────────────────────────────────────────────────────────────────────┐
│  TIER 1: GLOBAL SKILLS                    Scope: ALL agents        │
│  ─────────────────────────────────────────────────────────────     │
│  Developer best practices, security standards, code review         │
│  checklist, communication templates, ethical guidelines            │
│  Updated by: humans or optimizer agent                             │
│  Versioned: yes, with rollback                                     │
└───────────────────────────┬────────────────────────────────────────┘
                            │ inherited by
┌───────────────────────────▼────────────────────────────────────────┐
│  TIER 2: DOMAIN SKILLS                    Scope: Tagged agents     │
│  ─────────────────────────────────────────────────────────────     │
│  React conventions, Python idioms, DevOps patterns, API design,    │
│  database optimization, testing strategies                         │
│  Loaded: when agent's task matches domain tags                     │
│  Example: code agent on a React task gets React domain skills      │
└───────────────────────────┬────────────────────────────────────────┘
                            │ combined with
┌───────────────────────────▼────────────────────────────────────────┐
│  TIER 3: CLIENT / PROJECT CONTEXT         Scope: ISOLATED          │
│  ─────────────────────────────────────────────────────────────     │
│  Client A: tech stack, team preferences, past decisions,           │
│            approved libraries, deploy targets, naming conventions  │
│  Client B: entirely separate namespace — never mixed               │
│  Access control: strict tenant isolation                           │
└───────────────────────────┬────────────────────────────────────────┘
                            │ added at runtime
┌───────────────────────────▼────────────────────────────────────────┐
│  TIER 4: TASK / SESSION CONTEXT           Scope: Ephemeral         │
│  ─────────────────────────────────────────────────────────────     │
│  Current conversation, active file contents, tool call results,    │
│  working memory for this specific task                             │
│  Lifetime: single session, never persisted to higher tiers         │
└────────────────────────────────────────────────────────────────────┘
```

### Inheritance & Override Rules

| Rule | Example |
|------|---------|
| Lower tiers inherit upper tiers | Client A agents always see global skills |
| Lower tiers can override upper tiers | Client A can override a global naming convention |
| Sibling tiers are fully isolated | Client A and Client B never share context |
| Task context never promotes upward | A session conversation doesn't leak into client context |

---

## 4. Reference Architecture

```
                    ┌──────────────────────────────────┐
                    │      CONTEXT REGISTRY            │
                    │      (Directus + PostgreSQL)      │
                    │                                  │
                    │  ┌────────────────────────────┐  │
                    │  │  global_skills table        │  │
                    │  │  domain_skills table        │  │
                    │  │  client_contexts table      │  │
                    │  │  context_overrides table    │  │
                    │  └────────────────────────────┘  │
                    │  Exposed via: REST API + MCP      │
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │      CONTEXT LOADER SERVICE      │
                    │                                  │
                    │  Input:  agent_type, client_id,  │
                    │          task_tags               │
                    │  Output: merged context object   │
                    │                                  │
                    │  1. Fetch global skills          │
                    │  2. Fetch matching domain skills │
                    │  3. Fetch client context         │
                    │  4. Apply overrides              │
                    │  5. Return composed prompt ctx   │
                    └──────────────┬───────────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
┌──────────▼──────┐    ┌───────────▼──────┐    ┌──────────▼──────┐
│  Agent: Client A│    │  Agent: Client A │    │  Agent: Client B│
│  (Code task)    │    │  (Research task) │    │  (Code task)    │
│                 │    │                  │    │                 │
│ Context:        │    │ Context:         │    │ Context:        │
│ ✅ Global       │    │ ✅ Global        │    │ ✅ Global       │
│ ✅ React domain │    │ ✅ Research dom  │    │ ✅ Python domain│
│ ✅ Client A     │    │ ✅ Client A      │    │ ✅ Client B     │
│ ❌ Client B     │    │ ❌ Client B      │    │ ❌ Client A     │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

---

## 5. Implementation: Skills Registry

### Data Model

```sql
-- Global & Domain Skills
CREATE TABLE skills (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug         VARCHAR(100) UNIQUE NOT NULL,
    name         VARCHAR(200) NOT NULL,
    tier         VARCHAR(20) NOT NULL CHECK (tier IN ('global', 'domain')),
    tags         TEXT[],          -- ['react', 'typescript', 'frontend']
    instructions TEXT NOT NULL,   -- The actual skill content
    version      VARCHAR(20) NOT NULL DEFAULT '1.0.0',
    eval_score   FLOAT,           -- Effectiveness score from evaluations
    is_active    BOOLEAN DEFAULT true,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Client / Project Contexts (isolated)
CREATE TABLE client_contexts (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id    VARCHAR(100) NOT NULL,  -- 'client-acme', 'client-globex'
    project_id   VARCHAR(100),           -- optional project subdivision
    context_key  VARCHAR(100) NOT NULL,  -- 'tech_stack', 'conventions', 'history'
    context_val  JSONB NOT NULL,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(client_id, project_id, context_key)
);

-- Per-client overrides of global/domain skills
CREATE TABLE skill_overrides (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id    VARCHAR(100) NOT NULL,
    skill_slug   VARCHAR(100) REFERENCES skills(slug),
    override_val TEXT NOT NULL,   -- Replacement instructions for this client
    reason       TEXT,            -- Why this client diverges from global
    created_at   TIMESTAMPTZ DEFAULT NOW()
);
```

### Populating Global Skills (Example)

```yaml
# Global Skill: developer-best-practices
slug: developer-best-practices
name: Developer Best Practices
tier: global
tags: [all, development]
instructions: |
  ## Code Quality Standards
  - Write self-documenting code; add comments only for non-obvious logic
  - Functions should do one thing and do it well (Single Responsibility)
  - Write tests before fixing bugs (verify the bug, then fix, then verify fix)
  - Never commit secrets, API keys, or credentials to version control
  - All PRs require: passing CI, at least one reviewer approval, no TODO comments

  ## Security Baseline
  - Validate and sanitize all external inputs
  - Use parameterized queries — never string concatenation for SQL
  - Log security events; never log passwords or tokens
  - Dependencies must be pinned to exact versions in production

  ## Communication
  - Commit messages: imperative mood, < 72 chars, reference ticket ID
  - PR descriptions must include: what changed, why, how to test
  - Breaking changes require a migration guide in the PR
version: 2.1.0
```

```yaml
# Domain Skill: react-conventions
slug: react-conventions
name: React & TypeScript Conventions
tier: domain
tags: [react, typescript, frontend, nextjs]
instructions: |
  ## Component Design
  - Prefer functional components with hooks over class components
  - One component per file; filename matches component name (PascalCase)
  - Extract custom hooks for reusable stateful logic (use prefix `use`)
  - Props interfaces named `[ComponentName]Props`

  ## State Management
  - Local UI state: useState / useReducer
  - Server state: React Query / TanStack Query
  - Global app state: Zustand (avoid Redux unless already in use)

  ## Performance
  - Wrap expensive renders in React.memo
  - Memoize callbacks passed to children with useCallback
  - Lazy load routes with React.lazy + Suspense
version: 1.3.0
```

---

## 6. Implementation: Client Context Isolation

### Populating Client Context

```python
# When onboarding a new client, populate their context namespace
async def setup_client_context(client_id: str, onboarding_data: dict):
    """
    Call this once when a new client is onboarded.
    Their context is fully isolated from all other clients.
    """
    entries = [
        {
            "client_id": client_id,
            "context_key": "tech_stack",
            "context_val": {
                "frontend": onboarding_data["frontend"],       # e.g., "React 18 + TypeScript"
                "backend": onboarding_data["backend"],         # e.g., "FastAPI + Python 3.12"
                "database": onboarding_data["database"],       # e.g., "PostgreSQL 15"
                "deployment": onboarding_data["deployment"],   # e.g., "AWS ECS + RDS"
                "ci_cd": onboarding_data["ci_cd"]             # e.g., "GitHub Actions"
            }
        },
        {
            "client_id": client_id,
            "context_key": "conventions",
            "context_val": {
                "branch_naming": "feature/TICKET-description",
                "environment_names": ["dev", "staging", "prod"],
                "approved_libraries": onboarding_data.get("approved_libraries", []),
                "forbidden_libraries": onboarding_data.get("forbidden_libraries", [])
            }
        },
        {
            "client_id": client_id,
            "context_key": "preferences",
            "context_val": onboarding_data.get("preferences", {})
        }
    ]
    await db.bulk_insert("client_contexts", entries)
```

### Vector Store Isolation (Qdrant)

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue

client = QdrantClient(url="http://localhost:6333")

# OPTION A: Separate collections per client (hard isolation)
# Collection: "skills_global", "client_acme_docs", "client_globex_docs"

# OPTION B: Single collection + payload filter (soft isolation, simpler ops)
# All documents have a 'tenant' field; every query filters on it

def search_client_knowledge(query_vector: list, client_id: str, include_global: bool = True):
    """
    Search returns:
    - Always: results scoped to client_id
    - Optionally: results from global namespace (skills, best practices)
    Client A can NEVER see Client B's results.
    """
    filter_conditions = [
        FieldCondition(key="tenant", match=MatchValue(value=client_id))
    ]
    
    if include_global:
        filter_conditions.append(
            FieldCondition(key="tenant", match=MatchValue(value="global"))
        )
    
    results = client.search(
        collection_name="knowledge_base",
        query_vector=query_vector,
        query_filter=Filter(
            should=filter_conditions  # client_id OR global
        ),
        limit=10
    )
    return results

# Client A search: sees client-acme + global docs
client_a_results = search_client_knowledge(vector, client_id="client-acme")

# Client B search: sees client-globex + global docs. Client A docs: never returned.
client_b_results = search_client_knowledge(vector, client_id="client-globex")
```

---

## 7. Implementation: Agent Context Loading

This is the core function — it composes the full context for any agent at runtime:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class AgentContext:
    global_skills: list[dict]
    domain_skills: list[dict]
    client_context: dict
    overrides: dict  # Client-specific overrides to global skills
    
    def to_system_prompt(self) -> str:
        """
        Compose all context tiers into a single system prompt section.
        Client overrides take precedence over global skills.
        """
        sections = []
        
        # Apply global skills (with any client overrides)
        sections.append("## Organization Standards (applies to all work)")
        for skill in self.global_skills:
            if skill["slug"] in self.overrides:
                # This client has a custom version of this skill
                sections.append(f"### {skill['name']} (customized for this client)")
                sections.append(self.overrides[skill["slug"]])
            else:
                sections.append(f"### {skill['name']}")
                sections.append(skill["instructions"])
        
        # Add domain skills
        if self.domain_skills:
            sections.append("\n## Domain-Specific Standards")
            for skill in self.domain_skills:
                sections.append(f"### {skill['name']}")
                sections.append(skill["instructions"])
        
        # Add client context
        if self.client_context:
            sections.append("\n## Client Context")
            for key, val in self.client_context.items():
                sections.append(f"**{key.replace('_', ' ').title()}:** {val}")
        
        return "\n".join(sections)


async def load_agent_context(
    agent_type: str,
    task_tags: list[str],
    client_id: Optional[str] = None
) -> AgentContext:
    """
    The central context loading function.
    Called at the start of every agent invocation.
    
    Args:
        agent_type: 'code', 'research', 'analysis', etc.
        task_tags: ['react', 'typescript'] — determines domain skills to load
        client_id: 'client-acme' — if None, only global/domain skills loaded
    """
    
    # 1. Always load global skills (all active global-tier skills)
    global_skills = await db.query(
        "SELECT * FROM skills WHERE tier = 'global' AND is_active = true"
    )
    
    # 2. Load domain skills matching the task tags
    domain_skills = await db.query(
        "SELECT * FROM skills WHERE tier = 'domain' AND is_active = true "
        "AND tags && $1",  # PostgreSQL array overlap operator
        [task_tags]
    )
    
    # 3. Load client context (ONLY for this client — no cross-client access possible)
    client_context = {}
    overrides = {}
    if client_id:
        rows = await db.query(
            "SELECT context_key, context_val FROM client_contexts WHERE client_id = $1",
            [client_id]
        )
        client_context = {row["context_key"]: row["context_val"] for row in rows}
        
        # 4. Load any client-specific overrides to global skills
        override_rows = await db.query(
            "SELECT skill_slug, override_val FROM skill_overrides WHERE client_id = $1",
            [client_id]
        )
        overrides = {row["skill_slug"]: row["override_val"] for row in override_rows}
    
    return AgentContext(
        global_skills=global_skills,
        domain_skills=domain_skills,
        client_context=client_context,
        overrides=overrides
    )


# Usage in a LangGraph agent node:
async def code_agent_node(state: AgentState) -> AgentState:
    # Load full context for this invocation
    ctx = await load_agent_context(
        agent_type="code",
        task_tags=state["detected_tags"],  # e.g., ['react', 'typescript']
        client_id=state["client_id"]       # e.g., 'client-acme'
    )
    
    system_prompt = BASE_CODE_AGENT_PROMPT + "\n\n" + ctx.to_system_prompt()
    
    response = await llm.ainvoke(
        [{"role": "system", "content": system_prompt},
         {"role": "user",   "content": state["task"]}]
    )
    return {**state, "result": response}
```

---

## 8. Preventing Context Contamination

Context contamination is the risk that Client A's data leaks into Client B's context. Defense layers:

### Layer 1: Database-level Isolation
```sql
-- Row-level security (PostgreSQL RLS)
ALTER TABLE client_contexts ENABLE ROW LEVEL SECURITY;

CREATE POLICY client_isolation ON client_contexts
    USING (client_id = current_setting('app.current_client_id'));

-- Set at connection time — no code-level filtering needed
SET app.current_client_id = 'client-acme';
-- Now ANY query on client_contexts only returns client-acme rows
-- Even if a bug forgets to add a WHERE clause
```

### Layer 2: Namespace Enforcement in Vector DB
```python
# Qdrant: every document tagged with tenant at write time
def add_to_knowledge_base(content: str, client_id: str, metadata: dict):
    client.upsert(
        collection_name="knowledge_base",
        points=[PointStruct(
            id=str(uuid4()),
            vector=embed(content),
            payload={
                **metadata,
                "tenant": client_id,  # Always stamped at write time
                "created_at": datetime.utcnow().isoformat()
            }
        )]
    )

# Reads always filter by tenant — enforced in the service layer
# A query without a tenant filter is a programming error, not a feature
```

### Layer 3: Agent Scope Validation
```python
class ScopedAgent:
    """Base class ensuring agents never access outside their allowed scope."""
    
    def __init__(self, client_id: str, allowed_scopes: list[str]):
        self.client_id = client_id
        self.allowed_scopes = allowed_scopes + ["global"]  # global always allowed
    
    async def retrieve_context(self, query: str, scope: str) -> list:
        if scope not in self.allowed_scopes:
            raise PermissionError(
                f"Agent for {self.client_id} attempted to access scope '{scope}'. "
                f"Allowed: {self.allowed_scopes}"
            )
        return await search_knowledge(query, tenant=scope)
```

### Layer 4: Audit Logging
```python
# Every context access is logged — detect anomalies in who's accessing what
await audit_log.write({
    "event": "context_access",
    "agent_id": agent_id,
    "client_id": client_id,
    "tier": tier_accessed,
    "keys_accessed": keys,
    "timestamp": datetime.utcnow()
})
```

---

## 9. Scalability Improvements

### 9.1 Semantic Skill Routing (Don't Load Everything)

Instead of injecting all active skills into every agent, use **semantic search to find only relevant skills** for the current task:

```python
async def load_relevant_skills(task_description: str, top_k: int = 5) -> list:
    """
    Instead of dumping all 50 global skills into the prompt,
    semantically retrieve only the most relevant ones for this task.
    Reduces prompt bloat and improves signal-to-noise ratio.
    """
    task_embedding = await embed(task_description)
    
    # Skills are stored with embeddings of their descriptions
    relevant = await skills_vector_store.search(
        vector=task_embedding,
        filter={"tier": {"$in": ["global", "domain"]}},
        limit=top_k
    )
    return relevant
```

**Why this matters at scale:** At 100+ skills, injecting everything degrades performance and wastes tokens. Retrieving 5 most relevant skills per task keeps quality high and costs low.

---

### 9.2 Skill Effectiveness Tracking (Skills That Don't Help, Get Retired)

```python
# After each task, score which skills were actually referenced in the output
async def score_skill_effectiveness(task_id: str, skills_loaded: list, output: str):
    for skill in skills_loaded:
        # Did the agent's output reflect this skill's guidance?
        score = await llm_judge.score(
            question=f"Did the output follow the guidance in '{skill.name}'?",
            context=skill.instructions,
            output=output
        )
        await db.update_skill_score(skill.id, score)  # Rolling average

# Skills with consistently low scores get flagged for review or retirement
# Skills with high scores get promoted to "core" tier
```

---

### 9.3 Context Caching

Global skills rarely change. Cache compiled context objects to avoid recomputing on every agent call:

```python
import hashlib
from functools import lru_cache

@redis_cache(ttl=3600)  # Cache for 1 hour
async def get_global_context_compiled() -> str:
    """Global skills change rarely — cache the compiled prompt section."""
    skills = await db.query("SELECT * FROM skills WHERE tier='global' AND is_active=true")
    return compile_skills_to_prompt(skills)

# Client contexts are specific per client; cache with client_id as key
@redis_cache(key_fn=lambda cid: f"client_ctx:{cid}", ttl=300)
async def get_client_context_compiled(client_id: str) -> str:
    ...
```

---

### 9.4 Context Versioning with Rollback

```python
# Every skill update creates a new version — you can roll back globally
async def update_skill(slug: str, new_instructions: str, updated_by: str):
    # Archive current version
    current = await db.get_skill(slug)
    await db.insert("skill_versions", {
        "skill_id": current.id,
        "version": current.version,
        "instructions": current.instructions,
        "archived_at": datetime.utcnow(),
        "archived_by": updated_by
    })
    
    # Update to new version
    new_version = increment_semver(current.version)  # 1.2.0 -> 1.3.0
    await db.update_skill(slug, {
        "instructions": new_instructions,
        "version": new_version,
        "updated_by": updated_by
    })
    
    # Invalidate caches
    await cache.delete("global_context_compiled")

async def rollback_skill(slug: str, to_version: str):
    """One call to revert a skill update across ALL agents immediately."""
    archived = await db.get_skill_version(slug, to_version)
    await update_skill(slug, archived.instructions, updated_by="rollback")
```

---

### 9.5 Automatic Skill Extraction (Skills That Write Themselves)

An **Extractor Agent** monitors successful task outputs and proposes new skills:

```
Flow:
1. Agent completes a task with high quality score
2. Extractor Agent analyzes the output
3. Identifies reusable patterns ("the agent always validates inputs this way")
4. Drafts a proposed new skill or improvement to existing skill
5. Presents to human for review (or auto-promotes if confidence > threshold)
6. Approved skill becomes available to ALL agents immediately

Result: Your skills library grows automatically as agents do good work.
```

---

## 10. Productivity Patterns Beyond This System

### 10.1 Personal AI Workspace ("Second Brain" per Developer)

Each developer on your team gets a personal context namespace that persists across all their sessions — their preferences, past decisions, coding style, and domain expertise:

```
Developer: Ryan
├── Preferred patterns (functional over OOP, Zustand over Redux)
├── Past decisions ("chose FastAPI over Django for project X because...")
├── Expertise ("strong in systems design, working on ML ops")
└── Personal shortcuts ("always wants PR descriptions in bullet format")
```

This is different from client context — it follows the *developer*, not the *project*. Implemented via Mem0 with `user_id` scoping.

---

### 10.2 Automatic Knowledge Distillation (Org Learns Over Time)

As developers and agents work, the best patterns are automatically promoted:

```
Code Review Agent finishes 50 reviews
    ↓
Analyzer sees: agent consistently flags "missing error handling on async calls"
    ↓
This becomes a global skill: "Async Error Handling Best Practice"
    ↓
Now all agents proactively write better async code without being told
```

Tools: **DSPy** (auto-optimize the skill prompt), **RAGAS** (score effectiveness), **Reflexion** (agent learns from its own mistakes).

---

### 10.3 Context-Aware Tool Authorization

Different clients may require different tool access:

```yaml
client-acme:
  allowed_tools:
    - read_github    # Can access their GitHub org
    - write_github   # Can commit to their repos
    - read_jira      # Can read their Jira
  forbidden_tools:
    - write_database # Too risky; require human approval
    - send_email     # Requires human sign-off

client-globex:
  allowed_tools:
    - read_github
    - read_confluence  # Their internal wiki
  forbidden_tools:
    - write_github     # Read-only access only
```

Agents working on Client A literally cannot call Client B's tools — enforced by the context loader, not relying on the LLM to "remember" restrictions.

---

### 10.4 Async Background Agents (Work Happens While You Sleep)

Instead of waiting for agents to respond synchronously:

```
You submit: "Review all open PRs for security issues"
    ↓
Supervisor spawns 10 parallel Code Review Agents (one per open PR)
    ↓
Agents work asynchronously using the global security review skill
    ↓
30 minutes later: you receive a consolidated report
Each agent used the same security checklist; client context ensured
they understood each project's approved patterns
```

---

### 10.5 Skill Competitions (A/B Test Your Best Practices)

Rather than guessing which version of a best practice produces better code, run experiments:

```
Skill v1.2: "Always use async/await for I/O operations"
Skill v1.3: "Use async/await for I/O; prefer asyncio.gather() for concurrent ops"

Week 1: 50% of agents use v1.2, 50% use v1.3
Metrics: code review pass rate, PR revision count, test coverage
Result: v1.3 shows 12% fewer PR revisions → auto-promoted as new global standard
```

---

### 10.6 Cross-Client Pattern Mining (Anonymous Insights)

With proper anonymization, patterns that work across clients can be promoted to global skills:

```
Observation: In 8 different client projects, agents that used "defensive 
copy on object mutation" reduced bugs reported in QA by 30% on average
    ↓
This pattern gets anonymized (no client data exposed)
    ↓
Promoted to Global Skill: "Immutability Best Practices"
    ↓
All future agents benefit from what worked across all projects
```

Critical: client-specific details never leave the client namespace. Only abstract patterns are promoted.

---

## 11. Recommended Stack for This Pattern

| Component | Recommended Tool | Why |
|-----------|-----------------|-----|
| **Skills Registry** | Directus (PostgreSQL) | MCP-native, beautiful UI, version tracking built in |
| **Client Context Store** | PostgreSQL + Row-Level Security | Database-enforced isolation, no app-layer bugs possible |
| **Vector Search (global)** | Qdrant — `global` collection | Fast, filtering, open source |
| **Vector Search (client)** | Qdrant — per-client namespace | Same infra, fully isolated by payload filter |
| **Developer Memory** | Mem0 — `user_id` scoped | Automatic extraction, semantic search |
| **Temporal/Graph Memory** | Graphiti / Zep | Track how client context evolves over time |
| **Context Loader** | Custom FastAPI service | Single source of truth for context assembly |
| **Agent Framework** | LangGraph | State persistence, conditional branching, checkpointing |
| **Caching** | Redis | Context cache for global skills (changes rarely) |
| **Admin Dashboard** | Directus UI + Appsmith | Edit skills, view client context, manage overrides |
| **Skill Optimization** | DSPy | Auto-improve skill prompts from evaluation data |
| **Effectiveness Eval** | RAGAS + Langfuse | Score which skills are actually working |

---

## 12. Quick Start Implementation

### Step 1: Scaffold the Skills Registry
```bash
# Deploy Directus with PostgreSQL
docker compose up directus postgres

# Create the skills and client_contexts collections via Directus UI
# or import the schema:
curl -X POST http://localhost:8055/schema/apply \
  -H "Authorization: Bearer $DIRECTUS_TOKEN" \
  -d @schema/skills-registry-schema.json
```

### Step 2: Add Your First Global Skill
```python
import httpx

httpx.post("http://localhost:8055/items/skills", json={
    "slug": "developer-best-practices",
    "name": "Developer Best Practices",
    "tier": "global",
    "tags": ["all"],
    "instructions": """[Your organization's coding standards here]""",
    "version": "1.0.0"
}, headers={"Authorization": f"Bearer {token}"})
```

### Step 3: Wire Up the Context Loader
```python
# Every agent call goes through this
ctx = await load_agent_context(
    agent_type="code",
    task_tags=["react", "typescript"],
    client_id="client-acme"
)
system_prompt = f"{BASE_PROMPT}\n\n{ctx.to_system_prompt()}"
```

### Step 4: Add a Client
```python
await setup_client_context("client-acme", {
    "frontend": "React 18 + TypeScript",
    "backend": "Node.js + Express",
    "database": "PostgreSQL",
    "branch_naming": "feature/JIRA-description",
    "approved_libraries": ["react-query", "zustand", "zod"]
})
```

From this point on, any agent working on Client ACME automatically gets developer best practices + React conventions + ACME's specific context — and zero access to any other client's data.

---

*Last updated: April 2026 | [← System Design](06-system-design.md) | [↑ Back to README](../README.md)*
