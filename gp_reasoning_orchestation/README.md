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
│  BaseQAPipeline.solve()                                                     │
│  │                                                                          │
│  │  ┌──────────────────────────────────────┐                                │
│  ├─▶│  1. SemanticQueryPlannerFactory      │  (Strategy Pattern)            │
│  │  │     ├─ LLMOntologyDrivenPlanner      │  ← LLM-based planning         │
│  │  │     ├─ DeterministicQueryPlanner      │  ← BM25 + rules (no LLM)     │
│  │  │     └─ HybridQueryPlanner (TODO)      │  ← Deterministic + LLM fallbk│
│  │  └──────────────────────────────────────┘                                │
│  │         ║  (parallel)                                                    │
│  │  ┌──────────────────────────────────────┐                                │
│  │  │  ReasoningProtocolDesigner           │  ← Dynamic reasoning protocol  │
│  │  └──────────────────────────────────────┘                                │
│  │         ↓                                                                │
│  │  ┌──────────────────────────────────────┐                                │
│  ├─▶│  POST-PROCESS: _optimize_query_plan  │  (Working Memory Optimization) │
│  │  │     ├─ Shared data partition dedup   │                                │
│  │  │     └─ Partition max_rows override   │                                │
│  │  └──────────────────────────────────────┘                                │
│  │         ↓                                                                │
│  │  ┌──────────────────────────────────────┐                                │
│  ├─▶│  2. StructuredDataRetriever          │  ← Symbolic execution (pandas) │
│  │  └──────────────────────────────────────┘                                │
│  │         ↓                                                                │
│  │  ┌──────────────────────────────────────────────────────────────┐        │
│  ├─▶│  3. GroundedAnswerEngine                                     │        │
│  │  │     └─ Smart RouterAgent (5-level × 5-task model selection)  │        │
│  │  │         ├─ Stage 1: Fast regex exit (<1ms)                   │        │
│  │  │         ├─ Stage 2: Multi-signal weighted scoring (<5ms)     │        │
│  │  │         ├─ authorized_models validation                      │        │
│  │  │         └─ Cross-provider fallback chain                     │        │
│  │  └──────────────────────────────────────────────────────────────┘        │
│  │         ↓                                                                │
│  └─▶ _append_response_metadata()            ← Timestamp + response time     │
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
- Repair mechanism: Retries with context if first attempt fails
- **Smart RouterAgent**: Selects the optimal model via 5-level × 5-task-type routing (see section below)

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

### 7. Response Metadata (`_append_response_metadata` in `qa_base_pipeline.py`)

**Output format (italic markdown at bottom of answer):**
```
---
*Response generated at 2026-02-13 15:30:45 | Response time: 2.3s*
```

---

## Smart RouterAgent (Phase 1)

> **File:** `router_agent.py`
> **Design doc:** `smart_router_agent.md`
> **Test harness:** `check_router_agent.py`

The RouterAgent replaces the old binary Flash-vs-Pro routing with a deterministic 5-level × 5-task-type model selection system. It makes **zero LLM calls** — all routing decisions are regex + weighted scoring.

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

### Routing Table (Level × TaskType → Model)

| Level | Retrieval | Math | Comparison | Analysis | Creative |
|-------|-----------|------|------------|----------|----------|
| **L0** | Flash | Flash | Flash | Flash | Flash |
| **L1** | Flash | Flash | Flash | Flash | Haiku |
| **L2** | Pro 3.0 | Pro 3.0 | Sonnet | Sonnet | Sonnet |
| **L3** | Pro 3.1 | Pro 3.1 | Opus | Opus | Opus |
| **L4** | Pro 3.1 | Pro 3.1 | Opus | Opus | Opus |

**Design principle:** Gemini handles structured data tasks (retrieval, math). Claude handles language-quality tasks (comparison, analysis, creative).

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

### Routing Results (98 Test Questions)

See [routing_results.md](./routing_results.md) for the full table.

**Level distribution:**

| Level | Count | % |
|-------|------:|--:|
| L0_LOOKUP | 36 | 36.7% |
| L1_SIMPLE | 28 | 28.6% |
| L2_ANALYTICAL | 30 | 30.6% |
| L3_COMPLEX | 4 | 4.1% |

**Model distribution:**

| Model | Count | % |
|-------|------:|--:|
| gemini-3-flash | 62 | 63.3% |
| gemini-3-pro | 18 | 18.4% |
| claude-sonnet-4-6 | 12 | 12.2% |
| claude-opus-4-6 | 4 | 4.1% |
| claude-haiku-4-5 | 2 | 2.0% |

---

## Configuration

### GroundedAnswerEngineConfig (`question_answering_configurations.py`)

Controls model routing and LLM call parameters for the answer engine.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `temperature` | 0.3 | LLM sampling temperature |
| `max_output_tokens` | 200,000 | Max output tokens |
| `max_input_tokens` | 200,000 | Max input tokens (rejects if exceeded) |
| `max_attempts` | 2 | Retry count with repair prompt |
| `agent_request_time_out` | 60 | LLM call timeout (seconds) |
| `authorized_models` | 8 models | Models the router may use (see table above) |
| `fallback_chain` | 6 models | Cross-provider resilience ordering |

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

| Workflow ID | Pipeline Class |
|-------------|---------------|
| `competitive_pricing_intelligence_rheem_workflow` | `PricingCompetitiveIntelligenceQAPipeline` |
| `pricing_simulator_intelligence_workflow` | `PricingScenarioSimulatorQAPipeline` |
| `warranty_intelligence_rheem_workflow` | `WarrantyIntelligenceQAPipeline` |

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

## Key Files

| File | Purpose |
|------|---------|
| `qa_base_pipeline.py` | Abstract base class for all Q&A pipelines |
| `qa_pipelines.py` | Per-workflow pipeline implementations + registry |
| `question_answering_configurations.py` | All config dataclasses (QAToolConfig, SemanticQueryPlannerConfig, GroundedAnswerEngineConfig) |
| `engine/grounded_answer.py` | Final answer generation with RouterAgent integration |
| `engine/router_agent.py` | Smart 5-level model router (Phase 1) |
| `engine/check_router_agent.py` | Test harness for 98 questions across 5 workflows |
| `engine/semantic_query_planner.py` | LLM-based query planning |
| `engine/deterministic_query_planner.py` | Rule-based query planning (no LLM) |
| `engine/semantic_query_planner_factory.py` | Strategy pattern factory |
| `engine/structured_data_retriever.py` | Pandas-based data retrieval |
| `engine/reasoning_protocol_designer.py` | Dynamic reasoning protocol generation |

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
| 5-task-type detection (retrieval/math/comparison/analysis/creative) | Done | `TaskType` + `_detect_task_type_weighted()` |
| Two-stage routing (fast exit + multi-signal scoring) | Done | `_stage1_fast_exit()` + `_stage2_full_scoring()` |
| (Level × TaskType) → model routing table | Done | `_build_routing_table()` (Gemini for data, Claude for language) |
| Enterprise strategic pattern recognition | Done | `ENTERPRISE_STRATEGIC_PATTERNS` (29 regex patterns) |
| Gemini thinking config (model-aware: thinkingLevel vs thinkingBudget) | Done | `_build_thinking_config()` per model family |
| authorized_models validation | Done | `_is_authorized()` + `_find_authorized_alternative()` |
| Cross-provider fallback chain | Done | `_build_fallbacks()` with provider alternation |
| GroundedAnswerEngine integration | Done | `router.route()` replaces binary routing |
| Per-workflow config via QAToolConfig | Done | `grounded_answer_config` in `QAToolConfig` |
| Test harness (98 questions, 5 workflows) | Done | `check_router_agent.py` |

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
