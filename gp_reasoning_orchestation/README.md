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

**Purpose:** Fast, rule-based planning without LLM calls (Phase A implementation).

**Sub-components:**
- `DeterministicSearchIndex` — BM25 lexical + optional embedding search for section selection
- `DeterministicFieldSelector` — Token/synonym/description scoring for field selection
- `DeterministicRecordPlanner` — Rule-based intent detection for record plan (top_k, all, filter, aggregate)

**When to use:** For workflows where latency matters and questions follow predictable patterns.

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
| `all` | Return all rows (capped by `max_rows`, default 200) |
| `top_k` | Return top K rows sorted by a field |
| `filter` | Return rows matching filter conditions (==, !=, contains, >, <, etc.) |
| `aggregate` | Group-by with aggregations (sum, count, mean, min, max) |
| `none` | Skip this partition (no data retrieved) |

**Guardrails:**
- Row capping: `mode="all"` respects `max_rows` to prevent token explosion
- Field projection: Returns only selected columns
- Fail-safe: Configurable behavior on bad query plans (`fail_open_on_bad_plan`)
- `skip_reason` logging when a partition is intentionally skipped by optimizer

### 6. GroundedAnswerEngine (`grounded_answer.py`)

**Purpose:** Generate final answers grounded in retrieved evidence.

**Capabilities:**
- Non-negotiable rules: No fabrication, no external knowledge, no contradiction
- Token budget: Max 200K input tokens with overflow handling
- Map-reduce: Automatic chunking when evidence exceeds `max_input_tokens`
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
| **L2** | Pro 3.0 | Pro 3.0 | Sonnet | Sonnet | Sonnet | Opus |
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

**Default authorized models (8):**

| Model | Provider | Role |
|-------|----------|------|
| Gemini 3.0 Flash | Vertex AI | Fast/cheap (L0–L1) |
| Gemini 3.0 Pro | Vertex AI | Balanced structured data (L2) |
| Gemini 3.1 Pro | Vertex AI | Heavy structured data (L3–L4) |
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
| `engine/deterministic_query_planner.py` | Rule-based query planning (no LLM) |
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

### Phase B: Confidence Gating — NOT YET

| Item | Status |
|------|--------|
| Confidence gating + margin thresholds | Planned |
| Split batch planner into targeted mini-batches | Planned |
| Cache at the right granularity | Planned |
| `HybridQueryPlanner` (deterministic first, LLM fallback) | Scaffolded (`NotImplementedError`) |

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
