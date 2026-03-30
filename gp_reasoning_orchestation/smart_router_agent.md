# Smart Router Agent — Design Document

> **Status:** Phase 1 COMPLETED (Approach A + F hybrid). See `README.md` for implementation details.
> **Implementation:** `router_agent.py` — 5-level × 6-task-type deterministic routing, zero LLM calls.
> **Test harness:** `check_router_agent.py` + `test_routing_agent.py` — 139 questions across 7 workflow groups.

## Original State (Before Phase 1)

```
Question → Keyword Match → Binary: yes/no reasoning → Pick 1 of 2 models
```

**Problems:**
- Binary decision (only "fast" or "reasoning") — no granularity
- Keyword matching is fragile ("how are you?" triggers "how" → falsely routes to reasoning model)
- No concept of reasoning **levels** — Claude Opus 4.6 is overkill for "compare X and Y"
- No model **fitness** consideration — some questions suit Gemini better, others suit Claude better

## Implemented State

```
Question → Two-Stage Deterministic Router → Reasoning Level (L0-L4) × Task Type (6 types) → Best Model
                                                                                             ↑
                                          ASP context + modification patterns ──────► select_for_task()
                                          (explicit PROGRAM_EXECUTION routing)        → SIG-PRE + Claude
```

**What was delivered (Phase 1 + Phase 1.5 + Phase 1.6):**
- 5-level reasoning classification (L0–L4) via regex + weighted scoring
- 6 task types: RETRIEVAL, MATH, COMPARISON, ANALYSIS, CREATIVE, PROGRAM_EXECUTION
- Two-stage pipeline: Stage 1 fast regex exit (<1ms) + Stage 2 multi-signal weighted scoring (<5ms)
- `select_for_task()` for explicit task type routing (used by ASP pipeline for PROGRAM_EXECUTION)
- `PROGRAM_EXECUTION_PATTERNS` — 7 regex patterns for ASP modification detection (exported, NOT in Stage 1/Stage 2)
- Gemini thinking config (model-aware: thinkingLevel for 3.x, thinkingBudget for 2.5)
- Authorized models validation + cross-provider fallback chain
- External context enrichment: config-gated, two-layer detection (RouterDecision + domain patterns), parallel web search (Perplexity → Gemini), token-budgeted, domain search context per workflow
- 139-question test harness across 7 workflow groups (including ASP reasoning + composition workflow)

---

## Step 1: How Hard Is The Question?

### Proposed Reasoning Levels

| Level | Name | Description | Your Example | Best Model Tier |
|-------|------|-------------|-------------|-----------------|
| **L0** | Direct Lookup | Answer is directly in the data, no reasoning | "Give me top 10 records ranking by profit" | Gemini 3.0 Flash |
| **L1** | Single-Step Reasoning | One transformation/filter/aggregation | "What is the total profit for Rheem in Scenario 2?" | Gemini 3.0 Flash / Claude 4.5 Haiku |
| **L2** | Multi-Step Reasoning | 2-5 steps, cross-reference, synthesis | "Compare profit margins across Scenario 2, 3, 4 and highlight trends" | Claude 4.6 Sonnet / Gemini 3.1 Pro |
| **L3** | Complex Analytical | Deep reasoning, multi-constraint, cause-effect | "Why is Scenario 3 underperforming and what trade-offs should we optimize?" | Claude 4.6 Opus / Gemini 3.1 Pro |
| **L4** | Creative/Strategic | Novel strategies, what-if design, open-ended judgment | "Design a pricing strategy that maximizes margin while maintaining market share" | Claude 4.6 Opus / Gemini 3.1 Pro |

---

## Step 2: All Possible Approaches to Classify Complexity

### Approach A — Enhanced Rule-Based (Multi-Signal Scoring) — IMPLEMENTED

> **Status:** IMPLEMENTED in `_stage2_full_scoring()`. 9 signal groups with weighted scoring, length bonus, intent bonus, ambiguity bonus.

**How:** Combine multiple deterministic signals into a weighted complexity score instead of just keyword matching.

```python
complexity_score = (
    w1 * question_type_score +        # "what is"=0.1, "why"=0.7, "what if"=0.9
    w2 * clause_count_score +          # more clauses = more complex
    w3 * reasoning_connector_score +   # "because", "therefore", "assuming"
    w4 * entity_density_score +        # more entities to relate = harder
    w5 * action_verb_score +           # Bloom's taxonomy verb mapping
    w6 * question_length_score +       # longer questions tend to be harder
    w7 * conditional_structure_score   # "if...then", "given that"
)
```

| Pros | Cons |
|------|------|
| Zero latency (<1ms) | Still surface-level, misses semantic difficulty |
| No external dependencies | Weight tuning requires experimentation |
| Fully deterministic & explainable | Can't handle paraphrased complexity |
| Easy to implement incrementally | |

**Implementation complexity:** Low (1-2 days)
**Latency overhead:** <1ms

---

### Approach B — spaCy NLP Feature Extraction

**How:** Parse the question with spaCy to extract structural features:
- Dependency tree depth (deeper = more complex)
- Number of clauses (via subordinating conjunctions `mark`, `advcl`)
- Named entity count
- Question word type (WH-classification)
- Verb complexity (Bloom's mapping)

```python
import spacy
nlp = spacy.load("en_core_web_sm")  # ~15MB model

def classify_complexity(question: str) -> int:
    doc = nlp(question)
    tree_depth = max(len(list(token.ancestors)) for token in doc)
    clause_count = sum(1 for t in doc if t.dep_ in ("advcl", "ccomp", "xcomp"))
    entity_count = len(doc.ents)
    # ... combine into score
```

| Pros | Cons |
|------|------|
| Captures syntactic structure | Requires spaCy dependency (~15MB) |
| Better than pure keywords | Still syntactic, not semantic |
| Fast (2-5ms) | Fragile to unusual phrasing |
| Academic backing | |

**Implementation complexity:** Low-Medium (2-3 days)
**Latency overhead:** 2-5ms

---

### Approach C — Embedding-Based Semantic Similarity (Semantic Router Pattern)

**How:** Pre-define example questions for each reasoning level. Embed them. At runtime, embed the incoming question, find the most similar examples, and use their level.

Inspired by **Aurelio's Semantic Router** (open-source, MIT license).

```python
# Define route templates
L0_examples = [
    "Give me top 10 records by profit",
    "Show all data for Rheem in Q3",
    "List the warranty claims for model X",
]
L2_examples = [
    "Compare profit margins across scenarios and highlight trends",
    "What patterns emerge in warranty claims over the past 3 years?",
]
L3_examples = [
    "Why is Scenario 3 underperforming relative to Scenario 2?",
    "What trade-offs should we consider to optimize pricing?",
]

# At runtime: embed question → cosine similarity → closest level
```

| Pros | Cons |
|------|------|
| Captures semantic meaning | Needs embedding model (can use local sentence-transformers) |
| Robust to paraphrasing | Quality depends on example set |
| Very fast after init (<10ms) | Need 20-50 examples per level |
| Battle-tested (Semantic Router has 2.5k+ GitHub stars) | |

**Implementation complexity:** Medium (3-5 days)
**Latency overhead:** 5-15ms

---

### Approach D — Lightweight LLM Classification (Small Model as Router)

**How:** Use the cheapest/fastest model (Gemini 3.0 Flash or Haiku) to classify the question before routing.

```python
ROUTER_PROMPT = """Classify this question's reasoning complexity:
- L0_LOOKUP: Direct data retrieval, sorting, filtering
- L1_SIMPLE: Single-step calculation or transformation
- L2_ANALYTICAL: Multi-step reasoning, comparison, synthesis
- L3_COMPLEX: Deep analysis, cause-effect, optimization
- L4_STRATEGIC: Creative design, what-if scenarios, open-ended judgment

Question: {question}
Output ONLY the level code (e.g., L2_ANALYTICAL)."""
```

| Pros | Cons |
|------|------|
| Best semantic understanding | Adds LLM call latency (200-500ms) |
| Handles edge cases naturally | Extra cost per question |
| Easy to iterate (just change prompt) | Recursive problem (LLM to route LLM) |
| No training data needed | Classification errors possible |

**Implementation complexity:** Low (1 day)
**Latency overhead:** 200-500ms

---

### Approach E — Cascade / Escalation Pattern (AutoMix / FrugalGPT style)

**How:** Always start with the cheapest model. Check confidence of the response. If low confidence → escalate to the next more expensive model.

```
Question → Flash generates answer
  → Confidence check (self-verification or meta-model)
    → High confidence? Return answer
    → Low confidence? → Sonnet generates answer
      → Still low? → Opus generates answer
```

Papers: **FrugalGPT** (2023), **AutoMix** (NeurIPS 2024) — demonstrated 50%+ cost reduction.

| Pros | Cons |
|------|------|
| Naturally cost-optimized | Extra latency for escalated queries (2x-3x) |
| No classification errors (actual model tries first) | Needs confidence mechanism |
| Simple queries stay maximally cheap | Redundant work on hard questions |
| Proven at scale (Amazon Bedrock uses this) | |

**Implementation complexity:** Medium (1-2 weeks)
**Latency overhead:** 0ms (easy questions) to 2x (hard questions)

---

### Approach F — Hybrid Two-Stage Router — IMPLEMENTED

> **Status:** IMPLEMENTED as Stage 1 (`_stage1_fast_exit()`) + Stage 2 (`_stage2_full_scoring()`). Stage 2 uses weighted scoring (Approach A) instead of embeddings. ~60% of questions exit at Stage 1.

**How:** Combine fast deterministic filter (Stage 1) with semantic classification (Stage 2).

```
Stage 1 (< 1ms): Rule-based patterns
  → Obviously simple (e.g., "list all...", "show me...", "top N...") → L0, use Flash
  → Obviously complex (e.g., "design...", "what if...", "optimize...") → L3+, use Opus
  → Ambiguous → Stage 2

Stage 2 (< 5ms): Multi-signal weighted scoring (Approach A)
  → 9 signal groups + length/intent/ambiguity bonuses
  → Classify into L0-L4
  → Select model from (Level × TaskType) routing table
```

| Pros | Cons |
|------|------|
| Fast for obvious cases (<1ms) | Two systems to maintain |
| Accurate for ambiguous cases | More code complexity |
| Best of both worlds | |
| Minimizes overhead | |

**Implementation complexity:** Medium (1 week)
**Latency overhead:** <1ms (obvious) to <5ms (ambiguous)

---

## Step 2b: Model Fitness — Which Model for Which Level?

Beyond complexity, we need to consider **which model is best** for the specific task type:

### Model Capability Matrix (Available Models)

| Model | Provider | Speed | Reasoning | Math | Synthesis | Creative | ASP/Symbolic | Cost |
|-------|----------|-------|-----------|------|-----------|----------|-------------|------|
| **Gemini 3.0 Flash** | Vertex AI | ★★★★★ | ★★☆ | ★★★ | ★★☆ | ★★☆ | — | $ |
| **Claude 4.5 Haiku** | Vertex AI | ★★★★☆ | ★★★ | ★★★ | ★★★ | ★★☆ | — | $ |
| **Claude 4.6 Sonnet** | Vertex AI | ★★★☆☆ | ★★★★ | ★★★★ | ★★★★ | ★★★★ | ★★★ | $$ |
| **Claude 4.5 Sonnet** | Vertex AI | ★★★☆☆ | ★★★★ | ★★★★ | ★★★★ | ★★★☆ | ★★☆ | $$ |
| **Gemini 3.1 Pro** | Vertex AI | ★★★☆☆ | ★★★★ | ★★★★★ | ★★★★ | ★★★ | — | $$ |
| **Gemini 2.5 Pro** | Vertex AI | ★★☆☆☆ | ★★★★ | ★★★★★ | ★★★★ | ★★★ | — | $$$ |
| **Claude 4.6 Opus** | Vertex AI | ★★☆☆☆ | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★ | $$$$ |
| **SIG-PRE 0.1** | GP (self-hosted) | ★★★★ | ★★★★★ | ★★★★★ | — | — | ★★★★★ | — |

> **SIG-PRE 0.1** is the GP Contextual Reasoning Engine — a deterministic ASP/Clingo solver. It is not an LLM. It is the primary execution engine for `PROGRAM_EXECUTION` tasks, with Claude Opus 4.6 as the rule generation assistant and recovery fallback.

### Implemented Routing Table (5 × 6)

| Level | RETRIEVAL | MATH | COMPARISON | ANALYSIS | CREATIVE | PROGRAM_EXECUTION |
|-------|-----------|------|------------|----------|----------|-------------------|
| **L0** | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | SIG-PRE 0.1 (+ Opus) |
| **L1** | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | Claude 4.5 Haiku | SIG-PRE 0.1 (+ Opus) |
| **L2** | Gemini 3.1 Pro | Gemini 3.1 Pro | Claude 4.6 Sonnet | Claude 4.6 Sonnet | Claude 4.6 Sonnet | SIG-PRE 0.1 (+ Opus) |
| **L3** | Gemini 3.1 Pro | Gemini 3.1 Pro | Claude 4.6 Opus | Claude 4.6 Opus | Claude 4.6 Opus | SIG-PRE 0.1 (+ Opus) |
| **L4** | Gemini 3.1 Pro | Gemini 3.1 Pro | Claude 4.6 Opus | Claude 4.6 Opus | Claude 4.6 Opus | SIG-PRE 0.1 (+ Opus) |

**Design principles:**
- Gemini handles structured data tasks (retrieval, math, data analysis). Claude handles language-quality tasks (comparison, creative, strategic analysis). This means the router needs **two dimensions**: Reasoning Level + Task Type.
- `PROGRAM_EXECUTION` always routes to SIG-PRE 0.1 (deterministic ASP solver) with Claude Opus 4.6 as assistant for rule generation + recovery on failure. `select_for_task()` enforces at least L3_COMPLEX.
- `PROGRAM_EXECUTION` is never auto-detected by Stage 1/Stage 2 — it requires explicit invocation via `select_for_task()` by the ASP pipeline (`BaseQAPipelineWithASP`) after verifying both ASP context AND modification patterns.

---

## Summary: All Approaches Compared

| Approach | Accuracy | Latency | Impl. Effort | Training Data? | Best For | Status |
|----------|----------|---------|---------------|----------------|----------|--------|
| **A. Multi-Signal Rules** | ★★★ | <1ms | 1-2 days | No | Quick win, immediate improvement | **IMPLEMENTED** (Stage 2) |
| **B. spaCy NLP** | ★★★☆ | 2-5ms | 2-3 days | No | Structural complexity | Not needed |
| **C. Semantic Router** | ★★★★ | 5-15ms | 3-5 days | Examples only | Semantic understanding | Future (Phase 2) |
| **D. LLM Classifier** | ★★★★★ | 200-500ms | 1 day | No | Best accuracy, highest latency | Not planned |
| **E. Cascade/AutoMix** | ★★★★★ | 0-2x | 1-2 weeks | No | Cost optimization | Not planned |
| **F. Hybrid Two-Stage** | ★★★★☆ | 2-15ms | 1 week | No | Best balance | **IMPLEMENTED** (Stage 1 + 2) |

---

## Phased Approach

### Phase 1: Approach A + F hybrid — COMPLETED
1. Replaced binary keyword matching with **multi-signal weighted scoring** (`_stage2_full_scoring()`) → produces L0-L4
2. Added **task type detection** — 6 types: RETRIEVAL, MATH, COMPARISON, ANALYSIS, CREATIVE, PROGRAM_EXECUTION
3. Mapped (Level × TaskType) → specific model via `_build_routing_table()` (5 × 6 table)
4. Two-stage pipeline: `_stage1_fast_exit()` (<1ms, ~60% of questions) + `_stage2_full_scoring()` (<5ms)
5. Authorized models validation + cross-provider fallback chain
6. Gemini thinking config (thinkingLevel for 3.x, thinkingBudget for 2.5)
7. Test harness: 139 questions across 7 workflow groups (+ pytest regression in `test_routing_agent.py`)

### Phase 1.5: GP Reasoning Engine (SIG-PRE) Integration — COMPLETED
1. Added `PROGRAM_EXECUTION` task type + `PROGRAM_EXECUTION_PATTERNS` (7 regex patterns)
2. Added `select_for_task()` for explicit task type routing (bypasses Stage 1/Stage 2 task detection)
3. `PROGRAM_EXECUTION` always maps to SIG-PRE 0.1 + Claude Opus 4.6 at all levels (enforces >= L3_COMPLEX)
4. Two-condition safety: ASP context + modification patterns must BOTH match (checked by `BaseQAPipelineWithASP`, not by the router)
5. `ASPProgramExecutionAgent` with `RuleGenerator`, Option C execution (SIG-PRE primary + Claude recovery)

### Phase 1.6: External Context Enrichment — COMPLETED
1. Config-gated external context (`enable_external_context`, `external_context_timeout`, `external_context_max_tokens`) in `GroundedAnswerEngineConfig`
2. Two-layer detection via `_needs_external_context(decision, question)`:
   - Layer 1: RouterDecision-based (CREATIVE/COMPARISON → yes; ANALYSIS + low confidence → yes; L3+ + low confidence → yes)
   - Layer 2: Domain-specific keyword patterns keyed by `ZeusTool` workflow_id (`_DOMAIN_PATTERNS_BY_WORKFLOW`)
3. Parallel execution: search fires via `asyncio.create_task` after `router.route()`, collected before LLM call — near-zero added latency
4. Search provider fallback: Perplexity primary (`ask_perplexity_async`) → Gemini fallback (`gemini_web_search_async`)
5. Domain search context enrichment: `_DOMAIN_SEARCH_CONTEXT_BY_WORKFLOW` appends company/industry/domain keywords to search queries
6. Token budget enforcement via `trim_nl_text` (tiktoken-based, sentence boundary preservation)
7. Supplementary context prompt injection — clearly marked block instructing LLM to treat evidence as PRIMARY source of truth
8. Extensible architecture: `_gather_external_sources` uses `asyncio.gather(return_exceptions=True)` — add new `_fetch_*_context` methods for KG, etc.
9. Enabled for Composition Workflow (`RheemPricingWarrantyCompositionWorkflowQAPipeline`)

### Phase 2 (Next iteration): Add Approach C — NOT YET
- Add **embedding similarity** (Semantic Router) for the ambiguous middle zone where rules struggle
- Production log analysis → tune scoring weights
- Router confidence → dynamic fallback depth

### Phase 3 (Future): Advanced Routing — NOT YET
- REASONING task type for multi-step logical chains
- A/B testing framework for routing decisions
- Workflow-specific complexity bonus (from real production data)
- Multi-agent execution routing (SQL, Python, Prolog, Constraint Programming alongside ASP)

---

## Key References

### Academic Papers
- [RouteLLM (ICLR 2025)](https://arxiv.org/abs/2406.18665) — 85% cost reduction while maintaining 95% quality
- [GraphRouter (ICLR 2025)](https://arxiv.org/abs/2410.03834) — GNN-based routing, generalizes to new LLMs
- [EmbedLLM (ICLR 2025 Spotlight)](https://arxiv.org/abs/2410.02223) — Compact model embeddings for routing
- [Router-R1 (NeurIPS 2025)](https://arxiv.org/abs/2506.09033) — RL-based multi-round routing
- [AutoMix (NeurIPS 2024)](https://arxiv.org/abs/2310.12963) — Cascade with self-verification, 50% cost reduction
- [FrugalGPT](https://arxiv.org/abs/2305.05176) — Cascade routing pioneer
- [OptiRoute](https://arxiv.org/abs/2502.16696) — FLAN-T5 based task analysis
- [Unified Routing and Cascading](https://arxiv.org/abs/2410.10347) — Combined framework
- [Select-then-Route (EMNLP 2025)](https://aclanthology.org/2025.emnlp-industry.28/) — Taxonomy-guided routing
- [C3PO: Optimized LLM Cascades](https://arxiv.org/pdf/2511.07396)
- [Survey on Question Difficulty Estimation (ACM 2022)](https://dl.acm.org/doi/10.1145/3556538)
- [NLP Approaches to Question Difficulty Estimation](https://arxiv.org/abs/2305.10236)
- [Fine-Tuned Small LLMs Outperform Zero-Shot (2024)](https://arxiv.org/abs/2406.08660)

### Open-Source Projects
- [RouteLLM](https://github.com/lm-sys/RouteLLM) — Drop-in routing framework by LMSys
- [LLMRouter Library](https://github.com/ulab-uiuc/LLMRouter) — 16+ routing algorithms unified
- [Router-R1](https://github.com/ulab-uiuc/Router-R1) — RL-based LLM router
- [GraphRouter](https://github.com/ulab-uiuc/GraphRouter) — Graph neural network router
- [EmbedLLM](https://github.com/richardzhuang0412/EmbedLLM) — Model embedding framework
- [Semantic Router by Aurelio](https://github.com/aurelio-labs/semantic-router) — Ultra-fast semantic routing
- [NVIDIA LLM Router Blueprint](https://github.com/NVIDIA-AI-Blueprints/llm-router) — Production-grade router
- [AutoMix](https://github.com/automix-llm/automix) — Automatic model mixing
- [RouterBench by Martian](https://github.com/withmartian/routerbench) — Benchmarking framework
- [awesome-ai-model-routing](https://github.com/Not-Diamond/awesome-ai-model-routing) — Curated list

### Commercial Services
- [Martian Model Router](https://route.withmartian.com/) — 300+ enterprise customers, 98% cost savings
- [Not Diamond](https://www.notdiamond.ai/) — Powers OpenRouter's auto mode
- [Unify AI](https://unify.ai/) — Quality/cost/latency sliders
- [OpenRouter Auto Router](https://openrouter.ai/docs/guides/routing/routers/auto-router) — Zero-config routing
- [Amazon Bedrock Intelligent Prompt Routing](https://aws.amazon.com/bedrock/intelligent-prompt-routing/) — 35% cost savings

### Blog Posts and Guides
- [LMSYS: RouteLLM Blog](https://lmsys.org/blog/2024-07-01-routellm/)
- [Swfte AI: Intelligent LLM Routing](https://www.swfte.com/blog/intelligent-llm-routing-multi-model-ai)
- [Latitude: Dynamic LLM Routing Tools and Frameworks](https://latitude.so/blog/dynamic-llm-routing-tools-and-frameworks)
- [MarkTechPost: Comprehensive Guide to LLM Routing](https://www.marktechpost.com/2025/04/01/a-comprehensive-guide-to-llm-routing-tools-and-frameworks/)
- [Portkey: Task-Based LLM Routing](https://portkey.ai/blog/task-based-llm-routing/)
- [AWS: Multi-LLM Routing Strategies](https://aws.amazon.com/blogs/machine-learning/multi-llm-routing-strategies-for-generative-ai-applications-on-aws/)
- [Choosing the right Claude model](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)
