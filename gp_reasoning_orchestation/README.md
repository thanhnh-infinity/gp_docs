# Q&A Framework V2 — Technical Report

## Overview

The Q&A Framework is a grounded reasoning system that answers user questions based on structured workflow outputs. It ensures answers are derived solely from stored evidence (not external knowledge) while maintaining traceability and accuracy.

All Q&A pipelines inherit from `BaseQAPipeline` (`qa_base_pipeline.py`), which provides a unified interface for the three-step pipeline: **Plan → Retrieve → Answer**. Workflow-specific behavior is configured via `QAToolConfig` returned by each pipeline subclass.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Q&A Framework V2                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BaseQAPipeline.solve()  ──────────────────  BaseQAPipelineWithASP.solve()  │
│  │  (data-schema workflows)                  │  (ASP reasoning workflows)   │
│  │                                           │                              │
│  │  ┌──────────────────────────────────┐     │  ┌────────────────────────┐  │
│  ├─▶│  1. SemanticQueryPlannerFactory  │     ├─▶│  1. _build_query_plan  │  │
│  │  │    ├─ LLMOntologyDrivenPlanner   │     │  │  (bypass: no planner)  │  │
│  │  │    ├─ DeterministicQueryPlanner   │     │  └────────────────────────┘  │
│  │  │    └─ HybridQueryPlanner (TODO)   │     │           ↓                  │
│  │  └──────────────────────────────────┘     │  ┌────────────────────────┐  │
│  │       ║  (parallel)                       ├─▶│  2. _retrieve_evidence │  │
│  │  ┌──────────────────────────────────┐     │  │  (ASP program context) │  │
│  │  │  ReasoningProtocolDesigner       │     │  └────────────────────────┘  │
│  │  └──────────────────────────────────┘     │           ↓                  │
│  │       ↓                                   │  ┌────────────────────────┐  │
│  │  ┌──────────────────────────────────┐     ├─▶│  3. _generate_answer   │  │
│  ├─▶│  POST-PROCESS: _optimize_plan   │     │  │  Two-condition check:  │  │
│  │  │    ├─ Shared data partition dedup│     │  │  ├─ is_asp_program_ctx │  │
│  │  │    └─ Partition max_rows override│     │  │  ├─ PROGRAM_EXEC ptrns │  │
│  │  └──────────────────────────────────┘     │  │  │   ↓ both match      │  │
│  │       ↓                                   │  │  │ ASPProgramExecAgent  │  │
│  │  ┌──────────────────────────────────┐     │  │  │   ├─ RuleGenerator   │  │
│  ├─▶│  2. StructuredDataRetriever     │     │  │  │   ├─ SIG-PRE solver  │  │
│  │  │  (Symbolic execution — pandas)   │     │  │  │   ├─ Recovery        │  │
│  │  └──────────────────────────────────┘     │  │  │   └─ to_nlg() fmt   │  │
│  │       ↓                                   │  │  │   ↓ patterns miss   │  │
│  │  ┌──────────────────────────────────────┐ │  │  └─ GroundedAnswerEng  │  │
│  ├─▶│  3. GroundedAnswerEngine             │ │  └────────────────────────┘  │
│  │  │  ├─ RouterAgent (5×6 model routing)  │ │           ↓                  │
│  │  │  │   ├─ Stage 1: Fast regex (<1ms)   │ │                              │
│  │  │  │   ├─ Stage 2: Weighted scoring    │ │                              │
│  │  │  │   ├─ authorized_models validation │ │                              │
│  │  │  │   └─ Cross-provider fallback      │ │                              │
│  │  │  └─ External Context (config-gated)  │ │                              │
│  │  │      ├─ 2-layer detection            │ │                              │
│  │  │      ├─ Perplexity → Gemini fallback │ │                              │
│  │  │      └─ Parallel, token-budgeted     │ │                              │
│  │  └──────────────────────────────────────┘ │                              │
│  │       ↓                                   │                              │
│  └─▶ _append_response_metadata()             └─▶ _append_response_metadata()│
│                                                                             │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. BaseQAPipeline (`qa_base_pipeline.py`)

**Purpose:** Abstract base class that provides the unified Q&A pipeline for all workflows.

**Responsibilities:**
- Orchestrates the 3-step pipeline: Plan → Retrieve → Answer
- Applies working memory optimizations post-planning
- Passes `GroundedAnswerEngineConfig` (from `QAToolConfig`) to the answer engine
- Appends response metadata (timestamp, response time)
- Falls back to Perplexity when no workflow run is found

Each workflow implements:
- `get_workflow_qa_specific_config()` → returns `QAToolConfig`
- `get_workflow_id()` → returns workflow identifier

### 2. SemanticQueryPlannerFactory (`semantic_query_planner_factory.py`)

**Purpose:** Strategy pattern to select the query planning approach.

**Strategies:**

| Strategy | Class | Description |
|----------|-------|-------------|
| `LLM_ONTOLOGY_DRIVEN` | `LLMOntologyDrivenQueryPlanner` | LLM-based partition/field/record planning |
| `DETERMINISTIC` | `DeterministicQueryPlannerWrapper` | BM25 + rules, no LLM calls |
| `HYBRID` | `HybridQueryPlanner` | Deterministic first, LLM fallback (not yet implemented) |

Both planners return the same `SemanticQueryPlan` type, making them interchangeable.

#### 2a. LLM Ontology-Driven Planner (`semantic_query_planner.py`)

**Capabilities:**
- Batch mode: Single LLM call for all partitions (fewer API calls)
- Parallel mode: Per-partition LLM calls (more granular)
- Important partitions fast path: Skip LLM for critical partitions (e.g., summary tables)
- Field selection: LLM identifies relevant columns from large datasets
- Record plan: LLM determines row retrieval strategy (mode + filters)

#### 2b. Deterministic Query Planner (`deterministic_query_planner.py`)

**Purpose:** Fast, rule-based planning without LLM calls (Phase A implementation, tuned in Phase A.1).

**Sub-components:**
- `DeterministicSearchIndex` — BM25 lexical + optional embedding search for section selection (used in search mode only)
- `DeterministicFieldSelector` — Token/synonym/description scoring with content quality gate + adaptive thresholds for field selection
- `DeterministicRecordPlanner` — Rule-based intent detection with broad question patterns + forward-looking sort preference

**Key design decisions (from Phase A.1 tuning):**
- **Suggestion mode includes ALL suggested partitions** — no BM25 relevance gate. The content quality gate in field selection decides `mode="none"` vs real fields per section. This was the single biggest accuracy win (~15% gain).
- **Content quality gate** (`content_quality_gate=0.18`): At least one field must have meaningful content relevance (excluding structural importance) for a section to get fields. Prevents irrelevant sections from polluting the plan.
- **Config-driven domain affinity**: Domain keywords and section prefixes are externalized to `SemanticQueryPlannerConfig` — new domains scale without code changes.
- **Numbered variant routing**: Detects "scenario N" references to ensure matching sections are included.

**Measured accuracy** (vs Gemini 3.0 Flash reference, 40% section Jaccard + 60% field Jaccard):

| Workflow | Accuracy |
|----------|----------|
| Competitive Pricing | 83.4% |
| Warranty Intelligence | 81.8% |
| Pricing Simulator | 62.5% |
| Composition | 67.7% |

**When to use:** For workflows where latency matters and questions follow predictable patterns. Currently at 80%+ for 2/4 workflows.

### 3. Working Memory Optimizer (`_optimize_query_plan` in `qa_base_pipeline.py`)

**Purpose:** Post-process the semantic query plan to reduce duplicate data and optimize token usage.

**Optimizations applied:**

| Optimization | Config Field | Description |
|-------------|-------------|-------------|
| Shared data dedup | `shared_data_partitions` | When both primary and secondary partitions are active, sets secondary to `mode="none"` |
| Max rows override | `partition_max_rows` | Override default `max_rows=200` for specific partitions when `mode="all"` |

### 4. ReasoningProtocolDesigner (`reasoning_protocol_designer.py`)

**Purpose:** Generate a dynamic reasoning protocol based on the question and ontology.

**Capabilities:**
- Runs in **parallel** with query planning (no latency penalty)
- Produces a `protocol_markdown` that guides the answer engine's reasoning
- Controlled by `enable_reasoning_protocol` config flag
- Graceful degradation: falls back to static protocol on failure

### 5. StructuredDataRetriever (`structured_data_retriever.py`)

**Purpose:** Extract relevant data based on the semantic query plan using symbolic execution (pandas).

**Record modes:**

| Mode | Description |
|------|-------------|
| `all` | Return all rows (capped by `max_rows`, default 200), optionally sorted by `sort_by` field |
| `top_k` | Return top K rows sorted by a field |
| `filter` | Return rows matching filter conditions (==, !=, contains, >, <, etc.) |
| `aggregate` | Group-by with aggregations (sum, count, mean, min, max) |
| `none` | Skip this partition (no data retrieved) |

**Guardrails:**
- Row capping: `mode="all"` respects `max_rows` to prevent token explosion
- Relevance ordering: `mode="all"` with `sort_by` sorts rows by the most analytically important field before capping (uses `_apply_top_k` internally)
- Field projection: Returns only selected columns
- Fail-safe: Configurable behavior on bad query plans (`fail_open_on_bad_plan`)
- `skip_reason` logging when a partition is intentionally skipped by optimizer

### 6. GroundedAnswerEngine (`grounded_answer.py`)

**Purpose:** Generate final answers grounded in retrieved evidence.

**Capabilities:**
- Non-negotiable rules: No fabrication, no external knowledge, no contradiction
- Token budget: Max 200K input tokens with overflow handling
- **Structured Map-Reduce**: Automatic chunking when evidence exceeds `max_input_tokens`, with structured JSON intermediate output for reasoning preservation (see section below)
- Repair mechanism: Retries with context if first attempt fails
- **Smart RouterAgent**: Selects the optimal model via 5-level × 6-task-type routing (see section below)
- **External Context Enrichment**: Optional web search context for questions needing market/industry background (see section below)

**Integration with RouterAgent:**
```python
router = RouterAgent(
    workflow_id=self.workflow_id,
    cfg=RouterConfig(
        authorized_models=self.cfg.authorized_models,
        fallback_chain=self.cfg.fallback_chain,
    ),
)
decision = router.route(question)
selected_model = decision.selected_model
```

**Fallback building (two layers):**
1. **Router per-decision fallbacks** — cross-provider alternatives at the same reasoning level (max 3)
2. **Config fallback_chain** — remaining models appended as safety net, preserving cross-provider order

### 7. Fallback to Public Search (`_fallback_to_public_search` in `qa_base_pipeline.py`)

**Purpose:** When no workflow run is found (expired session or wrong tool routed), fall back to Perplexity web search with a gentle disclaimer.

**Two failure cases covered:**
1. **Workflow session expired** — user's workflow data is no longer available in Redis
2. **Wrong tool selected** — question was routed to a workflow that doesn't match the user's intent

**Disclaimer format:**
```
> **Workflow data is not available for this analysis**
>
> This can happen for a couple of reasons:
> - **Your workflow session may have expired.** Re-running the workflow will restore access to your data.
> - **Your question may have been directed to a different analysis.** Try refining your question...
>
> In the meantime, here is what we found from publicly available sources:
```

### 8. Response Metadata (`_append_response_metadata` in `qa_base_pipeline.py`)

**Output format (italic markdown at bottom of answer):**
```
---
*Response generated at 2026-02-13 15:30:45 | Response time: 2.3s*
```

---

## External Context Enrichment

> **File:** `engine/grounded_answer.py`
> **Config:** `question_answering_configurations.py` → `GroundedAnswerEngineConfig`

### Overview

Some questions (e.g., "Why didn't copper prices drop?", "How does our pricing compare to the market?") benefit from external context (market trends, industry benchmarks) that the workflow data doesn't contain. The External Context Enrichment feature optionally enriches the grounded answer prompt with web search results — without slowing down questions that don't need it.

**Design principles:**
- **Config-gated:** `enable_external_context = False` by default → zero overhead for existing pipelines
- **Smart two-layer detection:** Piggybacks on the existing `RouterAgent.route()` decision + domain-specific keyword patterns
- **Parallel execution:** Search fires concurrently with prompt building → near-zero added latency
- **Grounding preserved:** Search context is clearly marked as SUPPLEMENTARY — the LLM is instructed to never let it override evidence data
- **Token-budgeted:** External context is capped by `external_context_max_tokens` using tiktoken-based truncation (`trim_nl_text`)

### Config Fields

| Parameter | Default | Description |
|-----------|---------|-------------|
| `enable_external_context` | `False` | When True, allows the Q&A workflow to fetch external knowledge (but doesn't mean it always will) |
| `external_context_timeout` | `5.0` | Max seconds to wait for external sources |
| `external_context_max_tokens` | `2000` | Token budget cap for external context injected into prompt |

### Two-Layer Detection (`_needs_external_context`)

**Layer 1 — RouterDecision-based (task type + confidence signals):**

| Condition | Triggers? |
|-----------|-----------|
| `task_type` is CREATIVE or COMPARISON | Yes |
| `task_type` is ANALYSIS and `confidence < 0.6` | Yes |
| `reasoning_level >= L3_COMPLEX` and `confidence < 0.6` | Yes |
| RETRIEVAL, MATH with normal confidence | No |

**Layer 2 — Domain-specific keyword patterns:**

Regex patterns keyed by `workflow_id` via `_DOMAIN_PATTERNS_BY_WORKFLOW` (uses `ZeusTool` enum as keys). Each workflow gets its own curated set of patterns. Only workflows with registered patterns trigger this layer.

| Workflow | Pattern Category | Example Triggers |
|----------|-----------------|------------------|
| Composition + Competitive Pricing | Supply chain & market | commodity index, copper price, forex, inflation, tariff, industry benchmark |
| Warranty Intelligence | Safety & failure | recall, CPSC, failure rate benchmark |
| Pricing Simulator | Market & competition | market trend, competitor pricing, industry average |

**Detection summary:**

| TaskType | Confidence | Domain Pattern Match? | Search? |
|----------|-----------|----------------------|---------|
| RETRIEVAL | any | No | No |
| MATH | any | No | No |
| ANALYSIS | >= 0.6 | No | No |
| ANALYSIS | < 0.6 | — | **Yes** |
| CREATIVE | any | — | **Yes** |
| COMPARISON | any | — | **Yes** |
| any | any | **Yes** | **Yes** |

### Search Provider Fallback

```
_fetch_web_search_context(query)
    │
    ├─ Primary: ask_perplexity_async(query)
    │   └─ Returns: answer + citations
    │
    └─ Fallback: gemini_web_search_async(query)
        └─ Returns: answer + key_points
```

If Perplexity fails (error/empty), automatically falls back to Gemini web search. If both fail, returns None — answer proceeds without external context.

### Domain Search Context Enrichment

Raw user questions may lack domain context needed for good search results. The `_DOMAIN_SEARCH_CONTEXT_BY_WORKFLOW` dictionary (keyed by `ZeusTool` enum) provides compact company/industry/domain metadata per workflow, appended to the search query:

```
User question: "Why didn't copper prices drop?"
Enriched query: "Why didn't copper prices drop? [Context: company: Rheem Manufacturing | industry: HVAC, water heating, air conditioning | domain: pricing, supply chain, warranty, competitive intelligence | competitors: AO Smith, Carrier, Trane, Lennox | channels: Home Depot, Lowe's, wholesale distributors]"
```

This enrichment is only used to steer the search — never injected into the final grounded answer prompt.

### Execution Flow

```
answer() called
    │
    ├─ router.route(question) → decision        (already exists, 0ms added)
    │
    ├─ enable_external_context = False?
    │   └─ YES → skip entirely (current behavior)
    │
    ├─ _needs_external_context(decision, question)?
    │   ├─ Layer 1: task_type + confidence check
    │   └─ Layer 2: domain-specific keyword patterns
    │   └─ NO → skip (RETRIEVAL, MATH, etc.)
    │
    ├─ YES → asyncio.create_task(_fetch_external_context)
    │         └─ _gather_external_sources(question)
    │              └─ _build_search_query(question, domain_context)
    │              └─ _fetch_web_search_context(enriched_query)
    │                   ├─ Perplexity (primary)
    │                   └─ Gemini (fallback)
    │
    ├─ build_grounded_reasoning_user_prompt()     ← runs in parallel with search
    │
    ├─ await external_ctx_task                    ← collect result
    │   ├─ got result → _truncate_to_token_budget → append to user_prompt
    │   └─ timeout/error → skip silently
    │
    └─ LLM call with (possibly enriched) prompt
```

**Prompt injection format:**
```
--- SUPPLEMENTARY EXTERNAL CONTEXT (from web search) ---
{search_context}
--- END EXTERNAL CONTEXT ---
IMPORTANT: The structured evidence (in `evidence_outputs`) above is your PRIMARY source of truth.
Use this external context ONLY for market background, industry benchmarks,
or explanatory context. NEVER let it override evidence data values.
```

### Extensibility

To add a new external source (e.g., Knowledge Graph):
1. Write `_fetch_kg_context(question) -> Optional[str]`
2. Add it to `_gather_external_sources` tasks list
3. All sources run in parallel via `asyncio.gather(return_exceptions=True)`
4. Results are combined and token-truncated together

---

## Smart RouterAgent (Phase 1)

> **File:** `router_agent.py`
> **Design doc:** `smart_router_agent.md`
> **Test harness:** `check_router_agent.py`

The RouterAgent replaces the old binary Flash-vs-Pro routing with a deterministic 5-level × 6-task-type model selection system. It makes **zero LLM calls** — all routing decisions are regex + weighted scoring.

### Reasoning Levels

| Level | Name | Score Range | Description |
|-------|------|-------------|-------------|
| L0 | `L0_LOOKUP` | < 0.25 | Direct data retrieval, no reasoning |
| L1 | `L1_SIMPLE` | 0.25 – 0.45 | Single transformation / filter / aggregation |
| L2 | `L2_ANALYTICAL` | 0.45 – 0.90 | Multi-step reasoning, cross-reference, synthesis |
| L3 | `L3_COMPLEX` | 0.90 – 1.30 | Deep reasoning, multi-constraint, cause-effect |
| L4 | `L4_STRATEGIC` | >= 1.30 | Novel strategies, what-if design, open-ended |

### Task Types

| Task Type | Signals | Weight |
|-----------|---------|--------|
| `RETRIEVAL` | top N, list, show me, give me | 1.0× |
| `MATH` | calculate, sum, margin, profit, %, financial impact | 1.2× |
| `COMPARISON` | compare, vs, between, relative to, benchmark | 1.3× |
| `ANALYSIS` | summarize, insight, trend, pattern, analyze, impact | 1.1× |
| `CREATIVE` | design, create, build, strategy, what-if, brainstorm | 1.5× + planning 0.8× |
| `PROGRAM_EXECUTION` | add/remove/modify constraint/rule/fact, re-run, re-execute | N/A (explicit only) |

> **Note:** `PROGRAM_EXECUTION` is never auto-detected by Stage 1/Stage 2. It is set explicitly by `BaseQAPipelineWithASP._generate_answer()` via `router.select_for_task(question, TaskType.PROGRAM_EXECUTION)` when BOTH conditions are met: ASP reasoning context + modification patterns.

### Routing Table (Level × TaskType → Model)

| Level | Retrieval | Math | Comparison | Analysis | Creative | Program Execution |
|-------|-----------|------|------------|----------|----------|-------------------|
| **L0** | Flash | Flash | Flash | Flash | Flash | Opus |
| **L1** | Flash | Flash | Flash | Flash | Haiku | Opus |
| **L2** | Pro 3.1 | Pro 3.1 | Sonnet | Sonnet | Sonnet | Opus |
| **L3** | Pro 3.1 | Pro 3.1 | Opus | Opus | Opus | Opus |
| **L4** | Pro 3.1 | Pro 3.1 | Opus | Opus | Opus | Opus |

**Design principles:**
- Gemini handles structured data tasks (retrieval, math). Claude handles language-quality tasks (comparison, analysis, creative).
- `PROGRAM_EXECUTION` always maps to Claude Opus 4.6 at every level — ASP code generation requires the strongest reasoning model. `select_for_task()` also enforces at least L3_COMPLEX.

### Two-Stage Routing Pipeline

**Stage 1: Fast Regex Exit (<1ms)**
- Obvious simple patterns: `show me`, `what is the`, `top N`, `list`, `how many` → L0/L1
- Obvious complex patterns: `design`, `create.*scenario`, `optimize`, `maximize` → L3
- Simple blockers: `risk`, `if we/I`, `competitor`, `systemic`, `long-term` — prevent false L0

**Stage 2: Multi-Signal Weighted Scoring (<5ms)**

9 signal groups extracted via regex pattern matching:

| Signal Group | Patterns | Complexity Weight | Cap |
|-------------|----------|-------------------|-----|
| Explanation | why, explain, justify, root cause, drives | 0.28 | 2 |
| Planning | optimize, what-if, strategy, reduce cost/risk | 0.30 | 3 |
| Creative | design, create, build, brainstorm | 0.25 | 2 |
| Enterprise Strategic | competitor, risk, if we, market share, lifetime value | 0.22 | 3 |
| Comparison | compare, vs, between, benchmark | 0.15 | 2 |
| Analysis | summarize, insight, trend, pattern, analyze | 0.14 | 2 |
| Multi-Group | for each, by region/store, group by, across | 0.10 | 3 |
| Math | calculate, sum, margin, profit, % | 0.06 | 2 |
| Simple Retrieval | top N, list, show me (discount: -0.15) | — | — |

Additional scoring components:
- **Length bonus:** `min(word_count / 80, 0.30)` — longer questions tend to be harder
- **Intent bonus:** From `DeterministicRecordPlanner.detect_intent()` if available (aggregate, comparison, temporal signals)
- **Ambiguity bonus:** From `DeterministicSearchIndex` if available (0.30 × ambiguity score)
- **Workflow bonus:** Per-workflow adjustment (zeroed out for Phase 1, no production data yet)

### Explicit Task Routing (`select_for_task`)

When the caller already knows the task type (e.g., `BaseQAPipelineWithASP` has verified ASP context + modification patterns), it calls `router.select_for_task(question, TaskType.PROGRAM_EXECUTION)` instead of `router.route(question)`.

**Behavior:**
- Runs the normal scoring pipeline (Stage 1/Stage 2) for reasoning level only
- Overrides the task type with the caller-provided value
- For `PROGRAM_EXECUTION`, enforces at least `L3_COMPLEX` so the routing table yields Claude Opus 4.6

**`PROGRAM_EXECUTION_PATTERNS`** (exported from `router_agent.py`, consumed by `BaseQAPipelineWithASP`):

| Pattern | Example Match |
|---------|--------------|
| `add.*constraint\|rule\|fact` | "Add a constraint that no nurse works 2 consecutive days" |
| `remove.*constraint\|rule\|fact` | "Remove the overtime constraint" |
| `modify.*program\|rule\|constraint\|fact` | "Modify the scheduling rule for weekends" |
| `change.*rule\|constraint\|fact` | "Change the max shifts rule" |
| `re-?run\|execute\|solve` | "Re-run the solver" |
| `what.if.*add\|remove\|change\|we` | "What if we add a night shift constraint?" |
| `new constraint\|rule\|fact` | "New constraint: max 3 shifts per nurse" |

> **Important:** These patterns are NEVER used in the router's Stage 1/Stage 2. They are only checked by `BaseQAPipelineWithASP._generate_answer()` where ASP context is confirmed.

### Authorized Models

`GroundedAnswerEngineConfig.authorized_models` is the **single source of truth** for which models the router may use. If a routing table selection is not in the authorized list, the router gracefully degrades by walking down levels (same task type) until it finds an authorized alternative.

**Default authorized models (7):**

| Model | Provider | Role |
|-------|----------|------|
| Gemini 3.0 Flash | Vertex AI | Fast/cheap (L0–L1) |
| Gemini 3.1 Pro | Vertex AI | Structured data (L2–L4) |
| Claude 4.5 Haiku | Vertex AI | Fast creative (L1) |
| Claude 4.6 Sonnet | Vertex AI | Balanced language quality (L2) |
| Claude 4.6 Opus | Vertex AI | Best language quality (L3–L4) |
| Claude 4.5 Sonnet | Vertex AI | Fallback |
| Gemini 2.5 Pro | Vertex AI | Fallback |

### Fallback Chain (Cross-Provider Alternation)

The `fallback_chain` is separate from `authorized_models`. It defines the **resilience ordering** for LLM-call-level retries, alternating providers so that a Gemini outage doesn't waste retries on more Gemini:

```
Gemini 3.0 Flash → Claude 4.6 Opus → Gemini 3.1 Pro → Claude 4.5 Sonnet → Gemini 2.5 Pro → Claude 4.5 Haiku
```

**Per-decision fallbacks** (built by `_build_fallbacks`): cross-provider alternative at the same level, then step up/down. Max 3 models per decision. These are prepended to the config fallback_chain for LLM calls.

### Gemini Thinking Config (Model-Aware)

Different Gemini model families use different thinking control parameters. The router builds the correct API config per model.

**Gemini 3.x → `thinkingLevel` (string)**

| Level | Gemini 3 Flash | Gemini 3 Pro | Gemini 3.1 Pro |
|-------|---------------|--------------|----------------|
| L0 | minimal | low | low |
| L1 | low | low | low |
| L2 | medium | high | medium |
| L3 | high | high | high |
| L4 | high | high | high |

Note: Gemini 3 Pro only supports `low`/`high` (no `minimal`, no `medium`). Gemini 3.1 Pro supports `low`/`medium`/`high` (no `minimal`).

**Gemini 2.5 → `thinkingBudget` (token count)**

| Level | Gemini 2.5 Pro (128–32768) | Gemini 2.5 Flash (0–24576) |
|-------|---------------------------|---------------------------|
| L0 | 128 | 0 (disabled) |
| L1 | 512 | 512 |
| L2 | 4,096 | 4,096 |
| L3 | 16,384 | 12,288 |
| L4 | 32,768 | 24,576 |

Model family detection uses `LLMGeminiModels` enum as source of truth (strips `-preview` suffix to match).

Ref: https://ai.google.dev/gemini-api/docs/thinking

### Routing Results (139 Test Questions)

See [routing_results.md](./routing_results.md) for the full table.

**Test groups:**

| Group | Workflow | Count |
|-------|---------|------:|
| Competitive Pricing | `competitive_pricing_intelligence_rheem_workflow` | 14 |
| Warranty Intelligence | `warranty_intelligence_rheem_workflow` | 25 |
| Pricing Simulator | `pricing_simulator_intelligence_workflow` | 19 |
| ASP Reasoning | `gp_contextual_reasoning_engine_problems_workflow` | 12 |
| Deep Analytical | `default` | 20 |
| Strategic | `default` | 21 |
| Composition Workflow | `pricing_simulator_competitive_warranty_composition_workflow` | 28 |

---

## Configuration

### GroundedAnswerEngineConfig (`question_answering_configurations.py`)

Controls model routing and LLM call parameters for the answer engine.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `temperature` | 0.3 | LLM sampling temperature |
| `max_output_tokens` | 200,000 | Max output tokens |
| `max_input_tokens` | 200,000 | Max input tokens (triggers map-reduce if exceeded) |
| `max_attempts` | 2 | Retry count with repair prompt |
| `agent_request_time_out` | 60 | LLM call timeout (seconds) |
| `authorized_models` | 8 models | Models the router may use (see table above) |
| `fallback_chain` | 6 models | Cross-provider resilience ordering |
| `enable_actionable_applications` | False | Append "Actionable Applications" section to answers |
| `enable_investigation_suggestions` | False | Append "Suggested Next Questions" section to answers |
| `enable_external_context` | False | Allow external context enrichment (web search, etc.) |
| `external_context_timeout` | 5.0 | Max seconds to wait for external context sources |
| `external_context_max_tokens` | 2000 | Token budget cap for external context in prompt |

### QAToolConfig (`question_answering_configurations.py`)

#### Core Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| `grounded_answer_config` | default | `GroundedAnswerEngineConfig` instance (model routing, authorized models, fallback) |
| `semantic_query_planner_config` | default | Nested config for planner strategy, deterministic settings, LLM models |
| `enable_reasoning_protocol` | False | True = dynamic reasoning protocol, False = static |
| `use_suggestions_only` | True | True = use predefined partitions only, False = smart selection |
| `suggested_partitions` | None | Predefined section IDs for partition selection |
| `use_selection_batch_mode` | True | True = batch (single LLM call), False = parallel (per-partition) |
| `fail_open_on_bad_plan` | False | True = return all data on bad plan, False = strict |
| `include_ontology_in_answer` | False | Pass ontology context to final answer engine |

#### Important Partitions

| Parameter | Default | Description |
|-----------|---------|-------------|
| `important_partitions` | None | Partitions that always use `mode="all"` (e.g., summary tables) |
| `enable_important_partitions_fast_path` | True | True = skip LLM for important partitions |
| `enable_aggregate_mode` | False | Enable aggregate operations (sum, count, mean, etc.) |

#### Working Memory Optimization

| Parameter | Default | Description |
|-----------|---------|-------------|
| `shared_data_partitions` | None | Map secondary → primary partition. When both active, secondary skipped |
| `partition_max_rows` | None | Override `max_rows` (default 200) per partition |

### Per-Workflow Configuration (`qa_pipelines.py`)

Each workflow pipeline implements `get_workflow_qa_specific_config()` returning a `QAToolConfig`. This is the single place to configure per-workflow behavior, including model authorization:

```python
class PricingCompetitiveIntelligenceQAPipeline(BaseQAPipeline):
    def get_workflow_qa_specific_config(self) -> QAToolConfig:
        return QAToolConfig(
            semantic_query_planner_config=SemanticQueryPlannerConfig(...),
            grounded_answer_config=GroundedAnswerEngineConfig(
                # Example: restrict to cheap models for cost-sensitive workflow
                authorized_models=[
                    LLMVertexAIGeminiModels.GEMINI_3_0_FLASH.value,
                    LLMVertexAIClaudeModels.CLAUDE_4_5_HAIKU.value,
                ],
            ),
            ...
        )
```

**Registered pipelines:**

| Workflow ID | Pipeline Class | Base | External Context |
|-------------|---------------|------|:---:|
| `competitive_pricing_intelligence_rheem_workflow` | `PricingCompetitiveIntelligenceQAPipeline` | `BaseQAPipeline` | No |
| `pricing_simulator_intelligence_workflow` | `PricingScenarioSimulatorQAPipeline` | `BaseQAPipeline` | No |
| `warranty_intelligence_rheem_workflow` | `WarrantyIntelligenceQAPipeline` | `BaseQAPipeline` | No |
| `pricing_simulator_competitive_warranty_composition_workflow` | `RheemPricingWarrantyCompositionWorkflowQAPipeline` | `BaseQAPipeline` | **Yes** |
| `gp_contextual_reasoning_engine_problems_workflow` | `GPReasoningEngineQAPipeline` | `BaseQAPipelineWithASP` | No |

---

## Reliability Features

### LLM Call Protection

- **Timeout:** `agent_request_time_out` (default 60s)
- **Fallback chain:** Cross-provider alternation (Gemini → Claude → Gemini → Claude)
- **Error suppression:** `LiteLLMTimeoutFilter` converts verbose ERROR logs into clean WARNING logs

### Token Management

- **Max input:** 200K tokens with early rejection
- **Row capping:** `mode="all"` respects `max_rows` (default 200, configurable per-partition)
- **Field projection:** Returns only necessary columns
- **Shared data dedup:** Avoids loading duplicate data when partitions share the same dataset

---

## Data Flow Example

**User:**
"Can you design a new pricing scenario for Rheem to maximize the profit?"

**Execution flow:**

1. **BaseQAPipeline.solve()** → starts timer, loads workflow artifacts from Redis
2. **SemanticQueryPlannerFactory** → creates planner (LLM or Deterministic based on config)
3. **QueryPlanner** → selects partitions, fields, record plans
4. **_optimize_query_plan()** → applies shared data dedup + max rows overrides
5. **StructuredDataRetriever** → fetches evidence using pandas
6. **GroundedAnswerEngine** →
   - RouterAgent **Stage 1 fast exit**: matches `design.*scenario` + `maximize` → **L3_COMPLEX**, task=CREATIVE
   - Routing table: (L3, CREATIVE) → **Claude 4.6 Opus**
   - Thinking config: Claude → None (no Gemini thinking)
   - Confidence: 0.90 (fast exit = high confidence)
   - Fallbacks: [Gemini 3.1 Pro, Claude 4.6 Sonnet] + remaining fallback_chain
   - LLM call with selected model + full fallback chain
7. **_append_response_metadata()** → adds timestamp and response time footer
8. **Return** → grounded markdown answer with metadata

---

## ASP Reasoning Support

### Overview

The Q&A Framework supports ASP (Answer Set Programming) workflows via the GP Contextual Reasoning Engine (SIG-PRE). Unlike data-schema workflows that use tabular data + pandas, ASP workflows operate on symbolic reasoning programs with predicates, rules, constraints, and answer sets.

**Key design principle:** Two separate pipeline paths share a single `BaseQAPipeline.solve()` entry point. The split happens at the worker level — each of the three workers (`_build_query_plan`, `_retrieve_evidence`, `_generate_answer`) is overridden by `BaseQAPipelineWithASP`.

### BaseQAPipelineWithASP (`qa_base_pipeline.py`)

Intermediate class between `BaseQAPipeline` and workflow-specific pipelines (e.g., `GPReasoningEngineQAPipeline`).

**Overrides:**

| Worker | Data-Schema (BaseQAPipeline) | ASP Reasoning (BaseQAPipelineWithASP) |
|--------|-------|------|
| `_build_query_plan` | SemanticQueryPlannerFactory → LLM/Deterministic planner | Bypass: selects `asp_program_context` partition directly |
| `_retrieve_evidence` | StructuredDataRetriever (pandas) | Bypass: returns ASP section as-is |
| `_generate_answer` | GroundedAnswerEngine always | Two-condition routing (see below) |

**Two-condition routing in `_generate_answer`:**

```
 evidence_outputs
       │
       ▼
 is_asp_program_context? ──No──▶ super()._generate_answer() (data-schema)
       │Yes
       ▼
 PROGRAM_EXECUTION_PATTERNS match? ──No──▶ super()._generate_answer() (GroundedAnswerEngine)
       │Yes                                  "What is the total cost?"
       ▼
 ASPProgramExecutionAgent.execute()
   "Add constraint: no nurse works 2 consecutive days"
```

Both conditions must be met: ASP context exists **AND** question matches modification patterns. This prevents data-schema workflows from accidentally triggering ASP program execution (e.g., a Pricing workflow user asking "what if we add a constraint to the pricing model").

### ASPProgramExecutionAgent (`engine/asp_program_execution_agent.py`)

LLM-driven agent for modifying and re-executing ASP programs. Four-step pipeline:

```
┌─────────────────────────────────────────────────┐
│ A. COMPREHEND — Build program context from      │
│    ASPReasonerResponse (predicates, rules,       │
│    facts, constants, show directives, models)    │
│                                                  │
│ B. GENERATE  — RuleGenerator.generate()         │
│    Claude Opus 4.6 produces new ASP statements   │
│    (constraints, rules, facts)                   │
│                                                  │
│ C. EXECUTE   — Option C: SIG-PRE primary        │
│    ├─ _sig_pre_execute() → GP-DRE solver        │
│    │    (original facts + new statements)         │
│    │                                              │
│    └─ On failure → _recovery()                   │
│         ├─ Claude diagnoses error                │
│         ├─ Fixes ASP statements                  │
│         └─ Re-executes via SIG-PRE               │
│                                                  │
│ D. INTERPRET — _format_results()                │
│    ├─ Program Modification section (unique)      │
│    ├─ to_nlg() canonical reasoning report        │
│    ├─ New Rules Added by Modification            │
│    └─ Comparison with Original (cost delta)      │
└─────────────────────────────────────────────────┘
```

**Key classes:**

| Class | Purpose |
|-------|---------|
| `RuleGenerator` | Generates new ASP rules/constraints/facts. Phase 1: single LLM call. Future: post-training models, syntax validation, multi-pass generation |
| `ExecutionSource` | Enum: `SIG_PRE` (deterministic solver) or `LLM_RECOVERY` (Claude recovery) |
| `ExecutionResult` | Dataclass with `response`, `source`, `error_message`, `recovery_explanation`, `fixed_statements` |

**Option C execution strategy:**
1. **SIG-PRE primary** — deterministic, provably correct. Calls `gp_dre_neuro_contextual_reasoning_impl(context_content=facts+new_statements, gp_dre_agent=agent_id)`
2. **Recovery on failure** — Claude diagnoses the solver error, fixes the ASP statements, re-executes via SIG-PRE. Best-effort, marked as `LLM_RECOVERY` source

### ASPReasonerResponse.to_nlg() (`reasoning_engine_agent_ontology.py`)

Canonical NLG renderer living inside the `ASPReasonerResponse` data model in the SDK. Shared by:
- `REDataEntryModelContextualReasoning.nlg()` — initial reasoning report (title + `to_nlg()`)
- `ASPProgramExecutionAgent._format_results()` — modification report (wraps `to_nlg()` with modification-specific sections)

**Sections rendered:**

| # | Section | Content |
|---|---------|---------|
| 1 | General Overview | Agent, solver, optimization, model count |
| 2 | Preconditions & Assumptions | Predicate schemas (NL) + constant declarations + show directives |
| 3 | Reasoning Results | Answer sets rendered as tables: nurse schedules, course schedules, or generic planning/predicate views |
| 4 | Summary & Insights | Optimization costs, model comparison, feasibility notes |

**Rendering branches in Section 3:**
- `scheduling_` agents → `_render_nurse_schedule()` (pandas pivot: nurse × day → shift)
- `planning_` agents → `_render_course_schedule()` / `_render_generic_planning()`
- All others → `_render_generic_predicates()` (predicate → atoms list)

### Data Flow Example (ASP Modification)

**User:**
"Add a constraint that no nurse works more than 2 consecutive days"

**Execution flow:**

1. **BaseQAPipelineWithASP.solve()** → starts timer, loads workflow artifacts from Redis
2. **_build_query_plan()** → detects `asp_program_context` partition → selects directly (bypasses SemanticQueryPlanner)
3. **_retrieve_evidence()** → returns ASP section as-is (bypasses StructuredDataRetriever)
4. **_generate_answer()** →
   - Condition 1: `asp_evidence.is_asp_program_context` → True
   - Condition 2: `PROGRAM_EXECUTION_PATTERNS` match ("add" + "constraint") → True
   - `router.select_for_task(question, TaskType.PROGRAM_EXECUTION)` → **Claude 4.6 Opus**, L3_COMPLEX
5. **ASPProgramExecutionAgent.execute()** →
   - A. COMPREHEND: builds program context from `ASPReasonerResponse` (predicates, rules, facts, models)
   - B. GENERATE: `RuleGenerator.generate()` → Claude produces: `:- assign(N,S,D1), assign(N,S2,D2), assign(N,S3,D3), D2=D1+1, D3=D2+1.`
   - C. EXECUTE: `_sig_pre_execute()` → calls GP-DRE solver with original facts + new constraint → new `ASPReasonerResponse`
   - D. INTERPRET: `_format_results()` → Program Modification + `to_nlg()` report + New Rules Added + Comparison
6. **_append_response_metadata()** → adds timestamp and response time footer
7. **Return** → Markdown report with modification summary, full reasoning report, new rules, and cost comparison

---

## Key Files

| File | Purpose |
|------|---------|
| `qa_base_pipeline.py` | Abstract base class (`BaseQAPipeline`) + ASP intermediate class (`BaseQAPipelineWithASP`) |
| `qa_pipelines.py` | Per-workflow pipeline implementations + registry |
| `question_answering_configurations.py` | All config dataclasses (QAToolConfig, SemanticQueryPlannerConfig, GroundedAnswerEngineConfig) |
| `engine/grounded_answer.py` | Final answer generation with RouterAgent integration |
| `engine/router_agent.py` | Smart 5-level × 6-task-type model router, `PROGRAM_EXECUTION_PATTERNS`, `select_for_task()` |
| `engine/asp_program_execution_agent.py` | ASP program modification + re-execution agent (RuleGenerator, ExecutionResult, Option C) |
| `engine/check_router_agent.py` | Experiment harness for 139 questions across 6 workflows |
| `engine/semantic_query_planner.py` | LLM-based query planning |
| `engine/deterministic_query_planner.py` | Rule-based query planning orchestrator (no LLM) — section selection, domain affinity, config-driven |
| `engine/ontology_driven_deterministic/deterministic_field_selector.py` | Token/synonym/description scoring with content quality gate + adaptive thresholds |
| `engine/ontology_driven_deterministic/deterministic_record_planner.py` | Rule-based intent detection (top_k/all/filter/aggregate) with broad question patterns |
| `engine/ontology_driven_deterministic/deterministic_search_index.py` | BM25 + embedding search index for section selection (search mode) |
| `engine/semantic_query_planner_factory.py` | Strategy pattern factory |
| `engine/structured_data_retriever.py` | Pandas-based data retrieval |
| `engine/reasoning_protocol_designer.py` | Dynamic reasoning protocol generation |
| `tests/unit/semantic_test/test_routing_agent.py` | Pytest-parametrized routing regression tests (139 questions, 7 groups) |
| **SDK** | |
| `search_tools.py` | `ask_perplexity_async()`, `gemini_web_search_async()` — web search providers for external context |
| `reasoning_engine_agent_ontology.py` | `ASPReasonerResponse.to_nlg()` — canonical NLG renderer shared by initial report + modification report |
| `workflow_ontology.py` | `ReasoningContextSignature`, `ASP_PROGRAM_CONTEXT`, `PipelineStructuredOutput.is_asp_program_context` |
| `pipeline_gp_reasoning_engine_workflow.py` | `REDataEntryModelContextualReasoning` — delegates `nlg()` to `ASPReasonerResponse.to_nlg()` |

---

## Implementation Roadmap

### Phase A: Deterministic Planning — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| Retrieval-based section selection (BM25 + embeddings) | Done | `DeterministicSearchIndex` |
| Deterministic field scoring | Done | `DeterministicFieldSelector` |
| Rule-based record plan | Done | `DeterministicRecordPlanner` |
| LLM batch planner as fallback | Done | Factory pattern with `DETERMINISTIC` strategy |

### Phase A.1: Deterministic Planner Tuning — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| Include all suggested partitions (bypass BM25 gate) | Done | `_select_sections()` — +15% accuracy gain |
| Content quality gate for field selection | Done | `DeterministicFieldSelector.content_quality_gate=0.18` |
| Adaptive relative threshold for large sections | Done | `DeterministicFieldSelector.relative_threshold_ratio=0.40` + scaling |
| Config-driven domain affinity (keywords + section prefixes) | Done | `SemanticQueryPlannerConfig.domain_keywords`, `domain_section_prefixes` |
| Numbered variant routing (scenario N detection) | Done | `SemanticQueryPlannerConfig.numbered_variant_pattern` |
| Broad question detection expansion | Done | `DeterministicRecordPlanner` — design, optimal, optimize, highlight, etc. |
| Forward-looking sort field preference | Done | `DeterministicRecordPlanner._infer_sort_field()` — expected/optimal prefix preference |
| Business concept synonyms (lean set) | Done | `DeterministicFieldSelector` — opportunity, impact, segment, market, grossprofit |
| Experiment infrastructure (`--dqp-only` runner) | Done | `experiment_semantic_query_planner.py` with DQP-only mode |
| Accuracy: CP 83.4%, WI 81.8%, PS 62.5%, Comp 67.7% | Done | See `deterministic_semantic_query_planner_improvement.md` for details |

### Phase 1: Smart Router Agent — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| 5-level reasoning classification (L0–L4) | Done | `ReasoningLevel` enum + `_score_to_level()` |
| 6-task-type detection (retrieval/math/comparison/analysis/creative/program_execution) | Done | `TaskType` + `_detect_task_type_weighted()` + `select_for_task()` |
| Two-stage routing (fast exit + multi-signal scoring) | Done | `_stage1_fast_exit()` + `_stage2_full_scoring()` |
| (Level × TaskType) → model routing table | Done | `_build_routing_table()` (Gemini for data, Claude for language, Opus for ASP) |
| Enterprise strategic pattern recognition | Done | `ENTERPRISE_STRATEGIC_PATTERNS` (29 regex patterns) |
| Gemini thinking config (model-aware: thinkingLevel vs thinkingBudget) | Done | `_build_thinking_config()` per model family |
| authorized_models validation | Done | `_is_authorized()` + `_find_authorized_alternative()` |
| Cross-provider fallback chain | Done | `_build_fallbacks()` with provider alternation |
| GroundedAnswerEngine integration | Done | `router.route()` replaces binary routing |
| Per-workflow config via QAToolConfig | Done | `grounded_answer_config` in `QAToolConfig` |
| `PROGRAM_EXECUTION` task type + explicit routing | Done | `PROGRAM_EXECUTION_PATTERNS` + `select_for_task()` |
| Test harness (139 questions, 7 groups) | Done | `check_router_agent.py` + `test_routing_agent.py` |

### Phase 1.5: ASP Reasoning Support — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| `BaseQAPipelineWithASP` intermediate class | Done | Overrides `_build_query_plan`, `_retrieve_evidence`, `_generate_answer` in `qa_base_pipeline.py` |
| `ReasoningContextSignature` + `ASP_PROGRAM_CONTEXT` | Done | Frozen dataclass for namespaced section IDs in `workflow_ontology.py` |
| `PipelineStructuredOutput.is_asp_program_context` | Done | Centralized ASP detection property in `workflow_ontology.py` |
| Two-condition routing (ASP context + patterns) | Done | `_generate_answer()` checks `is_asp_program_context` + `PROGRAM_EXECUTION_PATTERNS` |
| `ASPProgramExecutionAgent` (4-step pipeline) | Done | COMPREHEND → GENERATE → EXECUTE → INTERPRET in `asp_program_execution_agent.py` |
| `RuleGenerator` class | Done | Standalone class for ASP rule generation (Phase 1: LLM, future: post-training models) |
| Option C execution (SIG-PRE primary + recovery) | Done | `_sig_pre_execute()` → on failure → `_recovery()` with `ExecutionResult` provenance |
| `ASPReasonerResponse.to_nlg()` canonical renderer | Done | 4-section NLG in `reasoning_engine_agent_ontology.py`, shared by `nlg()` + `_format_results()` |
| `REDataEntryModelContextualReasoning.nlg()` refactored | Done | Delegates to `ASPReasonerResponse.to_nlg()` (removed ~150 lines of duplicate rendering) |
| Modification report with New Rules section | Done | `_format_results()` renders modification + `to_nlg()` + New Rules Added + Comparison |

### Phase 1.6: External Context Enrichment — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| Config-gated external context (`enable_external_context`) | Done | `GroundedAnswerEngineConfig` with timeout + token budget |
| Two-layer detection (`_needs_external_context`) | Done | Layer 1: RouterDecision-based, Layer 2: domain-specific keyword patterns |
| Domain-specific patterns keyed by `ZeusTool` workflow_id | Done | `_DOMAIN_PATTERNS_BY_WORKFLOW` in `grounded_answer.py` |
| Parallel search execution via `asyncio.create_task` | Done | Fires after `router.route()`, collects before LLM call |
| Search provider fallback (Perplexity → Gemini) | Done | `_fetch_web_search_context()` with automatic fallback |
| Domain search context enrichment per workflow | Done | `_DOMAIN_SEARCH_CONTEXT_BY_WORKFLOW` + `_build_search_query()` |
| Token budget enforcement via `trim_nl_text` | Done | tiktoken-based truncation with sentence boundary preservation |
| Supplementary context prompt injection | Done | Clearly marked block with grounding instruction |
| Extensible source architecture (`_gather_external_sources`) | Done | `asyncio.gather(return_exceptions=True)` — add new `_fetch_*_context` methods |
| Composition workflow enabled | Done | `RheemPricingWarrantyCompositionWorkflowQAPipeline` with `enable_external_context=True` |
| Fallback to public search message improvement | Done | Two-case disclaimer (expired session + wrong tool) in `qa_base_pipeline.py` |

### Phase B: Hybrid Planner + Confidence Gating — NOT YET

With DQP at 80%+ for 2/4 workflows (Phase A.1), hybrid is now more viable — ~60-70% of questions can skip LLM entirely.

| Item | Status | Notes |
|------|--------|-------|
| Confidence scoring from field selection signals | Planned | `max_content_score`, none-section count, sort/filter match quality |
| `HybridQueryPlanner` (deterministic first, LLM fallback) | Scaffolded (`NotImplementedError`) | Enum exists: `SemanticQueryPlannerStrategy.HYBRID` |
| Embedding-based field selection (replace token overlap) | Planned | **Highest priority** — would close gap for PS (62.5%) and Comp (67.7%) |
| Distilled record mode classifier (from LLM outputs) | Planned | Training data available from experiment runs |
| Split batch planner into targeted mini-batches | Planned | |
| Cache at the right granularity | Planned | |

### Phase 2: Router Agent Improvements — NOT YET

| Item | Status |
|------|--------|
| Production log analysis → tune scoring weights | Planned |
| Workflow-specific complexity bonus (from real data) | Planned |
| REASONING task type for multi-step logical chains | Planned |
| Router confidence → dynamic fallback depth | Planned |
| A/B testing framework for routing decisions | Planned |

### Phase C: Distilled Planner Model — NOT YET

| Item | Status |
|------|--------|
| Distill planner into small local model | Planned |
| Log-based training data collection | Planned |
| Gemini as periodic auditor / fallback | Planned |

### Phase D: ASP Reasoning Enhancements — NOT YET

| Item | Status |
|------|--------|
| ASP syntax validation (ground before solve to catch LLM errors early) | Planned |
| State persistence: store modified `ASPReasonerResponse` in run store for follow-up questions | Planned |
| Multi-turn modifications: "Now also add constraint X" on already-modified program | Planned |
| Rollback: "Undo the last modification" | Planned |
| Rule modification/removal (requires GP-DRE service changes to accept full programs) | Planned |
| RuleGenerator post-training models (distilled ASP expert, syntax-aware generation) | Planned |
| Multi-agent execution (SQL, Python, Prolog, Constraint Programming alongside ASP) | Planned |

---

## Recent Enhancements

### 1. Relevance-Ordered `mode=all` (SQP + Data Retriever)

**Problem:** When `mode=all` is used, rows are returned in arbitrary storage order. If `max_rows` caps truncate data, the most relevant rows may be cut off — the grounded answer LLM reasons over less important data.

**Solution:** `RecordPlan` already has `sort_by` and `descending` fields. We now:
1. Instruct the SQP LLM to provide `sort_by` even for `mode=all` plans
2. Parse `sort_by`/`descending` in `_validate_record_plan` for `mode=all`
3. Preserve LLM's `sort_by` when heuristic overrides force `mode=all`
4. Sort rows in the data retriever before capping

**Files changed:**
- `semantic_query_planner.py` — prompt instructions (4 variants: parallel basic/aggregate, batch basic/aggregate), `_validate_record_plan` parser, heuristic override preservation
- `structured_data_retriever.py` — `mode=all` branch uses `_apply_top_k` for sorting when `sort_by` is set

**Prompt wording (important — affects LLM behavior):**
```
"When using mode=all, also provide sort_by with the most analytically important numeric field and descending=true/false so rows are ordered by relevance."
```
> **Lesson learned:** An earlier version used `"IMPORTANT: For ALL modes..."` which was too aggressive and caused the LLM to change its mode/filter decisions for aggregate questions. The softer, scoped wording above avoids this regression.

**Backward compatible:** If `sort_by` is `None` (LLM doesn't provide it, or old plans), the retriever behaves exactly as before.

**Flow:**
```
SQP decides mode=all for a section
  → LLM also provides sort_by=<important_field>, descending=true/false
  → _validate_record_plan parses sort_by for mode=all
  → Heuristic override preserves sort_by when forcing mode=all
  → Data retriever: _apply_top_k(data, k=max_rows, sort_by=...) → sorted + capped
  → Grounded answer LLM gets most relevant rows first
```

### 2. Structured Map-Reduce for GroundedAnswerEngine

**Problem:** When evidence exceeds token limits, the engine splits data into chunks and runs map-reduce. Previously, each map-phase chunk produced **free-text prose** — narrative essays with redundant boilerplate, imprecise numbers buried in paragraphs, and ambiguous framing that the reduce phase struggled to synthesize.

**Solution:** Each map-phase chunk now produces a **structured JSON reasoning artifact** instead of prose. The reduce phase then synthesizes these precise, structured findings into the final narrative answer.

**Structured output schema** (`_MAP_PHASE_STRUCTURED_SCHEMA`):

| Field | Purpose | Reasoning Type |
|-------|---------|---------------|
| `data_scope` | What data was analyzed (sections, row range, temporal/dimensional scope) | Context |
| `reasoning_chain` | Step-by-step thinking: identify → compute → compare → infer → project → evaluate | **How** (process reasoning) |
| `key_findings` | Precise facts with data points, confidence, source fields | **What** (factual) |
| `relationships_and_causality` | Entity connections: correlation, causal, dependency, contradiction, temporal sequence | **Why** (causal reasoning) |
| `patterns_and_anomalies` | Trends, seasonality, outliers, clusters, distribution skew, threshold breaches | **When/Where** (pattern recognition) |
| `hypotheses_and_what_if` | Counterfactual/forward-looking reasoning with supporting + counter evidence | **What-if** (scenario reasoning) |
| `quantitative_summary` | Exact numeric values with units and context (no rounding) | **How much** (quantitative) |
| `gaps_and_limitations` | What the chunk could NOT answer or did not have data for | Uncertainty management |

**Key design decisions:**
- **Reasoning-first**: `reasoning_chain` captures the full thinking process, not just conclusions
- **Evidence + interpretation always paired**: Every finding has its reasoning context
- **Uncertainty preserved**: Confidence levels, counter-evidence, gaps — reduce needs to reason about certainty
- **Flexible**: LLM includes applicable sections, omits empty ones — no wasted tokens
- **Fallback safe**: If JSON parsing fails, raw text is used (backward compatible)
- **Single chunk still reduces**: Even 1 structured artifact goes through reduce to produce user-facing prose

**Map-phase system prompt** (`_MAP_PHASE_SYSTEM_PROMPT`):
- Separate from the main grounded answer system prompt
- Instructs LLM to produce valid JSON matching the schema
- Emphasizes exhaustive reasoning: capture EVERYTHING for what, when, where, why, how, what-if
- Preserves exact numeric values (no rounding in quantitative_summary)

**Reduce-phase prompt** (`_REDUCE_PHASE_PROMPT_TEMPLATE`):
- Explicit merge instructions: sum quantities, chain reasoning steps, resolve conflicts
- Hypothesis strengthening/weakening across chunks
- Gap reconciliation: if all chunks share a limitation, it's real; if only one, others may have covered it

**Why structured is better than prose:**

| Aspect | Free text (before) | Structured JSON (now) |
|--------|-------------------|----------------------|
| Token efficiency | ~500-2000 tokens/chunk | ~100-300 tokens/chunk |
| Reduce accuracy | Must parse prose | Direct field access |
| Numeric precision | "$6,724.70 in claims" (lossy) | `6724.70` (exact) |
| Conflict resolution | Ambiguous framing | Compare confidence + specificity |
| Reasoning preservation | Lost in narrative | Full chain preserved |
| Scalability | Budget exhausted ~5 chunks | Handles 10-20+ chunks |

**Files changed:**
- `grounded_answer.py`:
  - Added `_MAP_PHASE_STRUCTURED_SCHEMA`, `_MAP_PHASE_SYSTEM_PROMPT`, `_REDUCE_PHASE_PROMPT_TEMPLATE`
  - Added `_build_map_phase_user_prompt()` — builds evidence payload per chunk
  - Added `_parse_map_result()` — JSON parser using `llmus.clean_json_from_markdown` + brace-scanning fallback
  - Rewrote `_answer_using_map_reduce()` — structured map phase, JSON parsing with fallback, new reduce phase

**What did NOT change:**
- Single-shot answer path (no token overflow) — completely untouched
- Chunking logic (`_chunk_evidence_by_token_budget`, `_split_large_evidence`)
- `build_grounded_reasoning_user_prompt()` — still used by the single-shot path
- System prompt, config, models, fallback chains
- Aggregate mode in SQP/retriever — zero impact

**Future consideration (deferred):**
- **Cross-chunk context sync**: Each chunk currently reasons independently. A future enhancement could do a two-pass map: first pass extracts structured summaries, second pass re-processes chunks WITH other chunks' summaries as context. The structured format makes this feasible without exploding tokens.

### 3. Computed Fields (Post-Aggregation Derived Columns)

**Problem:** Questions like "If you use last PO price × 2025 quantities, what is the projected 2026 spend?" require cross-field computation (`price × quantity`) per item group, then summing. The aggregation engine could only compute `sum(price)` and `sum(quantity)` separately — never `sum(price × quantity)`. The LLM received separate totals and produced incorrect results (e.g., $79.1M instead of the correct $10.5B).

**Solution:** Added `computed_fields` to `RecordPlan` — post-aggregation derived columns computed in pandas after group-by aggregation, before the grand total and row cap.

**Data model:**
```python
class ComputedField(BaseModel):
    expression: str  # e.g., "last_po_price * total_qty_2025"
    alias: str       # e.g., "estimated_spend_2026"
```

**Safe expression evaluator** (`_eval_computed_field` in `structured_data_retriever.py`):
- Uses Python's `ast` module — parses expression to AST, evaluates node-by-node
- **Allowed**: column references, numeric literals, all arithmetic operators (`+`, `-`, `*`, `/`, `//`, `%`, `**`), parentheses, unary `-`/`+`
- **Rejected**: function calls, imports, attribute access, list comprehensions, ternary expressions — all raise `ValueError`
- **Division/modulo by zero**: replaced with `NaN` (no crash)
- **Caller wraps in try/except**: if expression fails, aggregation continues without the computed field (best-effort)

**SQP prompt rule:**
```
11) When the question requires a cross-field computation after aggregation, use "computed_fields"
    to define derived columns. Expressions MUST reference aggregation aliases (not raw field names).
    Supports: +, -, *, /, **, parentheses, and numeric literals.
    Examples: "last_po_price * total_qty", "(revenue - cost) / revenue * 100"
```

**Grand total for computed fields:**
- Computed as `sum(computed_column)` across ALL groups BEFORE the 200-row cap
- This ensures the headline number is exact even when detail rows are capped

**Verification:** Tested against raw pandas computation on 41,704 groups / 1.1M filtered rows — grand total matches to the penny ($11,907,390,980.32).

**Files changed:**
- `semantic_query_planner.py` — `ComputedField` model, `computed_fields` on `RecordPlan`, SQP prompt rules + output schema, parser in `_validate_record_plan`
- `structured_data_retriever.py` — `_eval_computed_field()` (AST-based evaluator), computed field execution after aggregation, grand total computation for computed fields, passed through from record plan

### 4. Current Date Injection for Temporal References

**Problem:** "YTD" queries generated `>= 2025-01-01` (data start date) instead of `>= 2026-01-01` (current year) because the SQP LLM didn't know today's date.

**Solution:**
- **SQP**: Inject `"current_date": "2026-03-23"` into the user prompt JSON. Added rule: "Use current_date to resolve YTD, MTD, QTD, 'last year', etc."
- **DQP**: Added `_resolve_temporal_filters()` in `DeterministicRecordPlanner` — deterministic date resolution for detected temporal references (`ytd` → `>= YYYY-01-01`, `mtd` → `>= YYYY-MM-01`, etc.). Injected into the plan via wrapper `plan_records()` → `_plan_records_core()`.

**Date comparison fix in retriever:** The retriever's filter engine treated date strings like `"2026-01-01"` as numeric (failed with "not numeric" warning). Added YYYY-MM-DD pattern detection → `pd.to_datetime` comparison with timezone-aware localization (`col_date.dt.tz` match).

**Files changed:**
- `semantic_query_planner.py` — `current_date` in user prompts, rule 9 in system prompts
- `deterministic_record_planner.py` — `_resolve_temporal_filters()`, `plan_records()` wrapper
- `structured_data_retriever.py` — date comparison branch in filter engine with tz-aware localization

### 5. Row Cap Control (`enable_row_cap`)

**Problem:** S360 supply chain queries needed all 34,983 aggregation groups for correct computed field grand totals. The 200-row cap truncated groups before computation.

**Solution:** Added `enable_row_cap: bool = True` to `QAToolConfig` → passed to `StructuredDataRetriever` → controls all 4 cap locations (mode=all, mode=filter, aggregate groups, aggregate sample rows).

**Decision:** Reverted S360 to `enable_row_cap=True` (default) after implementing computed fields — the grand total is now computed from ALL groups before capping, so the 200-row cap only limits detail breakdown, not numerical accuracy.

**Files changed:**
- `question_answering_configurations.py` — `enable_row_cap` field
- `structured_data_retriever.py` — `enable_row_cap` in `__init__`, guards on all cap locations
- `qa_base_pipeline.py` — passes `config.enable_row_cap` to retriever
- `qa_pipelines.py` — S360 config (currently `True`, infrastructure ready for `False`)

### 6. Positional Aggregation Sort Fix (first/last)

**Problem:** `last()` aggregation returned the last row in CSV order, not the most recent value. For "last PO price", this gave a random price instead of the chronologically latest one. Affected 29% of items (10,044 out of 34,982).

**Solution:** In `_apply_aggregate_df`, before aggregation, if any op is `first`/`last`, auto-detect the best date/time column and sort by it ascending. Generic — uses `_infer_date_column()` which detects date columns by dtype, name keywords, or parsability.

**Files changed:**
- `structured_data_retriever.py` — `_infer_date_column()` function, pre-aggregation sort for `first`/`last` ops

### 7. Redis RunStore Gzip Compression

**Problem:** S360 workflow output grew to 668 MB after adding new supply chain data (`edp_spend_f_curated.csv` increased from 409 MB to 590 MB). `record_run_success` broke the TCP socket writing to Redis (`ConnectionResetError: Connection reset by peer`).

**Solution:** Added transparent gzip compression in `RedisRunStore`:
- `_set_json`: if payload > 1 MB, gzip compress and prepend `b"__gz__"` marker
- `_get_json`: detect `__gz__` prefix → decompress; otherwise read as plain JSON
- **Backwards compatible**: old uncompressed entries (no `__gz__` prefix) still read correctly

**Result:** 668 MB → 53 MB (12.5x compression ratio). Redis write succeeds.

**Files changed:**
- `run_store/redis_store.py` — `_GZIP_PREFIX`, `_GZIP_THRESHOLD_BYTES`, `_set_json` with compression, `_get_json` with decompression

---

## Conversational Context Carryover — IMPLEMENTED

### Problem

The Q&A pipeline is stateless — each question is processed independently. When a user asks a follow-up, implicit constraints from the previous turn (YTD, D3 filter, time range) are lost.

**Example:**
- Q1: "What is the YTD spend for D3 category?" → SQP adds YTD + D3 filters → returns $54.1M
- Q2: "Which vendors account for the highest portions of that spend?" → SQP generates plan WITHOUT YTD constraint (only D3 filter) → queries ALL time periods → wrong result

### Approaches Evaluated

| Approach | Description | Verdict |
|----------|-------------|---------|
| LangGraph built-in query rewriting | Let LangGraph LLM rewrite the question | **Rejected** — often doesn't rewrite at all; when it does, hallucinates badly ("above time" → "longest lead times") |
| Free-form LLM query rewriter | Custom LLM call to rewrite question | **Risky** — same hallucination risk |
| Using LangGraph's `question` param (tool enrichment) | Use `question` instead of `original_question` in tool functions | **Rejected** — LangGraph passes verbatim most of the time; when it rewrites, it hallucinates |
| **Constraint Memory + SQP Injection** | Store previous plan's filters, inject as structured context to SQP | **Implemented** — deterministic constraints, question unchanged, SQP decides |

### Architecture: Constraint Memory + SQP Context Injection

```
Q1 completes → extract filters from SemanticQueryPlan → store in Redis/InMemory
                                                         (key: workflow_id + thread_id)

Q2 arrives → load prior_context → inject into SQP prompt → SQP decides per-filter

SQP receives:
  {
    "current_date": "2026-03-24",
    "question": "Which vendors account for the highest portions of that spend?",
    "prior_context": {
      "prior_question": "What is the YTD spend for D3 category?",
      "prior_filters": [
        {"field": "direct_level", "op": "==", "value": "D3-Non-Ferrous and Electromechanical"},
        {"field": "transaction_date", "op": ">=", "value": "2026-01-01"},
        {"field": "transaction_date", "op": "<=", "value": "2026-03-23"}
      ],
      "prior_summary": "YTD spend = $54.1M for D3 category"
    },
    "sections": [...]
  }
```

### Data Model

`QAPriorContext` dataclass in `question_answering_configurations.py`:
- `question`: the previous question text
- `filters`: list of filter dicts extracted from the previous `SemanticQueryPlan` (field, op, value) — **filters only, not the full RecordPlan** (mode, group_by, aggregations, computed_fields should NOT carry forward)
- `sections_used`: which data sections were queried
- `result_summary`: first 200 chars of the answer (enough for SQP context)
- `timestamp`: ISO timestamp

### Storage

- **Redis** (`RedisRunStore`): key `qa-prior-context:{workflow_id}:{thread_id}`, TTL 24 hours
- **InMemory** (`InMemoryRunStore`): dict keyed by `(thread_id, workflow_id)`
- Cross-workflow isolation: key includes `workflow_id` — S360 context doesn't leak to Pricing workflow

### SQP Rule 12 (Constraint Inheritance)

```
12) If "prior_context" is provided, decide per-filter whether to inherit:
    INHERIT a prior filter when the question references the prior scope —
    dollar amounts from prior_summary, same categories, or implicit references
    ("that", "those", "it", "above", "same", "break down", "drill into").
    DROP a prior filter when the question explicitly specifies a DIFFERENT value
    for that dimension (e.g., prior has YTD but question says "last year").
    IGNORE prior_context entirely when the question is a new, unrelated topic.
    When inheriting, add the filter exactly as provided — do not modify values.
```

### Pipeline Flow

```
Q&A Tool (original_question) → pipeline.solve()
  → run_store.get_prior_context(thread_id, workflow_id)
  → _build_query_plan(question, ..., prior_context=prior_context)
    → SQP._build_user_prompt_batch_fields_and_records(..., prior_context)
      → JSON payload includes prior_context if present
  → _retrieve_evidence(...)
  → _generate_answer(...)
  → _extract_filters_from_plan(semantic_query_plan) → store prior context
  → return answer
```

### Config Flag

`enable_context_carryover: bool = False` on `QAToolConfig`. Currently enabled for S360 only (`RheemSupplyChainS360WorkflowQAPipeline`).

### Edge Cases

| Case | Behavior |
|------|----------|
| First question (no prior context) | `prior_context` is `None` → omitted from prompt → SQP behaves as before |
| Different topic | SQP sees no reference to prior scope → ignores prior_context |
| Explicit override ("full year" vs "YTD") | SQP drops prior date filter, uses new one |
| Different workflow | Key includes `workflow_id` → no cross-workflow leakage |
| Stale context (>24h) | TTL expires → treated as first question |

### Files Changed

| File | Change |
|------|--------|
| `question_answering_configurations.py` | `QAPriorContext` dataclass, `enable_context_carryover` flag |
| `run_store/redis_store.py` | `store_prior_context()`, `get_prior_context()` with 24h TTL |
| `run_store/memory.py` | Same interface for local dev |
| `qa_base_pipeline.py` | Load prior context before planning, `_extract_filters_from_plan()`, store after answer, thread `prior_context` through `_build_query_plan` |
| `semantic_query_planner.py` | `prior_context` param threaded through `build_semantic_query_plan_batch` → `select_fields_and_record_plans_batch_all_sections` → `_build_user_prompt_batch_fields_and_records`; injected into SQP JSON payload; rule 12 in both system prompts |
| `qa_pipelines.py` | `enable_context_carryover=True` for S360 |

### Remaining Work

| Item | Status | Priority |
|------|--------|----------|
| Enable for other pipelines | Not started | Medium |
| DQP support for prior constraints | Not started | Medium |

---

## Smart Aggregation Cap Sorting

### Problem

When aggregation produces more groups than `max_rows` (e.g., 41,704 item groups → capped to 200), the smart cap sorted by the **first aggregation column** (e.g., `total_qty_2025`). This meant items with high spend but lower quantity could be excluded from the top 200 — invisible to the LLM.

### Solution

Prefer computed fields for sorting when they exist. The sort priority is now:
1. **Computed field columns** (e.g., `estimated_spend_2026`) — the derived metric the user asked for
2. **Aggregation columns** (fallback) — first agg column as before

This ensures the top 200 rows contain the items most relevant to the user's question.

**No behavior change when computed fields are absent** — `computed_col_names` is empty, `candidate_cols` falls back to `agg_cols_renamed`.

**File:** `structured_data_retriever.py` — smart cap section in `_apply_aggregate_df`

---

## Redis Performance: orjson Serialization

### Problem

`json.dumps` on 4.7M rows (S360 workflow output) was extremely slow — pure Python iteration over every value. This was a significant portion of the ~6 minute processing + Redis store time.

### Solution

Replaced `json.dumps` → `orjson.dumps` and `json.loads` → `orjson.loads` in `RedisRunStore._set_json` / `_get_json`.

- **orjson**: Rust-based JSON serializer, ~8.8x faster than stdlib `json`
- Returns `bytes` directly (no `.encode("utf-8")` needed)
- `orjson.loads` accepts bytes directly (no `.decode("utf-8")` needed)
- Backwards compatible — produces standard JSON, reads standard JSON from old entries

**Benchmark (100K rows, extrapolated to 4.7M):**

| Serializer | Time (4.7M rows) |
|-----------|-------------------|
| `json.dumps` | ~4.0s |
| `orjson.dumps` | ~0.5s |

**File:** `run_store/redis_store.py` — `_set_json` and `_get_json`

---

## Implementation Roadmap (Updated)

### Phase 2.0: Computed Fields + Temporal Resolution — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| `ComputedField` model + `computed_fields` on `RecordPlan` | Done | `semantic_query_planner.py` |
| Safe AST-based expression evaluator (`_eval_computed_field`) | Done | `structured_data_retriever.py` — all arithmetic ops, parentheses, numeric literals; `ast` module, no `eval()` |
| SQP prompt rule 11 (computed fields) | Done | Both batch and non-batch system prompts |
| Computed field grand total (sum across ALL groups before cap) | Done | `_apply_aggregate_df` — try/except protected |
| Smart cap sort by computed fields | Done | Prefer computed field columns over agg columns for smart cap sorting |
| Current date injection in SQP prompts | Done | `datetime.now().strftime("%Y-%m-%d")` in user prompts |
| DQP temporal filter resolution (`_resolve_temporal_filters`) | Done | YTD, MTD, QTD, last year — deterministic |
| Date comparison in retriever filter engine | Done | YYYY-MM-DD pattern → `pd.to_datetime` with tz-aware localization |
| `enable_row_cap` config parameter | Done | Infrastructure ready, S360 currently `True` |
| Positional aggregation sort (first/last → date order) | Done | `_infer_date_column()` + pre-aggregation sort |
| Redis gzip compression for large payloads | Done | `_set_json`/`_get_json` with `__gz__` prefix marker |
| Redis orjson serialization (~8.8x faster) | Done | `orjson.dumps`/`orjson.loads` in `RedisRunStore` |

### Phase 2.1: Security Guardrails — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| `security_guardrails.py` module extraction | Done | `is_blocked_input()`, `sanitize_output_text()`, `BLOCKED_RESPONSE` |
| Regex-based capability discovery patterns (6 patterns) | Done | Require capability noun + self/system reference |
| Deterministic short-circuit in main.py + console_app.py | Done | Block before LangGraph, return canned response |
| Prompt guardrail for tool/data disclosure | Done | `GUARDRAIL_SYSTEM_INSTRUCTION` update |
| Removed redundant `sanitize_user_messages` middleware | Done | Entry-point blocking is sufficient |

### Phase 2.2: Conversational Context Carryover — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| `QAPriorContext` data model | Done | `question_answering_configurations.py` |
| Constraint extraction from `SemanticQueryPlan` | Done | `_extract_filters_from_plan()` in `qa_base_pipeline.py` |
| Redis/InMemory storage (24h TTL) | Done | `store_prior_context()` / `get_prior_context()` in both stores |
| `prior_context` injection into SQP prompt | Done | Threaded through batch plan builder, injected in user prompt JSON |
| SQP rule 12 (constraint inheritance) | Done | Per-filter INHERIT/DROP/IGNORE decision framework |
| `enable_context_carryover` config flag | Done | S360 only (`True`), all others `False` (default) |
| Enable for other pipelines | Not started | Medium priority |
| DQP support for prior constraints | Not started | Medium priority |

### Phase 2.3: Null Filters + Temporal Grouping + Derived Group-By — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| `is_null` / `is_not_null` filter ops | Done | Added to `FILTER_OPS`, `Op` Literal, `NULL_CHECK_OPS` in `question_answering_models.py` |
| Null check handling in retriever | Done | `_apply_single_filter` checks `col.isna()` + empty string + "nan"/"None" strings |
| SQP rule 9 (null filter guidance) | Done | Both system prompts: `is_null` for "no supplier", "missing vendor", etc. |
| Derived `group_by` (`year_of:`, `month_of:`, `quarter_of:`) | Done | `_DERIVED_GROUP_BY_EXTRACTORS` in `_apply_aggregate_df` — extracts time period from date columns |
| SQP rule 11 (derived group_by guidance) | Done | Both system prompts: prefix notation for temporal grouping |
| SQP rules renumbered (1–14) | Done | Both system prompts consistent |
| DQP null filter detection | Done | `NULL_FILTER_PATTERNS` + `_NULL_FIELD_KEYWORDS` → `_resolve_null_filters()` |
| DQP temporal grouping detection | Done | `TEMPORAL_GROUPING_PATTERNS` → auto-upgrade to aggregate mode + inject `year_of:date_field` |
| DQP `IntentSignals` extended | Done | `is_null_filter`, `is_temporal_grouping`, `null_filter_fields`, `temporal_grouping_period` |

### Phase 2.4: Gemini 3.0 Pro Deprecation — COMPLETED

| Item | Status | Implementation |
|------|--------|----------------|
| Remove `GEMINI_3_0_PRO` from model enums | Done | `model_ontology.py` |
| Router L2 → Gemini 3.1 Pro | Done | `router_agent.py` — L2 RETRIEVAL/MATH now use 3.1 Pro with `medium` thinking |
| Remove 3.0 Pro thinking level map | Done | `router_agent.py` — only Flash, 3.1 Pro, 2.5 Pro/Flash entries remain |
| L2 fallback pool cross-provider | Done | `[sonnet, 3.1_pro, opus, 2.5_pro]` — Gemini always available as fallback |
| Remove from authorized models | Done | `question_answering_configurations.py` — 7 models (was 8) |
| Update test expectations | Done | `test_routing_agent.py` — L2 model, thinking level, fallback chains |
| Synced to reasoning-engine | Done | All 8 files updated in reasoning-engine repo |
