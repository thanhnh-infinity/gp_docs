# Improving the Deterministic Semantic Query Planner to Match LLM Performance

## Current Architecture Overview

The deterministic pipeline has **4 stages** (updated after tuning session 2025-03):

1. **Stage 1 — Section Selection** (two modes):
   - **Suggestion mode** (`use_suggestions_only=True`): Includes ALL suggested partitions (no BM25 relevance gate). Delegates relevance decisions to Stage 2's content quality gate. This was the **single biggest accuracy win** (~15% gain) — previously, BM25 filtering removed sections that the reference LLM keeps as `mode="none"`, devastating section Jaccard.
   - **Search mode** (`use_suggestions_only=False`): BM25 + `all-MiniLM-L6-v2` embeddings (384d), weighted `0.6*BM25 + 0.4*cosine`.
   - **Domain affinity** (config-driven): Detects question domain via keyword matching (`domain_keywords` config), then boosts/prioritizes sections matching that domain's prefixes (`domain_section_prefixes` config). Domains and prefixes are externalised to config with built-in defaults as fallback — new domains can be added without code changes.
   - **Numbered variant routing**: Detects "scenario N" references and ensures matching sections are included (`numbered_variant_pattern` config).

2. **Stage 2 — Field Selection**: Token overlap + synonym expansion + description matching + importance scoring
   - Weights: `0.3*token + 0.2*synonym + 0.3*description + 0.2*importance`
   - **Content quality gate** (`content_quality_gate=0.18`): At least one field must have meaningful content relevance (token+synonym+description score, excluding structural importance). If no field passes, section gets `mode="none"` with 0 fields. This prevents structural fields (id, name) from keeping irrelevant sections alive.
   - **Adaptive relative threshold**: `relative_threshold_ratio=0.40` scales up for large sections (>15 fields: +0.05, >20: +0.12, >30: +0.20) to reduce noise.
   - **Conditional auto-include**: Structural fields (id, name, date) only auto-included if (a) content-relevant fields already selected AND (b) the structural field has some content relevance.
   - **Business concept synonyms**: Lean set of domain-aware synonyms (opportunity→profit/margin, impact→profit/revenue, segment→category, etc.).

3. **Stage 3 — Record Plan**: Regex-based intent detection → `all / top_k / filter / aggregate`
   - **Broad question detection** expanded: patterns for "design a/another/complete", "optimal", "optimize", "what actions would", "highlight", "financial impact", "which offers".
   - **Forward-looking sort field preference**: For questions with "expected", "optimal", "scenario", "forecast" keywords, prefers fields with "expected"/"optimal" prefixes over "current" prefixes in sort selection.

4. **Stage 4 — Config-Driven Orchestration** (`deterministic_query_planner.py`):
   - All domain-specific logic externalized to `SemanticQueryPlannerConfig` dataclass fields (`domain_keywords`, `domain_section_prefixes`, `numbered_variant_pattern`, `important_partitions`).
   - New domains scale by adding config — zero code changes required.

---

## Where the Deterministic Planner Falls Short vs LLMs

### What Was FIXED (session 2025-03)

1. **Section selection relevance gate** (FIXED — biggest win): Previously BM25 gate excluded sections the reference LLM keeps as `mode="none"`. Now all suggested partitions are included, and the content quality gate in field selection decides "none" vs real fields. **+15% accuracy gain.**
2. **Field over-selection in irrelevant sections** (FIXED): Content quality gate (`content_quality_gate=0.18`) prevents structural fields from keeping irrelevant sections alive.
3. **Field noise in large sections** (FIXED): Adaptive relative threshold tightens scoring for sections with many fields.
4. **Domain-specific hardcoding** (FIXED): Refactored from class constants to config-driven with defaults. New domains add config, not code.
5. **Missing broad question patterns** (FIXED): Added "design", "optimal", "optimize", "financial impact", "which offers" etc.
6. **Sort field for forward-looking questions** (FIXED): "expected"/"optimal" prefix preference for scenario/forecast questions.

### What Still Falls Short

#### Section Selection (search mode, non-suggestion path)
- `all-MiniLM-L6-v2` is a **2021-era 384d model** — misses nuanced semantic relationships
- No reranking — initial retrieval score is final score
- "Where is cannibalization eating gross profit?" requires understanding `cannibalization → SKU overlap → margin share analysis`, which BM25/MiniLM can't do
- **Note**: In suggestion mode (most common), this is largely bypassed since all suggested partitions are included

#### Field Selection — Still the Weakest Link
- Scoring is purely lexical — no semantic understanding
- "What is the average elasticity?" → needs `price_sensitivity`, `demand_curve_slope` — token overlap is 0
- Synonym dict helps but can't cover all domain relationships (aggressive synonyms hurt accuracy — tested and reverted)
- Description matching helps but is still token-based, not semantic
- **Key gap**: Composition workflow (67.7%) and Pricing Simulator (62.5%) suffer most here because field names don't lexically match question terms

#### Record Plan — Regex Still Brittle
- "Which SKUs truly belong together across systems?" → no intent detected → falls back to `top_k(20)` (should be `all`)
- "Can you calculate scenario 2 & 3's financial impact as a percentage..." → complex multi-step intent regex can't parse
- Entity extraction misses implicit references ("Home Depot" in "THD")
- **Broad question detection improved** but still pattern-based — misses novel phrasings

---

## Concrete Improvements That Could Close the Gap

Ranked by **impact / effort ratio**. Updated with notes on what's been partially addressed.

### High Impact, Low-Medium Effort

#### 1. Upgrade Embedding Model (Section + Field Selection) — STILL RECOMMENDED

Replace `all-MiniLM-L6-v2` (384d, 2021) with a modern model:

| Model | Dim | MTEB Score | Size | Latency |
|-------|-----|-----------|------|---------|
| `all-MiniLM-L6-v2` (current) | 384 | 56.3 | 80MB | ~1ms |
| `bge-large-en-v1.5` | 1024 | 64.2 | 1.3GB | ~5ms |
| `gte-large-en-v1.5` | 1024 | 65.4 | 1.3GB | ~5ms |
| `snowflake-arctic-embed-m-v2.0` | 768 | 63.2 | 300MB | ~3ms |

**Status**: Still the single biggest potential win for the **search mode** path and for **field selection**. In suggestion mode, section selection is mostly solved (all partitions included), but embedding-based field selection (#3) is where this matters most now. Consider prioritizing #3 over #1 if suggestion mode is dominant.

#### 2. Add Cross-Encoder Reranking (Section Selection) — LOWER PRIORITY NOW

After initial BM25+embedding retrieval of top 20 sections, rerank with a cross-encoder:

```python
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-12-v2")
pairs = [(question, section_text) for section_text in candidate_sections]
rerank_scores = reranker.predict(pairs)
```

**Status**: Lower priority because suggestion mode bypasses search entirely. Only valuable if non-suggestion mode becomes common. Cost: ~10-20ms for 20 candidates.

#### 3. Embedding-Based Field Selection (Replace/Augment Token Overlap) — HIGHEST PRIORITY

Instead of token overlap for fields, compute cosine similarity between question embedding and each field's `(name + description)` embedding:

```python
field_emb = model.encode(f"{field_name}: {field_description}")
question_emb = model.encode(question)
field_score = cosine(question_emb, field_emb)
```

**Status**: This is now the **#1 recommended improvement**. Field selection is the weakest link — it's purely lexical, and our tuning session showed that aggressive synonym expansion hurts more than it helps. Embedding-based scoring would fix cases like `"elasticity" → "price_sensitivity"` without fragile synonym dicts. Would likely close the gap for Pricing Simulator (62.5%) and Composition (67.7%).

### Medium Impact, Medium Effort

#### 4. Query Expansion with Ontology Context — PARTIALLY ADDRESSED

Before searching, expand the query using the ontology's own vocabulary:

```python
ontology_terms = {}  # "elasticity" → ["price_sensitivity", "demand_curve_slope"]
expanded_query = question + " " + " ".join(related_ontology_terms)
```

**Status**: Partially addressed by description matching in field selection (we already match question tokens against ontology metadata descriptions). A more aggressive version could pre-build a term→field index from all ontology descriptions. Worth exploring if #3 alone isn't enough.

#### 5. Small Classifier for Record Mode (Replace Regex Intent Detection) — STILL RECOMMENDED

Train a tiny classifier on LLM outputs:

```python
# Training data from experiment: (question, correct_mode) pairs
from sklearn.linear_model import LogisticRegression
clf = LogisticRegression()
clf.fit(X_train, y_train)  # ~50-200 examples from experiment runs
```

**Status**: Training data already exists from experiment runs. Broad question detection was improved with more regex patterns, but a classifier would be more robust. This is **knowledge distillation** from LLM to a tiny model — high ROI with the data we already have.

#### 6. NER-Based Entity Extraction (Replace Regex Entity Detection) — STILL RECOMMENDED

```python
from transformers import pipeline
ner = pipeline("ner", model="dslim/bert-base-NER", grouped_entities=True)
entities = ner("Which installers are driving warranty cost per region?")
```

**Status**: Entity extraction still misses implicit references ("THD" → "Home Depot"). Custom domain NER trained on ontology entities (store names, SKU patterns, region codes) would help. Medium priority — most impactful for filter mode accuracy.

### High Impact, High Effort (But Worth Considering)

#### 7. Distill the LLM into the Deterministic Pipeline — GOOD LONG-TERM DIRECTION

Use experiment data to systematically learn from LLM decisions:

- **Section selection**: Less critical now (suggestion mode solved it)
- **Field selection**: Collect (question + section, LLM-selected-fields) pairs → train a field relevance model. **This is the most impactful target for distillation.**
- **Record plan**: Covered in #5 above

**Status**: With experiment infrastructure in place, data collection is straightforward. Start with field selection distillation — it addresses the weakest link directly.

#### 8. Graph-Aware Section/Field Linking — EXPLORE AFTER BASICS

The ontology has `graph` and `rules` fields that the deterministic planner currently ignores. These contain relationships between sections.

**Status**: Worth exploring after #3 and #5 are implemented. Could help with cross-section questions but is complex to implement correctly.

---

## Measured Accuracy Results (2025-03 Tuning Session)

**Metric**: 40% section Jaccard + 60% field Jaccard (averaged over matching sections). Compared against Gemini 3.0 Flash reference.

### Current DQP Accuracy (after tuning)

| Workflow | Avg Accuracy | Median Accuracy | Combined (avg of both) | Status |
|----------|-------------|-----------------|----------------------|--------|
| Competitive Pricing | 83.4% | — | **83.4%** | Target met |
| Warranty Intelligence | 81.8% | — | **81.8%** | Target met |
| Pricing Simulator | 62.5% | — | **62.5%** | Below target |
| Composition | 67.7% | — | **67.7%** | Below target |

### Progression (key milestones)

| Change | CP | PS | WI | Comp |
|--------|-----|-----|-----|------|
| Original baseline | 67.3% | 52.1% | 69.3% | 61.8% |
| Include ALL suggested partitions | 82.6% | 60.0% | 75.7% | 65.9% |
| Content quality gate = 0.18 | **83.4%** | **62.5%** | **81.8%** | **67.7%** |

### Remaining Gap Analysis

| Improvement | Estimated Additional Gain | Effort | Priority |
|-------------|--------------------------|--------|----------|
| Embedding-based field selection (#3) | +8-12% (esp. PS, Comp) | 2 days | **#1** |
| Distilled record mode classifier (#5) | +3-7% | 2-3 days | **#2** |
| Better embedding model (#1) | +3-5% (mostly for search mode) | 1 day | #3 |
| NER entity extraction (#6) | +2-4% | 1-2 days | #4 |
| Query expansion (#4) | +1-3% (partially addressed) | 1 day | #5 |

**Applying #3 + #5 could bring Pricing Simulator to ~75-80% and Composition to ~78-83%.**

**Ceiling estimate**: Getting above 85% without any LLM is very hard because record plan decisions (filter values, aggregation fields, sort columns) require semantic reasoning about question intent relative to data schema — that's where LLMs have a fundamental advantage.

---

## Recommended Strategy: Hybrid Approach

The sweet spot is the **Hybrid** strategy (`SemanticQueryPlannerStrategy.HYBRID` — enum exists, not yet implemented):

1. Run deterministic first (0ms LLM cost, already 80%+ for 2/4 workflows)
2. Measure confidence using content quality scores and field coverage
3. Only call the LLM when confidence is low

**Updated assessment**: With DQP now at 80%+ for Competitive Pricing and Warranty Intelligence, the hybrid approach is even more compelling:
- **~60-70% of questions** can be handled deterministically with high confidence → zero LLM cost
- **~30-40% of questions** (complex reasoning, low field match scores) fall back to LLM → full accuracy
- **Net result**: LLM-level accuracy at ~30-40% of the LLM cost

**Confidence signal candidates**:
- `max_content_score` from field selection — low values indicate poor lexical match
- Number of sections with `mode="none"` — many "none" sections suggest question doesn't match the domain well
- Record plan mode confidence (already exists in `deterministic_record_planner.py`)
- Whether any sort/filter field was confidently matched

---

## HuggingFace Models Assessment

**Short answer**: HuggingFace models are valuable as **components within the deterministic pipeline** (embeddings, cross-encoders, classifiers), NOT as full LLM replacements.

### Where HuggingFace models ADD value (recommended):
- **Embedding models** (improvement #1, #3): `bge-large-en-v1.5` or `gte-large-en-v1.5` for semantic field matching — runs locally, ~5ms, no API cost
- **Cross-encoders** (#2): `cross-encoder/ms-marco-MiniLM-L-12-v2` for reranking — ~10-20ms, high accuracy
- **Small classifiers** (#5): sklearn/tiny NN for record mode — <1ms, distilled from LLM data
- **NER models** (#6): `dslim/bert-base-NER` or domain-fine-tuned for entity extraction

### Where HuggingFace models DON'T make sense:
- **Full LLM replacement** (e.g., Llama, Mistral for query planning): Worse quality than Gemini/Claude APIs, more expensive to operate than deterministic, adds GPU infrastructure burden
- **Open-source LLMs** sit in an awkward middle — not needed given the hybrid strategy

### When full HuggingFace LLMs would make sense:
1. **Data sovereignty** — if ontology data can't be sent to Google/Anthropic APIs
2. **Massive scale** — 100K+ queries/day where self-hosted becomes cheaper
3. **Fine-tuning** — if thousands of (question, plan) pairs are collected for a domain-specific model (but distillation into smaller components is usually more practical)
