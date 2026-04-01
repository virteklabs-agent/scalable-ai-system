# Guardrails, Safety & Iterative Improvement

Building autonomous AI systems requires layered safety mechanisms, continuous evaluation, and iterative improvement loops. This document covers the tools, frameworks, and patterns that keep your system reliable, safe, and self-improving.

---

## Table of Contents

1. [Guardrail Frameworks](#1-guardrail-frameworks)
2. [Iterative Reasoning Patterns](#2-iterative-reasoning-patterns)
3. [Evaluation & Testing Frameworks](#3-evaluation--testing-frameworks)
4. [Red-Teaming & Adversarial Testing](#4-red-teaming--adversarial-testing)
5. [Alignment Techniques](#5-alignment-techniques)
6. [Hallucination Detection & Mitigation](#6-hallucination-detection--mitigation)
7. [Safety Architecture Patterns](#7-safety-architecture-patterns)
8. [Regulatory & Compliance Context](#8-regulatory--compliance-context)
9. [Layered Defense Architecture](#9-layered-defense-architecture)

---

## 1. Guardrail Frameworks

### NVIDIA NeMo Guardrails
**GitHub:** https://github.com/NVIDIA/NeMo-Guardrails  
**Purpose:** Programmable guardrails for LLM-powered applications using Colang DSL

**Key Features:**
- Define topical rails (block off-topic conversations)
- Input/output rails (filter harmful content before/after LLM calls)
- Dialogue rails (enforce conversation flows)
- Fact-checking rails (verify LLM responses against knowledge bases)
- Integrates with LangChain, OpenAI, local models

**Use Cases:**
- Customer service bots: prevent agents from discussing competitors, pricing, or legal matters outside approved scope
- Healthcare agents: enforce evidence-based responses only, block unverified medical claims
- Financial agents: prevent agents from making investment recommendations beyond their authorization level
- Code agents: block generation of malicious code patterns, enforce license compliance

**Example Colang Rail:**
```colang
define user ask about competitors
  "Tell me about [competitor]"
  "How does [competitor] compare?"

define flow
  user ask about competitors
  bot say "I can only discuss our own products. How can I help you today?"
```

---

### Guardrails AI
**GitHub:** https://github.com/guardrails-ai/guardrails  
**Purpose:** Output validation and structured data extraction from LLM responses

**Key Features:**
- RAIL (Reliable AI Language) spec for defining output schemas
- 50+ pre-built validators (PII detection, toxicity, URL validity, JSON schema)
- Retry logic with corrective prompting on validation failure
- Async streaming support
- Hub of community validators

**Use Cases:**
- API response generation: enforce JSON schema compliance, retry if malformed
- Document summarization: validate summaries don't introduce new facts (faithfulness check)
- PII-sensitive applications: automatically redact or block responses containing email addresses, SSNs, phone numbers
- Legal document agents: ensure all citations follow a valid format before returning to user

**Example:**
```python
from guardrails import Guard
from guardrails.hub import ToxicLanguage, DetectPII

guard = Guard().use_many(
    ToxicLanguage(threshold=0.5, on_fail="exception"),
    DetectPII(pii_entities=["EMAIL", "PHONE_NUMBER"], on_fail="fix")
)

validated_output = guard(
    openai.chat.completions.create,
    model="gpt-4o",
    messages=[{"role": "user", "content": prompt}]
)
```

---

### Meta Llama Guard 3
**Paper:** https://arxiv.org/abs/2312.06674  
**Purpose:** LLM-based content safety classifier trained on Meta's safety taxonomy

**Key Features:**
- Classifies both prompts and responses against 13 safety categories
- Categories: violence, hate speech, sexual content, criminal planning, CBRN, etc.
- Available as a model (Llama-Guard-3-8B, Llama-Guard-3-1B for edge)
- Can be fine-tuned on custom safety taxonomies
- Multilingual support (8 languages in v3)

**Use Cases:**
- Content moderation pipeline: run Llama Guard as a pre/post filter on all LLM interactions
- Agentic pipelines: validate tool call arguments before execution (e.g., block shell commands that look malicious)
- Audit logging: tag all interactions with safety scores for compliance reporting
- Fine-tune on domain-specific unsafe content (e.g., pharma-specific unsafe outputs)

---

### LLM Guard
**GitHub:** https://github.com/protectai/llm-guard  
**Purpose:** Open-source security toolkit for LLM interactions

**Key Features:**
- Input scanners: prompt injection detection, PII anonymization, toxicity, ban topics, ban substrings
- Output scanners: bias, deanonymization, relevance, factual consistency, regex patterns
- Runs locally (no external API calls)
- FastAPI integration for drop-in REST deployment

**Use Cases:**
- Prompt injection defense: detect and block attempts to override system prompts
- Compliance logging: scan all I/O and store results in audit trail
- Air-gapped deployments: runs fully on-premise, ideal for regulated industries

---

### Lakera Guard
**Website:** https://www.lakera.ai  
**Purpose:** Real-time prompt injection and jailbreak detection as an API

**Key Features:**
- Sub-100ms latency API
- Detects prompt injection, jailbreaks, PII leakage, toxic content
- Drop-in proxy mode — sits between your app and the LLM
- Continuous model updates as new attack patterns emerge

**Use Cases:**
- Production API proxy: route all LLM calls through Lakera for real-time screening without code changes
- Multi-tenant SaaS: prevent one tenant's users from injecting prompts that affect other tenants

---

### Guardrail Framework Comparison

| Framework | Type | Deployment | Best For | Latency |
|-----------|------|------------|----------|---------|
| NeMo Guardrails | Rule + LLM hybrid | Self-hosted | Dialogue flow control | Medium |
| Guardrails AI | Output validation | Self-hosted / SDK | Schema enforcement, retries | Low |
| Llama Guard 3 | LLM classifier | Self-hosted | Content safety taxonomy | Medium |
| LLM Guard | Scanner toolkit | Self-hosted | Prompt injection, PII | Low |
| Lakera Guard | API service | Cloud API | Real-time proxy screening | Very Low |

---

## 2. Iterative Reasoning Patterns

### ReAct (Reasoning + Acting)
**Paper:** https://arxiv.org/abs/2210.03629

Agents interleave **Thought → Action → Observation** cycles, allowing them to reason about tool results before taking the next step.

```
Thought: I need to find the current weather in London.
Action: search_weather(city="London")
Observation: 15°C, partly cloudy
Thought: The weather is mild. I should recommend a light jacket.
Final Answer: It's 15°C and partly cloudy in London — a light jacket is recommended.
```

**Use Case:** Any agent that needs to use tools (web search, APIs, databases) and reason about their outputs before proceeding.

---

### Reflexion
**Paper:** https://arxiv.org/abs/2303.11366  
**GitHub:** https://github.com/noahshinn/reflexion

Agents reflect on failures and store **verbal reinforcement signals** in episodic memory to avoid repeating mistakes across episodes.

**Feedback Loop:**
```
Attempt 1 → Fail → Reflect: "I forgot to handle the edge case where input is None"
Attempt 2 → Fail → Reflect: "The API rate limit caused a timeout; add retry logic"
Attempt 3 → Success
```

**Use Case:**
- Code generation agents that self-correct based on test failures
- Research agents that refine search strategies based on relevance scores
- Agentic systems that improve over long-running sessions without retraining

---

### Self-RAG
**Paper:** https://arxiv.org/abs/2310.11511

The LLM decides **when to retrieve**, critiques retrieved passages for relevance, and critiques its own output for faithfulness using special reflection tokens.

**Reflection Tokens:** `[Retrieve]`, `[Relevant]`, `[Irrelevant]`, `[Fully supported]`, `[Partially supported]`, `[No support]`

**Use Case:**
- Knowledge-intensive agents where blind retrieval degrades quality
- Medical/legal agents that must cite sources and verify factual support

---

### Chain-of-Thought (CoT) & Tree of Thoughts (ToT)
**CoT Paper:** https://arxiv.org/abs/2201.11903  
**ToT Paper:** https://arxiv.org/abs/2305.10601

| Pattern | Structure | Best For |
|---------|-----------|----------|
| Standard CoT | Linear step-by-step | Math, logic, simple reasoning |
| Self-Consistency CoT | Multiple CoT paths + majority vote | High-stakes decisions |
| Tree of Thoughts | Tree search over reasoning paths | Complex planning, strategy |
| Graph of Thoughts | DAG of reasoning operations | Multi-step problem solving |

**Use Case for ToT:** Multi-step code refactoring where multiple approaches need to be explored and scored before committing to one.

---

### Feedback Loop Architecture

```
┌─────────────────────────────────────────────────────┐
│                   TASK INPUT                        │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────▼────────┐
              │   AGENT PLAN    │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  EXECUTE TOOLS  │◄──────────────────┐
              └────────┬────────┘                   │
                       │                            │
              ┌────────▼────────┐                   │
              │  EVALUATE       │   [Retry/Refine]  │
              │  (Critic LLM /  ├───────────────────┘
              │   Test Runner / │
              │   RAGAS Score)  │
              └────────┬────────┘
                       │ [Pass]
              ┌────────▼────────┐
              │  REFLECT &      │
              │  STORE MEMORY   │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  FINAL OUTPUT   │
              └─────────────────┘
```

---

## 3. Evaluation & Testing Frameworks

### RAGAS
**GitHub:** https://github.com/explodinggradients/ragas  
**Purpose:** RAG-specific evaluation framework

**Metrics:**
- **Faithfulness:** Does the answer stick to retrieved context?
- **Answer Relevancy:** Is the answer relevant to the question?
- **Context Precision:** How much of retrieved context is actually needed?
- **Context Recall:** Was all necessary information retrieved?
- **Answer Correctness:** Does the answer match ground truth?

**Use Case:** Automated nightly evaluation of your RAG pipeline — alert when faithfulness drops below 0.8, indicating context drift or retrieval degradation.

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall

result = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_recall]
)
print(result)  # {'faithfulness': 0.92, 'answer_relevancy': 0.88, ...}
```

---

### DeepEval
**GitHub:** https://github.com/confident-ai/deepeval  
**Purpose:** Unit testing framework for LLM applications

**Key Features:**
- pytest-compatible — run LLM evals like unit tests in CI/CD
- 14+ built-in metrics: G-Eval, hallucination, bias, toxicity, RAGAS metrics
- Confident AI cloud dashboard for tracking eval history
- Supports multi-turn conversation testing

**Use Case:** Gate deployments in CI/CD — if any LLM test fails (e.g., hallucination score > 0.2), block the deployment automatically.

```python
from deepeval import assert_test
from deepeval.metrics import HallucinationMetric
from deepeval.test_case import LLMTestCase

def test_no_hallucination():
    test_case = LLMTestCase(
        input="What is the capital of France?",
        actual_output=llm_response,
        context=["France is a country in Western Europe. Its capital is Paris."]
    )
    metric = HallucinationMetric(threshold=0.2)
    assert_test(test_case, [metric])
```

---

### LangSmith
**Website:** https://smith.langchain.com  
**Purpose:** LangChain's observability, testing, and prompt management platform

**Key Features:**
- Trace every LangChain/LangGraph run with full token-level visibility
- Dataset management for regression testing
- Automated evaluators (correctness, conciseness, harmfulness)
- Prompt hub for versioned prompt management
- A/B testing different prompts/models

**Use Case:** After deploying a new model version, run your regression dataset through LangSmith to compare quality scores against the previous baseline before switching traffic.

---

### Arize Phoenix
**GitHub:** https://github.com/Arize-ai/phoenix  
**Purpose:** Open-source AI observability and evaluation platform

**Key Features:**
- OpenTelemetry-native (plug in LangChain, LlamaIndex, OpenAI, Anthropic auto-instrumentation)
- Embedding drift detection — visualize when input distribution shifts
- LLM eval templates for hallucination, relevance, toxicity
- Runs locally or on Arize cloud

**Use Case:** Detect when your agent's input embeddings drift from training distribution, signaling that the knowledge base or user queries have shifted and the agent needs retraining or prompt updates.

---

### Evaluation Framework Comparison

| Tool | Type | CI/CD | RAG Metrics | Agent Metrics | Deployment |
|------|------|-------|-------------|---------------|------------|
| RAGAS | Metric library | Scriptable | ✅ Native | Partial | OSS |
| DeepEval | Test framework | ✅ pytest | ✅ Included | ✅ Multi-turn | OSS + Cloud |
| LangSmith | Platform | ✅ Via SDK | ✅ | ✅ Full traces | Cloud |
| Arize Phoenix | Observability | Via webhooks | Partial | ✅ Traces | OSS + Cloud |
| Langfuse | Observability+Eval | ✅ Via SDK | ✅ | ✅ Full traces | OSS + Cloud |
| W&B Weave | Platform | ✅ | ✅ | ✅ | Cloud |

---

## 4. Red-Teaming & Adversarial Testing

### PyRIT (Microsoft)
**GitHub:** https://github.com/Azure/PyRIT  
**Purpose:** Python Risk Identification Toolkit for Generative AI red-teaming

**Key Features:**
- Automated multi-turn attack orchestration
- Attack strategies: crescendo, PAIR (Prompt Automatic Iterative Refinement), skeleton key
- Scoring with OpenAI, Azure Content Safety, or local classifiers
- Memory of successful attacks for iterative refinement
- Async support for high-volume testing

**Use Case:** Before deploying a new agent, run PyRIT's crescendo attack (gradually escalating requests) to find the threshold where your guardrails break, then harden them before production.

```python
from pyrit.orchestrator import CrescendoOrchestrator
from pyrit.prompt_target import OpenAIChatTarget

orchestrator = CrescendoOrchestrator(
    objective_target=OpenAIChatTarget(),
    adversarial_chat=OpenAIChatTarget(),
    max_turns=10
)
await orchestrator.apply_crescendo_attack_async(attack_strategy="How to make explosives")
```

---

### Garak (NVIDIA)
**GitHub:** https://github.com/NVIDIA/garak  
**Purpose:** LLM vulnerability scanner — the "nmap for LLMs"

**Key Features:**
- 100+ probe types: jailbreaks, prompt injection, encoding attacks, hallucination induction
- Tests against any model endpoint (OpenAI, HuggingFace, local)
- HTML/JSONL reports of vulnerabilities found
- Continuous integration friendly

**Use Case:** Run garak as part of your model evaluation pipeline before promoting any new LLM to production. Scan for known jailbreaks and track vulnerability scores over time.

```bash
python -m garak --model_type openai --model_name gpt-4o \
  --probes jailbreak,prompt_injection,encoding \
  --report_prefix my_model_scan
```

---

### ARTKIT (BCG X)
**GitHub:** https://github.com/BCG-X-Official/artkit  
**Purpose:** Automated Red-Teaming toolkit with async, multi-turn support

**Key Features:**
- Async-first for high-throughput adversarial testing
- Built-in augmentation: rephrase attacks, translate to bypass filters
- Supports multi-turn conversation attacks
- Integrates with major LLM providers

---

### Promptfoo
**GitHub:** https://github.com/promptfoo/promptfoo  
**Purpose:** Prompt testing, evaluation, and red-teaming CLI/CI tool

**Key Features:**
- YAML-based test definitions
- Compare multiple models/prompts side-by-side
- Built-in red-team attack generation
- CI integration with GitHub Actions
- Output graders: LLM-as-judge, regex, JSON schema, similarity

**Use Case:** A/B test a new system prompt against the current one across 200 test cases, measuring quality metrics and safety metrics simultaneously before deploying the update.

---

## 5. Alignment Techniques

### Constitutional AI (CAI)
**Paper:** https://arxiv.org/abs/2212.08073  
**Organization:** Anthropic

A set of natural language principles defines acceptable behavior. The model critiques and revises its own outputs against these principles without human labels for each revision.

**Use Case:** Define a custom constitution for your domain (e.g., "Never recommend actions that could harm the user financially") and use it to guide RLHF or direct critique-revision loops.

---

### RLHF / RLAIF / DPO

| Technique | Requires Human Labels | Compute | Notes |
|-----------|----------------------|---------|-------|
| RLHF | ✅ Expensive | High | Gold standard for alignment |
| RLAIF | ❌ (AI feedback) | Medium | Scales well, uses LLM as judge |
| DPO | Preference pairs | Low | No RL training, simpler pipeline |
| KTO | Binary feedback | Very Low | Even simpler than DPO |
| ORPO | None (SFT + align) | Low | Single training stage |

**Use Case for RLAIF in scalable systems:**  
Use GPT-4o or Claude as an AI labeler to generate preference pairs from your agent's outputs, then use DPO to fine-tune a smaller local model. This creates a self-improving loop without expensive human annotation at scale.

---

## 6. Hallucination Detection & Mitigation

### Detection Tools

| Tool | Method | Use Case |
|------|--------|----------|
| **SelfCheckGPT** | Sample consistency | Check if model hallucinates by sampling multiple times |
| **RAGAS Faithfulness** | NLI-based | Verify answer is grounded in retrieved context |
| **Vectara HHEM** | Fine-tuned classifier | Production hallucination scoring API |
| **TruLens** | Triad evaluation | RAG relevance, groundedness, coherence |

### Mitigation Strategies

1. **Retrieval Grounding:** Force the LLM to cite specific retrieved passages; use NLI to verify each claim against citations
2. **Self-Consistency Sampling:** Generate N responses, take majority vote on factual claims
3. **Confidence Calibration:** Prompt the model to express uncertainty; filter or flag low-confidence outputs
4. **Semantic Caching:** Cache verified responses for identical/near-identical queries — reduces hallucination surface area and costs
5. **Structured Output Enforcement:** Use JSON mode + schema validation to prevent unstructured hallucinations in data extraction tasks

---

## 7. Safety Architecture Patterns

### Circuit Breakers

Inspired by distributed systems, circuit breakers in AI systems detect repeated failures or safety violations and temporarily halt the offending agent or pipeline.

```python
class AgentCircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None

    def call(self, agent_fn, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError("Agent circuit is open — too many failures")
        try:
            result = agent_fn(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
```

**Use Case:** An optimizer agent that runs code repeatedly — if it triggers sandbox violations 5 times in a row, the circuit opens and an alert is sent to the human operator.

---

### Sandboxing

| Approach | Tool | Isolation Level | Use Case |
|----------|------|----------------|----------|
| Container | Docker, Podman | Process | Code execution agents |
| MicroVM | Firecracker, gVisor | Kernel | Untrusted code at scale |
| WASM sandbox | Wasmtime | Instruction | Plugin execution |
| E2B Code Interpreter | E2B SDK | Cloud VM | Safe code execution for agents |
| Modal | Modal SDK | Serverless VM | On-demand sandboxed execution |

**Use Case for E2B:** An autonomous code agent generates and executes Python to analyze data. Each execution runs in an E2B ephemeral VM that is destroyed after the task, preventing any persistent access to the host system.

---

### Rate Limiting & Quota Management

```yaml
# Example: Per-agent rate limits via LiteLLM Proxy
router_settings:
  num_retries: 3
  timeout: 30

model_list:
  - model_name: gpt-4o
    litellm_params:
      model: gpt-4o
      api_key: ${OPENAI_API_KEY}
      tpm: 100000   # tokens per minute limit
      rpm: 500       # requests per minute limit

# Per-team budget
general_settings:
  alerting: ["slack"]
  budget_duration: "30d"
  max_budget: 50.00  # $50/month hard cap
```

---

### Human-in-the-Loop (HITL) Kill Switches

```python
class HITLGate:
    """Pause agent execution for human review before high-stakes actions."""
    
    HIGH_STAKES_PATTERNS = [
        r"DELETE.*FROM",      # Database deletions
        r"rm -rf",            # File system destruction
        r"TRANSFER.*FUNDS",   # Financial transactions
        r"DEPLOY.*PRODUCTION"  # Production deployments
    ]
    
    async def check_action(self, proposed_action: str, agent_id: str) -> bool:
        for pattern in self.HIGH_STAKES_PATTERNS:
            if re.search(pattern, proposed_action, re.IGNORECASE):
                approval = await self.request_human_approval(
                    agent_id=agent_id,
                    action=proposed_action,
                    timeout=300  # 5 minute timeout
                )
                return approval
        return True  # Low-stakes: auto-approve
```

---

### Principle of Least Privilege for Agents

Each agent should only have access to the tools and data it needs:

```yaml
# Agent capability matrix
agents:
  research_agent:
    allowed_tools: [web_search, read_file]
    forbidden_tools: [write_file, execute_code, send_email]
    data_access: [public_knowledge_base]
    
  code_agent:
    allowed_tools: [read_file, write_file, execute_code_sandboxed]
    forbidden_tools: [web_search, send_email, database_write]
    data_access: [project_codebase]
    
  orchestrator:
    allowed_tools: [delegate_task, read_status, aggregate_results]
    forbidden_tools: [execute_code, database_write]
    data_access: [all_agent_outputs]
```

---

## 8. Regulatory & Compliance Context

| Framework | Scope | Key Requirements for AI Agents |
|-----------|-------|--------------------------------|
| **EU AI Act** | EU | Risk classification, transparency, human oversight for high-risk AI |
| **OWASP Top 10 for LLMs** | Global | Prompt injection, insecure output handling, training data poisoning |
| **MITRE ATLAS** | Global | Adversarial ML threat matrix — attack tactics and mitigations |
| **NIST AI RMF** | US | Govern, Map, Measure, Manage risk framework |
| **ISO/IEC 42001** | Global | AI Management System standard |
| **SOC 2 Type II** | US | Security, availability, confidentiality for SaaS AI products |

**Key compliance actions for this system:**
1. Maintain tamper-evident audit logs of all agent actions (OWASP LLM08)
2. Implement input validation to prevent prompt injection (OWASP LLM01)
3. Document AI system capabilities and limitations (EU AI Act Article 13)
4. Provide human override mechanisms (EU AI Act Article 14)
5. Run regular adversarial testing (NIST AI RMF - Measure)

---

## 9. Layered Defense Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER / CLIENT REQUEST                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│               LAYER 1: PERIMETER DEFENSE                        │
│  • Lakera Guard / LLM Guard — prompt injection detection        │
│  • Rate limiting (LiteLLM Proxy)                                │
│  • Authentication & authorization (API keys, JWT, OAuth)        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│               LAYER 2: INPUT GUARDRAILS                         │
│  • NeMo Guardrails — topical/dialogue rails                     │
│  • Guardrails AI — schema/PII validation                        │
│  • Llama Guard — safety category classification                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│               LAYER 3: AGENT EXECUTION SAFETY                   │
│  • Sandboxed code execution (E2B, Firecracker)                  │
│  • Least privilege tool access per agent                        │
│  • HITL gates for high-stakes actions                           │
│  • Circuit breakers for repeated failures                       │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│               LAYER 4: OUTPUT GUARDRAILS                        │
│  • Hallucination detection (RAGAS Faithfulness, Vectara HHEM)  │
│  • Output schema validation (Guardrails AI)                     │
│  • Content safety check (Llama Guard on output)                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────┐
│               LAYER 5: CONTINUOUS MONITORING                    │
│  • Arize Phoenix / Langfuse — drift detection, eval scoring    │
│  • Tamper-evident audit logs                                    │
│  • Regular adversarial testing (PyRIT, Garak)                  │
│  • RAGAS / DeepEval regression test suite in CI/CD             │
└─────────────────────────────────────────────────────────────────┘
```

---

*Last updated: April 2026 | [← Context & Memory](02-context-and-memory.md) | [Dashboards & Observability →](04-dashboards-and-observability.md)*
