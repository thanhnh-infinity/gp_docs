# Improving the Deterministic Semantic Query Planner to Match LLM Performance

## Current Architecture Overview

The deterministic pipeline has **3 levels**, each with specific strengths and weaknesses:

1. **Level 1 — Section Selection**: BM25 + `all-MiniLM-L6-v2` embeddings (384d)
2. **Level 2 — Field Selection**: Token overlap + synonym expansion + description matching + importance scoring
3. **Level 3 — Record Plan**: Regex-based intent detection → `all / top_k / filter / aggregate`

---

## Where the Deterministic Planner Falls Short vs LLMs

### Level 1: Section Selection (BM25 + all-MiniLM-L6-v2)

**Current**: `0.6 * BM25 + 0.4 * cosine(MiniLM-384d)` → top 10 sections

**Problem**: This is actually the **strongest** deterministic layer — BM25 + embeddings works well for section-level matching. But:
- `all-MiniLM-L6-v2` is a **2021-era 384d model** — it misses nuanced semantic relationships
- No reranking — the initial retrieval score is the final score
- Query "Where is cannibalization eating gross profit?" requires understanding that `cannibalization → SKU overlap → margin share analysis`, which BM25/MiniLM can't do

### Level 2: Field Selection (Token Overlap + Synonyms)

**Current**: `0.3*token + 0.2*synonym + 0.3*description + 0.2*importance` → weighted sum per field

**This is the weakest link.** The scoring is purely lexical:
- "What is the average elasticity?" → needs fields like `price_sensitivity`, `demand_curve_slope` — but token overlap with "elasticity" is 0 for both
- Synonym dict has 94 entries — covers common terms but misses domain-specific relationships
- No semantic understanding of field descriptions (just token overlap against them)

### Level 3: Record Plan (Regex Intent Detection)

**Current**: Pattern matching → `top_k/filter/aggregate/all`

**Second weakest.** The regex rules are brittle:
- "Which SKUs truly belong together across systems?" → no ranking/aggregate/filter intent detected → falls back to `top_k(20)` which is wrong (should be `all`)
- "Can you calculate scenario 2 & 3's financial impact as a percentage..." → complex multi-step intent that regex can't parse
- Entity extraction misses implicit references ("Home Depot" in "THD")

---

## Concrete Improvements That Could Close the Gap

Ranked by **impact / effort ratio**:

### High Impact, Low-Medium Effort

#### 1. Upgrade Embedding Model (Section + Field Selection)

Replace `all-MiniLM-L6-v2` (384d, 2021) with a modern model:

| Model | Dim | MTEB Score | Size | Latency |
|-------|-----|-----------|------|---------|
| `all-MiniLM-L6-v2` (current) | 384 | 56.3 | 80MB | ~1ms |
| `bge-large-en-v1.5` | 1024 | 64.2 | 1.3GB | ~5ms |
| `gte-large-en-v1.5` | 1024 | 65.4 | 1.3GB | ~5ms |
| `snowflake-arctic-embed-m-v2.0` | 768 | 63.2 | 300MB | ~3ms |

A +8 point MTEB improvement means dramatically better semantic matching for both section selection AND field selection. This is probably the **single biggest win**.

#### 2. Add Cross-Encoder Reranking (Section Selection)

After initial BM25+embedding retrieval of top 20 sections, rerank with a cross-encoder:

```python
# Cross-encoder scores (question, section_text) pairs jointly
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-12-v2")  # or BGE reranker

pairs = [(question, section_text) for section_text in candidate_sections]
rerank_scores = reranker.predict(pairs)
```

Cross-encoders are **much more accurate** than bi-encoders because they see question + document together. Cost: ~10-20ms for 20 candidates.

#### 3. Embedding-Based Field Selection (Replace/Augment Token Overlap)

Instead of token overlap for fields, compute cosine similarity between question embedding and each field's `(name + description)` embedding:

```python
# Pre-compute field embeddings at index time
field_emb = model.encode(f"{field_name}: {field_description}")

# At query time
question_emb = model.encode(question)
field_score = cosine(question_emb, field_emb)
```

This alone would fix cases like `"elasticity" → "price_sensitivity"` which token overlap completely misses.

### Medium Impact, Medium Effort

#### 4. Query Expansion with Ontology Context

Before searching, expand the query using the ontology's own vocabulary:

```python
# Build a term→field mapping from ontology metadata
ontology_terms = {}  # "elasticity" → ["price_sensitivity", "demand_curve_slope"]

# Expand query: "average elasticity" → "average elasticity price_sensitivity demand_curve_slope"
expanded_query = question + " " + " ".join(related_ontology_terms)
```

This is like a domain-aware synonym expansion but driven by the actual ontology data rather than a static dict.

#### 5. Small Classifier for Record Mode (Replace Regex Intent Detection)

Train a tiny classifier (logistic regression or small NN) on LLM outputs:

```python
# Training data: (question, correct_mode) pairs from Gemini 3.0 Flash results
# Features: question tokens, field types, row_count, section_type
# Labels: "all", "top_k", "filter", "aggregate"

from sklearn.linear_model import LogisticRegression
clf = LogisticRegression()
clf.fit(X_train, y_train)  # ~50-200 examples from your experiment
```

The training data already exists — the experiment generates (question, LLM-chosen-mode) pairs. This is **knowledge distillation** from LLM to a tiny model.

#### 6. NER-Based Entity Extraction (Replace Regex Entity Detection)

Use a HuggingFace NER model for entity extraction instead of regex:

```python
from transformers import pipeline
ner = pipeline("ner", model="dslim/bert-base-NER", grouped_entities=True)
entities = ner("Which installers are driving warranty cost per region?")
# → [{"entity_group": "MISC", "word": "warranty"}, ...]
```

Or better — a custom domain NER that recognizes ontology entities (store names, SKU patterns, region codes).

### High Impact, High Effort (But Worth Considering)

#### 7. Distill the LLM into the Deterministic Pipeline

This is the most powerful approach. Use experiment data to systematically learn from LLM decisions:

- **Section selection**: Collect (question, LLM-selected-sections) pairs → fine-tune a bi-encoder to rank sections the way Gemini does
- **Field selection**: Collect (question + section, LLM-selected-fields) pairs → train a field relevance model
- **Record plan**: Already covered in #5 above

With ~200 experiment runs (50 questions × 4 strategies), there is enough signal to bootstrap. Over time, as production data accumulates, the distilled model keeps improving.

#### 8. Graph-Aware Section/Field Linking

The ontology has `graph` and `rules` fields that the deterministic planner currently ignores. These contain relationships between sections — e.g., "gross_margin_share_analysis" relates to "competitive_pricing". Exploiting these could help with questions that span multiple related sections.

---

## Realistic Accuracy Expectations

| Improvement | Estimated Accuracy Gain | Effort |
|-------------|------------------------|--------|
| Better embeddings (#1) | +10-15% | 1 day |
| Cross-encoder reranking (#2) | +5-8% | 1 day |
| Embedding-based field selection (#3) | +10-15% | 2 days |
| Query expansion (#4) | +3-5% | 1 day |
| Distilled record mode classifier (#5) | +5-10% | 2-3 days |
| NER entity extraction (#6) | +3-5% | 1-2 days |
| Full LLM distillation (#7) | +10-20% | 1-2 weeks |

If deterministic is currently at ~45-60% accuracy vs Gemini 3.0 Flash (which the experiment will reveal), applying improvements #1-#5 could realistically bring it to **75-85%**.

**However**: getting above 85% without any LLM is very hard because the record plan level (choosing filter values, aggregation fields, sort columns) requires **semantic reasoning about the question's intent relative to the data schema**. That's where LLMs have a fundamental advantage.

---

## Recommended Strategy: Hybrid Approach

The sweet spot is the **Hybrid** strategy (the `HYBRID` strategy type that's not yet implemented):

1. Run deterministic first
2. Measure confidence (the confidence scoring already exists in `deterministic_record_planner.py`)
3. Only call the LLM when confidence is low

This gives LLM-level accuracy with deterministic-level cost for the easy 60-70% of questions.

---

## HuggingFace Models Assessment

**Short answer**: No HuggingFace solution can beat the current LLM + Deterministic combination when balancing all trade-offs.

- **Frontier APIs** (Gemini, GPT) win on quality — especially for Level-3 record plan reasoning
- **Deterministic** wins on cost/latency — 0ms, no LLM, no GPU
- **Open-source HuggingFace LLMs** sit in an awkward middle — worse quality than APIs, more expensive to operate than deterministic, and adds infrastructure burden

HuggingFace models would only make sense for:
1. **Data sovereignty** — if ontology data can't be sent to Google/OpenAI APIs
2. **Massive scale** — 100K+ queries/day where self-hosted becomes cheaper
3. **Fine-tuning** — if thousands of (question, plan) pairs are collected to fine-tune a domain-specific model
