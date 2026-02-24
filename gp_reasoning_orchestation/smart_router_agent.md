# Smart Router Agent — All Possible Solutions (Level 1)

## Current State

```
Question → Keyword Match → Binary: yes/no reasoning → Pick 1 of 2 models
```

**Problems:**
- Binary decision (only "fast" or "reasoning") — no granularity
- Keyword matching is fragile ("how are you?" triggers "how" → falsely routes to reasoning model)
- No concept of reasoning **levels** — Claude Opus 4.6 is overkill for "compare X and Y"
- No model **fitness** consideration — some questions suit Gemini better, others suit Claude better

## Target State

```
Question → Complexity Analysis → Reasoning Level (L0-L4) → Model Fitness Scoring → Best Model
```

---

## Step 1: How Hard Is The Question?

### Proposed Reasoning Levels

| Level | Name | Description | Your Example | Best Model Tier |
|-------|------|-------------|-------------|-----------------|
| **L0** | Direct Lookup | Answer is directly in the data, no reasoning | "Give me top 10 records ranking by profit" | Gemini 3.0 Flash |
| **L1** | Single-Step Reasoning | One transformation/filter/aggregation | "What is the total profit for Rheem in Scenario 2?" | Gemini 3.0 Flash / Claude 4.5 Haiku |
| **L2** | Multi-Step Reasoning | 2-5 steps, cross-reference, synthesis | "Compare profit margins across Scenario 2, 3, 4 and highlight trends" | Claude 4.6 Sonnet / Gemini 3.0 Pro |
| **L3** | Complex Analytical | Deep reasoning, multi-constraint, cause-effect | "Why is Scenario 3 underperforming and what trade-offs should we optimize?" | Claude 4.6 Opus / Gemini 3.1 Pro |
| **L4** | Creative/Strategic | Novel strategies, what-if design, open-ended judgment | "Design a pricing strategy that maximizes margin while maintaining market share" | Claude 4.6 Opus / Gemini 3.1 Pro |

---

## Step 2: All Possible Approaches to Classify Complexity

### Approach A — Enhanced Rule-Based (Multi-Signal Scoring)

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

### Approach F — Hybrid Two-Stage Router

**How:** Combine fast deterministic filter (Stage 1) with semantic classification (Stage 2).

```
Stage 1 (< 2ms): Rule-based patterns
  → Obviously simple (e.g., "list all...", "show me...", "top N...") → L0, use Flash
  → Obviously complex (e.g., "design...", "what if...", "optimize...") → L3+, use Opus
  → Ambiguous → Stage 2

Stage 2 (5-15ms): Embedding similarity OR spaCy features
  → Classify into L0-L4
  → Select model
```

| Pros | Cons |
|------|------|
| Fast for obvious cases (<2ms) | Two systems to maintain |
| Accurate for ambiguous cases | More code complexity |
| Best of both worlds | |
| Minimizes overhead | |

**Implementation complexity:** Medium (1 week)
**Latency overhead:** 2ms (obvious) to 15ms (ambiguous)

---

## Step 2b: Model Fitness — Which Model for Which Level?

Beyond complexity, we need to consider **which model is best** for the specific task type:

### Model Capability Matrix (Your Available Models)

| Model | Speed | Reasoning | Math | Synthesis | Creative | Cost |
|-------|-------|-----------|------|-----------|----------|------|
| **Gemini 3.0 Flash** | ★★★★★ | ★★☆ | ★★★ | ★★☆ | ★★☆ | $ |
| **Claude 4.5 Haiku** | ★★★★☆ | ★★★ | ★★★ | ★★★ | ★★☆ | $ |
| **Claude 4.6 Sonnet** | ★★★☆☆ | ★★★★ | ★★★★ | ★★★★ | ★★★★ | $$ |
| **Claude 4.5 Sonnet** | ★★★☆☆ | ★★★★ | ★★★★ | ★★★★ | ★★★☆ | $$ |
| **Gemini 3.1 Pro** | ★★★☆☆ | ★★★★ | ★★★★★ | ★★★★ | ★★★ | $$ |
| **Gemini 2.5 Pro** | ★★☆☆☆ | ★★★★ | ★★★★★ | ★★★★ | ★★★ | $$$ |
| **Claude 4.6 Opus** | ★★☆☆☆ | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | $$$$ |

### Proposed Routing Table

| Level | RETRIEVAL | MATH | COMPARISON | ANALYSIS | CREATIVE |
|-------|-----------|------|------------|----------|----------|
| **L0** | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash |
| **L1** | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | Gemini 3.0 Flash | Claude 4.5 Haiku |
| **L2** | Gemini 3.0 Pro | Gemini 3.0 Pro | Claude 4.6 Sonnet | Claude 4.6 Sonnet | Claude 4.6 Sonnet |
| **L3** | Gemini 3.1 Pro | Gemini 3.1 Pro | Claude 4.6 Opus | Claude 4.6 Opus | Claude 4.6 Opus |
| **L4** | Gemini 3.1 Pro | Gemini 3.1 Pro | Claude 4.6 Opus | Claude 4.6 Opus | Claude 4.6 Opus |

**Principle**: Gemini handles structured data tasks (retrieval, math, data analysis). Claude handles language-quality tasks (comparison, creative, strategic analysis). This means the router needs **two dimensions**: Reasoning Level + Task Type.

---

## Summary: All Approaches Compared

| Approach | Accuracy | Latency | Impl. Effort | Training Data? | Best For |
|----------|----------|---------|---------------|----------------|----------|
| **A. Multi-Signal Rules** | ★★★ | <1ms | 1-2 days | No | Quick win, immediate improvement |
| **B. spaCy NLP** | ★★★☆ | 2-5ms | 2-3 days | No | Structural complexity |
| **C. Semantic Router** | ★★★★ | 5-15ms | 3-5 days | Examples only | Semantic understanding |
| **D. LLM Classifier** | ★★★★★ | 200-500ms | 1 day | No | Best accuracy, highest latency |
| **E. Cascade/AutoMix** | ★★★★★ | 0-2x | 1-2 weeks | No | Cost optimization |
| **F. Hybrid Two-Stage** | ★★★★☆ | 2-15ms | 1 week | No | Best balance |

---

## Recommended Phased Approach

### Phase 1 (Immediate — Level 1): Approach A + F hybrid
1. Replace binary keyword matching with **multi-signal weighted scoring** → produces L0-L4
2. Add **task type detection** (math/calculation vs. comparison/synthesis vs. creative) using simple patterns
3. Map (Level, TaskType) → specific model from your bucket
4. Keep existing fallback chain as safety net

### Phase 2 (Next iteration): Add Approach C
- Add **embedding similarity** (Semantic Router) for the ambiguous middle zone where rules struggle

### Phase 3 (Level 2 — future): GP Reasoning Engine
- Integrate GP Reasoning Engine as an option alongside LLMs in the routing table

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
