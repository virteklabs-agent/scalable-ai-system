# 02 — Context & Memory Management

> How to store, share, and retrieve context across agents, projects, and timelines — so skills and best practices defined once are available everywhere.

---

## Table of Contents

1. [Centralized Memory / Context Stores](#1-centralized-memory--context-stores)
2. [Vector Databases for Agent Memory](#2-vector-databases-for-agent-memory)
3. [Graph-Based Knowledge Stores](#3-graph-based-knowledge-stores)
4. [Cross-Project Skill & Knowledge Sharing](#4-cross-project-skill--knowledge-sharing)
5. [Context Versioning & Management](#5-context-versioning--management)
6. [RAG Patterns for Agent Context](#6-rag-patterns-for-agent-context)
7. [State Management for Long-Running Sessions](#7-state-management-for-long-running-sessions)
8. [Multi-Tenant Context Isolation](#8-multi-tenant-context-isolation)
9. [Decision Matrix](#9-decision-matrix)

---

## 1. Centralized Memory / Context Stores

### Mem0 ("mem-zero")
- **What it is:** The most production-ready shared memory layer as of 2026. Dynamically extracts, consolidates, and retrieves salient information from conversations. Combines vector-based semantic search with optional graph memory for entity relationships.
- **Memory scopes:** `user_id`, `agent_id`, `run_id` — multiple independent projects point to one Mem0 instance, safely partitioned
- **Cross-project use case:** User preferences learned in Project A are immediately available to Project B. No sync step — Mem0 merges and deduplicates facts automatically across sessions.
- **Benchmarks:** +26% accuracy over OpenAI Memory (LOCOMO benchmark); 91% lower latency; 90%+ token cost savings; graph variant (Mem0g) scores 58.13% on temporal reasoning vs. OpenAI's 21.71%
- **Scalability:** Designed for billions of memories. AWS integration (Jan 2026) pairs with ElastiCache (sub-ms retrieval) and Neptune Analytics (graph traversal at scale). Kubernetes-ready.
- **GitHub:** [mem0ai/mem0](https://github.com/mem0ai/mem0)
- **Best for:** Personalized AI assistants, customer support agents, B2B copilots — anywhere per-user long-term memory is required across multiple products

### Zep
- **What it is:** Enterprise-grade context engineering platform built on Graphiti (temporal knowledge graph). Ingests both unstructured conversations and structured business data (CRM, app events, documents) and builds a unified context graph updated in real time.
- **Cross-project use case:** The same entity (user, organization, product) can be referenced from any project. The temporal model tracks "what was true when" — new projects automatically see the current state of any entity without conflicts.
- **Benchmarks:** Deep Memory Retrieval 94.8% vs MemGPT's 93.4%; LongMemEval +18.5% accuracy, 90% latency reduction vs. baseline
- **Architecture highlights:** Graphiti engine, hybrid retrieval (time + full-text + semantic + graph algorithm), automated summarization, SOC 2 compliant
- **GitHub:** [getzep/graphiti](https://github.com/getzep/graphiti)
- **Best for:** Enterprise agents that need to reason about how facts and relationships change over time; multi-system data integration

### Letta (formerly MemGPT)
- **What it is:** Open-source stateful agent framework from UC Berkeley. Treats the LLM context window like a CPU register — manages a memory hierarchy (core in-context, archival in a searchable DB) via self-editing memory tools the agent invokes.
- **Cross-project use case:** The Agent File (.af) format is an open serialization standard for stateful agents. An agent's full state — prompts, memory blocks, tool configs — can be checkpointed, versioned, and loaded by any compatible framework. Multiple projects share memory blocks by connecting agents to the same backend.
- **GitHub:** [letta-ai/letta](https://github.com/letta-ai/letta) | [letta-ai/agent-file](https://github.com/letta-ai/agent-file)
- **Best for:** Long-running autonomous agents that need persistent, self-managed, unlimited memory

---

## 2. Vector Databases for Agent Memory

### Quick Comparison

| Database | Type | Best For | Realistic Scale | Multi-tenancy |
|----------|------|----------|-----------------|---------------|
| **Pinecone** | Managed SaaS | Zero-ops enterprise, compliance (SOC 2, HIPAA) | Billions of vectors | Namespaces per tenant |
| **Weaviate** | OSS + Managed | Hybrid search, GraphQL, multi-modal | Very large | Tenant isolation classes |
| **Qdrant** | OSS + Managed | Complex filters, Rust performance | Billions | Collections per tenant |
| **Milvus** | OSS + Managed | GPU-accelerated large-scale | Billions (GPU) | Partitions per tenant |
| **Chroma** | Open-source | Prototyping, local dev | Small–Medium | Collections per tenant |
| **pgvector** | PostgreSQL ext. | SQL-native teams, existing PG stack | 10–100M vectors | Row-level RLS |

### When to Use Each

**Pinecone** — *Use when:* You want fully managed infrastructure with enterprise compliance and zero operational overhead. Cross-project sharing via namespaces within a shared index. [pinecone.io](https://www.pinecone.io)

**Qdrant** — *Use when:* You need strong filtered search on metadata (filter by `project_id`, `skill_type`, `agent_id`). Cost-sensitive production workloads. Written in Rust for performance. [github.com/qdrant/qdrant](https://github.com/qdrant/qdrant)

**Weaviate** — *Use when:* You need multi-modal search (text + images) or a rich GraphQL interface. HIPAA-compliant on AWS. [github.com/weaviate/weaviate](https://github.com/weaviate/weaviate)

**pgvector** — *Use when:* Your team already uses PostgreSQL. Row-level security (RLS) handles multi-tenant isolation natively. Achieve 471 QPS at 99% recall on 50M vectors with pgvectorscale. Graduate to Qdrant/Weaviate beyond ~100M vectors. [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector)

**Chroma** — *Use when:* Prototyping locally or building a proof of concept. Not suitable for regulated multi-tenant production loads. [github.com/chroma-core/chroma](https://github.com/chroma-core/chroma)

**Scalability rule of thumb:** pgvector/Chroma for prototyping → Qdrant/Weaviate for production mid-scale → Pinecone/Milvus at billions of vectors or zero-ops requirements.

---

## 3. Graph-Based Knowledge Stores

### Neo4j + GraphRAG
- **What it is:** Industry-leading graph database. GraphRAG uses a knowledge graph as the retrieval path — the graph structure compactly encodes entity relationships, letting more relevant context fit in a limited context window.
- **Cross-project use case:** The same entity node (user, org, concept) is traversed from any agent project. Neo4j's MCP server exposes the graph as MCP resources — any MCP-compatible agent framework can query and update shared knowledge.
- **Context Graphs:** Capture decision traces — the full reasoning chain, causal relationships, and provenance behind every significant agent decision. Enables a persistent, traversable history of "why the system decided what it did."
- **Use case:** "What decisions have all our agents made about this customer across all projects?" — answered in milliseconds by traversing the context graph
- **GitHub:** [neo4j-labs/create-context-graph](https://github.com/neo4j-labs/create-context-graph)

### Graphiti (by Zep)
- **What it is:** Open-source temporal knowledge graph library powering Zep. Tracks fact evolution over time — old facts versioned as historical edges, new facts as active edges. Supports fusion queries combining time, full-text, semantic, and graph algorithms.
- **Use case:** Systems that need to reason about how facts change — e.g., "What did we know about this customer 3 months ago vs. today?" or "What changed about our security policy last quarter?"
- **GitHub:** [getzep/graphiti](https://github.com/getzep/graphiti)

### Microsoft GraphRAG
- **What it is:** Constructs entity-relation graphs from document corpora. Supports local search (specific entities) and global search (thematic questions spanning the entire corpus).
- **Use case:** Enterprise knowledge bases — e.g., querying across thousands of internal documents: "What are our standard security policies across all teams?"
- **GitHub:** [microsoft/graphrag](https://github.com/microsoft/graphrag)

---

## 4. Cross-Project Skill & Knowledge Sharing

### Model Context Protocol (MCP)
- **What it is:** Open standard (Linux Foundation/Anthropic) defining how agents exchange prompts, tools, and resources with shared context stores. MCP Servers expose resources/tools; MCP Clients (agents) consume them.
- **Cross-project use case:** Any project using an MCP-compatible framework connects to the same MCP Server (e.g., a Neo4j memory server, Mem0 server, or internal knowledge base). The November 2025 spec adds enterprise IdP (SSO-based authorization), enabling auditable cross-project agent access without per-server credentials.
- **2025/2026 adoption:** OpenAI adopted MCP March 2025; Gartner predicts 75% of gateway vendors will have MCP features by 2026
- **Spec:** [modelcontextprotocol.io](https://modelcontextprotocol.io)

### Agent Skills (Markdown-Based Knowledge Packages)
- **What it is:** Modular knowledge packages that ship as Markdown files with YAML frontmatter. A skill is a folder containing a `SKILL.md` with name, description, and instructions, plus optional examples, templates, and reference files.
- **Cross-project use case:** Define a "Security Code Review" skill once in a shared Git repo. Import into any project. Works with Claude Code, Spring AI, and custom LLM agents — model-agnostic and framework-agnostic.
- **Progressive disclosure architecture:** At startup only descriptions load (~200 tokens per skill); when a task matches, the full skill body loads (~1,500 tokens); detailed references load on demand (~500 tokens each). The agent carries exactly the knowledge it needs at each moment.

```yaml
# skills/security-review/SKILL.md
---
name: "Security Code Review"
description: "Performs OWASP Top 10 security-focused code review"
version: "1.2.0"
tags: [security, code-review, owasp]
frameworks: [any]
---
# Instructions
1. Check for SQL injection, XSS, CSRF vulnerabilities...
2. Validate input sanitization on all user-facing fields...
```

**Highest-value reusable skill categories:**
- **Convention skills:** Code style, naming patterns, architectural decisions your team has made
- **Process skills:** Deployment workflows, ticket-writing standards, review procedures
- **Domain skills:** Security patterns, compliance requirements, industry-specific reasoning
- **Integration skills:** How to work with your specific APIs, databases, and internal systems

### Shared Memory Blocks (Letta Pattern)
- Multiple agents across different projects connect to the same named memory block (e.g., `research_results`, `shared_context`)
- When one agent writes to the block, all connected agents see the update immediately
- **Use case:** Project A's research agent writes findings to a `research_results` block; Project B's analysis agent reads the same block — no message bus, no sync step needed

---

## 5. Context Versioning & Management

### Context Engineering (The Discipline Replacing Prompt Engineering)
- **Key Anthropic finding (2025):** In a 100-turn web research evaluation, context editing (intelligently pruning stale tool outputs) reduced token consumption by **84%**. Combining memory tools with context editing improved agent performance **39%** over baseline. Context editing alone delivered **29%** improvement.
- Building capable agents is now less about finding the right words and more about deciding *what configuration of context* most reliably produces the desired behavior.

### Hierarchical Memory Tiers

| Tier | Contents | Storage | Retention |
|------|----------|---------|----------|
| **Short-term** | Recent conversation turns verbatim | In-context or session store | Current session |
| **Medium-term** | Compressed summaries of recent sessions | Summary DB / vector store | Days to weeks |
| **Long-term** | Key facts, entity relationships, decisions | Knowledge graph / relational DB | Indefinite |

> **Context drift warning:** 65% of enterprise AI failures in 2025 were attributed to context drift, not raw context exhaustion. A 2% misalignment introduced early in an agent chain can compound to a **40% failure rate** by the end. Context *quality* — not just quantity — is the primary reliability lever.

### Git-Context-Controller (GCC) Pattern
- Applies version control semantics to agent context: `COMMIT` (save checkpoint), `BRANCH` (fork for parallel exploration), `MERGE` (reconcile branches), `CONTEXT` (inspect current state)
- **Use case:** Multiple projects experiment with variations of shared base context without interfering with each other; stable contexts tagged and shared as baselines for new projects

### Google ADK Artifacts
- Named, versioned binary or text objects — agents reference artifacts by name and version instead of pasting large data into prompts
- Multiple agents across projects reference the same artifact version; storage scales independently of agent compute
- **Use case:** A large document processed once and stored as an artifact v1.0; every agent that needs it references the same artifact without re-processing

---

## 6. RAG Patterns for Agent Context

### RAG Architecture Taxonomy (2025/2026)

| Pattern | Description | When to Use |
|---------|-------------|-------------|
| **Standard RAG** | Fixed retrieve-then-generate | Simple factual lookup |
| **Agentic RAG** | Agent decides what/when to retrieve, loops until grounded | Complex multi-hop reasoning |
| **GraphRAG** | Entity-relation graph as retrieval path | Thematic questions across large corpora |
| **HippoRAG** | Mimics hippocampal indexing via Personalized PageRank | Efficient multi-hop reasoning |
| **LightRAG** | Combines knowledge graphs with vector retrieval | Both specific entity and thematic search |
| **HyDE** | Generates hypothetical ideal document, uses its embedding for retrieval | Sparse/zero-shot queries |
| **Hierarchical A-RAG** | Three retrieval tools at multiple granularities | Consistent 5–13% improvement over flat RAG |

### Recommended Cross-Project RAG Architecture

```
For cross-project agent context via RAG:

1. Global knowledge layer (vector DB + graph)
   - Organizational standards, past decisions, domain knowledge
   - Namespace: "global" or "shared"

2. Project-specific layer (same or separate index)
   - Project files, conversations, task history
   - Namespace: project_id

3. User/session layer (in-context or small vector store)
   - Recent turns, active task context
   - Evicted to project layer on session end

Retrieval query flow:
  a. Classify query scope (session / project / global)
  b. Retrieve from appropriate layer(s)
  c. Rerank and deduplicate across layers
  d. Pack into context window with tier-appropriate weighting
```

---

## 7. State Management for Long-Running Sessions

### LangGraph Stateful Architecture
- **State schema:** Typed Pydantic model defining all agent memory fields
- **Reducers:** Functions defining how state fields update (e.g., `add_messages` for deduplication by ID)
- **Checkpointers:** Persist complete graph state to PostgreSQL/Redis after every node execution — enables pause/resume and fault recovery
- **Use case:** A research agent that runs for hours, pauses for human review at decision gates, and resumes from exactly where it stopped

### Sub-Agent Context Decomposition (Best Practice)
- **Orchestrator agent:** Maintains high-level plan in a compact context (~5k tokens)
- **Specialist subagents:** Each takes a focused subtask with a clean, fresh context window (can explore extensively — tens of thousands of tokens)
- **Return contract:** Each subagent returns a condensed summary (1,000–2,000 tokens) to the orchestrator
- This sidesteps context limits by design rather than fighting them with compression techniques

### Techniques for Long-Running Sessions
1. **Intelligent pruning:** Drop oldest tool outputs when approaching context limit; preserve the dialogue flow
2. **Rolling summaries:** Periodically summarize completed work phases; replace verbose history with compact summaries
3. **External checkpoints:** Write interim results to persistent storage; re-inject selectively rather than keeping everything in-context
4. **Pause/resume:** Serialize full state; resume from checkpoint after human review or external resource availability

---

## 8. Multi-Tenant Context Isolation

### Isolation Model by Threat Level

| Threat Level | Model | Implementation |
|-------------|-------|----------------|
| **Low** (internal projects, trusted data) | Logical isolation | Namespaces, filtered queries, RBAC |
| **Medium** (SaaS, external customers) | Namespace + hardware boundary | Kubernetes namespaces + VM-level agent execution |
| **High** (finance, healthcare, regulated) | Physical isolation | Separate clusters, separate indexes, separate DBs |

### Core Isolation Principles

1. **Mandatory metadata injection:** Embed `{tenant_id, user_id, agent_id, session_id}` into every interaction at the *infrastructure layer* — enforced as a backend filter, not a suggestion to the LLM
2. **Vector DB namespace enforcement:** Unique namespace per tenant; stamp every vector with `tenant_id`; require tenant filter on *every* query — never rely on the LLM to maintain boundaries
3. **Memory system partitions:** All memory systems (Mem0, Zep, Letta) must enforce tenant-bound partitions. Shared memory without namespace enforcement is a **critical vulnerability**.
4. **Short-lived tokens:** Replace static API keys with ephemeral tokens. MCP enterprise IdP (SEP-990, Nov 2025 spec) enables SSO-integrated, auditable cross-project authorization.
5. **Kernel isolation warning:** Kubernetes namespaces share the host kernel — insufficient for hostile multi-tenancy. Use microVMs (Firecracker) or full VMs for code execution by agents.

### Threat Detection for Multi-Tenant Systems
Monitor for: abnormal authentication patterns, cross-tenant query attempts, excessive resource consumption per tenant, anomalous tool invocation volumes, high-branching delegation chains.

---

## 9. Decision Matrix

```
What is the primary use case?
├── Conversational memory across sessions → Mem0, Zep, Letta
├── Document/knowledge retrieval → RAG (Weaviate, Pinecone, Qdrant)
├── Relationship/reasoning across entities → Neo4j + GraphRAG, Graphiti
├── Cross-project skill reuse → Agent Skills (SKILL.md), MCP servers
└── Long-running stateful workflows → LangGraph + PostgreSQL checkpointer

What is the scale?
├── Prototype / small team → pgvector, Chroma, Letta local, Mem0 cloud
├── Production / single project → Qdrant/Weaviate + Mem0 or Zep
└── Enterprise / multi-project → Pinecone + Neo4j + Zep + MCP + OAuth

What is the tenant isolation requirement?
├── Internal/trusted → Namespace filtering (logical isolation)
├── SaaS customers → Namespace + VM execution boundary
└── Regulated industries → Physical isolation (separate indexes, clusters, DBs)

What is the context lifetime?
├── Single session only → LangGraph in-memory state
├── Cross-session persistence → Mem0, Zep, or Letta with external store
└── Indefinite organizational memory → Knowledge graph (Neo4j/Graphiti) + vector layer
```
