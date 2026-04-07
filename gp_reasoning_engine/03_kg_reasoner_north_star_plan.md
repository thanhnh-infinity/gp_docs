# Knowledge Graph Reasoner — North Star Plan

## Date: 2026-03-19

---

## Executive Summary

We have a **production-grade Knowledge Graph** in Neo4j (~15K entities, ~80K typed relationships, confidence-scored, provenance-tracked, community-hierarchied). We have a **production Q&A engine** (RouterAgent L0-L4, SemanticQueryPlanner, StructuredDataRetriever, GroundedAnswerEngine) that reasons over tabular data. We have **symbolic reasoning** (ASP/Clingo, ClingCon, NeurASP, LPMLN).

**What's missing:** A unified **Knowledge Graph Reasoner** that takes a natural language question, retrieves evidence from the KG (not just flat tables), reasons over graph structure, and produces a grounded answer — at the right cost/latency for the question's complexity.

This plan is the north star for building that system.

---

## I. State-of-the-Art KG Reasoning Approaches

### Comprehensive Landscape

| # | Approach | Mechanism | Strength | Weakness |
|---|---------|-----------|----------|----------|
| 1 | **GraphRAG** (Microsoft) | Community summaries → global synthesis | Broad holistic answers | Expensive index build, lossy compression |
| 2 | **Think-on-Graph v1/v2/v3** | LLM beam-searches KG iteratively | Deep multi-hop, dynamic exploration | High LLM call count = latency/cost |
| 3 | **Graph-Constrained Reasoning (GCR)** | KG-Trie constrains LLM decoding | Zero hallucination at token level | Requires self-hosted model (not API) |
| 4 | **Text-to-Cypher / Text-to-SPARQL** | NL → executable graph query | Precise, deterministic, auditable | Brittle on unseen schema, 30-40% error rate |
| 5 | **KG-RAG (Subgraph Retrieval)** | Entity link → subgraph → LLM prompt | Token-efficient, 97% retrieval accuracy | Requires good entity linker |
| 6 | **SubgraphRAG** | Lightweight subgraph retrieval | Optimized retrieval/efficiency tradeoff | Limited reasoning depth |
| 7 | **Reasoning on Graphs (RoG)** | Faithful reasoning plans as relation paths | Interpretable, +22% over prior SOTA | Requires fine-tuned model |
| 8 | **Paths-over-Graph (PoG)** | Reasoning paths injected into LLM prompt | Explainable graph traversals | Path explosion on dense graphs |
| 9 | **ReAct KG Agents** | Iterative tool-use (reason + act + observe) | Dynamic, handles open-ended questions | Non-deterministic, hard to debug |
| 10 | **AriGraph** | Episodic + semantic memory KG | Cross-session reasoning | Complex memory management |

### What We Take From Each

| Source | What We Take | Where It Goes |
|--------|-------------|---------------|
| **GraphRAG** | Community summaries for global synthesis | Tier 4 (already have COMMUNITY nodes + LLM summaries from KG Builder) |
| **Think-on-Graph** | Iterative beam search over KG paths | Tier 3 ReAct agent with beam search |
| **Text-to-Cypher** | Schema-constrained query generation | Tier 1 (template-based, not free-form) |
| **KG-RAG** | Entity linking + subgraph retrieval + prompt grounding | Tier 2 (leveraging our structural layer) |
| **RoG** | Faithful reasoning plans as relation paths | Tier 2-3 (path planning before traversal) |
| **ReAct** | Iterative tool-use reasoning loop | Tier 3 orchestration pattern |
| **PoG** | Explicit graph paths in LLM context | Subgraph serialization format |

### What We Do That None of Them Do

- **Dual-source reasoning:** KG (graph) + tabular (structured data) in the same pipeline
- **ASP/symbolic verification:** Provably correct constraint checking on KG subgraphs
- **Confidence-weighted reasoning:** Every fact has extraction_type + confidence → source-aware trust
- **Structural scoping:** WORKFLOW→DOMAIN→SECTION→ENTITY navigational hierarchy for retrieval
- **Relationship axioms:** OWL-like query expansion (inverse, symmetric, transitive) at query time

---

## II. Critical Enterprise Requirements

Every production KG reasoner MUST have:

| # | Requirement | Why | Our Approach |
|---|-------------|-----|-------------|
| 1 | **Query Complexity Classification** | Not every question needs multi-hop reasoning | RouterAgent L0-L4 + KG tier router |
| 2 | **Entity Linking & Disambiguation** | Bridge between NL and graph nodes | 3-stage cascade: deterministic ID → Neo4j full-text (Lucene fuzzy) → Neo4j vector (HNSW cosine). Structural type filtering, type-word guard, sub-word dedup. |
| 3 | **Faithful Grounding & Provenance** | Every claim must cite a graph path | extraction_type, confidence, via_entity on every artifact |
| 4 | **Hallucination Prevention** | KGR must not invent non-existent paths | Template-based Cypher (not free-form) + confidence filtering |
| 5 | **Iterative Query Refinement** | First query may return empty/wrong results | Tier escalation chain: Tier 1 → 2 → 3 |
| 6 | **Multi-Hop Reasoning** | Real questions require 2-5 hop traversals | Configurable hop depth with beam search pruning |
| 7 | **Schema Awareness** | LLM needs to know entity/relationship types | Compiled schema descriptor from WorkflowKGSchemaConfig |
| 8 | **Semantic Caching** | Multi-hop reasoning is expensive | 5-level cache hierarchy (entity → Cypher → subgraph → community → answer) |
| 9 | **Observability & Tracing** | KG reasoning fails in non-obvious ways | ReasoningTrace per request, OpenTelemetry spans |
| 10 | **Audit Logging** | Regulatory compliance | Immutable log of queries, paths, LLM calls, answers |
| 11 | **Latency Management** | Users expect <5s for simple, <30s for complex | Tiered routing sends 80% of queries through fast path |
| 12 | **Modular Architecture** | Components evolve independently | Pluggable: entity linker, query planner, reasoner, answer generator |

---

## III. The Architecture: 5-Tier Reasoning System

### The Core Insight

Not every question needs the KG, and not every KG question needs deep reasoning. **80% of questions are Tier 0-1** (tabular or simple Cypher). **15% are Tier 2** (subgraph + prompt grounding). **5% are Tier 3-4** (iterative reasoning or global synthesis). Route accordingly.

### Tier Definitions

```
Question
  │
  ▼
┌───────────────────────────────┐
│ RouterAgent (existing L0-L4)  │
│ + KG Tier Classifier (new)    │
│   Relational patterns detect  │
│   entity references, graph    │
│   traversal intent, and       │
│   cross-entity reasoning need │
└───────────┬───────────────────┘
            │
  ┌─────────┼────────────┬──────────────┬──────────────┐
  ▼         ▼            ▼              ▼              ▼
Tier 0    Tier 1       Tier 2        Tier 3         Tier 4
Tabular   Text-to-     KG-RAG        Think-on-      GraphRAG
Retrieval Cypher       Subgraph      Graph          Global
(existing)(new)        (new)         ReAct Agent    Synthesis
                                     (new)          (new)
  │         │            │              │              │
  ▼         ▼            ▼              ▼              ▼
GroundedAnswerEngine (existing — reused across all tiers)
  │
  ▼
Answer with provenance, confidence, graph path citations
```

### Tier 0 — Tabular Retrieval (existing, unchanged)

**Trigger:** RouterAgent L0/L1, question resolves to single partition, no relational intent.
**Engine:** SemanticQueryPlanner → StructuredDataRetriever → GroundedAnswerEngine.
**Latency:** <3s. No KG involvement.
**Example:** "Show me the top 10 SKUs by margin in the pricing simulator"

### Tier 1 — Text-to-Cypher (KG-assisted lookup)

**Trigger:** Question references KG-recognizable entities + typed relationships (1-2 hops). RouterAgent L1/L2 + relational signal detected.
**Engine:** EntityLinker → KGQueryPlanner (template-based Cypher) → Neo4j execute → GroundedAnswerEngine.
**Latency:** <5s.
**Example:** "Which suppliers provide to Rheem's Water Heater Division?", "What markets does SKU 1002306458 compete in?"

**Key design: Template-based Cypher, NOT free-form generation.**
- 21 domain-agnostic Cypher templates across 11 QueryIntent types (entity_lookup, single_hop, multi_hop, aggregation, ranking, temporal, filtered_lookup, cross_domain, comparison, path_finding, existence)
- Entity IDs pre-resolved via 3-stage EntityLinker — templates use `{id: $entity_id}` not `{label: 'guessed name'}`
- Relationship types from schema via `CypherSchemaCompiler.relation_info` — not LLM-guessed
- Cypher injection prevention: character blocklist on injected slots, entity IDs as Neo4j `$params`
- GraphGlot client-side syntax validation before Neo4j execution
- Phase 0 (free-form LLM Cypher) as fallback when no template matches — with SQL syntax pre-validator
- Tabular override patterns catch design/what-if/summarization questions before entity linking
- Error rate: ~0% for template path vs 30-40% for free-form

### Tier 2 — Subgraph Retrieval + Prompt Grounding (KG-RAG)

**Trigger:** RouterAgent L2; multi-hop reasoning needed but traversal path is bounded.
**Engine:** EntityLinker → Subgraph extraction (2-3 hops) → Structural context injection → Serialize as triples → GroundedAnswerEngine.
**Latency:** <8s.
**Example:** "Compare warranty costs for SKUs manufactured by Rheem vs A.O. Smith in the Southeast"

**Key design: Structural layer as scope filter.**
- Use WORKFLOW→DOMAIN→SECTION→CONTAINS_ENTITY to scope subgraph to relevant domains
- Community-aware pruning when subgraph exceeds token budget
- Serialize as relation-triples with confidence + extraction_type (not raw JSON)
- Include DOMAIN/SECTION metadata for disambiguation

### Tier 3 — Iterative KG Reasoning (Think-on-Graph + ReAct)

**Trigger:** RouterAgent L3; dynamic exploration needed, path cannot be predetermined. Multi-constraint, causal analysis.
**Engine:** ReAct agent loop with tools:
  - `search_neighbors(entity_id, predicate_filter, max_hops)` — KG traversal
  - `execute_cypher(query)` — arbitrary Cypher
  - `retrieve_community_summary(community_id)` — global context
  - `check_constraints_asp(subgraph, constraints)` — symbolic verification
  - `lookup_entity(name)` — entity linking
**Latency:** <20s. Budget: max 8 reasoning steps.
**Example:** "Why are warranty claims higher in Texas than California for heat pump SKUs, and what supply chain factors contribute?"

**Key design: ASP as verification tool within the ReAct loop.**
- Agent can compile KG subgraph → ASP facts → check constraints → use result to decide next exploration step
- This is neuro-symbolic reasoning: LLM explores, ASP verifies

### Tier 4 — Global Synthesis (GraphRAG + Community Summaries)

**Trigger:** RouterAgent L4 or L3+ with no specific entity focus; holistic/strategic reasoning.
**Engine:** Retrieve community summaries (hierarchical, map-reduce) → ASP constraint checking → LLM synthesis.
**Latency:** <30s.
**Example:** "Design a pricing strategy that considers competitive positioning, supply chain risks, and warranty exposure across all regions"

**Key design: Community summaries pre-built at KG build time.**
- COMMUNITY nodes already exist from community_detector.py
- Summaries cached in Redis, not generated at query time
- Hierarchical: low-level communities → mid-level → top-level → global synthesis

### Tier Escalation

If Tier 1 (Cypher) returns empty or low-confidence results → escalate to Tier 2.
If Tier 2 (subgraph) is insufficient → escalate to Tier 3.
This is a **fail-up chain**, not retry. The cheapest sufficient tier always wins.

---

## IV. Core Components to Build

### New Modules (3-Layer Architecture)

```
app/
├── reasoner/
│   └── kg_reasoner/                              # REASONING PRIMITIVES
│       ├── __init__.py                           # (graph ops — no LLM/user knowledge)
│       ├── kg_traversal.py                       # k-hop ego-network, beam search, path scoring
│       ├── kg_axiom_expander.py                  # inverse/symmetric/transitive query expansion
│       ├── kg_confidence_propagator.py           # confidence-weighted path evaluation + filtering
│       ├── kg_subgraph_serializer.py             # Subgraph → triple text for LLM prompt
│       └── cypher_templates/                     # Parameterized Cypher patterns
│           └── template_registry.py              # ✅ 21 templates, 11 intents, CypherTemplateRegistry
│
├── agents/reasoning_orchestrator/
│   ├── engine/
│   │   └── kg_reasoner/                          # ORCHESTRATION
│   │       ├── __init__.py
│   │       ├── kg_cypher_engine.py               # ✅ Phase 0: schema-aware LLM Cypher + SQL validator
│   │       ├── kg_tier_router.py                 # ✅ 5-tier, 8-signal scoring, tabular override
│   │       ├── entity_linker.py                  # ✅ 3-stage: deterministic → fulltext → vector
│   │       ├── kg_query_planner.py               # ✅ Template selection + slot filling + Phase 0 fallback
│   │       ├── kg_subgraph_retriever.py          # Phase 2: subgraph extraction
│   │       ├── kg_reasoning_agent.py             # Phase 3: ReAct agent with KG tools
│   │       ├── kg_global_reasoner.py             # Phase 3: community summary synthesis
│   │       └── kg_answer_grounding.py            # Phase 2+: post-generation fact-checking
│   │
│   └── kg_qa_pipeline.py                         # KG-aware Q&A pipeline (extends BaseQAPipeline)
│
├── kr_engine/
│   ├── knowlegde_graph/
│   │   ├── cypher_schema_compiler.py             # ✅ Schema → LLM-ready descriptor
│   │   ├── neo4j_entity_index.py                 # ✅ Full-text (Lucene) + vector (HNSW) index
│   │   └── check_setup_neo4j.py                  # ✅ Standalone setup: --setup --populate-embeddings
│   ├── compiler/
│   │   └── kg_to_asp_compiler.py                 # KG subgraph → ASP facts
│   └── pipeline/knowledge_graph_builder/         # EXISTING (KG Builder — upstream)
│
└── reasoner/                                     # EXISTING (Clingo, ClingCon, NeurASP, LPMLN)
```

**Layer separation principle:**
- **Orchestration** (engine/kg_reasoner/) → calls → **Primitives** (reasoner/kg_reasoner/) → calls → **Data** (kr_engine/) → calls → Neo4j/Redis
- **Never upward:** Primitives never import from Orchestration. Data never imports from Primitives.
- Same pattern as existing: GroundedAnswerEngine (orchestration) → ClingoASPReasoner (primitive) → GPKnowledgeGraph (data)

### Component Details

#### A. Entity Linker (`entity_linker.py`)

Three-stage resolution:

1. **Deterministic** — compute `make_entity_id(candidate_type, mention)` for plausible types, check Neo4j. Resolves ~70-80%.
2. **Alias index** — Redis hash `entity_aliases:{namespace}` with normalized variants, populated at KG build time. O(1) lookup.
3. **Embedding fallback** — pre-computed bge-small embeddings for all entity labels, cosine similarity > 0.80. For the remaining 20-30%.

Type disambiguation: if "Rheem" matches both MANUFACTURER and partial SKU, use relational context from the question ("manufactured by Rheem" → MANUFACTURER).

#### B. KG Query Planner (`kg_query_planner.py`)

Schema-constrained Cypher generation:

1. Compile `WorkflowKGSchemaConfig` → compact schema descriptor (entity types, relationship types, properties)
2. Entity extraction from question → deterministic anchors
3. Template selection based on (source_type, relationship, target_type, aggregation)
4. LLM slot-filling (not arbitrary Cypher) — fill entity values, direction, filters
5. Axiom expansion — RelationshipAxiom (inverse_of, symmetric, transitive)
6. Confidence filtering — `WHERE r.confidence >= $threshold`
7. Execute → validate result shape → return or escalate

#### C. Subgraph Retriever (`kg_subgraph_retriever.py`)

1. Domain scoping via WORKFLOW→DOMAIN→SECTION (reuse SemanticQueryPlanner domain selection)
2. k-hop ego-network from seed entities (k=2 default, k=3 for L3)
3. Predicate filtering — follow only relevant relationship types
4. Community-aware pruning when exceeding token budget
5. Serialize as relation-triples with confidence + extraction_type + structural context

#### D. KG-to-ASP Compiler (`kg_to_asp_compiler.py`)

The neuro-symbolic bridge:

```
Entity: (id=gp_entity_abc, type=SKU, label="XYZ123", properties={retail_price: 1299})
  → entity(gp_entity_abc, "SKU", "XYZ123").
  → attribute(gp_entity_abc, retail_price, 1299).

Relation: (subject=gp_entity_abc, predicate=MANUFACTURED_BY, object=gp_entity_def)
  → rel(gp_entity_abc, manufactured_by, gp_entity_def).
  → rel_confidence(gp_entity_abc, manufactured_by, gp_entity_def, 95).
```

Three integration patterns:
- **ASP as constraint checker:** Compile subgraph → add integrity constraints from question → solve → report violations
- **ASP as inference engine:** Transitive closure, reachability, cardinality — provably correct where LLMs hallucinate
- **ASP within ReAct loop:** Agent compiles partial subgraph → checks hypothesis via ASP → decides next exploration step

#### E. ReAct Reasoning Agent (`kg_reasoning_agent.py`)

Tool-using agent for Tier 3:

```
Tools available:
  search_neighbors(entity_id, predicates, max_hops) → List[Triple]
  execute_cypher(query) → List[Row]
  retrieve_community_summary(community_id) → str
  check_constraints_asp(subgraph, constraints) → SatisfactionResult
  lookup_entity(name) → LinkedEntity
```

ReAct loop: Reason → select tool → execute → observe → reason again → ... → answer.
Budget: max 8 steps. Each step logged to ReasoningTrace.

---

## V. Confidence-Weighted Reasoning

### At Retrieval

Default confidence thresholds by extraction_type:
- `structured` → 0.0 (always trust deterministic)
- `structural` → 0.0 (framework scaffold)
- `inferred` → 0.5 (transitive may introduce noise)
- `nl` → 0.6 (LLM extraction less reliable)

Adjustable per tier. Tier 1 uses strict thresholds; Tier 3 starts strict and relaxes if insufficient evidence.

### At Reasoning

Subgraph serialization includes confidence + extraction_type per triple. LLM system prompt: "Treat high-confidence structured facts as ground truth. Low-confidence NL facts are hypotheses needing corroboration."

For ASP: encode confidence as optimization weight — prefer high-confidence reasoning paths.

### At Answer

Include confidence summary: "Based on X structured facts (high confidence) and Y NL-extracted facts (moderate confidence)." Flag answers where critical facts have confidence < 0.6.

---

## V-b. KG Evidence Integration Philosophy (Phase 0 → Phase 1+ Evolution)

### Phase 0: KG as Supplementary Retrieval

In Phase 0, the KG Reasoner is purely **retrieval** — it generates Cypher, executes against Neo4j, and converts results to `PipelineStructuredOutput` so GroundedAnswerEngine can consume them alongside tabular evidence. The LLM in GroundedAnswerEngine does the reasoning.

**Why this is acceptable for Phase 0:**
- Validates that KG retrieval adds value to answers
- Cypher query encodes the graph traversal (joins, filters already happened in Neo4j)
- Zero changes to GroundedAnswerEngine needed
- Failure data informs Phase 1 design

**What gets lost in Phase 0:**
- **Relationships** — edges flattened into columns
- **Confidence** — per-entity/per-edge scores disappear
- **Provenance** — extraction_type not preserved in flat rows
- **Graph structure** — multi-hop paths become flat joins

### Phase 1+: KG as Reasoning Component

In Phase 1+, the KG Reasoner produces **graph-aware evidence narratives** — not flat rows. Instead of dumping Cypher results as a table, it produces structured text that preserves graph semantics:

*"SKU XYZ is manufactured by Rheem (confidence=1.0, structured) and sold in JACKSONVILLE market (confidence=1.0) with optimal_retail_price=$659. The JACKSONVILLE market is in the SOUTHEAST region (3 hops from root) which has 12 stores with total claims of $4.5M."*

This is a separate evidence format from `PipelineStructuredOutput` — carrying provenance, confidence, and traversal paths.

### Data Volume Mitigation

KG evidence is **supplementary** to tabular evidence — it adds cross-entity context, not duplicate rows:
- `max_results` config (default 50) limits Cypher output
- If KG returns 500 rows → Cypher is too broad → Phase 1 templates fix this
- KG evidence should add 10-20 rows of cross-domain context, not thousands

### Conflict Resolution: KG vs Tabular

Three scenarios:
1. **Same data** — KG built from same PipelineStructuredOutput → redundant, not conflicting. LLM uses whichever is more relevant.
2. **Cross-domain relationships** — KG has what tabular doesn't (SKU→MARKET→REGION→DC across domains). Complementary.
3. **Different numbers** — If KG has different metric values (due to entity resolution or aggregation), **always trust tabular as ground truth** (raw data). KG adds structural context, not numeric authority.

### Architecture Evolution Path

```
Phase 0:  KG Retrieval → flat rows → PipelineStructuredOutput → GroundedAnswerEngine (LLM reasons)
Phase 1:  KG Retrieval → graph-aware narrative → separate evidence format → GroundedAnswerEngine
Phase 2:  KG Subgraph → serialized triples with confidence → LLM reasons over graph structure
Phase 3:  KG ReAct Agent → iterative graph reasoning with ASP verification → structured answer
```

Each phase adds reasoning capability to the KG component itself, reducing reliance on GroundedAnswerEngine for graph-specific reasoning.

---

## VI. Caching Strategy (5-Level)

| Level | What | Key | TTL | Hit Rate |
|-------|------|-----|-----|----------|
| L1 | Entity linking results | `el:{ns}:{question_hash}` | KG refresh | 60-70% |
| L2 | Cypher query results | `cypher:{ns}:{query_hash}` | KG refresh | 40-50% |
| L3 | Extracted subgraphs | `sg:{ns}:{seeds}:{hops}` | KG refresh | 30-40% |
| L4 | Community summaries | `cs:{ns}:{community_id}` | KG refresh | 90%+ (pre-built) |
| L5 | Full answers | `ans:{ns}:{question_hash}` | 1 hour | 20-30% |

Invalidation: KG rebuild publishes to `kg:invalidate:{namespace}` → clears L1-L4. L5 by TTL only.

---

## VII. Observability

### ReasoningTrace (per request)

```python
@dataclass
class ReasoningTrace:
    trace_id: str
    question: str
    tier_selected: int
    tier_escalations: List[int]
    entities_linked: List[Dict]
    cypher_queries: List[str]
    subgraph_size: Optional[Dict]  # {nodes, edges, tokens}
    asp_invocations: int
    llm_calls: List[Dict]  # {model, tokens, latency, purpose}
    cache_hits: Dict[str, bool]
    answer_confidence: float
    total_latency_ms: float
    stage_latencies: Dict[str, float]
```

Every reasoning step traceable: which entities linked, which Cypher executed, what subgraph retrieved, which LLM called, how answer constructed.

---

## VIII. Existing Components We Reuse

| Component | Current Location | Reuse For |
|-----------|-----------------|-----------|
| **RouterAgent** | `engine/router_agent.py` | L0-L4 complexity + KG tier classification |
| **SemanticQueryPlanner** | `engine/semantic_query_planner.py` | Domain scoping for subgraph retrieval |
| **StructuredDataRetriever** | `engine/structured_data_retriever.py` | Tier 0 tabular path (unchanged) |
| **GroundedAnswerEngine** | `engine/grounded_answer.py` | Answer generation across all tiers |
| **ASP Reasoners** | `reasoner/` (Clingo, ClingCon, NeurASP, LPMLN) | Constraint checking + symbolic verification |
| **GPKnowledgeGraph** | `kr_engine/knowlegde_graph/knowledge_graph.py` | Cypher execution against Neo4j |
| **WorkflowKGSchemaConfig** | `kg_models/schema_config.py` | Schema compilation for Cypher generation |
| **RelationshipAxiom** | `kg_models/schema_config.py` | Query expansion (inverse, symmetric, transitive) |
| **Entity IDs** | `make_entity_id()` | Deterministic entity linking |
| **Community nodes** | `community_detector.py` | Tier 4 global summaries |
| **RedisRunStore** | `run_store/redis_store.py` | Caching infrastructure |
| **LiteLLM** | `llm_engine/llm_utils.py` | Model routing across all tiers |

---

## IX. Implementation Phases

### Phase 0: Quick Win — Schema-Aware Cypher Generation — ✅ COMPLETED

**Goal:** Working KG reasoner prototype that learns from real failures to inform Phase 1 design.

**The insight:** Before building templates, entity linkers, and tier routers — just let the LLM generate Cypher directly. It will have a 30-40% error rate. **That's the point.** The failures tell us exactly which questions need templates, which entity references users actually type, and which Cypher patterns are most common.

| Step | File | What | Status |
|------|------|------|--------|
| 0.1 | `app/kr_engine/knowlegde_graph/cypher_schema_compiler.py` | Compile `WorkflowKGSchemaConfig` → LLM-ready schema descriptor | ✅ |
| 0.2 | `app/agents/.../question_answering_configurations.py` | `KGCypherEngineConfig` added to `QAToolConfig` | ✅ |
| 0.3 | `app/agents/.../engine/kg_reasoner/kg_cypher_engine.py` | Schema-aware Cypher generation + execution + self-correction | ✅ |
| 0.4 | `app/agents/.../qa_base_pipeline.py` + `qa_pipelines.py` | Wire Stage 2b + Rheem subclass override | ✅ |
| 0.5 | N/A | Not needed — Zeus calls via LangGraph tools | ✅ |
| 0.6 | `app/agents/.../engine/kg_reasoner/run_kg_reasoner_test.py` | Test harness: 30 questions, 6 categories, JSON export | ✅ |

**What this produces:**
- A working (if imperfect) KG reasoner that answers real questions
- Schema-aware Cypher generation constrained by domain schema (entity types, relationship types, properties)
- KG evidence as `PipelineStructuredOutput` — injected alongside tabular evidence for GroundedAnswerEngine
- Domain-specific KG question detection (entity terms + relation terms from schema, minimal generic keywords)
- Self-correction loop (up to `max_retries` on Cypher execution failure)
- Full Cypher logging for Phase 1 failure analysis
- Test harness with 30 questions across 6 difficulty categories
- **Failure data drives Phase 1 template design** — templates built to cover actual failure patterns

**What this does NOT produce:**
- No entity linker (LLM guesses entity names — some will be wrong)
- No templates (free-form Cypher — some will have syntax errors)
- No tiers (everything goes through LLM Cypher generation)
- No caching, no observability (just logs)

**Architecture (replaced by Phase 1+):**
```
Question → BaseQAPipeline.solve()
    ├── Stage 1: Semantic Query Planning (unchanged)
    ├── Stage 2: Tabular Retrieval (unchanged)
    ├── Stage 2b: _retrieve_kg_evidence() — NEW
    │     ├── CypherSchemaCompiler(WorkflowKGSchemaConfig) → schema descriptor
    │     ├── KGCypherEngine.is_kg_relevant(question) — 3-source detection
    │     ├── KGCypherEngine.query(question) → Cypher → Neo4j → rows
    │     └── _kg_evidence_to_structured_output() → PipelineStructuredOutput
    └── GroundedAnswerEngine.answer(tabular + KG evidence)
```

---

### Phase 1: Foundation (3-4 weeks) — ✅ COMPLETED

**Goal:** Tier 0 (unchanged) + Tier 1 (Text-to-Cypher) working end-to-end. **Informed by Phase 0 failure analysis.**

#### Phase 0 Failure Analysis → Phase 1 Design Decisions

Phase 0 testing (16 real Rheem questions) revealed three failure patterns that directly drive Phase 1 design:

**Failure #1: Entity name guessing (caused 60%+ of 0-row results)**
- LLM generates `{label: 'Water Heater Division'}` but Neo4j has `"WATER HEATER"`
- LLM generates `{label: '1002306458'}` but matching requires entity ID, not label string
- User says "D3" but LLM doesn't know this maps to "D3-Non-Ferrous and Electromechanical"
- **Fix:** Entity Alias Index (Step 1.1) + Entity Linker (Step 1.2) — pre-resolve mentions to entity IDs before Cypher generation

**Failure #2: Wrong relationship types (caused 30%+ of failures)**
- LLM generates `-[:SUPPLIES_CATEGORY]->` but the real relationship is `-[:HAS_SPEND_IN_CATEGORY]->`
- LLM invents relationships that don't exist in the schema
- **Fix:** Template Cypher (Step 1.3) — constrained Cypher patterns with known-valid relationship types, LLM only fills slots

**Failure #3: Latency (3-36 seconds per query)**
- Every question triggers a full LLM call to generate Cypher from scratch
- **Fix:** Template matching is sub-second when entities are pre-linked; LLM only called for slot-filling, not full Cypher generation

#### Implementation Steps

| Step | File | What | Status |
|------|------|------|--------|
| 1.1 | `neo4j_entity_index.py` | Neo4j full-text + vector index — replaces Redis. Auto-indexed, no populate step | ✅ |
| 1.2 | `entity_linker.py` | 3-stage cascade: deterministic → alias → embedding. Links NL mentions to KG entity IDs | ✅ |
| 1.3 | `kg_query_planner.py` + `cypher_templates/` | Template-based Cypher with LLM slot-filling. Phase 0 free-form as fallback | ✅ |
| 1.4 | `kg_tier_router.py` | KG tier classification (Tier 0=tabular, Tier 1=Cypher). 5-tier, 8-signal scoring | ✅ |
| 1.5 | Pipeline integration | Wire Phase 1 into qa_base_pipeline + API + populate_alias_index | ✅ |

**Deliverable:** User asks "Which suppliers provide to Water Heater Division?" → "Water Heater Division" found via Neo4j full-text index → template Cypher with `{id: $entity_id}` → executed → grounded answer returned.

#### Entity Linking Architecture (Final — Neo4j Native)

Redis dependency eliminated. All entity linking uses Neo4j native indexes:

```
Stage 1: Deterministic — make_entity_id() → Neo4j RANGE index exists check
Stage 2: Neo4j Full-Text — CALL db.index.fulltext.queryNodes("entity_label_fulltext", "Rheem~")
          Lucene fuzzy matching (~), auto-indexed on entity insert, zero populate step
Stage 3: Neo4j Vector — CALL db.index.vector.queryNodes("entity_label_vector", 5, $embedding)
          HNSW cosine similarity, requires label_embedding property on entities
          Falls back to local embedding model if vector index not populated
```

**Why Neo4j instead of Redis:**
- Zero extra dependency — Neo4j is the shared contract between KG Builder and KG Reasoner
- No separate populate step — full-text index auto-indexes on insert
- No eviction risk — persistent, not cache
- Shared across Cloud Run services — Zeus and RE read the same Neo4j
- Always consistent — indexes reflect exactly what's in the graph

**Setup for existing KG (no rebuild needed):**
```bash
python -m app.kr_engine.knowlegde_graph.check_setup_neo4j --setup --populate-embeddings
```

#### Phase 1 Test Results (28 Rheem questions)

| Mode | Template matches | Phase 0 fallbacks | Avg latency |
|------|-----------------|-------------------|-------------|
| Phase 0 only | 0/24 | 24/24 | 4799ms |
| Phase 1 (with Neo4j indexes) | 10+/24 | ~14/24 | **~2900ms** |

**Key findings:**
- Template matches execute at **150-400ms** (10-50x faster than Phase 0's 3-8s)
- Full-text index enables fuzzy entity matching without Redis or embedding model
- Phase 0 fallback handles everything templates can't — zero regressions
- Graceful degradation: no indexes → deterministic + local embedding fallback, no crashes

**External libraries integrated:**
- **GraphGlot** — Cypher syntax validation before Neo4j execution (<1ms client-side)
- **Static embeddings** (`sentence-transformers/static-retrieval-mrl-en-v1`) — 100-400x faster for entity label encoding
- **RapidFuzz** — superseded by Neo4j Lucene fuzzy matching for entity linking

**Remaining template coverage gaps (Phase 2+ territory):**
- Aggregation/ranking with only 1 entity type (needs simpler templates)
- Cross-domain questions (needs Tier 2 subgraph retrieval)
- Questions where entity is linked but no schema relationship found between source/target types

### Phase 2: Subgraph + Symbolic (2-3 weeks)

**Goal:** Tier 2 (KG-RAG subgraph) + KG-to-ASP bridge.

| Task | File | Effort |
|------|------|--------|
| Subgraph retriever | `kg_subgraph_retriever.py` | 4 days |
| Subgraph serialization (triples + structural context) | same | 2 days |
| KG-to-ASP compiler | `kg_to_asp_compiler.py` | 3 days |
| Subgraph cache layer (Redis) | cache integration | 2 days |
| Tier escalation logic (1 → 2) | `kg_tier_router.py` | 1 day |
| Entity linker embedding fallback | `entity_linker.py` | 2 days |

**Deliverable:** User asks "Compare warranty exposure for heat pump vs gas products across Southeast markets" → subgraph extracted across warranty + pricing domains → serialized → LLM synthesizes cross-domain answer.

### Phase 3: Advanced Reasoning (3-4 weeks)

**Goal:** Tier 3 (ReAct agent) + Tier 4 (global synthesis).

| Task | File | Effort |
|------|------|--------|
| ReAct reasoning agent with KG tools | `kg_reasoning_agent.py` | 5 days |
| ASP tool integration in ReAct loop | `kg_reasoning_agent.py` | 3 days |
| Global synthesis from community summaries | `kg_global_reasoner.py` | 3 days |
| Full observability (ReasoningTrace) | tracing integration | 3 days |
| Tier escalation 2 → 3, 3 → 4 | `kg_tier_router.py` | 2 days |
| Cost budget enforcement per tier | same | 1 day |

**Deliverable:** User asks "Why are warranty claims increasing in Texas, and what pricing and supply chain factors contribute?" → ReAct agent traverses warranty→pricing→supply chain domains iteratively, invokes ASP to check constraint hypotheses, produces a multi-source grounded answer.

### Phase 4: Production Hardening (2 weeks)

**Goal:** Enterprise-ready.

| Task | Effort |
|------|--------|
| Full caching (5-level) | 3 days |
| Evaluation pipeline (golden set of test questions per tier) | 3 days |
| Cost tracking and budget alerts | 1 day |
| Client generalization (Bayer/Suntory configs) | 3 days |
| Documentation and architecture doc update | 1 day |

---

## X. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Template-based Cypher over free-form** | Free-form has 30-40% error rate. Templates with LLM slot-filling: <5%. |
| **Subgraph as triples, not JSON** | Natural KG primitive, compresses well, LLM reasons directly |
| **ASP as verification, not primary reasoning** | ASP for constraints/optimization; LLM for open-ended synthesis |
| **Confidence as first-class citizen** | Our metadata is unusually rich — propagate it, don't ignore it |
| **Tier escalation over parallelism** | Cheapest sufficient tier wins — saves cost, simplifies answer arbitration |
| **No GCR/KG-Trie** | Requires self-hosted model; we use API-based models (Vertex AI) |
| **Community summaries at build time** | Already supported in community_detector.py; query-time generation too slow |
| **Provider-agnostic from day 1** | Abstract Neo4j behind interface — ready for TigerGraph, Neptune, etc. |

---

## XI. Success Metrics

| Metric | Target |
|--------|--------|
| Tier 0 latency (tabular) | <3s p95 |
| Tier 1 latency (Cypher) | <5s p95 |
| Tier 2 latency (subgraph) | <8s p95 |
| Tier 3 latency (iterative) | <20s p95 |
| Tier 4 latency (global) | <30s p95 |
| Cypher generation correctness | >95% |
| Entity linking accuracy | >90% |
| Answer faithfulness (grounded in graph) | >95% |
| Tier escalation rate (1→2) | <20% |
| Cache hit rate (L1-L4 combined) | >50% |
| Hallucination rate | <2% |

---

## XII. The Unique Advantage — Our Moat

```
Other KG Reasoners:                Our KG Reasoner:

LLM → generates Cypher →           RouterAgent → selects cheapest tier
      maybe correct                 EntityLinker → deterministic + embedding
      no confidence                 KG Query/Subgraph → confidence-weighted
      no provenance                 ASP Solver → provably correct verification
      no symbolic verification      Community Summaries → global synthesis
                                    GroundedAnswerEngine → cited, confidence-scored
                                    ReasoningTrace → fully auditable

= Best-effort                      = Correct + Efficient + Explainable + Auditable
```

**The differentiator:** We don't just retrieve from the KG — we **reason** over it with symbolic verification, confidence weighting, and tiered cost optimization. No other system combines graph traversal + ASP constraint checking + confidence propagation + community summaries in a single framework.

---

---

## XIII. Leveraging Our Exclusive System Components

Our reasoning-engine has capabilities that NO other KG reasoner system has access to. The KG Reasoner MUST deeply leverage all of them:

### A. Four ASP Solvers (Symbolic Reasoning Arsenal)

| Solver | Strength | KG Reasoner Use Case |
|--------|----------|---------------------|
| **ClingoASPReasoner** | Standard ASP — answer sets, integrity constraints, optimization with weak constraints | Tier 3: constraint checking on KG subgraphs, reachability, cardinality validation |
| **ClingconASPReasoner** | Constraint solving with numeric domains | Tier 3: "find suppliers where total spend < $1M AND lead time < 30 days" — numeric constraints on KG properties |
| **NeuralASPReasoner** (NeurASP) | Neural-symbolic hybrid — learns from data + reasons formally | Tier 3: when pure symbolic rules are insufficient, NeurASP learns patterns from KG data + applies formal reasoning |
| **ProbabilisticASPReasoner** (LPMLN) | Probabilistic logic programming — handles uncertain facts | Tier 2-3: when KG facts have mixed confidence (structured=1.0, NL=0.7), LPMLN reasons over probabilistic answer sets weighted by confidence scores |

**KG-to-ASP compilation** uses these directly:
- Entity triples → ASP facts: `entity(gp_entity_abc, "SKU", "XYZ123").`
- Relationship triples → ASP relations: `rel(gp_entity_abc, manufactured_by, gp_entity_def).`
- Confidence → weak constraints: `#minimize { (100-C)@1 : rel_confidence(S,P,O,C) }.`
- `merged_from` metadata → provenance tracking in answer sets
- `via_entity`, `via_predicate` → inference chain reconstruction in ASP

The `ASPProgramExecutionAgent` (existing, production) handles modify-and-rerun patterns. The KG Reasoner extends this with KG subgraph compilation as input.

### B. OWL/RDF Ontology Engine

| Component | KG Reasoner Use Case |
|-----------|---------------------|
| **owlready2 + PELLET** reasoner | OWL consistency checking on reasoning results — validate that inferred facts don't violate ontology constraints |
| **SPARQL** | Alternative query language for RDF-backed KGs (provider-agnostic — same `KGQueryPlanner` interface, Cypher for Neo4j, SPARQL for RDF stores) |
| **Description logic** inference | Entity type hierarchy resolution — if question asks for "products" and ontology says SKU is-a Product, expand the query |
| **OntologyManager** | Runtime ontology access for entity type relationships, property constraints, domain/range declarations |
| **Client Ontology** (from OSI Engine) | Client-specific vocabulary mappings — "ss_sales_price IS-A price" bridges client terms to GP base. KG Reasoner uses these to understand client-specific questions |

### C. KG Builder Metadata (Provenance Chain)

Every entity and relation from the KG Builder carries rich metadata that the Reasoner MUST leverage:

| Metadata | How Reasoner Uses It |
|----------|---------------------|
| `extraction_type` (STRUCTURED/NL/INFERRED/STRUCTURAL) | Source-aware trust: structured facts are ground truth, NL facts need corroboration, inferred facts need path validation |
| `confidence` (0.0-1.0) | Confidence-weighted reasoning: prefer high-confidence paths, flag low-confidence critical facts |
| `merged_from: [entity_ids]` | Audit trail: when answer references a merged entity, show which originals were combined |
| `via_entity`, `via_predicate` | Inference explainability: when using an inferred relationship, cite the intermediate hop |
| `DomainVocabulary` + `DomainVocabularyMode` | Schema context for entity linking — reasoner knows the domain's entity types and known names |
| `use_typed_relationships` | Query optimization — native Neo4j types (`:MANUFACTURED_BY`) for O(1) traversal |
| `COMMUNITY` + `HAS_SUB_COMMUNITY` + `BELONGS_TO_COMMUNITY` | Tier 4 global synthesis: traverse community hierarchy for multi-level summarization |
| `RelationshipAxiom` (inverse_of, symmetric, transitive) | Query expansion at runtime — automatically follow inverse/symmetric/transitive edges |

### D. Existing Q&A Infrastructure (Reuse, Don't Rebuild)

| Component | Role in KG Reasoner |
|-----------|-------------------|
| **RouterAgent** (L0-L4 + TaskType) | Extended with relational signal → KG tier classification. The SAME router handles both tabular and KG questions. |
| **SemanticQueryPlanner** (LLM + deterministic) | Domain scoping for Tier 2 subgraph retrieval — selects which DOMAINs are relevant |
| **StructuredDataRetriever** | Tier 0 tabular path (unchanged). KG tiers 1-4 ADD to this, don't replace it. |
| **GroundedAnswerEngine** | Answer generation across ALL tiers. Receives evidence from tabular, Cypher, subgraph, or community summaries — same grounding logic. |
| **`deterministic_reasoner.py`** | Rule-based reasoning with BM25 + embedding scoring. Reused for deterministic entity linking (alias matching). |
| **RedisRunStore** | All 5 cache levels + session state + reasoning trace persistence |
| **LiteLLM** (100+ providers, Vertex AI) | Model routing across all tiers — same provider-agnostic LLM access |

---

## XIV. Gap Analysis: SOTA Coverage Verification

Systematic check — every approach and requirement from `03_theory_kg_reasoner_state_of_the_art.md` mapped to our plan:

### SOTA Approaches Coverage

| # | Approach | Our Tier | How We Cover It | How We Do It BETTER |
|---|---------|---------|-----------------|---------------------|
| 1 | **GraphRAG** (community summarization, map-reduce) | Tier 4 | Community summaries from KG Builder (COMMUNITY + BELONGS_TO_COMMUNITY + HAS_SUB_COMMUNITY hierarchy) | Pre-built at KG build time (not query time), hierarchical bottom-up aggregation, ASP constraint verification on synthesis results |
| 2 | **Think-on-Graph v1 (ToG-1)** — LLM beam search on KG | Tier 3 | ReAct agent with beam search, configurable hop depth, relevance pruning | + ASP symbolic verification at each hop (ClingoASPReasoner as tool-using agent tool) |
| 3 | **Think-on-Graph v2 (ToG-2)** — tightly coupled KG×Text hybrid | Tier 3 | ReAct agent traverses KG + retrieves NL text from SECTION `summary_snippet` | SECTION nodes carry summary text alongside graph structure — KG navigation → text deep-dive built into structural layer. No "loose coupling" failure. |
| 4 | **Think-on-Graph v3 (ToG-3)** — heterogeneous graph architecture | Tier 2-4 | Chunk-level (SECTION text) + triplet-level (typed relationships) + community-level (COMMUNITY summaries) — all integrated in the same Neo4j graph | Already built into KG structure — not a separate index. Heterogeneous graph is our native storage model. |
| 5 | **Graph-Constrained Reasoning (GCR)** — KG-Trie decoding constraint | Deferred | Requires self-hosted model (not compatible with API-based Vertex AI/LiteLLM) | Template-based Cypher achieves similar hallucination prevention for structured queries without custom inference infrastructure |
| 6 | **Text-to-Cypher** (semantic parsing: NL → structured query) | Tier 1 | Template-based + LLM slot-filling + iterative correction feedback loop (Multi-Agent GraphRAG pattern) | <5% error rate vs 30-40% free-form; axiom expansion (inverse_of, symmetric, transitive via RelationshipAxiom); CypherBench as evaluation benchmark |
| 7 | **Text-to-SPARQL** | Tier 1 | Same `KGQueryPlanner` interface, different query language | Provider-agnostic: Cypher for Neo4j, SPARQL for RDF stores (via owlready2/PELLET) |
| 8 | **KG-RAG** (subgraph-based prompt grounding, 97% retrieval, 65% fewer tokens) | Tier 2 | Entity linking + structural scoping + subgraph retrieval | WORKFLOW→DOMAIN→SECTION scoping = better precision than generic entity linking. Token-efficient: selective schema injection reduces prompt by ~60%. |
| 9 | **SubgraphRAG** (lightweight subgraph) | Tier 2 | Lightweight subgraph with token budget cap | Community-aware pruning when exceeding budget (entities in same community prioritized) |
| 10 | **Reasoning on Graphs (RoG)** — faithful reasoning plans as relation paths | Tier 2-3 | Path planning before traversal: generate relation-path plan, then execute | + confidence scoring per hop + extraction_type awareness + merged_from provenance |
| 11 | **Paths-over-Graph (PoG)** — reasoning paths in LLM prompt | Tier 2-3 | Explicit graph paths serialized as triples in LLM context | Paths carry full provenance (extraction_type, confidence, via_entity, via_predicate) — not just path structure |
| 12 | **ReAct KG Agents** — iterative tool-using agents (Reason + Act loop) | Tier 3 | ReAct loop with 5 KG tools (search_neighbors, execute_cypher, retrieve_community, check_constraints_asp, lookup_entity) | + ASP tool (ClingoASPReasoner + ClingconASPReasoner) for symbolic verification within the loop. No LangGraph dependency — our own orchestration. |
| 13 | **AriGraph** (episodic + semantic memory, world model) | Cross-session | Redis-backed session state + reasoning trace history | Existing `RedisRunStore` stores ReasoningTrace per session; cross-session reasoning via trace retrieval. Semantic KG IS the world model. |

### Enterprise Requirements Coverage (12/12 from SOTA)

| # | Requirement | Our Coverage | Implementation |
|---|-------------|:---:|---------------|
| 1 | **Security & RBAC** | ✅ | Multi-tenancy via namespace isolation (every query scoped by `{namespace: $ns}`). Tenant-level and role-based access control (RBAC) enforced at API layer before KG query execution. Property-level security via Cypher template constraints (templates never expose unauthorized entity types). |
| 2 | **Hallucination guardrails (faithful grounding)** | ✅ | **Faithfulness over fluency** is our core principle. Template Cypher prevents hallucinated paths structurally. Confidence filtering excludes low-confidence facts. **Post-generation fact-checking:** validate all answer entities exist in retrieved subgraph/Cypher results — cite specific graph paths as provenance for every claim. extraction_type transparency. NeMo Guardrails / Guardrails.ai integration ready at API layer. |
| 3 | **Entity linking & disambiguation** | ✅ | 3-stage: deterministic ID → alias index → dense retrieval (embedding fallback with bge-small). Handles synonyms, abbreviations, typos via alias index. Type disambiguation via relational context ("manufactured by Rheem" → MANUFACTURER). |
| 4 | **Iterative query refinement & self-correction** | ✅ | **Within Tier 1:** Cypher self-correction with content-aware feedback loop — if Cypher returns empty or errors, LLM revises slot-filling (up to 2 retries) before escalating. **Across tiers:** escalation chain 1→2→3. Same pattern as Multi-Agent GraphRAG's iterative correction. |
| 5 | **Observability & tracing** | ✅ | **Observability from day one.** ReasoningTrace per request. OpenTelemetry spans per tier/step. Stage latencies, cache hits, LLM calls, Cypher queries, subgraph sizes — all logged. Failure mode classification (retrieval failure vs context-window overflow vs hallucination) for targeted debugging. |
| 6 | **Latency management & semantic caching** | ✅ | 5-level cache hierarchy. **Semantic query caching** (like GPTCache): uses embedding similarity on question text (not just exact hash) to find cached answers for paraphrased questions. **Cypher result caching** for deterministic queries. **Speculative subgraph prefetching** for common entity neighborhoods. **Fast-path/slow-path routing** via tier classifier — 80% of queries through fast Tier 0-1. |
| 7 | **Query complexity classification** | ✅ | RouterAgent L0-L4 + relational signal patterns → KG tier (0-4). **Tiered routing by complexity** — simple entity lookups → direct Cypher, single-hop → KG-RAG, multi-hop → Think-on-Graph/ReAct, global → GraphRAG community summarization. |
| 8 | **Multi-hop reasoning engine** | ✅ | Configurable hop depth (k=2 default, k=3 for L3). **Beam search** over candidate paths with **relevance pruning**. Max hops parameter with fallback behavior when paths exceed limit. |
| 9 | **Schema awareness & dynamic schema introspection** | ✅ | Schema compiler from `WorkflowKGSchemaConfig`. **Selective schema injection / dynamic schema introspection:** based on entity types detected in the question, only inject relevant schema elements (not full 20-predicate schema). Reduces token usage by ~60%. **Neo4j vector index** ready for semantic entity search. |
| 10 | **Evaluation pipeline (KGQA benchmarking)** | ✅ | Golden sets per tier: 50+ questions per domain (Rheem) with expected answers. Metrics: **faithfulness** (all claims grounded in graph paths?), **completeness** (all correct answers returned?), **Cypher correctness** (execute vs expected), **latency P50/P95/P99** per tier, **hallucination rate** (citations to non-existent nodes). **CypherBench** as external benchmark. Continuous evaluation in CI. Domain-specific golden sets per client. |
| 11 | **Modular, pluggable architecture** | ✅ | **Swappable components:** entity linker, query planner (NL → structured query — semantic parsing), subgraph retriever, reasoning engine (agent-based or symbolic), answer generator — all independently replaceable. Provider-agnostic graph interface (Neo4j today, TigerGraph/Neptune/Amazon Neptune ready). Orchestration via our own agent framework (not LangGraph dependency — concepts stolen, not adopted). |
| 12 | **Audit logging & compliance** | ✅ | Every ReasoningTrace immutably persisted to Redis + optional cloud storage. Includes: **user identity**, **timestamps**, all Cypher queries, all graph paths traversed, all LLM calls with input/output, final answer. Regulatory audit export capability for finance/healthcare/legal compliance. |

### The 3 Non-Negotiables (from SOTA) — All Addressed

1. **Faithfulness over fluency** — Template Cypher prevents hallucinated paths. Post-generation fact-checking validates every claim against retrieved context. Confidence thresholds exclude unreliable facts. An honest "insufficient evidence" is better than an eloquent hallucination.

2. **Tiered routing by complexity** — 80% of queries through Tier 0-1 (cheap, <5s). Only 5% through Tier 3-4 (expensive, <30s). RouterAgent + KG tier classifier route intelligently. Sending every query through a full agentic loop burns cost and kills latency.

3. **Observability from day one** — ReasoningTrace captures every step. OpenTelemetry spans. Failure mode classification. You cannot fix wrong entity linking or correct Cypher on wrong subgraph without tracing. Built into Phase 1, not retrofitted.

### Integration with Existing System

| Component | How KG Reasoner Connects |
|-----------|-------------------------|
| **KG Builder** (upstream) | KG Builder builds and writes the graph. KG Reasoner reads it. On KG rebuild, reasoner's caches are invalidated via `kg:invalidate:{namespace}` pub/sub. Entity alias index populated during KG write. |
| **RouterAgent** | Extended with relational signal patterns to detect KG-relevant questions. Returns both `ReasoningLevel` (L0-L4) and `KGTier` (0-4). |
| **SemanticQueryPlanner** | Reused for domain scoping in Tier 2 — determines which DOMAINs are relevant to the question. |
| **StructuredDataRetriever** | Tier 0 (tabular) path unchanged. KG Reasoner adds Tier 1-4 as alternatives, not replacements. |
| **GroundedAnswerEngine** | Receives evidence from ALL tiers — tabular rows, Cypher results, serialized subgraphs, community summaries. Generates grounded answer regardless of source. |
| **ASP/Clingo Reasoners** | KG-to-ASP compiler bridges the KG and symbolic reasoning. Three patterns: constraint checking, inference engine, ReAct tool. |
| **OWL/RDF Ontology Engine** | Used for: (1) ontology-driven entity linking (entity type hierarchies from OWL), (2) SPARQL as alternative query language for RDF-backed KGs, (3) OWL consistency checking on reasoning results. |
| **MCP Server** | Expose KG Reasoner as MCP tools: `query_kg(question)`, `traverse_graph(entity, hops)`, `check_constraint(subgraph, constraint)`. Any MCP client (Claude, IDE, external agent) can invoke our reasoning. |
| **RedisRunStore** | Caching infrastructure for all 5 cache levels. Session state for cross-session reasoning (AriGraph pattern). |

---

## XIV. Dominant Position: Why We Win Against Every SOTA Approach

| SOTA | Their Best Feature | We Match It | We EXCEED It With |
|------|-------------------|:---:|-------------------|
| **GraphRAG** | Community summarization | ✅ Tier 4 | Pre-built summaries (faster) + hierarchical aggregation + ASP verification |
| **Think-on-Graph v2** | KG×Text hybrid | ✅ Tier 3 | SECTION summary_snippet in structural layer = text alongside graph natively |
| **GCR** | Zero hallucination | ✅ Template Cypher | Template slot-filling (<5% error) + post-generation fact-checking + confidence thresholds |
| **Multi-Agent GraphRAG** | Iterative Cypher correction | ✅ Tier 1 self-correction | Self-correction WITHIN tier + escalation ACROSS tiers = two levels of recovery |
| **KG-RAG** | 97% retrieval accuracy | ✅ Tier 2 | Structural scoping (DOMAIN→SECTION) gives precision unavailable to generic entity linking |
| **RoG** | Faithful reasoning plans | ✅ Path planning | + confidence per hop + extraction_type awareness = reasoning quality metadata |
| **ReAct agents** | Dynamic exploration | ✅ Tier 3 | + ASP symbolic verification tool = provably correct constraint checking within the loop |
| **AriGraph** | Cross-session memory | ✅ Redis traces | Existing infrastructure, no custom memory system needed |

**What NO other system has — Our 10 Exclusive Advantages:**

1. **Dual-source reasoning** — KG graph + tabular structured data in the SAME pipeline. Tier 0 (tabular) and Tier 1-4 (KG) share the same RouterAgent, GroundedAnswerEngine, and answer format. User doesn't need to know which source answered their question.

2. **Four symbolic reasoners as verification tools** — ClingoASPReasoner (standard ASP with answer sets + integrity constraints + weak constraints + optimization), ClingconASPReasoner (numeric constraint solving), NeuralASPReasoner (NeurASP neural-symbolic hybrid), ProbabilisticASPReasoner (LPMLN probabilistic logic). The ReAct agent can invoke ANY of these within the reasoning loop to **prove** constraint satisfaction, not just estimate it with an LLM.

3. **Confidence-weighted provenance chain** — Every entity and relation carries `extraction_type` + `confidence` + `merged_from` + `via_entity` + `via_predicate`. The reasoner propagates these through retrieval → reasoning → answer. The user knows "this answer is based on 8 structured facts (confidence=1.0) and 2 NL-extracted facts (confidence=0.7). Entity X was merged from 3 variants. Relationship Y was inferred via intermediate entity Z."

4. **5-tier cost-optimized routing** — 80% of questions go through Tier 0-1 (cheap, <5s). Only 5% need Tier 3-4 (expensive, <30s). RouterAgent L0-L4 + KG relational patterns → precise tier selection. No other system has this level of cost optimization.

5. **Structural navigational layer** — WORKFLOW→DOMAIN→SECTION→CONTAINS_ENTITY hierarchy enables deterministic, complete traversal from root to any fact. This is our query scoping superpower — Tier 2 subgraph retrieval uses DOMAIN→SECTION to scope, achieving higher precision than generic entity linking.

6. **OWL/RDF ontology engine integration** — RelationshipAxiom (inverse_of, symmetric, transitive) expands queries at runtime. owlready2 + PELLET for description logic inference. OntologyManager for entity type hierarchies. Client Ontology from OSI Engine for client-specific vocabulary bridges. SPARQL as alternative query language. OWL consistency checking validates reasoning results. No other production KG reasoner integrates formal ontology reasoning.

7. **Domain-aware KG** — The KG itself was built with DomainVocabulary + DomainVocabularyMode (HINT mode) that taught the SemanticaEngine domain types during extraction. NL-extracted entities already carry correct domain types (MANUFACTURER, not ORG) — reducing entity linking errors at reasoning time.

8. **Native typed Neo4j relationships** — `use_typed_relationships=True` creates `:MANUFACTURED_BY`, `:SOLD_IN_MARKET` etc. O(1) type-based traversal. The reasoner's Cypher templates leverage this for fast pattern matching.

9. **KG-to-ASP compilation bridge** — No other system can compile a Neo4j subgraph into ASP facts, apply formal integrity constraints, solve with answer sets, and feed the results back into the reasoning loop. This is the neuro-symbolic reasoning bridge.

10. **Client-agnostic from day 1** — Rheem config is one file. Bayer/Suntory = add another config file. The KG Reasoner framework (entity linker, query planner, subgraph retriever, reasoning agent, answer generator) is generic. Client-specific knowledge lives in `WorkflowKGSchemaConfig` and domain configs only.

---

---

## XVI. Additional Production Concerns (Beyond SOTA Literature)

These are real-world production challenges that academic KG reasoning papers don't cover but enterprise systems MUST handle:

### Query Understanding (Deep Reasoning)

| Concern | How We Handle It |
|---------|-----------------|
| **Question decomposition** — complex questions → sub-questions | Tier 3 ReAct agent naturally decomposes: each reasoning step is a sub-question. Tier 4 uses community hierarchy for decomposition (global → domain → section). SemanticQueryPlanner already decomposes tabular questions into section selections. |
| **Temporal reasoning** — "what changed between FY2024 and FY2025?", "how did warranty claims trend?" | KG edges carry `valid_from`/`valid_to` from TemporalColumnConfig. `context_columns` like TTM_YEAR/period on edges enable time-scoped queries. Cypher templates include temporal filters: `WHERE r.TTM_YEAR = $year`. |
| **Numerical reasoning** — arithmetic on KG properties (sum, diff, ratio, %, ranking) | All metric_columns stored as numeric floats (via `_parse_numeric`). Cypher supports `SUM()`, `AVG()`, `COUNT()`, `MAX()`, `MIN()`. ClingconASPReasoner handles numeric constraint solving. Template Cypher includes aggregation patterns. |
| **Negation handling** — "which SKUs are NOT sold in Jacksonville?" | Cypher natively supports: `WHERE NOT (s)-[:SOLD_IN_MARKET]->(m {label: "JACKSONVILLE"})`. ASP supports negation-as-failure: `not_in_market(S,M) :- entity(S,"SKU",_), entity(M,"MARKET",_), not rel(S,sold_in_market,M).` |
| **Counter-factual / what-if reasoning** — "what if we drop supplier X?" | ASP `#minimize` / `#maximize` + weak constraints model what-if scenarios. ClingconASPReasoner handles numeric constraint optimization. Existing ASPProgramExecutionAgent supports "what-if add/remove constraint" patterns. |

### KG-Specific Challenges

| Concern | How We Handle It |
|---------|-----------------|
| **Incomplete KG handling** — missing edges/nodes | Confidence-weighted reasoning: if a path requires a low-confidence or missing edge, report "insufficient evidence" rather than hallucinate. Tier escalation: if Tier 1 returns empty, escalate to Tier 2 (broader subgraph may find alternative paths). |
| **KG freshness / staleness detection** | `content_hash` on SECTION nodes detects data changes. `updated_at` timestamp on every node/edge. Cache invalidation via `kg:invalidate:{namespace}` pub/sub on KG rebuild. ReasoningTrace records which KG version was queried. |
| **Schema evolution** — new entity/relationship types added over time | `merge_ontology_relationships()` on WorkflowKGSchemaConfig automatically incorporates new relationship types from ontology. Cypher schema compiler regenerates on each pipeline run. `_validate_schema_columns` catches config-vs-data mismatches at load time. |
| **Cross-namespace reasoning** — questions spanning multiple workflows | Entity linking searches across namespaces when explicitly requested. Cypher templates support multi-namespace joins: `MATCH (a {namespace: $ns1})-[r]->(b {namespace: $ns2})`. Phase 4 (production hardening) validates cross-namespace safety. |

### Production Operations

| Concern | How We Handle It |
|---------|-----------------|
| **Rate limiting per tier** | Cost budget per tier (Tier 1: $0.01, Tier 3: $0.20, Tier 4: $0.50). Max concurrent ReAct steps = 8. LiteLLM already supports rate limiting per model. Per-user rate limits at API layer. |
| **Graceful degradation** — KG unavailable → fallback | If Neo4j connection fails, KG tier router falls back to Tier 0 (tabular retrieval from PipelineStructuredOutput). GroundedAnswerEngine works with both KG and tabular evidence. Logged as degraded mode in ReasoningTrace. |
| **A/B testing** | Tier selection is deterministic from question features — can run A/B by routing 50% of questions through Tier 2 vs Tier 3 for the same complexity level. ReasoningTrace tracks which tier was used → offline analysis of answer quality per tier. |
| **Batch reasoning** — process N questions efficiently | Entity linking cache (L1) amortizes across batch. Cypher result cache (L2) deduplicates repeated subgraph patterns. Batch API endpoint accepts `List[Question]`, processes concurrently with shared cache. |
| **Webhook/event notification** | ReasoningTrace published to Redis pub/sub on completion. Consumers (dashboard, analytics, monitoring) subscribe to `reasoning:complete:{namespace}`. Integrates with existing FastAPI WebSocket support. |

---

*This plan is the north star. Each phase builds on the previous. The architecture is modular — we can ship Tier 0+1 and iterate toward Tier 3+4. Every component reuses existing infrastructure where possible. Every SOTA approach is covered. Every enterprise requirement is addressed. Every production operational concern is planned for.*

---

## Architectural Assessment — April 2026

### What's Built vs What's Planned

| Component | Status | Power Contribution |
|-----------|--------|-------------------|
| Schema-aware Cypher generation (Phase 0) | ✅ Production | Foundation |
| 3-stage Entity Linking (Neo4j fulltext + vector) | ✅ Production | High — grounds mentions to IDs |
| 21 Cypher Templates + CypherTemplateRegistry | ✅ Production | Medium — handles simple structural queries |
| 5-Tier Router (8-signal scoring + tabular override) | ✅ Production | High — correct question classification |
| Phase 0 LLM fallback (with SQL pre-validator) | ✅ Production | Critical — catches everything templates can't |
| Pipeline + API integration (Zeus ↔ RE) | ✅ Production | Deployment-ready |
| Subgraph Retrieval (Tier 2) | 📋 Planned | **Critical** — cross-domain reasoning |
| ReAct Agent + ASP Verification (Tier 3) | 📋 Planned | Long-term moat |
| Community Synthesis (Tier 4) | 📋 Planned | Strategic questions |
| Axiom Expansion (inverse, symmetric, transitive) | 📋 Planned | Query completeness |
| Confidence Propagation | 📋 Planned | Answer trustworthiness |
| 5-Level Semantic Cache | 📋 Planned | Performance at scale |

### Honest Power Assessment

**Current (Phase 0+1): 40% of full potential.** Reliable infrastructure, zero crashes, correct classification. But 50% of KG questions fall to Phase 0 (full LLM Cypher). The system is a smart query router with a good entity linker, not yet a full reasoner.

**With Phase 2 (Subgraph): 70%.** This is the inflection point. Cross-domain joins, multi-hop reasoning, subgraph-grounded answers. The KG starts answering questions that tabular fundamentally can't. Highest-ROI next investment.

**With Phase 3 (ReAct + ASP): 90%.** Neuro-symbolic reasoning — LLM explores, ASP verifies. Provably correct constraint checking on KG subgraphs. No competitor has this in production. Publishable research.

**With Phase 4 (Community): 100%.** Full vision. Strategic questions answered via hierarchical community summarization.

### What Makes This Architecture Unique

No existing system — academic or commercial — combines all of these:

1. **Schema-driven everything.** Detection terms, entity aliases, relationship types, Cypher templates — all derived from `WorkflowKGSchemaConfig`. New client = new config file, zero framework changes.

2. **Dual-source reasoning.** KG evidence + tabular evidence in the same pipeline. GroundedAnswerEngine doesn't care where evidence came from. No question is worse off than before KG was added.

3. **Graceful degradation chain.** Template → Phase 0 LLM → type inference → local embedding → tabular. Every component fails gracefully. Enterprise-level resilience that academic systems don't have.

4. **Neuro-symbolic integration (planned).** ClingoASP, ClingCon, NeurASP, LPMLN already in the codebase. Wiring them into the KG reasoning loop (Tier 3) creates provably correct reasoning that pure-LLM systems can't match.

5. **Structural hierarchy as scope filter.** WORKFLOW→DOMAIN→SECTION→ENTITY navigational layer enables cross-domain reasoning (Tier 2) without full-graph scans. No other system has this.

6. **Multi-domain namespace isolation.** Rheem, Bayer, Suntory — different KGs, different schemas, same framework. Namespace-scoped queries prevent cross-contamination.

### Key Risks

| Risk | Phase | Mitigation |
|------|-------|------------|
| Subgraph token budget overflow | Phase 2 | Community-aware pruning, structural scoping via SECTION |
| ReAct agent non-determinism | Phase 3 | Max 8 steps, ASP verification at each step, tier escalation budget |
| Community detection on heterogeneous KG | Phase 4 | Test Leiden/Louvain on actual Rheem KG before building summarization |
| No published benchmarks | All | Run on WebQSP/CWQ/MetaQA before claiming superiority |
| Semantic cache invalidation | Cross-phase | Defer caching until query patterns are observed in production |
| `model.encode()` blocks event loop | Phase 1 | Known. Fix with `asyncio.to_thread` before high-concurrency production. |

### Recommended Priority

1. **Phase 1.1** — Property-aware slot filling (`metric_property`, `filter_property`, `rank_property`). Increases template match rate from 50% to ~70%. Low effort, high ROI.
2. **Phase 2** — Subgraph retrieval. The inflection point. Makes the KG actually useful for complex questions. Medium effort, highest business value.
3. **Benchmarking** — Run on standard KG-QA datasets. Required for paper and client credibility.
4. **Phase 3** — ReAct + ASP. High effort, high risk, highest long-term moat. Design tools from real Phase 2 question patterns.
