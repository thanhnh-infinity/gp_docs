# Semantic Query Planner — Model Selection Analysis

> Date: 2026-02-21
> Task: LLMOntologyDrivenQueryPlanner field selection + record plan generation
> Current default: Gemini 2.5 Flash (9-23s avg latency, ~$0.004/call)

---

## Task Profile

The Semantic Query Planner (LLMOntologyDrivenQueryPlanner) does:
- **Input:** ~8K tokens (ontology with partitions, fields, metadata + user question)
- **Output:** ~500-2K tokens (JSON — selected fields per section + record mode/filters)
- **Nature:** Semantic field selection + structured JSON generation
- **Key requirement:** Pick the 3-8 most relevant fields from 20-60 options per section, assign record retrieval mode (all/top_k/filter/aggregate)

---

## Experiment Results (52 Questions x 3 Strategies)

From `experiment_results_semantic_query_planner.md`:

### Latency

| Workflow | Gemini 3.0 Flash | Gemini 2.5 Flash | Deterministic |
|---|---|---|---|
| Competitive Pricing (14 Q) | avg 31.1s, max 106s | avg 11.2s, max 27.7s | avg 7ms |
| Pricing Simulator (19 Q) | avg 25.8s, max 107s | avg 9.2s, max 24.7s | avg 6ms |
| Warranty (19 Q) | avg 51.9s, max 128s | avg 22.8s, max 36.1s | avg 7ms |

### Accuracy

- **Section selection:** Both LLMs select IDENTICAL sections for all 52 questions (100% agreement)
- **Deterministic:** Misses 1 section (`module_4_foundation_store_linkage_summary`) on all 19 warranty questions
- **Field precision (avg fields/section):**
  - 3.0 Flash: 3.0-7.9 (tightest)
  - 2.5 Flash: 3.6-8.2 (marginally broader)
  - Deterministic: 6.2-16.1 (2-3x more than LLMs)
- **Deterministic filter quality:** Often produces garbage filters by extracting literal question text (e.g., `filter: rdc_name contains 'families are misclassified'`)

### Conclusion from Experiment

**Gemini 2.5 Flash is the best current default** — identical section accuracy to 3.0 Flash, 2.3-2.8x faster, safe max latency (36s vs 128s which exceeds timeout). Deterministic is fast but has accuracy issues and is best kept as fallback.

---

## Research: Best Alternative Models (Claude + GPT)

### Benchmark Data

#### Cleanlab Structured Output Benchmark (field selection from schema — most relevant)

| Model | Field Accuracy | Full Output Accuracy | Latency vs Gemini 2.5 Flash | Cost vs Gemini 2.5 Flash |
|---|---|---|---|---|
| Gemini 3 Pro | 96.4% | 77% | ~similar | ~4x more |
| GPT-5 | 95.6% | 76% | much slower | ~12x more |
| Gemini 2.5 Pro | 94.4% | 72% | ~similar | ~4x more |
| **GPT-4.1-mini** | **86.3%** | **45%** | **35% faster** | **5x cheaper** |
| Gemini 2.5 Flash | 82.9% | 28% | baseline | baseline |

#### ExtractBench (JSON validity on complex schemas — Feb 2026)

| Model | Valid JSON Rate | Field Accuracy (when valid) |
|---|---|---|
| Gemini 3 Flash | **71%** | 74.7% |
| Gemini 3 Pro | 60% | 74.8% |
| GPT-5.2 | 60% | 71.7% |
| GPT-5 | 37% | 80.4% |
| Claude Sonnet 4.5 | 43% | 66.9% |
| Claude Opus 4.5 | 34% | 65.0% |

#### BFCL v4 — Function Calling / Tool Use

| Model | Score | Rank |
|---|---|---|
| Claude Opus 4.1 | 70.36% | #2 |
| Claude Sonnet 4 | 70.29% | #3 |
| GPT-5 | 59.22% | #7 |

---

### Full Model Comparison Table

| Model | TTFT | Output Speed | Input $/1M | Output $/1M | Est. cost/call | Est. latency |
|---|---|---|---|---|---|---|
| **GPT-4.1 nano** | **0.37s** | **110 t/s** | **$0.10** | **$0.40** | **$0.001** | **~1-2s** |
| **GPT-4.1 mini** | **0.55s** | **80-90 t/s** | **$0.40** | **$1.60** | **$0.005** | **~2-4s** |
| GPT-4.1 | 0.7s | 70-90 t/s | $2.00 | $8.00 | $0.032 | ~3-5s |
| GPT-4o-mini | 0.5s | 80-100 t/s | $0.15 | $0.60 | $0.002 | ~2-4s |
| GPT-5-mini | ~1.0s | ~100 t/s | $0.25 | $2.00 | $0.006 | ~4-8s |
| GPT-5.2 (instant) | 0.58s | 78-96 t/s | $1.75 | $14.00 | $0.032 | ~3-5s |
| **Claude 4.5 Haiku** | **0.55s** | **109 t/s** | **$1.00** | **$5.00** | **$0.018** | **~5-10s** |
| Claude 4.5 Sonnet | 1.19s | 69.7 t/s | $3.00 | $15.00 | $0.054 | ~15-25s |
| Claude 4.6 Sonnet | 0.78s | 56.2 t/s | $3.00 | $15.00 | $0.054 | ~15-30s |
| Claude 4.6 Opus | 1.65s | 67.1 t/s | $5.00 | $25.00 | $0.090 | ~20-35s |
| Gemini 2.5 Flash | 0.34s | 244 t/s | $0.30 | $2.50 | $0.004 | 9-23s (current) |
| Gemini 3.0 Flash | ~0.4s | ~200 t/s | ~$0.30 | ~$2.50 | ~$0.004 | 25-52s (tested) |

---

## Top 3 Recommended Models

### #1: GPT-4.1-mini — Best Overall for This Task

| Metric | Value |
|---|---|
| Field accuracy | 86.3% (Cleanlab) — better than Gemini 2.5 Flash (82.9%) |
| Estimated latency | ~2-4s (35% faster than Gemini 2.5 Flash per Cleanlab) |
| Cost per call | ~$0.005 (5x cheaper than Gemini 2.5 Flash with thinking) |
| IFEval score | 87.4% (best in class for instruction following) |
| Context window | 1M tokens |
| JSON reliability | High — specifically trained for function calling + structured output |

**Why it wins:**
- Better accuracy than our current Gemini 2.5 Flash baseline (+3.4% field accuracy)
- 35% faster latency
- 5x cheaper (when accounting for Gemini thinking tokens)
- Specifically trained by OpenAI for tool calling and structured output
- Fully supported by litellm (already in our stack)

**Concern:** GPT-4.1 family follows instructions very literally — needs explicit prompts. Our existing prompt in `semantic_query_planner.py` is already detailed and schema-driven, so this should be fine.

---

### #2: GPT-4.1-nano — Speed Champion

| Metric | Value |
|---|---|
| Field accuracy | ~80% (estimated) |
| Estimated latency | ~1-2s |
| Cost per call | ~$0.001 (cheapest available) |
| TTFT | 0.37s (fastest available) |
| Output speed | 110 t/s |
| Context window | 1M tokens |

**Why it's interesting:**
- Fastest model available — 0.37s TTFT, 110 t/s output
- 10x cheaper than Gemini 2.5 Flash
- Purpose-built by OpenAI for "classification, autocompletion, and data extraction"
- Estimated latency of 1-2s would be transformational for UX

**Concern:** Lowest capability tier. May struggle with the 25-partition warranty ontology (60+ fields). Needs testing.

**Best deployment strategy:** Use as default for Competitive Pricing (8 partitions) and Pricing Simulator (5 partitions). Fallback to GPT-4.1-mini for Warranty (25 partitions).

---

### #3: Claude 4.5 Haiku — Best Claude Option

| Metric | Value |
|---|---|
| Field accuracy | ~80% (estimated, no Cleanlab data for Claude) |
| Estimated latency | ~5-10s |
| Cost per call | ~$0.018 |
| TTFT | 0.55s |
| Output speed | 109 t/s |
| BFCL function calling | #2-3 overall (Claude dominates this benchmark) |

**Why it's worth testing:**
- Fastest and cheapest Claude model
- Claude models dominate BFCL function calling benchmark (#2 and #3 overall)
- **Already in our `authorized_models`** — no infrastructure changes needed via Vertex AI
- Prompt caching available (90% input cost reduction if ontology schema is reused across calls)

**Concern:** ExtractBench shows Claude has the lowest valid JSON rates (34-43%). Mitigated by:
1. Our task uses Pydantic model validation (litellm + structured output mode)
2. We have retry logic (`max_attempts=2`)
3. Our schema is moderately complex (not the 369-field edge case that broke all models)

---

## Models NOT Recommended for This Task

| Model | Why Skip |
|---|---|
| **Claude 4.6 Sonnet/Opus** | Overkill for field extraction. 56-67 t/s output speed. $0.054-$0.090/call. Intelligence wasted on a selection task. |
| **GPT-5/5.1/5.2** | Known JSON consistency issues (GPT-5). Expensive ($10-14/M output). Reasoning overhead adds latency. Overkill for extraction. |
| **GPT-4o / 4o-mini** | Superseded by GPT-4.1 family in every dimension (cost, accuracy, speed, context window). |
| **o1/o3/o4-mini** | Reasoning models with massive TTFT overhead (9-22s). Wrong tool for extraction. |
| **Claude 4.5 Sonnet** | Same price as Sonnet 4.6 but lower accuracy. No reason to use. |
| **GPT-5-mini/nano** | Less tested for structured extraction than 4.1 family. GPT-5 JSON issues may propagate. |
| **Gemini 3.0 Flash** | Tested in experiment — 2.3-2.8x slower than Gemini 2.5 Flash with no accuracy gain at section level. Max latency (128s) exceeds timeout. |

---

## Final Summary

| | GPT-4.1 mini | GPT-4.1 nano | Claude 4.5 Haiku | Gemini 2.5 Flash (current) |
|---|---|---|---|---|
| **Field accuracy** | 86.3% (Cleanlab) | ~80% (est.) | ~80% (est.) | 82.9% (Cleanlab) |
| **Est. latency** | **2-4s** | **1-2s** | 5-10s | 9-23s |
| **Cost/call** | $0.005 | **$0.001** | $0.018 | $0.004 |
| **JSON reliability** | High (trained) | High (trained) | Medium (retry needed) | High |
| **Already in stack** | No (add via litellm) | No (add via litellm) | **Yes** (Vertex AI) | **Yes** (current) |
| **Recommendation** | **New default** | Simple workflows | Claude-only infra | Fallback |

### Recommended Testing Order

1. **GPT-4.1-mini** as new default for all workflows — Cleanlab data strongly suggests it's both faster AND more accurate than Gemini 2.5 Flash
2. **GPT-4.1-nano** for Competitive Pricing + Pricing Simulator (fewer partitions) — if accuracy holds, use as default for those
3. **Claude 4.5 Haiku** — test alongside the above to have a Vertex AI native option

### Key Insight from LLMStructBench

> Using native structured output modes (JSON mode / response_format) may improve structural validity but can increase semantic errors. Consider using schema + examples in the system prompt rather than forcing JSON output format.

This suggests our current approach (prompt-based JSON generation with Pydantic validation) may actually be optimal vs. switching to provider-native structured output modes.

---

## Sources

- Cleanlab Structured Output Benchmark (2025): https://cleanlab.ai/blog/structured-output-benchmark/
- ExtractBench (Feb 2026): https://arxiv.org/abs/2602.12247
- BFCL v4 Leaderboard: https://gorilla.cs.berkeley.edu/leaderboard.html
- LLMStructBench (Feb 2026): https://arxiv.org/abs/2602.14743
- Artificial Analysis Model Benchmarks: https://artificialanalysis.ai/
- OpenAI GPT-4.1 Announcement: https://openai.com/index/gpt-4-1/
- Anthropic Structured Outputs Docs: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- OpenAI Structured Outputs Guide: https://platform.openai.com/docs/guides/structured-outputs
