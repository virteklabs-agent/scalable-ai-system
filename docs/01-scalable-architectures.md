# 01 — Scalable Multi-Agent Architectures

> How to design and run many agents in parallel, including agents that optimize the system itself.

---

## Table of Contents

1. [Multi-Agent Frameworks](#1-multi-agent-frameworks)
2. [Architectures for Parallel Agent Execution](#2-architectures-for-parallel-agent-execution)
3. [Agent Communication Protocols](#3-agent-communication-protocols)
4. [Self-Optimizing & Meta-Learning Systems](#4-self-optimizing--meta-learning-systems)
5. [Human-in-the-Loop vs Autonomous Agents](#5-human-in-the-loop-vs-autonomous-agents)
6. [Orchestration Patterns](#6-orchestration-patterns)
7. [Scalability Best Practices](#7-scalability-best-practices)
8. [Key Research Papers](#8-key-research-papers)

---

## 1. Multi-Agent Frameworks

### LangGraph
- **What it is:** Models agent workflows as stateful directed graphs (nodes = steps, edges = transitions). Supports cycles, branching, parallel subgraph execution, and checkpointing. Reached v1.0 in late 2025.
- **Use case:** Complex, stateful production workflows — e.g., a research pipeline that retrieves, reasons, verifies, and writes in a loop until confidence is met
- **Scalability:** LangGraph Platform provides managed horizontal scaling; state is persisted to PostgreSQL/Redis checkpointers
- **GitHub:** [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)

### AutoGen / AG2 (Microsoft)
- **What it is:** Structures multi-agent collaboration as conversations. Agents can take dynamic roles, delegate tasks, and include humans via a configurable proxy agent. Layered architecture: Core API (event-driven), AgentChat API (rapid prototyping), Extensions API.
- **Use case:** Dialogue-heavy coordination — e.g., a writing team where a Researcher, Editor, and Critic debate content quality before publishing
- **GitHub:** [microsoft/autogen](https://github.com/microsoft/autogen)

### CrewAI
- **What it is:** Role-based agent framework inspired by real-world teams. YAML-driven configuration. Each agent has a defined role, goal, and backstory. Growing enterprise adoption via CrewAI+.
- **Use case:** Structured business pipelines — e.g., a sales intelligence crew with a lead researcher, data enricher, and outreach writer
- **GitHub:** [crewAIInc/crewAI](https://github.com/crewAIInc/crewAI)

### OpenAI Agents SDK
- **What it is:** Lightweight production framework with three primitives — Agents (LLMs + tools), Handoffs (delegation to other agents), Guardrails (input/output validation). Built-in tracing.
- **Use case:** OpenAI-native workflows where simplicity and speed matter — e.g., a customer support agent that hands off to a billing specialist
- **Docs:** [openai.github.io/openai-agents-python](https://openai.github.io/openai-agents-python/)

### Agency Swarm
- **What it is:** Extends the OpenAI Agents SDK with organizational role structures and explicit directional communication flows. Multi-LLM support via LiteLLM (Claude, Gemini, Grok, Azure OpenAI).
- **Use case:** Org-chart-style agent teams — e.g., CEO agent delegates to Developer and QA agents with strictly enforced communication rules
- **GitHub:** [VRSEN/agency-swarm](https://github.com/VRSEN/agency-swarm)

### Google Agent Development Kit (ADK)
- **What it is:** Code-first Python toolkit with native A2A protocol support for cross-framework agent interoperability. Apache 2.0 licensed.
- **Use case:** Systems that need to mix agent frameworks — e.g., a LangGraph workflow that delegates a subtask to a CrewAI crew via A2A
- **GitHub:** [google/adk-python](https://github.com/google/adk-python)

### MetaGPT
- **What it is:** Encodes Standardized Operating Procedures (SOPs) into agent pipelines. Simulates a software development organization (PM, architect, engineer, QA). ICLR 2024 oral presentation (top 1.2%).
- **Use case:** Automated software development workflows; generating entire codebases from a single natural language specification
- **GitHub:** [FoundationAgents/MetaGPT](https://github.com/FoundationAgents/MetaGPT) — 59k+ stars

### Kyegomez/Swarms
- **What it is:** Enterprise-grade orchestration framework supporting Sequential, Concurrent, Hierarchical, and Mixture-of-Agents topologies through a single unified interface.
- **Use case:** High-throughput production deployments needing dynamic topology switching without code changes
- **GitHub:** [kyegomez/swarms](https://github.com/kyegomez/swarms)

### MassGen
- **What it is:** Open-source multi-agent scaling system that autonomously orchestrates frontier models to collaborate, reason, and produce high-quality results.
- **Use case:** Scaling quality through diversity — running many agents in parallel and building consensus across outputs
- **GitHub:** [massgen/MassGen](https://github.com/massgen/MassGen)

### Framework Comparison

| Framework | Architecture | State | Scalability | Best For |
|-----------|-------------|-------|-------------|----------|
| LangGraph | Graph state machine | Checkpointing | High | Complex stateful workflows |
| AutoGen / AG2 | Conversational | Conversation history | High | Dialogue, group decisions |
| CrewAI | Role-based teams | RAG + role memory | Medium-High | Business pipelines |
| OpenAI Agents SDK | Handoff chains | Stateless-first | High | Lightweight, OpenAI-native |
| Agency Swarm | Org-structure | Session persistence | Medium-High | Org-modeled teams |
| Google ADK | Code-first + A2A | Custom | High | Cross-framework interop |
| MetaGPT | SOP pipelines | Intermediate results | Medium | Software dev automation |
| Kyegomez/Swarms | Universal orchestrator | Topology-dependent | High | Enterprise, large-scale |

---

## 2. Architectures for Parallel Agent Execution

### Actor Model (Ray)
- **What it is:** Each agent is an independent "actor" with its own mailbox and state. Agents communicate exclusively via asynchronous message passing. No shared memory eliminates lock contention.
- **Why it scales:** Actors distribute transparently across machines; failure is isolated to the actor, not the system
- **Use case:** Running thousands of agents in parallel on a research or data generation task — e.g., generating a training dataset across 10,000 concurrent agent instances
- **Key paper:** Matrix (arXiv 2511.21686) demonstrates scaling to 10,000+ concurrent agent workflows using Ray Actors + Ray Serve + vLLM
- **Tool:** [ray-project/ray](https://github.com/ray-project/ray)

### Message Queue Architecture
- **What it is:** Agents publish/consume tasks via a message broker (Kafka, Redis Streams, SQS). Decouples producers from consumers.
- **Why it scales:** Brokers buffer workload spikes; agents scale horizontally by adding consumers; dead-letter queues handle failures gracefully; event sourcing enables full replay
- **Use case:** Async document processing pipeline — e.g., thousands of documents queued in Kafka, consumed by a pool of analysis agents at their own pace
- **Tools:** Apache Kafka, Redis Streams, AWS SQS/SNS, Google Pub/Sub

### Serverless / Container Orchestration
- **Serverless (Lambda, Cloud Run):** Zero idle cost, instant scale-out for stateless agents; best for event-triggered, short-lived tasks
- **Kubernetes with HPA:** GPU scheduling, horizontal pod autoscaling triggered by queue depth, rolling deployments
- **Use case:** Kubernetes for persistent long-running multi-agent systems; serverless for bursty, event-driven workloads

---

## 3. Agent Communication Protocols

The defining 2025 development: standardized interoperability protocols are replacing framework-specific integrations.

### Model Context Protocol (MCP) — Anthropic
- **What it is:** Standard for how agents integrate with external tools, APIs, and data sources. JSON-RPC 2.0 transport (inspired by the Language Server Protocol). OpenAI adopted MCP March 2025. Donated to Linux Foundation December 2025.
- **Role in scalability:** Vertical connector — reduces the M×N integration problem to M+N
- **Use case:** Any agent connecting to a shared database, file system, or API without writing custom integration code
- **Spec:** [modelcontextprotocol.io](https://modelcontextprotocol.io)

### Agent-to-Agent Protocol (A2A) — Google
- **What it is:** Governs peer coordination, negotiation, and delegation between autonomous agents across different frameworks and vendors. HTTP + SSE + JSON-RPC. Agents advertise capabilities via "Agent Cards" in JSON.
- **Role:** Horizontal connector — standardizes agent-to-agent interface, enabling cross-vendor multi-agent systems
- **Use case:** A LangGraph orchestrator delegating a specialized subtask to a CrewAI crew built by a different team or vendor
- **50+ launch partners:** Atlassian, Salesforce, SAP, ServiceNow

### Agent Communication Protocol (ACP) — IBM/BeeAI
- **Role:** Local-first, low-latency coordination for on-premise or edge deployments where cloud round-trips are unacceptable
- **Use case:** Private enterprise systems in regulated industries (finance, healthcare) that cannot route through cloud intermediaries

### Agent Network Protocol (ANP)
- **Role:** Decentralized agent discovery using DIDs (Decentralized Identifiers); no central registry
- **Use case:** Internet-scale agent marketplaces where agents discover and collaborate without central coordination

| Protocol | Layer | Transport | Key Use Case |
|----------|-------|-----------|-------------|
| MCP | Agent → Tool | JSON-RPC 2.0 | Tool and data access standardization |
| A2A | Agent → Agent | HTTP/SSE/JSON-RPC | Cross-framework delegation |
| ACP | Local agents | REST | Low-latency, privacy-first, on-prem |
| ANP | Decentralized | DID-based | Internet-scale agent discovery |

---

## 4. Self-Optimizing & Meta-Learning Systems

### Automated Design of Agentic Systems (ADAS) — ICLR 2025
- **What it is:** A meta-agent that automatically discovers and programs better agent architectures in code. Building blocks (chain-of-thought, memory structures, tool use, self-reflection) are composed and optimized automatically rather than hand-designed.
- **Use case:** Let the system continuously discover better versions of itself without manual prompt engineering iteration

### AFlow: Automating Agentic Workflow Generation — ICLR 2025 (oral, top 1.8%)
- **What it is:** Rather than manually designing execution graphs, AFlow searches the workflow space automatically. Discovered workflows consistently outperform hand-crafted ones.
- **Use case:** Bootstrap high-quality agent workflows from natural language task descriptions
- **GitHub:** Part of [FoundationAgents/MetaGPT](https://github.com/FoundationAgents/MetaGPT)

### Self-Evolving Agent Patterns
- **Reflexion (arXiv 2303.11366):** Agents generate verbal self-critiques of failures, store them in episodic memory, use them as context on next attempt. Boosted GPT-4 accuracy on HumanEval from 80% to 91%. No retraining required.
- **MEMRL:** Runtime reinforcement learning on episodic memory
- **MemEvolve:** Meta-evolution of agent memory systems
- **CoMAS (2025):** Co-evolving multi-agent systems — agents evolve collectively via RL-based inter-agent discussion, generating intrinsic rewards without external supervision
- **Self-Rewarding Language Models (ICML 2024):** Agents generate their own training signal
- **Resource:** [EvoAgentX/Awesome-Self-Evolving-Agents](https://github.com/EvoAgentX/Awesome-Self-Evolving-Agents)

### MetaAgent (ICML 2025)
- Automatically constructs multi-agent systems using finite state machines as the design language
- Shifts agent design from art to engineering; opens the door to search-based architecture optimization

---

## 5. Human-in-the-Loop vs Autonomous Agents

| Mode | Description | When to Use |
|------|-------------|-------------|
| **Human-in-the-loop (HITL)** | Human must approve before agent proceeds | Irreversible actions (financial, data deletion, deployments), regulated domains |
| **Human-on-the-loop (HOTL)** | Agent runs autonomously; human monitors and can intervene | Moderate-risk workflows with audit requirements |
| **Human-out-of-the-loop** | Fully autonomous | High-volume, low-risk, reversible, well-scoped tasks |

**The enterprise consensus: risk-based hybrid model**
1. Classify workflows by business impact, regulatory exposure, and failure tolerance
2. Route low-risk, high-volume work to autonomous agents
3. Use confidence-based routing to escalate ambiguous or high-impact cases to humans
4. Implement full audit logging at all decision points
5. Define clear override authority (who can pause or undo agent actions)
6. Create feedback loops where human corrections improve future agent behavior

> **Regulatory context:** EU AI Act Article 14 and NIST AI RMF mandate human oversight for high-risk AI systems. Gartner predicts 33% of enterprise software will include agentic AI by 2028 (up from <1% in 2024).

---

## 6. Orchestration Patterns

### Supervisor / Orchestrator-Worker
- Central orchestrator decomposes tasks, delegates to specialized workers, monitors, validates, and synthesizes
- **Best for:** Complex multi-domain workflows; easiest to debug and trace
- **Bottleneck:** Orchestrator becomes single point of failure at very high load
- **Supported by:** LangGraph (native), CrewAI (default mode)

### Hierarchical
- Tree structure: top-level manager → mid-level supervisors → leaf workers
- **Best for:** Large enterprise deployments with clearly separable domains and different governance requirements per unit
- **Fault tolerance:** Supervisor failures affect sub-trees only, not the entire system

### Peer-to-Peer (Mesh/Swarm)
- Agents operate as equals, communicate directly, negotiate task ownership
- **Best for:** Research simulations, swarm intelligence tasks, highly resilient systems
- **Fundamental limitation:** O(N²) communication complexity — 50 agents = 1,225 connections
- **Supported by:** AutoGen (group chats), OpenAI Swarm (handoffs)

### Pipeline / Sequential
- Fixed execution order; each agent processes the output of the previous
- **Best for:** ETL-style workflows, document processing, content generation (research → draft → review)

### Hybrid (Recommended for Production)
- Combines patterns: hierarchical control at top, peer-to-peer within teams, pipeline for sequential stages
- **Principle:** Start centralized and decentralize only where concrete scalability bottlenecks are measured

| Pattern | Fault Tolerance | Scalability | Communication Complexity | Debugging Ease |
|---------|----------------|-------------|------------------------|-----------------|
| Supervisor | Low (single PoF) | High (add workers) | O(N) | High |
| Hierarchical | Moderate (sub-tree) | High (multi-tier) | O(N log N) | Medium |
| Peer-to-Peer | High (no PoF) | Limited (O(N²)) | O(N²) | Low |
| Pipeline | Medium | Medium | O(N) | High |
| **Hybrid** | **Balanced** | **Best overall** | Variable | Medium |

---

## 7. Scalability Best Practices

1. **Design for statelessness** — stateless agents enable simple horizontal scaling with standard load balancers; externalize all state to Redis, PostgreSQL, or a vector DB
2. **Horizontal scaling** — containerize agents; use Kubernetes HPA triggered by queue depth or request rate; configure memory limits at 1–2 GB minimum with generous timeouts
3. **Load balancing** — round-robin for stateless agents; consistent hashing for stateful sessions
4. **Async processing** — message queues between agent layers decouple producers/consumers; backpressure prevents cascade failures during spikes
5. **Hard limits** — max agent iterations, total token budget per task, and execution timeouts to prevent runaway loops (Denial-of-Wallet attacks)
6. **Observability first** — instrument every agent with structured logs, distributed traces (OpenTelemetry), and metrics (token usage, latency, cost, error rate)
7. **Memory as architecture** — explicitly design short-term (in-context), episodic (session), long-term (persistent), and shared (cross-agent) memory tiers
8. **Governance & cost controls** — implement budget limits per agent per session; alert on anomalies; scope API keys per agent type

---

## 8. Key Research Papers

| Title | Venue | Key Finding | Link |
|-------|-------|-------------|------|
| Matrix: P2P Multi-Agent Data Generation | arXiv Nov 2025 | Ray-based P2P scales to 10,000s of agents; 2–15x throughput vs centralized | [arXiv:2511.21686](https://arxiv.org/abs/2511.21686) |
| Towards a Science of Scaling Agent Systems | arXiv Dec 2025 | First quantitative scaling laws for agents; hybrid coordination wins | [arXiv:2512.08296](https://arxiv.org/html/2512.08296v1) |
| ADAS: Automated Design of Agentic Systems | ICLR 2025 | Meta-agent can discover and program better agent architectures automatically | [EvoAgentX](https://github.com/EvoAgentX/Awesome-Self-Evolving-Agents) |
| AFlow: Automated Agentic Workflow Generation | ICLR 2025 (oral, top 1.8%) | Automated workflow search outperforms hand-crafted workflows | [MetaGPT](https://github.com/FoundationAgents/MetaGPT) |
| AgentsNet: Coordination in Multi-Agent LLMs | arXiv Jul 2025 | First benchmark scaling to 100+ agents; grounded in distributed computing theory | [arXiv:2507.08616](https://arxiv.org/html/2507.08616v1) |
| Multi-Agent Memory: CS Architecture Perspective | arXiv 2025 | 3-layer memory hierarchy; consistency is the key open problem | [arXiv:2603.10062](https://arxiv.org/html/2603.10062v1) |
| Survey: Orchestration of Multi-Agent Systems | arXiv Jan 2025 | Comprehensive taxonomy of architectures and enterprise adoption patterns | [arXiv:2601.13671](https://arxiv.org/html/2601.13671v1) |
| Survey: Agent Interoperability Protocols | arXiv May 2025 | First unified analysis of MCP, A2A, ACP, ANP | [arXiv:2505.02279](https://arxiv.org/html/2505.02279v1) |
| MetaGPT: Meta Programming for Multi-Agent Collaboration | ICLR 2024 (oral, top 1.2%) | SOP-driven architecture reduces hallucination in multi-step workflows | [OpenReview](https://openreview.net/forum?id=VtmBAGCN7o) |
