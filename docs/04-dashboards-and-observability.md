# Dashboards, Observability & Management Tools

A scalable AI system is only as good as your ability to see what it's doing, understand why it's behaving a certain way, and intervene when needed. This document covers the full observability stack — from LLM traces to cost dashboards to no-code admin panels.

---

## Table of Contents

1. [AI Observability Platforms](#1-ai-observability-platforms)
2. [Agent-Specific Monitoring](#2-agent-specific-monitoring)
3. [Dashboard & Admin UI Frameworks](#3-dashboard--admin-ui-frameworks)
4. [Prompt Management](#4-prompt-management)
5. [API Gateway & Cost Management](#5-api-gateway--cost-management)
6. [CMS & Context Management UIs](#6-cms--context-management-uis)
7. [Infrastructure Observability](#7-infrastructure-observability)
8. [OpenTelemetry for AI](#8-opentelemetry-for-ai)
9. [Full Platform Comparison Matrix](#9-full-platform-comparison-matrix)

---

## 1. AI Observability Platforms

### Langfuse
**GitHub:** https://github.com/langfuse/langfuse  
**Purpose:** Open-source LLM observability, evaluation, and prompt management

**Key Features:**
- Full trace visualization: spans, token counts, latency, costs per step
- Prompt versioning and A/B testing built in
- Dataset management for automated evaluations
- LLM-as-judge evaluators with custom scoring functions
- Self-hostable (Docker) or cloud (langfuse.com)
- SDKs: Python, JavaScript, OpenAI drop-in, LangChain auto-instrumentation

**Use Cases:**
- **Production debugging:** A user reports an incorrect answer. Trace the exact span where the retrieval returned irrelevant context and the LLM hallucinated the answer.
- **Cost attribution:** Break down token costs by agent, project, user, or team to identify expensive workflows.
- **Prompt A/B testing:** Deploy two prompt versions to 50/50 of traffic and compare quality scores automatically.
- **Regression testing:** After every code push, run your evaluation dataset and receive a Slack alert if any metric degrades.

**Integration Example:**
```python
from langfuse.decorators import observe, langfuse_context

@observe()
def research_agent(query: str) -> str:
    langfuse_context.update_current_observation(
        metadata={"agent": "research", "project": "virtek-labs"}
    )
    # ... agent logic ...
    return response
```

---

### Arize Phoenix
**GitHub:** https://github.com/Arize-ai/phoenix  
**Purpose:** Open-source AI observability with embedding-native analysis

**Key Features:**
- OpenTelemetry-native — auto-instruments OpenAI, Anthropic, LangChain, LlamaIndex
- Embedding drift visualization (UMAP projections)
- Span-level evaluation with built-in templates
- Prompt playground for interactive debugging
- Runs locally on a laptop or deployed as a service

**Use Cases:**
- **Embedding drift detection:** Visualize when your users' questions are clustering in new regions not covered by your knowledge base — time to update the RAG corpus.
- **Latency analysis:** Identify which tool call in a multi-step agent pipeline accounts for 80% of latency.
- **Session debugging:** Reconstruct a full multi-turn conversation with all intermediate tool calls and reasoning steps.

---

### W&B Weave
**Website:** https://wandb.ai/site/weave  
**Purpose:** Weights & Biases' LLM evaluation and tracing platform

**Key Features:**
- Call tracing with automatic cost/latency tracking
- Dataset versioning and evaluation pipelines
- Model comparison dashboards
- Integrates with W&B Sweeps for hyperparameter optimization of prompts
- Leaderboard-style eval comparison

**Use Cases:**
- **Model comparison:** Compare Claude 3.5 Sonnet vs GPT-4o vs Llama 3.1 70B across your specific task distribution before committing to a model.
- **Continuous eval pipeline:** Every PR triggers a Weave evaluation run; results are posted as GitHub PR comments.

---

### Maxim AI
**Website:** https://www.getmaxim.ai  
**Purpose:** End-to-end AI quality platform for product teams

**Key Features:**
- No-code evaluation workflow builder
- Agent simulation for testing before deployment
- Real-time production monitoring with anomaly detection
- Human review queues for flagged outputs
- Supports multi-agent pipeline evaluation

**Use Cases:**
- **QA for non-engineers:** Product managers can define evaluation criteria in natural language without writing code.
- **Agent simulation:** Test how your multi-agent pipeline handles 1,000 synthetic user scenarios before launch.

---

### Comet Opik
**GitHub:** https://github.com/comet-ml/opik  
**Purpose:** Open-source LLM evaluation and tracing (Comet ML's OSS offering)

**Key Features:**
- Automatic tracing for OpenAI, LangChain, LlamaIndex
- Built-in hallucination, moderation, and PII detection evaluators
- Online evaluation — score production traces in real time
- Integrates with Comet ML for full MLOps pipeline

---

### AgentOps
**GitHub:** https://github.com/AgentOps-AI/agentops  
**Purpose:** Observability specifically designed for agent frameworks

**Key Features:**
- One-line integration: `agentops.init(api_key=...)`
- Tracks agent sessions, LLM calls, tool executions, errors
- Replay agent sessions in a visual timeline
- Cost and token analytics per agent session
- Supports LangChain, AutoGen, CrewAI, OpenAI Agents SDK

**Use Cases:**
- **Agent debugging:** An autonomous agent ran for 2 hours and produced an incorrect output. Replay the full session in AgentOps to find where it went wrong.
- **Multi-agent cost tracking:** See exactly how much each agent in your CrewAI crew costs per task execution.

```python
import agentops
agentops.init(api_key="your-key")
# All subsequent LLM calls and agent actions are automatically tracked
```

---

## 2. Agent-Specific Monitoring

### Memory Store UIs

| Memory System | Dashboard/UI | Key Metrics to Monitor |
|---------------|-------------|------------------------|
| **Mem0** | Mem0 Dashboard (cloud) | Memory hit rate, memory freshness, entity counts |
| **Zep** | Built-in REST + Grafana | Session count, fact extraction rate, memory search latency |
| **Letta (MemGPT)** | ADE (Agent Development Environment) | Memory block utilization, context window usage |
| **Graphiti** | Neo4j Browser | Graph node/edge counts, query latency, entity resolution |

### What to Monitor for Agents

```yaml
key_agent_metrics:
  performance:
    - task_completion_rate
    - average_turns_per_task
    - tool_call_success_rate
    - p50_p95_p99_latency
  quality:
    - hallucination_score_mean
    - user_feedback_rating
    - goal_achievement_rate
    - context_utilization_ratio
  cost:
    - tokens_per_task
    - cost_per_task
    - tool_call_frequency
    - cache_hit_rate
  safety:
    - guardrail_trigger_rate
    - circuit_breaker_activations
    - hitl_intervention_rate
    - anomaly_detection_alerts
```

---

## 3. Dashboard & Admin UI Frameworks

### Dify
**GitHub:** https://github.com/langgenius/dify  
**Purpose:** Open-source LLM application development platform with visual workflow builder

**Key Features:**
- Visual drag-and-drop workflow editor for agent pipelines
- Built-in RAG pipeline with multiple embedding/retrieval options
- Knowledge base management with document upload and chunking
- Prompt IDE with versioning
- API/webhook deployment of workflows
- Built-in analytics and conversation logs
- Self-hostable with Docker Compose

**Use Cases:**
- **Context management dashboard:** Upload and manage your shared skills library through Dify's knowledge base UI without writing code.
- **Workflow visualization:** Non-technical team members can view, understand, and modify agent workflows through the visual editor.
- **Rapid prototyping:** Build and test new agent patterns in hours using the visual builder before implementing them in code.

---

### Flowise
**GitHub:** https://github.com/FlowiseAI/Flowise  
**Purpose:** Drag-and-drop UI for building LangChain/LlamaIndex flows

**Key Features:**
- Node-based visual editor (300+ LangChain components as nodes)
- Export flows as API endpoints
- Chatbot embed widget
- Agentflow v2 for agentic workflows with loops and conditionals
- Self-hostable

**Use Cases:**
- **Prototyping shared context pipelines:** Visually wire together Mem0 → LangChain Agent → Vector DB → Response before writing production code.
- **Non-developer workflow creation:** Business users build their own data retrieval workflows using no-code interface.

---

### Grafana
**Website:** https://grafana.com  
**Purpose:** Universal observability and dashboard platform

**Key Features:**
- Connects to 100+ data sources (Prometheus, Elasticsearch, PostgreSQL, CloudWatch)
- Pre-built dashboards for Kubernetes, LiteLLM, and custom metrics
- Alerting with Slack/PagerDuty/email integration
- Grafana Loki for log aggregation
- Annotation support for marking deployment events on charts

**Use Cases:**
- **System-level AI monitoring:** Combine GPU utilization, LLM latency metrics, and agent task completion rates in a single unified dashboard.
- **Cost alerting:** Alert when daily LLM spend exceeds budget thresholds.
- **Agent fleet monitoring:** Track which agents are active, idle, or failing across your entire deployment.

**Recommended Grafana Dashboard for LLM Systems:**
```
Panels:
1. Request Rate (req/min) — by agent type
2. P95 Latency — by agent and model
3. Token Consumption — hourly, by project
4. Error Rate — by error type (guardrail, timeout, model error)
5. Cost Accumulation — running total vs budget
6. Active Agent Sessions — real-time count
7. Memory Operations — reads vs writes to shared context store
8. Queue Depth — pending tasks in message queue
```

---

### Appsmith
**GitHub:** https://github.com/appsmithorg/appsmith  
**Purpose:** Open-source low-code platform for internal tools and dashboards

**Key Features:**
- Drag-and-drop UI builder connected to APIs, databases, GraphQL
- 45+ pre-built widgets (tables, charts, forms, file uploads)
- JavaScript logic between UI components
- Git sync for version-controlled apps
- Role-based access control

**Use Cases:**
- **Context registry dashboard:** Build an internal UI to view, edit, and version your shared skills/contexts stored in your centralized database — without building a custom frontend from scratch.
- **Agent task manager:** Create an admin panel to view running agent tasks, their status, memory usage, and manually trigger retries.
- **Approval workflows:** Build HITL interfaces where operators can approve or reject high-stakes agent actions.

---

### Retool
**Website:** https://retool.com  
**Purpose:** Commercial low-code internal tool builder

**Key Features:**
- Similar to Appsmith but with more polish and enterprise SSO
- AI Components: built-in AI table, AI text input, Retool AI for connecting to any LLM
- Pre-built components for common admin patterns

**Use Cases:** Same as Appsmith — Retool is the commercial alternative with better enterprise support.

---

## 4. Prompt Management

### LangSmith Prompt Hub
**Purpose:** Versioned prompt management integrated with LangChain observability

**Key Features:**
- Push/pull prompts by name and version tag
- A/B test prompts by sending traffic to different versions
- Commit history and diffing between prompt versions
- Associate eval results with specific prompt versions

**Use Case:** Your system prompt for the research agent lives in LangSmith Hub at `virtek-labs/research-agent-v3`. When you update it, the old version is preserved and you can roll back instantly if quality metrics drop.

```python
from langchain import hub

prompt = hub.pull("virtek-labs/research-agent", api_key=LANGSMITH_KEY)
# Or pin to a specific version:
prompt = hub.pull("virtek-labs/research-agent:v3", api_key=LANGSMITH_KEY)
```

---

### PromptLayer
**Website:** https://promptlayer.com  
**Purpose:** Prompt versioning, management, and monitoring

**Key Features:**
- Track every prompt call with metadata and scores
- Visual prompt editor with version history
- Template variables with type enforcement
- Team collaboration on prompts

---

### Agenta
**GitHub:** https://github.com/Agenta-AI/agenta  
**Purpose:** Open-source LLM application management platform

**Key Features:**
- Prompt playground with parameter comparison
- Custom evaluation pipeline builder
- A/B testing with traffic splitting
- Self-hostable

**Use Case:** Teams experiment with 5 different prompt variants simultaneously, scoring each against your ground-truth dataset, then promote the winner to production with a single click.

---

### Langfuse Prompt Management
**Purpose:** Integrated into Langfuse's observability platform

**Key Features:**
- Link prompts directly to traces for attribution
- See which prompt version produced which trace
- Stage management: development → staging → production
- Cached prompt fetching for low-latency retrieval

---

## 5. API Gateway & Cost Management

### LiteLLM Proxy
**GitHub:** https://github.com/BerriAI/litellm  
**Purpose:** Unified OpenAI-compatible proxy for 100+ LLM providers

**Key Features:**
- Single endpoint, consistent API regardless of underlying model
- Load balancing across multiple providers/models
- Per-team/per-key budget enforcement
- Built-in retry logic, fallbacks between providers
- Prometheus metrics export
- Virtual keys with spend tracking

**Use Cases:**
- **Model routing:** Route cheap tasks to Haiku/Gemini Flash, complex tasks to GPT-4o/Claude Sonnet — automatically, based on cost or latency thresholds.
- **Budget enforcement:** Assign each project/team a monthly token budget; LiteLLM enforces it automatically and sends alerts.
- **Provider redundancy:** If OpenAI is down, automatically failover to Anthropic with zero code changes.

```yaml
# litellm_config.yaml
model_list:
  - model_name: fast-model
    litellm_params:
      model: gemini/gemini-2.0-flash
  - model_name: fast-model  # Fallback
    litellm_params:
      model: claude-haiku-3-5

router_settings:
  routing_strategy: "cost-based-routing"
  fallbacks: [{"fast-model": ["gpt-4o-mini"]}]
```

---

### Portkey
**Website:** https://portkey.ai  
**GitHub:** https://github.com/Portkey-AI/gateway  
**Purpose:** AI gateway with guardrails, observability, and reliability features

**Key Features:**
- 250+ model integrations via unified API
- Real-time guardrails (content filtering, regex, custom validators)
- Semantic caching to reduce costs
- Prompt versioning and management
- Virtual keys for access control
- Detailed analytics dashboard built in

**Use Cases:**
- **Semantic caching:** Cache semantically similar questions and return stored answers, reducing costs by 20-40% for repetitive queries.
- **Request tracing:** Every API call gets a unique trace ID automatically — link to your observability platform without additional instrumentation.

---

### Kong AI Gateway
**Website:** https://konghq.com/products/kong-ai-gateway  
**Purpose:** Enterprise API gateway with AI-specific plugins

**Key Features:**
- AI Proxy plugin: unified LLM endpoint
- AI Rate Limiting, AI Request Transformer, AI Response Transformer
- Semantic Caching plugin
- Prompt Injection Detection plugin
- Full Kong Gateway enterprise features (mTLS, OAuth, plugins ecosystem)

**Use Cases:**
- **Enterprise deployment:** Large organizations with existing Kong deployments can add AI capabilities without a new gateway layer.
- **Compliance:** Use Kong's audit logging and mTLS to meet enterprise security requirements for AI traffic.

---

### Helicone
**Website:** https://www.helicone.ai  
**GitHub:** https://github.com/Helicone/helicone  
**Purpose:** LLM observability and cost management — change one URL to instrument

**Key Features:**
- One-line integration (change base URL to Helicone's proxy)
- Automatic cost tracking per user/session/custom property
- Request caching and rate limiting
- Custom properties for grouping by team/project/feature
- Prompt templates with versioning

**Use Case:** Add `base_url="https://oai.helicone.ai/v1"` and every OpenAI call is automatically tracked, costed, and visible in the Helicone dashboard. Zero code changes required.

---

## 6. CMS & Context Management UIs

### Directus
**GitHub:** https://github.com/directus/directus  
**Purpose:** Headless CMS / data platform with MCP-native integration

**Key Features:**
- Instant REST + GraphQL API over any SQL database
- MCP server built-in — Claude and other agents can directly read/write the CMS via MCP protocol
- Fine-grained role-based access control per collection and item
- File management with metadata
- No-code data model editing
- Webhooks and event-driven automation flows

**Use Cases:**
- **Skills/context registry:** Store your shared agent skills, best practices, and contexts as structured data in Directus. Agents access them via MCP server without custom API code.
- **Knowledge base management:** Non-technical team members manage content through Directus's UI; agents consume it via API or MCP.
- **Context versioning:** Use Directus's revision tracking to maintain a full history of every change to shared contexts.

```json
// Directus collection: agent_skills
{
  "id": "security-code-review",
  "name": "Security Code Review",
  "version": "1.2.0",
  "tags": ["security", "code-review", "owasp"],
  "instructions": "1. Check for SQL injection...\n2. Verify input validation...",
  "frameworks": ["any"],
  "last_updated": "2026-03-15"
}
```

---

### Payload CMS
**GitHub:** https://github.com/payloadcms/payload  
**Purpose:** TypeScript-native headless CMS with full code control

**Key Features:**
- Define collections in TypeScript/code (not GUI) — great for developers
- Built-in REST + GraphQL API
- Access control functions per field/collection
- Rich relationship management
- Plugin ecosystem

**Use Cases:**
- **Developer-controlled context registry:** Define agent skills and contexts as typed TypeScript schemas with full IDE support; deploy as an API.
- **Tight Next.js integration:** If your dashboard is a Next.js app, Payload can be embedded directly in the same codebase.

---

### Strapi
**GitHub:** https://github.com/strapi/strapi  
**Purpose:** Popular open-source headless CMS

**Key Features:**
- Visual content type builder
- REST and GraphQL APIs auto-generated
- Plugin marketplace (i18n, email, upload, etc.)
- Role-based access control

**Use Case:** Teams already familiar with Strapi can use it as the backend for their context registry, with agents consuming contexts via REST API.

---

## 7. Infrastructure Observability

### Datadog
**Purpose:** Enterprise-grade full-stack observability

**AI-specific features:**
- LLM Observability module: trace LLM calls, evaluate quality, track costs
- APM for instrumenting agent Python/Node services
- Log Management with ML-based anomaly detection
- AI-powered alert correlation

**Use Case:** Large-scale production deployment where you need unified infra + AI observability in one platform with enterprise SLA support.

---

### New Relic
**Purpose:** Full-stack observability with built-in AI assistant (NRQL)

**AI-specific features:**
- AI monitoring module for LLM traces
- Entity relationships showing how agents connect to services/databases
- Intelligent alerting with ML baseline detection

---

### OpenLIT
**GitHub:** https://github.com/openlit/openlit  
**Purpose:** Open-source GPU and LLM observability using OpenTelemetry

**Key Features:**
- Auto-instruments OpenAI, Anthropic, LangChain, LlamaIndex, Pinecone, ChromaDB
- GPU monitoring (NVIDIA DCGM integration)
- OpenTelemetry-native — export to any OTLP-compatible backend
- Self-hostable dashboard

**Use Cases:**
- **GPU + LLM co-monitoring:** See GPU utilization alongside LLM call latency to identify bottlenecks in self-hosted model deployments.
- **Vendor-neutral:** Export to Grafana, Jaeger, Tempo, or any OTLP backend — no lock-in.

---

## 8. OpenTelemetry for AI

OpenTelemetry is the emerging standard for AI observability. The GenAI semantic conventions define standardized attribute names for LLM traces.

**Key Semantic Conventions:**
```
gen_ai.system           = "openai" | "anthropic" | "aws.bedrock"
gen_ai.request.model    = "gpt-4o" | "claude-3-5-sonnet"
gen_ai.request.max_tokens
gen_ai.response.model
gen_ai.usage.input_tokens
gen_ai.usage.output_tokens
gen_ai.operation.name   = "chat" | "text_completion" | "embeddings"
```

**OpenTelemetry-Native AI Libraries:**
- **OpenLLMetry / Traceloop:** https://github.com/traceloop/openllmetry — auto-instruments LLM SDKs, exports to any OTLP backend
- **OpenLIT:** GPU + LLM metrics via OTel
- **Arize Phoenix:** Full OTel support
- **Langfuse:** Accepts OTel traces via its SDK

**Architecture:**
```
Agent Code
    │
    │ (auto-instrumentation)
    ▼
OpenTelemetry SDK
    │
    │ OTLP/HTTP or OTLP/gRPC
    ▼
OTel Collector
    ├──► Langfuse (traces + evals)
    ├──► Grafana Tempo (trace storage)
    ├──► Prometheus (metrics)
    └──► Elasticsearch (logs)
```

---

## 9. Full Platform Comparison Matrix

| Tool | Category | Self-Host | Agents | RAG | Cost Track | Eval | Open Source |
|------|----------|-----------|--------|-----|------------|------|-------------|
| Langfuse | Observability | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Arize Phoenix | Observability | ✅ | ✅ | ✅ | Partial | ✅ | ✅ |
| AgentOps | Agent monitoring | ✅ | ✅ Native | ❌ | ✅ | ❌ | ✅ |
| W&B Weave | ML + LLM platform | ❌ | ✅ | ✅ | ✅ | ✅ | Partial |
| Comet Opik | Observability | ✅ | ✅ | ✅ | Partial | ✅ | ✅ |
| Maxim AI | Quality platform | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ |
| Dify | App builder | ✅ | ✅ | ✅ Native | Partial | ✅ | ✅ |
| Flowise | Visual builder | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| Appsmith | Admin UI | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| LiteLLM | API Gateway | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ |
| Portkey | API Gateway | ✅ | ❌ | ❌ | ✅ | Partial | ✅ |
| Helicone | Cost monitoring | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ |
| Directus | CMS / Context DB | ✅ | Via MCP | ❌ | ❌ | ❌ | ✅ |
| Grafana | Infrastructure | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ |
| OpenLIT | OTel observability | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |

---

## Recommended Starter Stack

```
For a production scalable AI system:

Observability:    Langfuse (self-hosted) — traces, evals, prompts
Agent Monitoring: AgentOps — agent session replay
Infrastructure:   Grafana + Prometheus — system metrics
API Gateway:      LiteLLM Proxy — model routing + budgets
Context UI:       Directus — shared context/skills management
App Builder:      Dify — workflow visualization + no-code iteration
Cost Tracking:    Helicone — per-user/per-project cost breakdown
```

All of the above are open-source and self-hostable, giving you full data control while providing enterprise-grade capabilities.

---

*Last updated: April 2026 | [← Guardrails & Safety](03-guardrails-and-safety.md) | [GitHub Resources →](05-github-resources.md)*
