# Workflow Output → Knowledge Graph Framework

## Why This Exists

Every GP workflow produces `PipelineStructuredOutput` — structured tables, NL summaries, tooltips, ontology metadata. Previously this output only fed the Q&A pipeline (LLM-grounded answers). This framework transforms **any** workflow output into a Knowledge Graph in Neo4j, enabling KG-based reasoning alongside LLM reasoning.

**The critical design constraint:** the framework is **client-agnostic**. Rheem is the first use case, but Bayer, Suntory, and future clients use the exact same framework code. Client-specific knowledge lives entirely in a **declarative schema configuration file** — never in framework code.

---

## Architecture Overview

```
  ANY Workflow Output
  (List[PipelineStructuredOutput])
       │
       ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 1: input_loader.py                            │
  │  Classify partitions by ExtractionMode               │
  │  + Incremental change detection (content hashing)    │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 2: structured_extractor.py                    │
  │  Schema-driven deterministic extraction              │
  │  + Confidence = 1.0                                  │
  │  + Temporal versioning + Implicit entities           │
  │  + context_columns (string) + metric_columns (float) │
  └──────────────────────┬───────────────────────────────┘
                         │
                    domain context
                    (entity types, relation types,
                     domain-scoped + cross-domain
                     known entity names as HINTS)
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 3: nl_extractor.py (domain-aware)             │
  │  SemanticaEngine LLM-based NER/REL                   │
  │  + Ontology-driven domain context preamble           │
  │  + Confidence from LLM mentions                      │
  │  + ontology.metadata column definitions              │
  │  (domain context is optional — off = same as before) │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 4: knowledge_graph_assembler.py               │
  │  Merge structured + NL → unified graph               │
  │  + Structural layer (WORKFLOW → DOMAIN → SECTION)    │
  │  + CONTAINS_ENTITY / JOINS_WITH links                │
  │  + Content hash stamping on SECTION nodes            │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 5: entity_resolver.py (P0)                    │
  │  Embedding-based fuzzy entity deduplication          │
  │  BAAI/bge-small-en-v1.5 → cosine similarity + validation │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 6: relationship_inferrer.py (P3)              │
  │  Multi-hop transitive relationship materialization   │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 7: graph_schema_validator.py (P2)             │
  │  SHACL-like pre-insertion validation                 │
  │  Cardinality / orphans / dangling / property types   │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 8: community_detector.py (P1)                 │
  │  Louvain community detection → COMMUNITY nodes       │
  │  Hierarchical (multi-level) + optional LLM summaries │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 9-10: knowledge_graph_writer.py               │
  │  Batched Neo4j insertion (5K nodes / 10K rels)       │
  │  + 6-check post-insertion validation                 │
  │  + Incremental: clear only changed partitions        │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Phase 11: Export D3 JSON (optional)                 │
  └──────────────────────────────────────────────────────┘
```

**Orchestrated by:** `workflow_to_kg_pipeline.py` → `WorkflowToKnowledgeGraphPipeline(KnowledgeGraphPipeline)` — 11-phase pipeline with all advanced features opt-in via `WorkflowKGSchemaConfig`

---

## File Map

All under `app/kr_engine/pipeline/knowledge_graph_builder/`:

> **Note:** Folder was renamed from `knowledge_graph_builder` → `knowledge_graph_builder` to reflect that the framework is generic and will handle non-workflow inputs in the future.

```
knowledge_graph_builder/
├── __init__.py                              # Package re-exports
├── run_workflow_to_knowledge_graph_pipeline.py  # Entry point (CLI + convenience functions)
│
├── kg_models/                               # Data models + schema config + domain configs
│   ├── __init__.py                          # Re-exports all model types + PIPELINE_CLASS_NAME
│   ├── models.py                            # Enums, PartitionProfile, make_entity_id()
│   ├── schema_config.py                     # WorkflowKGSchemaConfig + declarative types
│   └── specific_domain_configs/
│       └── rheem_composition.py             # Rheem schema: 30 partition schemas, 4 domains
│
├── workers/                                 # Loader, extractors, assembler, writer, advanced
│   ├── input_loader.py                      # JSON loading, partition classification
│   ├── structured_extractor.py              # Schema-driven extraction (+ confidence + temporal)
│   ├── nl_extractor.py                      # SemanticaEngine wrapper (+ confidence propagation)
│   ├── knowledge_graph_assembler.py         # Entity resolution, merge, structural nodes
│   ├── knowledge_graph_writer.py            # Batched Neo4j insertion + 6-check validation + incremental
│   ├── entity_resolver.py                   # P0: Embedding-based fuzzy entity deduplication
│   ├── community_detector.py                # P1: Louvain community detection + COMMUNITY nodes
│   ├── graph_schema_validator.py            # P2: SHACL-like pre-insertion validation
│   └── relationship_inferrer.py             # P3: Multi-hop transitive relationship inference
│
└── pipeline/
    └── workflow_to_kg_pipeline.py           # WorkflowToKnowledgeGraphPipeline orchestrator
```

**Integration points (no existing files modified):**
- Subclassing `KnowledgeGraphPipeline` from `app/kr_engine/pipeline/kd_pipeline.py`
- Using `SemanticaEngine` from `app/kr_engine/semantic_framework/semantica/semantica_engine.py`
- Producing `SemanticaGraphArtifact` / `SemanticaEntity` / `SemanticaRelation` from `semantica_models.py`
- Inserting via `GPKnowledgeGraph.insert_semantica_graph_v2()` from `knowlegde_graph/knowledge_graph.py`
- Querying via `GPKnowledgeGraph.execute_query()` (added for validation)

> **All imports use `knowledge_graph_builder`** (not the old `knowledge_graph_builder`).

---

## Type System (Enums)

All entity types, relationship predicates, and extraction tags are `str, Enum` — catches typos at import time, enables IDE autocomplete, and works as drop-in string replacements.

### Framework-level (in `kg_models/models.py`)

```python
class StructuralEntityType(str, Enum):
    WORKFLOW = "WORKFLOW"    # Root node — entry point for Q&A retrieval
    SECTION = "SECTION"      # One per partition
    DOMAIN = "DOMAIN"        # One per business domain

class StructuralRelationType(str, Enum):
    HAS_DOMAIN = "HAS_DOMAIN"               # WORKFLOW → DOMAIN
    BELONGS_TO_DOMAIN = "BELONGS_TO_DOMAIN"  # SECTION → DOMAIN
    CONTAINS_ENTITY = "CONTAINS_ENTITY"      # SECTION → extracted entities
    JOINS_WITH = "JOINS_WITH"                # SECTION → SECTION (cross-partition)

class ExtractionMode(str, Enum):
    STRUCTURED_ONLY = "structured_only"
    NL_ONLY = "nl_only"
    HYBRID = "hybrid"
    SKIP = "skip"

class ExtractionType(str, Enum):
    STRUCTURED = "structured"   # Deterministic extraction from tabular data
    NL = "nl"                   # LLM-based extraction from natural language
    INFERRED = "inferred"       # Derived by rule-based inference (transitive closure)
    STRUCTURAL = "structural"   # Framework scaffold nodes (WORKFLOW, DOMAIN, SECTION)

class CardinalityDirection(str, Enum):
    OUTGOING = "outgoing"
    INCOMING = "incoming"

class ViolationType(str, Enum):
    CARDINALITY_UNDER = "cardinality_under"
    CARDINALITY_OVER = "cardinality_over"
    DANGLING_SUBJECT = "dangling_subject"
    DANGLING_OBJECT = "dangling_object"
    ORPHANED_ENTITY = "orphaned_entity"
    INVALID_CONFIDENCE = "invalid_confidence"
```

**Every entity and relation in Neo4j carries `extraction_type`** — enabling the reasoning engine to apply source-aware confidence weighting and provenance queries like `MATCH (n {extraction_type: "nl"})`.


### Client-level (in `kg_models/specific_domain_configs/rheem_composition.py`)

```python
class RheemEntityType(str, Enum):
    SKU = "SKU"
    MANUFACTURER = "MANUFACTURER"
    RETAILER = "RETAILER"
    MARKET = "MARKET"
    REGION = "REGION"
    PRODUCT_CATEGORY = "PRODUCT_CATEGORY"
    PRODUCT_SEGMENT = "PRODUCT_SEGMENT"
    PRODUCT_FAMILY = "PRODUCT_FAMILY"
    PERFORMANCE = "PERFORMANCE"
    FUEL_TYPE = "FUEL_TYPE"
    STORE = "STORE"
    DISTRIBUTION_CENTER = "DISTRIBUTION_CENTER"
    PRODUCT_TYPE = "PRODUCT_TYPE"
    MODEL = "MODEL"
    SUPPLIER = "SUPPLIER"
    PLANT = "PLANT"
    OPERATING_UNIT = "OPERATING_UNIT"
    SPEND_CATEGORY = "SPEND_CATEGORY"
    PRICING_SCENARIO = "PRICING_SCENARIO"
    ZIP_CODE = "ZIP_CODE"
    STATE = "STATE"

class RheemRelationType(str, Enum):
    MANUFACTURED_BY = "MANUFACTURED_BY"
    SOLD_IN_MARKET = "SOLD_IN_MARKET"
    SOLD_BY_RETAILER = "SOLD_BY_RETAILER"
    IN_MARKET = "IN_MARKET"
    IN_REGION = "IN_REGION"
    HAS_CATEGORY = "HAS_CATEGORY"
    HAS_MODEL = "HAS_MODEL"
    HAS_PRODUCT_SEGMENT = "HAS_PRODUCT_SEGMENT"
    HAS_PRODUCT_FAMILY = "HAS_PRODUCT_FAMILY"
    HAS_PERFORMANCE = "HAS_PERFORMANCE"
    COMPETES_WITH_MANUFACTURER = "COMPETES_WITH_MANUFACTURER"
    HAS_FUEL_TYPE = "HAS_FUEL_TYPE"
    SERVED_BY_DC = "SERVED_BY_DC"
    SUPPLIES_TO = "SUPPLIES_TO"
    SUPPLIES_CATEGORY = "SUPPLIES_CATEGORY"
    IN_STATE = "IN_STATE"
    PART_OF_OPERATING_UNIT = "PART_OF_OPERATING_UNIT"
    PRICED_UNDER_SCENARIO = "PRICED_UNDER_SCENARIO"
    WARRANTY_IMPACT_AT = "WARRANTY_IMPACT_AT"
```

Adding a new client = define `ClientEntityType` + `ClientRelationType` enums in their config file.

---

## Core Design Decisions

### 1. Dual-Path Extraction (Structured → NL, Sequential)

The most important design decision. Workflow output contains two fundamentally different data types:

- **Tabular data** (`item.data`): Rows of SKUs, prices, transactions. Millions of rows. Deterministic extraction is faster, cheaper, and more accurate than LLM-based extraction for structured data.
- **NL text** (`item.summary`, `item.tooltip`, `ontology.nlp_methodology_description`, `ontology.title`, `ontology.metadata`): Strategic analysis, methodology descriptions, column semantic definitions. LLM-based NER/REL (SemanticaEngine) is appropriate here.

**Phase ordering:** Structured runs FIRST (Phase 2), then NL (Phase 3). This is intentional — structured results + ontology schema provide **domain context as hints** to make NL extraction smarter (see §1b below).

**The framework classifies each partition:**
```
Has schema config + has data rows + has text  → HYBRID    (both paths)
Has schema config + has data rows             → STRUCTURED_ONLY
Has text but no schema config                 → NL_ONLY
Neither                                       → SKIP
```

### 1b. Ontology-Driven Domain-Aware NL Extraction

**Problem:** Without domain context, the LLM invents generic entity types (`CONCEPT`, `LOCATION`) and predicates (`RELATES_TO`) that don't align with the structured schema. Entities like `MARKET "JACKSONVILLE"` get extracted as `LOCATION "Jacksonville"` — different type, different ID, no merge.

**Solution:** After structured extraction completes, `build_domain_context()` assembles domain knowledge as **hints (not constraints)** for the LLM:

- **Entity types + relation types** — whole namespace vocabulary (all types across all domains)
- **Known entity names** — domain-scoped (entities from the partition's domain) + cross-domain shared entities (entities appearing in 2+ domains, e.g. MANUFACTURER "Rheem", RETAILER "Home Depot", shared MARKET names)
- **Cap:** 20 names per type to keep the preamble concise (~200-300 tokens)

**The critical design principle:** Ontology + structured results = **context to make the LLM smarter, NOT constraints that limit it.** The preamble explicitly states: *"You are NOT limited to these — extract ALL entities and relationships you find, including types and predicates not listed below."* The LLM should use known types when it sees them, but freely discover novel entity types (PRICING_TREND, CAUSAL_INSIGHT) and relationships that no schema anticipated.

**On/off toggle:** `domain_context=None` → NL extraction works exactly as before (zero domain awareness). When provided, the text is enriched with a domain preamble. Currently always on in the pipeline — to disable, set `domain_ctx = None` in `workflow_to_kg_pipeline.py` (one line change).

**Cross-domain entity grounding:** When a warranty summary mentions "Jacksonville market", the LLM sees `MARKET: ATLANTA, CHICAGO, JACKSONVILLE, ...` in the preamble and uses the correct type (`MARKET`) with a name that aligns for deterministic ID merging. Without this, the LLM might call it `LOCATION "Jacksonville"` — wrong type, different ID, no merge.

### 2. Declarative Schema Config

Client-specific knowledge is declared, not coded. Entity types and relationship predicates use enums:

```python
PartitionSchemaConfig(
    entity_mappings=[
        ColumnEntityMapping("Base SKU", RheemEntityType.SKU,
                           entity_attributes=["Base Fuel Type", "Base Warranty"]),
        ColumnEntityMapping("Base Company", RheemEntityType.MANUFACTURER,
                           entity_attributes=[]),
    ],
    relationship_rules=[
        RelationshipRule("Base SKU", "Base Company", RheemRelationType.MANUFACTURED_BY,
                        metric_columns=["Dollar Diff ($)"]),
    ],
    implicit_entities=[
        ImplicitEntityConfig(
            entity_type="RETAILER", entity_value="Home Depot",
            link_from_column="Base SKU", relation_type="SOLD_BY_RETAILER",
        ),
    ],
    aggregation=AggregationStrategy(
        group_by_columns=["vendor_id", "organization_id"],
        sum_columns=["transaction_spend"],
    ),
    attribute_columns=[],
)
```

This means:
- Adding a new client = creating one config file. Zero framework changes.
- Changing extraction rules = editing the config. No code changes.
- The framework handles all the HOW; the config declares all the WHAT.

### 3. Per-Entity Attribute Scoping

**Problem:** In a row with multiple entity columns (e.g., SKU + MARKET + MANUFACTURER), flat `attribute_columns` leak onto ALL entities. `company_avg_cogs` appearing on a MARKET node is nonsensical.

**Solution:** `entity_attributes` field on `ColumnEntityMapping`:

```python
# Per-entity scoping — SKU gets cost attributes, MARKET gets nothing
ColumnEntityMapping("sku", "SKU", entity_attributes=["company_avg_cogs", "company_avg_grossprice"])
ColumnEntityMapping("market", "MARKET", entity_attributes=[])
```

**Rules:**
- If `entity_attributes` is set (even to `[]`) → ONLY those columns become properties on that entity
- If `entity_attributes` is `None` (default) → falls back to partition-level `attribute_columns` (backward compatible)

Implemented in `structured_extractor.py` via `_resolve_attr_columns()`.

### 4. Implicit Entities (Cross-Partition Linking + Contextual Entities)

**Problem:** Some entities exist from context, not from data columns. Examples:
- All pricing simulator SKUs are Rheem products at Home Depot — but there's no "company"/"retailer" column
- Scenario detail partitions need to link SKUs to the PRICING_SCENARIO summary node — but there's no "scenario" column in detail data
- All warranty stores are Home Depot stores — but there's no "retailer" column

**Solution:** `ImplicitEntityConfig` on `PartitionSchemaConfig`:

```python
ImplicitEntityConfig(
    entity_type="PRICING_SCENARIO",
    entity_value="Scenario 1: Flat 10%",   # MUST match summary partition's column value
    link_from_column="sku",                 # Links FROM every unique SKU entity
    relation_type="PRICED_UNDER_SCENARIO",
)
```

**How it works:**
1. After row processing, the framework creates/merges the implicit entity node using `make_entity_id(entity_type, entity_value)`
2. Creates a deduplicated relationship from each unique entity matching `link_from_column` to the implicit entity
3. `reverse_direction=True` flips the edge direction if needed
4. The entity ID is deterministic — if another partition creates the same entity (e.g., summary creates PRICING_SCENARIO "Scenario 1: Flat 10%"), they merge into the same node

**This is the cross-partition linking mechanism.** Detail partitions declare implicit entities whose `entity_value` matches values from other partitions. `make_entity_id()` produces the same ID → nodes merge in the assembled graph.

### 5. Metrics on Edges vs Entity Properties

**Problem:** Market-specific metrics like `optimal_retail_price` or `claims_usd_total` are NOT intrinsic to the SKU or STORE — they depend on the relationship context (which market, which scenario). Storing them as flat entity properties loses context.

**Solution:** Two types of edge properties on `RelationshipRule`:

- **`metric_columns`** — numeric values enforced as `float` via `_parse_numeric()`. Handles `%` → decimal (`"65%"` → `0.65`), `$` stripping, comma thousands. Non-numeric values are logged + skipped.
- **`context_columns`** — string/label values stored as-is. For time dimension labels (`TTM_YEAR="FY2025"`), categorical flags, or any non-numeric edge property.

```python
# Market-specific economics live on the SKU→MARKET edge
RelationshipRule("sku", "market", "SOLD_IN_MARKET",
    metric_columns=["optimal_retail_price", "expected_units", "current_competitor_price"])

# Warranty KPI edges carry time dimension as string label
RelationshipRule("str_nbr", "mkt_nm", "IN_MARKET",
    context_columns=["TTM_YEAR", "period"],         # "FY2025" stored as-is
    metric_columns=["claims_usd_total", "ordered_sales_usd"])  # enforced float

# SKU-intrinsic attributes (not market-dependent) stay on the SKU node
ColumnEntityMapping("sku", "SKU", entity_attributes=["company_avg_cogs"])
```

**Query pattern:** "What's the optimal price for SKU X in JACKSONVILLE?"
```cypher
MATCH (s:SKU)-[r:SOLD_IN_MARKET]->(m:MARKET {label: "JACKSONVILLE"})
WHERE s.label = "X"
RETURN r.optimal_retail_price, r.expected_units
```

### 6. 3-Way Relationship Modeling (SKU × MARKET × SCENARIO)

Pricing simulator detail rows are 3-way facts: (SKU, MARKET, SCENARIO). Neo4j doesn't have hyperedges. Modeled pragmatically with 2 edges:

```
SKU ──SOLD_IN_MARKET──→ MARKET     (carries all market-specific economics)
SKU ──PRICED_UNDER_SCENARIO──→ PRICING_SCENARIO   (via implicit entity)
```

The `source_partition` property on the SOLD_IN_MARKET edge identifies which scenario produced it. Query correlates via:
```cypher
MATCH (s:SKU)-[r:SOLD_IN_MARKET]->(m:MARKET),
      (s)-[:PRICED_UNDER_SCENARIO]->(sc:PRICING_SCENARIO)
WHERE sc.label CONTAINS "Scenario 4"
  AND r.source_partition CONTAINS "scenario_4"
RETURN r.optimal_retail_price, sc.company_gross_profit_dollar_diff
```

### 7. Aggregation Strategy for Large Tables

The `edp_spend_f_curated` partition has **1.6M transaction rows**. Creating one node per row would produce an unusable graph. Instead:

```
AggregationStrategy(
    group_by_columns=["vendor_id", "vendor_name", "organization_id", "category_id"],
    sum_columns=["transaction_spend"],
    count_column="transaction_spend",
)
```

This reduces 1.6M rows → ~6K supplier-plant-category groups with aggregated spend totals.

### 8. Deterministic Entity IDs

Entity IDs are generated as: `gp_entity_<sha1(TYPE::normalized_name)[:16]>`

This is the **same pattern** used by `SemanticaEngine`, so entities created by the structured extractor merge cleanly with entities from NL extraction.

**Cross-domain merging** is the key power of deterministic IDs:
- Rheem MANUFACTURER node: `gp_entity_c5f2ad4471d33a36` — same ID whether created by Pricing Simulator, Pricing CI, or Warranty schemas
- Home Depot RETAILER node: `gp_entity_2b9337daf052a6fd` — same ID from Pricing Simulator's implicit entity and Warranty store schemas
- MODEL entities from `module_7_warranty_curve_dataset` merge with MODULE `module_3c_top_models_by_impact_table` — same make_entity_id("MODEL", model_value)

### 9. Column Name Flexibility

The Rheem data has a known issue: column headers in `item.data` use Title Case (`"Base SKU"`) while `ontology.metadata` keys use snake_case (`"base_sku"`). The `_find_column()` function handles this:

```python
def _find_column(headers, target):
    # Normalizes both to lowercase + underscores for comparison
    # "Base SKU" matches "base_sku" and vice versa
```

### 10. WORKFLOW Root Node + Structural Graph Layer

The framework creates a **hierarchical structural layer** with a WORKFLOW root node as the entry point for KG retrieval:

```
WORKFLOW (rheem_composition)              ← root node, entry point from Q&A
   │
   ├── HAS_DOMAIN → DOMAIN (Pricing CI)
   │     │
   │     ├── BELONGS_TO_DOMAIN ← SECTION (competitive_pricing...)
   │     │     └── CONTAINS_ENTITY → SKU, MANUFACTURER, MARKET ...
   │     │
   │     └── BELONGS_TO_DOMAIN ← SECTION (price_recommendations...)
   │           └── CONTAINS_ENTITY → SKU, MARKET ...
   │
   ├── HAS_DOMAIN → DOMAIN (Warranty Intelligence)
   │     └── ...
   │
   └── HAS_DOMAIN → DOMAIN (Supply Chain)
         └── ...

Cross-partition: SECTION ──JOINS_WITH──→ SECTION (from data_set_join_logic)
```

**Why a WORKFLOW root node:**
- **Query scoping**: From Q&A, you know the workflow → start traversal from WORKFLOW root → entire query is scoped to that subgraph. No property scan across all nodes.
- **Multi-tenant isolation**: Rheem, Bayer, Suntory KGs in the same Neo4j instance. Each has its own WORKFLOW root — no cross-contamination.
- **Graph-native**: Neo4j is optimized for traversal from a known starting node (indexed lookup → O(1)), not for property filters across all nodes.
- **Double scoping**: Every node carries both `namespace` and `workflow_name` properties. Queries can scope by graph traversal (fast) AND/OR property filter (safety net).

---

## OOP Hierarchy

```
KnowledgeGraphPipeline (ABC)                    ← NEW base class in kd_pipeline.py
├── semantic_artifact, config, export_to_d3_json()
├── abstract: configure_pipeline(), update_with_gp_knowledge_graph()
│
├── KnowledgeDiscoveryPipeline                   ← existing (Bayer, etc.)
│   ├── collectors, extractors, parser, aligner
│   └── SemanticaKnowledgeGraphDiscoveryPipeline
│
└── WorkflowToKnowledgeGraphPipeline             ← this framework
    ├── structured_extractor, nl_extractor, assembler, writer
    └── run_kg_pipeline()
```

`WorkflowToKnowledgeGraphPipeline` extends `KnowledgeGraphPipeline` directly — not `KnowledgeDiscoveryPipeline` — because it doesn't use collectors, extractors, parser, or aligner. Clean separation.

---

## Entity Resolution Strategy

The `knowledge_graph_assembler.py` merges artifacts from all sources:

1. **Entity deduplication by ID**: First occurrence's properties win, subsequent supplement
2. **Relation deduplication by (subject, predicate, object)**: First occurrence wins, properties merge
3. **Source tracking**: Every entity has `source_partition` property listing all partitions it appeared in (e.g., `"competitive_pricing,competitive_pricing_geographic"`)

**Priority**: Structured properties win over NL properties (structured data is ground truth; NL extraction may hallucinate)

---

## Rheem Composition: Domain Breakdown

### Domain Overview

| Domain | Data Partitions | NL-Only Partitions | Key Entity Types | Key Relationships |
|--------|:-:|:-:|-----------------|-------------------|
| **Pricing CI** | 5 | 4 | SKU, MANUFACTURER, MARKET, REGION, MODEL, STATE | MANUFACTURED_BY, SOLD_IN_MARKET, COMPETES_WITH_MANUFACTURER, HAS_MODEL |
| **Pricing Simulator** | 5 | 0 | SKU, MARKET, PRICING_SCENARIO, PRODUCT_CATEGORY, FUEL_TYPE, PRODUCT_SEGMENT, PERFORMANCE, PRODUCT_FAMILY | SOLD_IN_MARKET, PRICED_UNDER_SCENARIO, HAS_CATEGORY, HAS_FUEL_TYPE |
| **Warranty** | 5 | 22 | STORE, MARKET, REGION, DISTRIBUTION_CENTER, MODEL, PRODUCT_TYPE, ZIP_CODE, STATE | IN_MARKET, SERVED_BY_DC, WARRANTY_IMPACT_AT, IN_STATE |
| **Supply Chain** | 6 | 0 | SUPPLIER, PLANT, OPERATING_UNIT, SPEND_CATEGORY, STATE, REGION | SUPPLIES_TO, SUPPLIES_CATEGORY, IN_REGION, IN_STATE |

### Cross-Domain Entity Merging

Entities merge across domains via deterministic IDs:

| Entity | Created By | Merged Across |
|--------|-----------|--------------|
| MANUFACTURER "Rheem" | Pricing CI (explicit column), Pricing Sim (implicit), Warranty (implicit) | All 3 domains → single node `gp_entity_c5f2ad4471d33a36` |
| RETAILER "Home Depot" | Pricing Sim (implicit), Warranty stores (implicit) | Pricing Sim + Warranty → single node `gp_entity_2b9337daf052a6fd` |
| MARKET entities | Pricing CI, Pricing Sim, Warranty KPIs, Warranty scatter | All domains with market data |
| REGION entities | Pricing CI, Warranty KPIs, Supply Chain | Multiple domains |
| STATE entities | Pricing CI, Warranty ZIP, Supply Chain | Multiple domains |
| MODEL entities | Warranty curve (module 7), Warranty impact (module 3c) | Within Warranty domain |
| DC entities | Warranty KPIs, Warranty impact (module 3b), Warranty return profiles | Within Warranty domain |

### Implicit Entities

Implicit entities inject contextual nodes where no data column exists:

| Implicit Entity | Linked From | Relation | Used In |
|----------------|------------|----------|---------|
| MANUFACTURER "Rheem" | SKU (sku) | MANUFACTURED_BY | Pricing Simulator (4 scenario schemas) |
| MANUFACTURER "Rheem" | MODEL (model/Model) | MANUFACTURED_BY | Warranty curve, Warranty model impact |
| MANUFACTURER "Rheem" | PRODUCT_TYPE (product_type/segment_label) | MANUFACTURED_BY | Warranty claims product, Warranty return profile |
| RETAILER "Home Depot" | SKU (sku) | SOLD_BY_RETAILER | Pricing Simulator (4 scenario schemas) |
| RETAILER "Home Depot" | STORE (str_nbr/Store Number) | SOLD_BY_RETAILER | Warranty KPIs, store coverage, prioritization, scatter plot (6 schemas) |
| PRICING_SCENARIO per label | SKU (sku) | PRICED_UNDER_SCENARIO | Pricing Simulator (4 scenario schemas) |

### Pricing Competitive Intelligence Knowledge Architecture

```
MANUFACTURER "Rheem" (explicit from "Base Company" column)
     ↑
     │ MANUFACTURED_BY
     │
Base SKU ──COMPETES_WITH──→ Competitor SKU (direct SKU-to-SKU competition edge)
  │  │                          │
  │  │                          ├──MANUFACTURED_BY──→ MANUFACTURER (competitor company)
  │  │                          ├──SOLD_BY_RETAILER──→ RETAILER "Lowe's" (implicit)
  │  │                          └── entity_attributes: Competitor Rating, Competitor Sentiment scores...
  │  │
  │  ├──SOLD_IN_MARKET──→ MARKET ──IN_REGION──→ REGION
  │  │     (price metrics, date on edge)
  │  │
  │  ├──SOLD_BY_RETAILER──→ RETAILER "Home Depot" (implicit)
  │  ├──HAS_MODEL──→ MODEL
  │  └── entity_attributes: Base Core SKU, Rating, Sentiment scores...
  │
  └──COMPETES_WITH_MANUFACTURER──→ MANUFACTURER "A.O. Smith" (implicit, marker edge)

"Base Company" ──COMPETES_WITH_MANUFACTURER──→ MANUFACTURER "A.O. Smith" (implicit)
  (creates Rheem → A.O. Smith manufacturer-level competition edge)
```

**Key design patterns:**

1. **Dual SKU entities per row**: Each pricing row has BOTH a base SKU and a competitor SKU. These are separate entity mappings with **separate `entity_attributes`** lists — base gets base scores, competitor gets competitor-prefixed scores.

2. **COMPETES_WITH relationship**: Direct SKU-to-SKU product competition edge carrying price differential metrics (`Dollar Diff ($)`, `Percent Diff`). Enables queries like "Which competitor SKU is closest in price to Base SKU X?"

3. **COMPETES_WITH_MANUFACTURER**: Company-level competition marker. The implicit `_IMPLICIT_CI_COMPETITOR_MFR` creates a Rheem → A.O. Smith manufacturer-level competition edge (from "Base Company" to implicit "A.O. Smith"). No metrics — purely relational.

4. **Geographic partition aggregation**: The `competitive_pricing_geographic` partition has ~39K rows. Aggregated by `(Base SKU, Competitor SKU, Base Market Name, State)` with `avg_columns` for prices and diffs. Reduces to ~10K groups while preserving state-level granularity.

5. **Date on edges (partition 0 only)**: The non-geographic pricing partition includes `Date` as a metric_column on SOLD_IN_MARKET, enabling temporal queries. The geographic partition intentionally excludes Date (lost in aggregation — temporal tracking happens in partition 0).

6. **Price recommendations partition**: Pure Rheem SKU data (no competitors). Adds implicit MANUFACTURER "Rheem" and RETAILER "Home Depot" for cross-domain merging.

### Pricing Simulator Knowledge Architecture

```
PRICING_SCENARIO (from summary partition — 4 nodes with portfolio-level financials)
     ↑
     │ PRICED_UNDER_SCENARIO (via implicit entity, cross-partition link)
     │
SKU ──SOLD_IN_MARKET──→ MARKET (19 market-specific metrics on edge)
 │       └── source_partition identifies which scenario
 │
 ├──MANUFACTURED_BY──→ MANUFACTURER "Rheem" (implicit)
 ├──SOLD_BY_RETAILER──→ RETAILER "Home Depot" (implicit)
 ├──HAS_CATEGORY──→ PRODUCT_CATEGORY
 ├──HAS_FUEL_TYPE──→ FUEL_TYPE
 ├──HAS_PRODUCT_SEGMENT──→ PRODUCT_SEGMENT
 ├──HAS_PERFORMANCE──→ PERFORMANCE
 └──HAS_PRODUCT_FAMILY──→ PRODUCT_FAMILY (Scenario 1 only)

SKU node properties: company_avg_grossprice, company_avg_cogs (per-entity scoped)
MARKET node properties: [] (no leaked attributes)
PRICING_SCENARIO node properties: 26 portfolio-level financial metrics
```

**Cross-partition linking mechanism:** Each scenario detail partition creates an implicit PRICING_SCENARIO entity whose `entity_value` exactly matches the summary partition's "scenario" column value. `make_entity_id("PRICING_SCENARIO", "Scenario 1: Flat 10%")` produces the same ID in both partitions → nodes merge with combined properties.

### Warranty Intelligence Knowledge Architecture

```
MANUFACTURER "Rheem" (implicit, cross-domain — same node as Pricing)
     ↑
     │ MANUFACTURED_BY
     │
MODEL ──WARRANTY_IMPACT_AT──→ DISTRIBUTION_CENTER (impact metrics on edge)
  │                                    ↑
  │ (q1_2017..q4_2025 quarterly        │ SERVED_BY_DC
  │  claim $ as node properties)        │
  │                                    │
  │         PRODUCT_TYPE ──MANUFACTURED_BY──→ MANUFACTURER "Rheem"
  │         (claims_diff, paid_amount, number_of_claims)
  │         (lifecycle percentages for sub-class return profiles)
  │
STORE ──IN_MARKET──→ MARKET (21 metric columns including time dimension)
  │                    │
  ├──IN_REGION──→ REGION
  ├──SERVED_BY_DC──→ DISTRIBUTION_CENTER
  └──SOLD_BY_RETAILER──→ RETAILER "Home Depot" (implicit, cross-domain)

ZIP_CODE ──IN_STATE──→ STATE (7 metric columns: paid amounts by performance tier)

STORE node properties: address1, rlc_number (per-entity scoped)
DC node properties: RDC_NBR, tier_label, lifecycle percentages (per-entity scoped)
```

**Key design notes:**
- WARRANTY_IMPACT_AT is distinct from SERVED_BY_DC — it captures warranty-specific model×DC impact (expected vs observed claim %, dollar impact)
- `TTM_YEAR` and `period` are both included as edge metrics on STORE→MARKET — `_find_column` matches whichever exists (TTM partition uses TTM_YEAR, CY partition uses period)
- Warranty curve MODEL entities merge with module 3c MODEL entities via deterministic IDs — a MODEL node can have both quarterly time series (from curve) and DC impact data (from hot spot analysis)
- 22 of 27 warranty items have 0 data rows — their knowledge is in markdown tables within `summary`, captured by the NL extractor

### Supply Chain Knowledge Architecture

```
                    ┌─────────────────────────────────────────────────────┐
                    │              Cross-Partition Merging                │
                    │                                                     │
                    │  SUPPLIER (vendor_id)  ← merges across:            │
                    │    • edp_supplier_data  (attributes: risk scores)   │
                    │    • edp_lead_time      (edge: lead time to PLANT)  │
                    │    • edp_spend_f_curated (edge: spend to OP_UNIT)   │
                    │                                                     │
                    │  PLANT (organization_name) ← merges across:        │
                    │    • edp_org_location   (attributes: city, country) │
                    │    • edp_lead_time      (via SUPPLIES_TO edge)      │
                    │                                                     │
                    │  SPEND_CATEGORY (category_id) ← merges across:     │
                    │    • edp_spend_categories_data (hierarchy attrs)    │
                    │    • edp_spend_f_curated (via SUPPLIES_CATEGORY)    │
                    └─────────────────────────────────────────────────────┘

SUPPLIER ──SUPPLIES_TO──→ PLANT (lead time + product count on edge, from edp_lead_time)
  │              │
  │              ├──IN_STATE──→ STATE (from edp_org_location)
  │              └──PART_OF_OPERATING_UNIT──→ OPERATING_UNIT (from edp_org_location)
  │
  ├──SUPPLIES_TO──→ OPERATING_UNIT (spend + quantity on edge, from edp_spend_f_curated)
  ├──SUPPLIES_CATEGORY──→ SPEND_CATEGORY (spend + quantity on edge)
  │                          │
  │                          └── attributes: segment2..5_description, direct_level,
  │                                          manager_level (from edp_spend_categories_data)
  │
  └── attributes: country, active_flag, risk scores, stability scores,
                   commentary (from edp_supplier_data)

OPERATING_UNIT ── attributes: town_or_city, country (from edp_operating_unit_data)
  └──IN_REGION──→ REGION
```

**Key design decisions:**

1. **Three-table SUPPLIER merge**: A single SUPPLIER node (e.g., vendor_id "V12345") gets:
   - **Attributes** from `edp_supplier_data` (risk scores, stability, commentary)
   - **SUPPLIES_TO → PLANT** edges from `edp_lead_time` (avg lead time, product count)
   - **SUPPLIES_TO → OPERATING_UNIT** edges from `edp_spend_f_curated` (spend totals)
   - All three partitions use `vendor_id` → `make_entity_id("SUPPLIER", "V12345")` → same node

2. **Aggregation on large tables**:
   - `edp_lead_time` (~60K rows): Grouped by `(vendor_id, organization_name)`, avg `full_lead_time`, count `segment1` (products). Reduces to ~unique vendor×plant pairs.
   - `edp_spend_f_curated` (~1.67M rows): Grouped by `(vendor_id, vendor_name, operating_unit_id, category_id, category_description, spend_type)`, sum spend/quantity, avg unit_price. The `spend_type` in group_by preserves Direct vs Indirect spend breakdown per supplier.

3. **OPERATING_UNIT cross-partition merge limitation (KNOWN)**:
   - `edp_org_location` maps PLANT → OPERATING_UNIT via `operating_unit_wid` (joins to `edp_operating_unit_data.row_wid`)
   - `edp_spend_f_curated` maps SUPPLIER → OPERATING_UNIT via `operating_unit_id` (joins to `edp_operating_unit_data.organization_id`)
   - These are **different keys** for the same logical entity → `make_entity_id("OPERATING_UNIT", "12345")` vs `make_entity_id("OPERATING_UNIT", "ORG_456")` → **entities do NOT merge**
   - This is a framework limitation — no lookup-based entity resolution. Would require a join table or ID normalization step. Documented, not fixable without framework changes.

4. **Transaction-level fields excluded from entity attributes**: `spend_type` and `rheem_global_bu` vary per transaction (a supplier can have BOTH Direct and Indirect spend). Placing them as entity_attributes on SUPPLIER would capture only the first row's value, which is misleading. Instead, `spend_type` is in `group_by_columns` so it creates separate aggregation groups.

5. **Reference data partitions** (`edp_supplier_data`, `edp_spend_categories_data`, `edp_operating_unit_data`): One row per entity, no aggregation needed. All columns become entity attributes. These enrich the entity nodes that transaction tables create relationships between.

---

## NL Extraction Details

NL extraction uses `SemanticaEngine` (same as the Bayer consumer health pipeline):

- **LLM**: GPT-4.1 via OpenAI provider (configurable via `nl_llm_model` / `nl_llm_provider`)
- **Pipeline**: NER → REL → Triplet → dedup → GraphBuilder
- **Concurrency**: `asyncio.Semaphore(10)` — max 10 concurrent LLM calls
- **Domain-aware** (optional): When `domain_context` is provided, a preamble with entity type hints, relation type hints, and known entity names is prepended to the text before sending to SemanticaEngine (see §1b above)
- **Text assembly** (via `_build_nl_text()`): Concatenates per partition:
  1. Domain knowledge preamble (if domain_context enabled) — entity types, relation types, known entity names as hints
  2. `item.title` / `item.name` — heading
  3. `ontology.title` — included if different from `item.title` (alternative phrasing with additional context)
  4. `item.description` — workflow description (often empty)
  5. `item.summary` — executive summary / analysis output
  6. `item.tooltip` / `ontology.nlp_methodology_description` — methodology text (deduplicated when identical)
  7. `ontology.metadata` — **semantic column definitions** formatted as `## Data Column Definitions` with bullet points per column (e.g., `weighted_average_elasticity: Final selected SKU x market demand elasticity from log-log regression on POS data`)
- **Fallback**: If LLM extraction fails for a partition, it logs a warning and continues
- **Provenance**: Every NL entity/relation gets `extraction_type=NL`, `namespace`, `confidence`, `source_partition`, `domain`

NL extraction produces entities like INSIGHT, TREND, CONCEPT, METHODOLOGY — complementing the structured extraction's data-derived entities. With domain context enabled, the LLM also uses correct domain-specific types (MARKET instead of LOCATION, MANUFACTURER instead of ORGANIZATION) for entities it recognizes from structured data, while freely discovering novel types for insights and trends.

---

## Expected Output Scale (Rheem)

```
┌─────────────────────┬─────────────────┐
│     Entity Type     │ Estimated Count │
├─────────────────────┼─────────────────┤
│ WORKFLOW            │ 1               │
├─────────────────────┼─────────────────┤
│ DOMAIN              │ 4               │
├─────────────────────┼─────────────────┤
│ SECTION             │ 47              │
├─────────────────────┼─────────────────┤
│ SKU                 │ ~7,200          │
├─────────────────────┼─────────────────┤
│ MANUFACTURER        │ ~5              │
├─────────────────────┼─────────────────┤
│ RETAILER            │ ~3              │
├─────────────────────┼─────────────────┤
│ MARKET              │ ~200+           │
├─────────────────────┼─────────────────┤
│ REGION              │ ~15-20          │
├─────────────────────┼─────────────────┤
│ PRICING_SCENARIO    │ 4               │
├─────────────────────┼─────────────────┤
│ SUPPLIER            │ ~6,000          │
├─────────────────────┼─────────────────┤
│ PLANT               │ ~233            │
├─────────────────────┼─────────────────┤
│ SPEND_CATEGORY      │ ~716            │
├─────────────────────┼─────────────────┤
│ STORE               │ ~3,000+         │
├─────────────────────┼─────────────────┤
│ DISTRIBUTION_CENTER │ ~50+            │
├─────────────────────┼─────────────────┤
│ MODEL               │ ~20+            │
├─────────────────────┼─────────────────┤
│ PRODUCT_TYPE        │ ~10+            │
├─────────────────────┼─────────────────┤
│ NL-extracted        │ ~200-500        │
├─────────────────────┼─────────────────┤
│ Total nodes         │ ~15,000-20,000  │
├─────────────────────┼─────────────────┤
│ Total relationships │ ~50,000-100,000 │
└─────────────────────┴─────────────────┘
```

---

## Schema Config Types Reference

```python
@dataclass
class ColumnEntityMapping:
    column_name: str                          # Column in the data
    entity_type: str                          # e.g., "SKU", "SUPPLIER"
    display_column: Optional[str] = None      # Column to use as label
    entity_attributes: Optional[List[str]] = None  # Per-entity attribute scoping

@dataclass
class RelationshipRule:
    source_entity_column: str
    target_entity_column: str
    relation_type: str                        # Predicate, e.g., "MANUFACTURED_BY"
    metric_columns: List[str] = field(default_factory=list)  # Numeric edge properties (enforced float)
    context_columns: List[str] = field(default_factory=list) # String edge properties (stored as-is)

@dataclass
class ImplicitEntityConfig:
    entity_type: str                          # e.g., "MANUFACTURER"
    entity_value: str                         # e.g., "Rheem"
    link_from_column: str                     # Which column's entities to link FROM
    relation_type: str                        # e.g., "MANUFACTURED_BY"
    entity_label: Optional[str] = None        # Display label (defaults to entity_value)
    reverse_direction: bool = False           # If True: implicit→link_from
    properties: Dict[str, Any] = field(default_factory=dict)

@dataclass
class AggregationStrategy:
    group_by_columns: List[str]
    sum_columns: List[str] = field(default_factory=list)
    count_column: Optional[str] = None
    avg_columns: List[str] = field(default_factory=list)

@dataclass
class PartitionSchemaConfig:
    entity_mappings: List[ColumnEntityMapping]
    relationship_rules: List[RelationshipRule] = field(default_factory=list)
    attribute_columns: List[str] = field(default_factory=list)
    aggregation: Optional[AggregationStrategy] = None
    implicit_entities: List[ImplicitEntityConfig] = field(default_factory=list)

@dataclass
class RelationshipAxiom:
    """OWL-like axioms for symbolic reasoning — NOT materialized, consumed at query time."""
    predicate: str
    inverse_of: Optional[str] = None   # A -[P]-> B implies B -[Q]-> A
    symmetric: bool = False             # A -[P]-> B implies B -[P]-> A
    transitive: bool = False            # A -[P]-> B and B -[P]-> C implies A -[P]-> C

@dataclass
class WorkflowKGSchemaConfig:
    workflow_name: str
    graph_namespace: str
    domains: List[DomainConfig]
    partition_schemas: Dict[str, PartitionSchemaConfig]
    # Advanced features (all opt-in)
    entity_resolution: EntityResolutionConfig
    community_detection: CommunityDetectionConfig
    cardinality_constraints: List[CardinalityConstraint]
    inferred_relationships: List[InferredRelationshipRule]
    relationship_axioms: List[RelationshipAxiom]          # NEW: OWL-like axioms for reasoning
```

---

## How to Add a New Client

1. Create `kg_models/specific_domain_configs/your_client.py`:

```python
from enum import Enum
from app.kr_engine.pipeline.knowledge_graph_builder.kg_models.schema_config import *

class YourClientEntityType(str, Enum):
    PRODUCT = "PRODUCT"
    CATEGORY = "CATEGORY"

class YourClientRelationType(str, Enum):
    BELONGS_TO = "BELONGS_TO"

YOUR_CONFIG = WorkflowKGSchemaConfig(
    workflow_name="your_client_workflow",
    graph_namespace="your_client_namespace",
    domains=[
        DomainConfig(name="Domain A", partition_names=["partition_1", "partition_2"]),
    ],
    partition_schemas={
        "partition_1": PartitionSchemaConfig(
            entity_mappings=[
                ColumnEntityMapping("column_a", YourClientEntityType.PRODUCT,
                                   entity_attributes=["price", "weight"]),
                ColumnEntityMapping("column_b", YourClientEntityType.CATEGORY,
                                   entity_attributes=[]),
            ],
            relationship_rules=[
                RelationshipRule("column_a", "column_b",
                                YourClientRelationType.BELONGS_TO,
                                metric_columns=["revenue"]),
            ],
            implicit_entities=[
                ImplicitEntityConfig(
                    entity_type="BRAND", entity_value="YourBrand",
                    link_from_column="column_a", relation_type="BRANDED_AS",
                ),
            ],
        ),
    },
)
```

2. Run:

```python
from app.kr_engine.pipeline.knowledge_graph_builder.run_workflow_to_knowledge_graph_pipeline import (
    run_workflow_kg_pipeline,
)
from app.kr_engine.pipeline.knowledge_graph_builder.kg_models.specific_domain_configs.your_client import YOUR_CONFIG

items = load_workflow_output_from_json(Path("your_data.json"))
await run_workflow_kg_pipeline(items, YOUR_CONFIG)
```

3. Done. Zero framework code changes.

---

## Neo4j Schema & Query Patterns

All entities are stored as `:SemanticaEntity` nodes with `(namespace, id)` uniqueness constraint. All relationships are `:SEMANTICA_REL` with `(namespace, predicate)`.

### Scoped queries via WORKFLOW root (recommended for Q&A retrieval)

```cypher
-- Start from WORKFLOW root → scoped to this workflow's subgraph
MATCH (w:SemanticaEntity {type: "WORKFLOW", workflow_name: $workflow_name, namespace: $ns})
      -[:SEMANTICA_REL {predicate: "HAS_DOMAIN"}]->(d:SemanticaEntity {type: "DOMAIN"})
      -[:SEMANTICA_REL {predicate: "BELONGS_TO_DOMAIN"}]<-(s:SemanticaEntity {type: "SECTION"})
      -[:SEMANTICA_REL {predicate: "CONTAINS_ENTITY"}]->(e:SemanticaEntity)
RETURN d.label AS domain, s.label AS section, e.type, count(e)
ORDER BY d.label, count(e) DESC

-- What's the optimal price for SKU X in JACKSONVILLE under Scenario 4?
MATCH (s:SemanticaEntity {type: "SKU", namespace: $ns})-[r:SEMANTICA_REL {predicate: "SOLD_IN_MARKET"}]->(m:SemanticaEntity {type: "MARKET", label: "JACKSONVILLE"}),
      (s)-[:SEMANTICA_REL {predicate: "PRICED_UNDER_SCENARIO"}]->(sc:SemanticaEntity {type: "PRICING_SCENARIO"})
WHERE s.label = "X"
  AND sc.label CONTAINS "Scenario 4"
  AND r.source_partition CONTAINS "scenario_4"
RETURN r.optimal_retail_price, r.expected_units, sc.company_gross_profit_dollar_diff

-- Which Rheem models have highest warranty impact at ONTARIO DC?
MATCH (m:SemanticaEntity {type: "MODEL", namespace: $ns})-[r:SEMANTICA_REL {predicate: "WARRANTY_IMPACT_AT"}]->(dc:SemanticaEntity {type: "DISTRIBUTION_CENTER", label: "ONTARIO"}),
      (m)-[:SEMANTICA_REL {predicate: "MANUFACTURED_BY"}]->(mfr:SemanticaEntity {type: "MANUFACTURER", label: "Rheem"})
RETURN m.label, r.impact, r.observed_claim_amount_by_sales
ORDER BY r.impact DESC

-- Which Home Depot stores have highest warranty claims in the LA market?
MATCH (st:SemanticaEntity {type: "STORE", namespace: $ns})-[r:SEMANTICA_REL {predicate: "IN_MARKET"}]->(mk:SemanticaEntity {type: "MARKET", label: "LA"}),
      (st)-[:SEMANTICA_REL {predicate: "SOLD_BY_RETAILER"}]->(ret:SemanticaEntity {type: "RETAILER", label: "Home Depot"})
RETURN st.label, r.claims_usd_total, r.ordered_sales_usd, r.TTM_YEAR
ORDER BY r.claims_usd_total DESC
```

### Direct property-based queries (simpler, still scoped)

```cypher
-- Query all entities in a namespace (flat filter)
MATCH (n:SemanticaEntity {namespace: $ns})
RETURN n.type, count(n) ORDER BY count(n) DESC

-- Find cross-partition joins
MATCH (s1:SemanticaEntity {type: "SECTION", namespace: $ns})
      -[r:SEMANTICA_REL {predicate: "JOINS_WITH"}]->(s2:SemanticaEntity {type: "SECTION"})
RETURN s1.label, s2.label, r.join_name, r.formula
```

### Why both approaches

- **WORKFLOW traversal**: Preferred for Q&A retrieval. Graph-native scoping — Neo4j follows edges from an indexed root node, never scans unrelated nodes. Guarantees isolation between workflows.
- **Property filter**: Simpler syntax for ad-hoc queries, validation, debugging. Every node carries `namespace` + `workflow_name` as a safety net.

---

## Batched Insertion

The `WorkflowKnowledgeGraphWriter` splits large artifacts into chunks to avoid Neo4j transaction timeouts:

- **Node batches**: 5,000 per transaction (configurable)
- **Relation batches**: 10,000 per transaction (configurable)
- Uses `GPKnowledgeGraph.insert_semantica_graph_v2()` which uses Cypher `UNWIND` for bulk operations
- Validation queries run after insertion to compare actual vs expected counts (10% tolerance)

---

## Key Dependencies

| Dependency | Used For |
|-----------|---------|
| `SemanticaEngine` | NL text extraction (NER, relation extraction, triplets) |
| `SemanticaGraphArtifact` | Universal output model (entities + relations) |
| `GPKnowledgeGraph` | Neo4j async driver with `insert_semantica_graph_v2()` + `execute_query()` |
| `KnowledgeGraphPipeline` | ABC base class providing `semantic_artifact`, `config`, `export_to_d3_json()` |
| `PipelineStructuredOutput` | Input model from workflow output |

---

## Running the Pipeline

```bash
# Full Rheem pipeline (requires Neo4j + OpenAI API)
python -m app.kr_engine.pipeline.knowledge_graph_builder.run_workflow_to_knowledge_graph_pipeline

# Or programmatically
import asyncio
from app.kr_engine.pipeline.knowledge_graph_builder.run_workflow_to_knowledge_graph_pipeline import (
    run_workflow_to_knowledge_graph_from_json,
)

asyncio.run(run_workflow_to_knowledge_graph_from_json(
    update_kg=True,      # Insert into Neo4j
    export_json=True,    # Export D3 JSON
))
```

---

## Maintenance Notes

- **Column name changes**: If a workflow's column headers change, update the corresponding `PartitionSchemaConfig` in the config file. The `_find_column()` function handles case/format normalization, but the semantic column name must still match.
- **New partitions**: If a workflow adds new partitions, add their names to the appropriate `DomainConfig.partition_names` and optionally add a `PartitionSchemaConfig` entry. Partitions without a schema config will auto-classify as NL_ONLY or SKIP.
- **Aggregation tuning**: If a fact table grows beyond expected size, adjust the `AggregationStrategy` to group at a coarser level.

---

## Production-Grade Review: Bugs Fixed & Improvements (March 2026)

All 5 framework files + the pipeline orchestrator were reviewed and hardened for production-grade quality. Key changes:

### Critical Bugs Fixed

| Bug | File | Impact | Fix |
|-----|------|--------|-----|
| **NL entities missing CONTAINS_ENTITY edges** | `nl_extractor.py` | SemanticaEngine does NOT set `source_partition` on entities. Assembler relied on this to build SECTION→CONTAINS_ENTITY→entity edges → **zero NL entity links** | Added `entity.properties.setdefault("source_partition", item.name)` after `build_graph_from_inputs` returns |
| **source_partition leading-comma bug** | `structured_extractor.py` | `f"{existing_src},{part}"` when `existing_src=""` → produces `",partition_b"` with leading comma | Extracted `_merge_source_partition()` with `if existing_src else part` guard |
| **Empty-text threshold drops valid NL partitions** | `nl_extractor.py` | `len(parts) <= 2` counted header parts as content → partitions with only `description` were dropped | Split into `header_parts` + `content_parts`, gate on `if not content_parts: return None` |
| **Cross-partition NL entity dedup loses source_partition** | `nl_extractor.py` | First-seen entity wins in `extract_all` → second partition's `source_partition` lost → missing CONTAINS_ENTITY edges | Added `_merge_nl_entity()` with `source_partition` union-merge |
| **Dead code: SemanticaSemanticStage** | `workflow_to_kg_pipeline.py` | Unused import and instantiation of `SemanticaSemanticStage` | Removed |

### Hardening Improvements

**`workflow_to_kg_pipeline.py`:**
- Added empty `workflow_output` guard: `if not workflow_output: raise ValueError(...)`
- Added batch failure awareness after insertion (checks `entity_batches_failed + rel_batches_failed`)
- Added validation result warning when `validation["valid"] is False`
- Added warning when `export_json=True` but `export_path=None`

**`input_loader.py`:**
- Added `_LOG_PREFIX = "[Workflow-To-KG Loader]"` consistent log prefix
- `load_workflow_output_from_json`: Added `FileNotFoundError`, `json.JSONDecodeError`, `isinstance(raw, list)` guards, per-item parse error with index + partition name
- `_classify_extraction_mode`: Added warning when `has_schema=True` but no data and no text → SKIP
- Added `_validate_schema_columns()` — collects every column referenced in entity_mappings, relationship_rules, attribute_columns, aggregation, diffs against actual data headers. Catches config-vs-data mismatches at classification time.
- Added `_extract_column_metadata` guard for `isinstance(raw_meta, dict)`
- Changed to `Counter` for mode_counts

**`knowledge_graph_assembler.py`:**
- Extracted `_build_contains_entity_relations()` method using `Dict[str, Set[str]]` for deduplication
- Pre-built `all_partition_names` set for join detection (avoid iterating all items per partition)
- Renamed `merge_artifacts` loop var from `artifact` to `art` to avoid parameter shadowing

**`nl_extractor.py`:**
- Added `ontology.metadata` nested dict handling: `elif isinstance(col_desc, dict) and "description" in col_desc`
- Added succeeded/failed/skipped counters in `extract_all`
- Added relation deduplication via `relation_set: Dict[tuple, Any]`

**`structured_extractor.py`:**
- Added `_NULL_SENTINELS = frozenset({"none", "null", "nan", ""})` and `_is_null_value()` helper
- `_aggregate_rows`: Added `_resolve()` helper that warns on missing columns, guard when zero group_by columns resolve
- `extract_all`: Changed from `all_relations: List` to `relation_set: Dict[tuple, SemanticaRelation]` with metric property merging
- Added `processed` and `skipped_no_schema` counters with warning log

**`knowledge_graph_writer.py`:**
- Replaced all hardcoded strings in Cypher queries with `StructuralEntityType` and `StructuralRelationType` enums
- Updated mode comparison to use `ExtractionMode.NL_ONLY.value` instead of `"nl_only"`
- All log messages use enum references

---

## Validation System (6-Check Comprehensive)

The `WorkflowKnowledgeGraphWriter.validate_insertion()` runs 6 post-insertion checks:

| Check | What It Does | Tolerance |
|-------|-------------|-----------|
| **1. Entity counts** | Query Neo4j entity counts by type, compare vs graph_stats | 90%-150% range |
| **2. Relation counts** | Query Neo4j relation counts by predicate, compare vs graph_stats | 90%-150% range |
| **3. Orphaned relations** | Find relations where one endpoint is in namespace, other isn't | 0 expected |
| **4. Dangling edges** | Check all entity IDs referenced in relations exist in Neo4j | 0 missing expected |
| **5. Structural integrity** | WORKFLOW root exists + unique, SECTION→CONTAINS_ENTITY coverage, flag empty STRUCTURED/HYBRID sections | Exactly 1 WORKFLOW root |
| **6. Entity reachability** | % of content entities reachable via WORKFLOW→DOMAIN→SECTION→CONTAINS_ENTITY traversal | **Hard fail < 50%**, warn < 70% |

**Verdict:** Valid only if ALL checks pass, none were skipped, AND `workflow_root_exists is True` (strict — `None` from query failure is rejected). Partial data (e.g., failed batches) triggers a warning before validation runs.

---

## State-of-the-Art Analysis: How We Compare

### Research Sources

Analysis based on: Neo4j LLM Knowledge Graph Builder, Microsoft GraphRAG (Leiden community detection, map-reduce summarization), LangChain GraphTransformers, LlamaIndex KnowledgeGraphIndex/PropertyGraphIndex, W3C R2RML/Direct Mapping standards, YARRRML tabular→KG mapping, and academic KG construction literature.

### What We Already Do Better Than Most

| Strength | Detail | Compared To |
|----------|--------|-------------|
| **Dual-path extraction** | Structured (deterministic) + NL (LLM-based) merged into unified graph | Neo4j/GraphRAG are LLM-only; R2RML is mapping-only |
| **Declarative schema config** | New client = 1 config file, zero framework changes | Neo4j uses prompt engineering; LangChain needs code changes |
| **Structural graph layer** | WORKFLOW→DOMAIN→SECTION→CONTAINS_ENTITY navigational backbone | Missing from GraphRAG, LangChain, LlamaIndex — unique to us |
| **Per-entity attribute scoping** | `entity_attributes` prevents metric leakage across entity types | No equivalent in any open-source KG builder |
| **6-check validation** | Entity counts, relation counts, orphans, dangling edges, structural integrity, reachability | Most frameworks have zero post-insertion validation |
| **Implicit entities** | Cross-partition linking via deterministic IDs + ImplicitEntityConfig | Manual in other frameworks; our mechanism is declarative |
| **Aggregation strategy** | Handles 1.6M-row fact tables by grouping before extraction | Most frameworks choke on large tables or require preprocessing |

### Key Techniques from State-of-the-Art (Gap Analysis → Implementation Status)

#### 1. Embedding-Based Entity Resolution (Neo4j KG Builder, GraphRAG) — ✅ IMPLEMENTED

**What it is:** After extraction, compute embeddings for entity labels within the same type. Entities with cosine similarity > threshold (e.g., 0.85) get merged even if names differ.

**Implementation:** `entity_resolver.py` — Phase 5 of pipeline. Uses `BAAI/bge-small-en-v1.5` (384d) via `get_pre_trained_embedding_model(task=FIELD_IN_DATASET_MATCHING)` (Cloud-Run compatible). Greedy BFS clustering with transitive cohesion validation + canonical selection + relation redirect + `merged_from` provenance tracking.

#### 2. Community Detection — Leiden/Louvain (Microsoft GraphRAG) — ✅ IMPLEMENTED

**What it is:** Hierarchical clustering of graph entities into communities at multiple resolution levels. Each community gets an LLM-generated summary.

**Implementation:** `community_detector.py` — Phase 8 of pipeline. Uses `networkx.algorithms.community.louvain_communities`. Multi-level hierarchical with COMMUNITY nodes, BELONGS_TO_COMMUNITY + HAS_SUB_COMMUNITY edges. Optional LLM summaries.

#### 3. Confidence Scoring (Microsoft GraphRAG) — ✅ IMPLEMENTED

**What it is:** Attach confidence scores to extracted entities and relationships. Structured = 1.0 (deterministic). NL-extracted = confidence from LLM.

**Implementation:** `structured_extractor.py` sets `confidence=1.0` on all entities/relations. `nl_extractor.py` propagates max mention-level confidence from SemanticaEngine (default 0.9). Inferred relations get `confidence=min(hop1, hop2)` — conservative propagation through the inference chain. Structural nodes (WORKFLOW, DOMAIN, SECTION, COMMUNITY) get `confidence=1.0`.

#### 4. Incremental Updates (Production KG Systems) — ✅ IMPLEMENTED

**What it is:** Only re-extract changed partitions instead of full rebuild. Compute content hashes per partition, store in SECTION node properties.

**Implementation:** `knowledge_graph_writer.py` added `compute_partition_hash()` (SHA256 of data+summary+tooltip+description), `get_existing_partition_hashes()`, `clear_partition_data()`. Pipeline stamps `content_hash` on SECTION nodes. Enabled via `run_kg_pipeline(incremental=True)`.

#### 5. Temporal Versioning — ✅ IMPLEMENTED

**What it is:** Track when facts were true (`valid_from`, `valid_to` on relationships). Essential for time-series data like pricing.

**Implementation:** `TemporalColumnConfig` dataclass on `RelationshipRule.temporal`. `structured_extractor.py` stamps `valid_from`/`valid_to`/`timestamp` on relation properties from declared columns.

#### 6. Graph Embeddings — Node2Vec / TransE — ⏳ DEFERRED

**What it is:** Compute vector representations of graph nodes based on their neighborhood structure. Enables similarity search without traversal.

**Status:** Not implemented. Requires research on Neo4j GDS integration, embedding storage, and reasoning agent retrieval pipeline integration.

#### 6b. Ontology-Driven NL Extraction (Domain-Aware SemanticaEngine) — ✅ IMPLEMENTED

**What it is:** Use the ontology schema and structured extraction results as **domain context** (not constraints) to make NL extraction smarter and more complete.

**The problem today:** NL extraction (Phase 3) runs blind — no awareness of entity types, relationship predicates, or entities already extracted from structured data. The LLM invents generic types (`CONCEPT`, `LOCATION`) and predicates (`RELATES_TO`) that don't align with the structured schema. Entities like `MARKET "JACKSONVILLE"` get extracted as `LOCATION "Jacksonville"` — different type, different ID, no merge.

**The design principle:** Ontology + structured results = **context to make the LLM smarter, NOT constraints that limit it.** The LLM should think: *"I know SKU, MARKET, MANUFACTURED_BY exist in this domain — I can use those types when I see them. But I also see a PRICING_TREND and a CAUSAL_RELATIONSHIP that nobody told me about — I should extract those too."* The hints make extraction more accurate without narrowing its scope.

**What this is NOT:**
- NOT "extract ONLY these entity types" (too restrictive — misses novel insights)
- NOT "don't extract what structured already has" (too conservative — misses nuanced relationships)
- NOT constraints or confirmation — purely reference knowledge to do a BETTER, MORE COMPLETE job

**Three levels of domain context (all as hints, never constraints):**

1. **Schema-guided prompting** (low effort, high impact): Pass `entity_types` and `relationship_types` from `WorkflowKGSchemaConfig` as hints to SemanticaEngine. The LLM knows the domain vocabulary but remains free to extract beyond it.

2. **Entity grounding** (medium effort, very high impact): After Phase 2 completes, pass known entity names from structured extraction as a reference vocabulary. When NL text mentions "Jacksonville market", the LLM can match it to the existing `MARKET "JACKSONVILLE"` entity (same ID → merges) while still extracting the novel insight about price elasticity in that market.

3. **Ontology metadata enrichment** (medium effort): Feed `ontology.metadata` column definitions alongside the text. The LLM understands that "weighted_average_elasticity" is a specific metric, not a generic concept — leading to more precise entity/relationship extraction.

**Architectural change required:** NL extraction must run AFTER structured extraction (currently parallel):
```
Current:  Phase 2 (structured) ─┐
                                 ├─→ Phase 4 (assemble)
          Phase 3 (NL)      ────┘

Improved: Phase 2 (structured) ──→ Phase 3 (NL, domain-aware) ──→ Phase 4 (assemble)
```

**Key implementation question:** Does SemanticaEngine's API support guided extraction (entity type hints, known entity vocabulary)? If not, we wrap it with a system prompt that includes the domain context. Requires careful prompt engineering and testing against real Rheem NL text to ensure hints help rather than constrain.

**Implementation:** `nl_extractor.py` — `build_domain_context()` assembles hints from schema config + structured artifact. `_prepend_domain_context()` injects them as text preamble. `_get_known_entities_for_domain()` filters entity names per domain + cross-domain shared. Pipeline builds context after Phase 2, passes to Phase 3. Toggled via `domain_context` parameter (None = off).

#### 7. SHACL-like Schema Validation — ✅ IMPLEMENTED

**What it is:** Graph topology constraints enforced before insertion. E.g., "every SKU MUST have exactly 1 MANUFACTURED_BY".

**Implementation:** `graph_schema_validator.py` — Phase 7 of pipeline. 4 checks: cardinality constraints, dangling relations, orphaned entities, property type validation. Configured via `WorkflowKGSchemaConfig.cardinality_constraints`.

#### 8. Multi-Hop Relationship Inference — ✅ IMPLEMENTED

**What it is:** Materialize transitive relationships. If SKU→MARKET and MARKET→REGION exist in different partitions, create SKU→REGION.

**Implementation:** `relationship_inferrer.py` — Phase 6 of pipeline. Builds typed adjacency index, finds 2-hop chains matching `InferredRelationshipRule` patterns, creates direct edges with `inferred=True` + `via_entity` provenance. Configured via `WorkflowKGSchemaConfig.inferred_relationships`.

---

## Prioritized Improvement Roadmap

| Priority | Feature | Effort | Impact | Status |
|----------|---------|--------|--------|--------|
| **P0** | Embedding-based entity resolution | Medium | Prevents duplicate entities across NL+structured | ✅ IMPLEMENTED — `entity_resolver.py` |
| **P0** | Incremental updates | Medium | Avoids full rebuild on every run | ✅ IMPLEMENTED — `knowledge_graph_writer.py` + pipeline |
| **P1** | Community detection (Louvain) | Medium | Enables global search / multi-resolution reasoning | ✅ IMPLEMENTED — `community_detector.py` |
| **P1** | Confidence scoring | Low | Better downstream reasoning quality | ✅ IMPLEMENTED — `structured_extractor.py` + `nl_extractor.py` |
| **P2** | Temporal versioning | Low | Critical for pricing/supply chain time-series | ✅ IMPLEMENTED — `structured_extractor.py` + `TemporalColumnConfig` |
| **P2** | Graph schema validation (SHACL-like) | Medium | Catches errors before Neo4j insertion | ✅ IMPLEMENTED — `graph_schema_validator.py` |
| **P3** | Graph embeddings (Node2Vec) | Medium | Enables hybrid KG+vector retrieval | ⏳ DEFERRED — requires further research |
| **P3** | Multi-hop relationship inference | Low | Materializes transitive relationships | ✅ IMPLEMENTED — `relationship_inferrer.py` |
| **P4** | Ontology-driven NL extraction | Medium | Domain-aware LLM extraction: better type alignment, entity grounding, novel insight capture | ✅ IMPLEMENTED — `nl_extractor.py` `build_domain_context()` + preamble injection |

---

## Known Limitations (Current — Updated March 2026)

1. ~~**No fuzzy entity resolution**~~ → ✅ RESOLVED: `entity_resolver.py` uses `BAAI/bge-small-en-v1.5` embeddings + cosine similarity clustering with transitive cohesion validation + `merged_from` provenance tracking.
2. ~~**No incremental updates**~~ → ✅ RESOLVED: SHA256 content hash per partition stored on SECTION nodes. On re-run, only changed partitions are re-extracted + cleared in Neo4j.
3. ~~**No temporal dimensions**~~ → ✅ RESOLVED: `TemporalColumnConfig` on `RelationshipRule` stamps `valid_from`/`valid_to`/`timestamp` on relation properties.
4. **Simple join detection** — only exact-match partition names in `data_set_join_logic` formula
5. **No hierarchical entity modeling** — spend categories with 4-level hierarchy flattened to single entity
6. ~~**No batch rollback**~~ → ✅ RESOLVED: Entity batch failure now **aborts relation insertion entirely** to prevent dangling edges. Returns `aborted_reason: "entity_batch_failure"` for audit.
7. **No NL extraction control** — can't hint entity types or relationship types to guide SemanticaEngine
8. **Sub-section not supported** — `PipelineStructuredOutput.sub_section` logged as warning but not processed
9. **No multi-valued attributes** — entity properties are scalar only (except `aliases` from entity resolution)
10. **OPERATING_UNIT cross-partition merge limitation** — different keys (`operating_unit_wid` vs `organization_id`) for same logical entity → separate nodes
11. **Graph embeddings (Node2Vec) deferred** — not yet implemented; requires research on integration with Neo4j + reasoning agent retrieval
- **Entity ID collisions**: The SHA1-based ID means two different entity types with the same name are distinct (type is part of the hash). But two entities of the SAME type with the SAME name (e.g., same SKU in different partitions) merge — which is the intended behavior.
- **New entity/relationship types**: Add to the client's enum classes (`RheemEntityType`, `RheemRelationType`). Framework structural types (`StructuralEntityType`, `StructuralRelationType`) should rarely change.
- **Implicit entity matching**: When using implicit entities for cross-partition linking, `entity_value` MUST exactly match the value in the other partition's data column (before normalization). Check with `make_entity_id()` to verify both produce the same ID.
- **Per-entity scoping**: Any multi-entity schema with `attribute_columns` should use `entity_attributes` on each `ColumnEntityMapping` to prevent property leaking. Single-entity schemas can safely use flat `attribute_columns`.
- **0-data partitions**: Many warranty partitions have 0 data rows (content in summary markdown). Their structured schemas are placeholders for future data. The NL extractor captures the summary text. This is by design.

---

## Advanced Features Implementation (March 2026)

All advanced features are **backward-compatible opt-ins** — every feature defaults to disabled. Existing pipeline behavior is 100% preserved. Features are enabled via fields on `WorkflowKGSchemaConfig`.

### New Model Types Added to `kg_models/schema_config.py`

```python
EntityResolutionConfig      # Controls embedding-based fuzzy dedup
CommunityDetectionConfig    # Controls Louvain community detection
TemporalColumnConfig        # Declares temporal columns on relationships
CardinalityConstraint       # Declares cardinality rules for validation
InferredRelationshipRule    # Declares multi-hop transitive patterns
```

New structural types in `kg_models/models.py`:
- `StructuralEntityType.COMMUNITY` — community cluster nodes
- `StructuralRelationType.BELONGS_TO_COMMUNITY` — entity → community membership
- `StructuralRelationType.HAS_SUB_COMMUNITY` — parent → child community hierarchy

### P0: Embedding-Based Entity Resolution (`entity_resolver.py`)

**What:** Groups entities by type, computes `BAAI/bge-small-en-v1.5` (384d) embeddings for labels, clusters by cosine similarity above threshold with transitive cohesion validation, merges each cluster into a single canonical entity.

**Key design decisions:**
- Uses `get_pre_trained_embedding_model(task=PretrainedTransformerTask.FIELD_IN_DATASET_MATCHING)` — `bge-small-en-v1.5` selected over `all-MiniLM-L6-v2` and `gte-small` via benchmark on real Rheem entity data (0.90 avg positive score, 0.64 avg negative, zero false positives at 0.85 threshold)
- Greedy single-linkage BFS clustering with **transitive cohesion validation**: after BFS expansion, members whose average similarity to the cluster falls below threshold are ejected into singletons — prevents A~B, B~C → A≁C over-merging
- Skips structural types (WORKFLOW, SECTION, DOMAIN, COMMUNITY)
- Canonical selection heuristic: highest confidence → most properties → longest label
- Merged entities get `aliases` property with all variant labels
- **`merged_from: List[str]`** on canonical entities tracks which entity IDs were merged — critical for reasoning provenance & audit trail
- All relations pointing to merged entities are redirected + deduplicated
- Self-loops created by merging are automatically removed
- Max confidence propagation: canonical gets `max(all merged confidences)`

**Config:** `WorkflowKGSchemaConfig.entity_resolution: EntityResolutionConfig`
- `enabled: bool = False` — must opt in
- `similarity_threshold: float = 0.85` — cosine similarity cutoff
- `embedding_model: str = "BAAI/bge-small-en-v1.5"` — embedding model
- `entity_types: List[str] = []` — empty = all content types
- `batch_size: int = 512` — embedding batch size

**Why bge-small over gte-small:** `gte-small` scored 0.87 for "ATLANTA" vs "CHICAGO" (false positive) and 0.84 for "Home Depot" vs "Lowe's". `bge-small-en-v1.5` correctly scored these as low similarity (0.64, 0.57). For short entity names where false positives are catastrophic, `bge-small` is the right choice.

### P0: Incremental Updates (`knowledge_graph_writer.py` + pipeline)

**What:** SHA256 content hash per partition. Stored as `content_hash` on SECTION nodes. On re-run, pipeline compares hashes → only re-extracts + re-inserts changed partitions.

**Key design decisions:**
- Hash computed from `data + summary + tooltip + description` (all content that affects extraction)
- `get_existing_partition_hashes()` queries Neo4j SECTION nodes for stored hashes
- `clear_partition_data()` deletes entities whose `source_partition` matches changed partitions (batched, 5K per batch)
- `insert_semantica_graph_v2()` already uses MERGE-based batch UNWIND — idempotent by design
- Pipeline stamps `content_hash` on SECTION entities after assembly (Phase 4)

**Config:** `run_kg_pipeline(incremental=True)` — no schema config needed

### P1: Community Detection (`community_detector.py`)

**What:** Builds a networkx graph from content entities + relations, runs Louvain community detection at multiple resolution levels, creates COMMUNITY nodes with BELONGS_TO_COMMUNITY edges.

**Key design decisions:**
- Uses `networkx.algorithms.community.louvain_communities` (not Leiden — no `igraph` dependency)
- Multi-level hierarchical: runs at `resolution * (level + 1)` for each level up to `max_levels`
- Sub-community → parent community linked via `HAS_SUB_COMMUNITY`
- COMMUNITY node properties: `level`, `community_index`, `member_count`, `entity_type_distribution`, `sample_members`
- Optional LLM summaries per community via `get_lite_llm_response_async`
- Excludes structural nodes from community graph

**Config:** `WorkflowKGSchemaConfig.community_detection: CommunityDetectionConfig`
- `enabled: bool = False`
- `algorithm: str = "louvain"`
- `resolution: float = 1.0`
- `max_levels: int = 2`
- `min_community_size: int = 3`
- `generate_summaries: bool = False`

### P1: Confidence Scoring (all workers)

**What:** All entities and relations carry a `confidence` property. Every creation point verified.

| Source | Confidence Value | Rationale |
|--------|-----------------|-----------|
| Structured entities/relations | `1.0` | Deterministic extraction |
| NL entities | `max(mention_confidences)` or `0.9` default | LLM uncertainty |
| NL relations | `0.9` default | LLM uncertainty |
| Inferred relations | `min(hop1, hop2)` | Conservative propagation |
| Structural nodes (WORKFLOW, DOMAIN, SECTION) | `1.0` | Framework-guaranteed |
| Community nodes + relations | `1.0` | Deterministic from topology |

### P2: Temporal Versioning (`structured_extractor.py` + `TemporalColumnConfig`)

**What:** Schema config declares which columns carry temporal semantics. The structured extractor stamps `valid_from`, `valid_to`, and/or `timestamp` on relationship properties.

**Config:** `RelationshipRule.temporal: Optional[TemporalColumnConfig]`
```python
TemporalColumnConfig(
    timestamp_column="period",
    valid_from_column="effective_date",
    valid_to_column="expiry_date",
)
```

### P2: Graph Schema Validation (`graph_schema_validator.py`)

**What:** SHACL-like pre-insertion validation. Runs BEFORE Neo4j insertion (Phase 7). Catches structural errors early.

**4 checks:**
1. **Cardinality constraints** — e.g., "every SKU must have exactly 1 MANUFACTURED_BY outgoing"
2. **Dangling relations** — relations referencing entity IDs that don't exist (hard error)
3. **Orphaned entities** — content entities with zero relationships (warning)
4. **Property type validation** — confidence must be numeric (warning)

**Config:** `WorkflowKGSchemaConfig.cardinality_constraints: List[CardinalityConstraint]`
```python
CardinalityConstraint(
    entity_type="SKU",
    relation_type="MANUFACTURED_BY",
    direction=CardinalityDirection.OUTGOING,  # enum, not string
    min_count=1,
    max_count=1,
)
```

All violation types use `ViolationType` enum (CARDINALITY_UNDER, CARDINALITY_OVER, DANGLING_SUBJECT, DANGLING_OBJECT, ORPHANED_ENTITY, INVALID_CONFIDENCE).

### P3: Multi-Hop Relationship Inference (`relationship_inferrer.py`)

**What:** Materializes transitive 2-hop relationships. E.g., if SKU→MARKET via IN_MARKET and MARKET→REGION via IN_REGION exist, creates SKU→REGION via IN_REGION.

**Key design decisions:**
- Builds typed adjacency index `{(source_id, predicate) → set(target_ids)}` + relation property index for O(1) lookups
- Verifies entity types at each hop (not just following any edge)
- **Self-loop guard**: skips if source == target (prevents A→B→A inferring A→A)
- Inferred relations carry `extraction_type=INFERRED`, `inferred=True`, `via_entity`, `via_predicate`, `target_predicate`
- **Confidence propagation**: `min(hop1_confidence, hop2_confidence)` — conservative, not hardcoded
- Deduplicates against existing relations
- Returns a NEW artifact (merged by assembler in Phase 6)

**Config:** `WorkflowKGSchemaConfig.inferred_relationships: List[InferredRelationshipRule]`
```python
InferredRelationshipRule(
    source_type="SKU",
    intermediate_type="MARKET",
    target_type="REGION",
    via_predicate="IN_MARKET",
    target_predicate="IN_REGION",
    inferred_predicate="IN_REGION",
)
```

### P3: Graph Embeddings (Node2Vec) — DEFERRED

Not implemented. Requires further research on:
- Integration with Neo4j GDS library vs standalone Node2Vec
- Storage of node embeddings (Neo4j property vs vector index)
- Integration with reasoning agent retrieval pipeline

### Files Modified for Advanced Features

| File | Changes |
|------|---------|
| `kg_models/models.py` | Added `COMMUNITY` entity type, `BELONGS_TO_COMMUNITY` + `HAS_SUB_COMMUNITY` relation types |
| `kg_models/schema_config.py` | Added 5 new dataclasses + fields on `WorkflowKGSchemaConfig` |
| `kg_models/__init__.py` | Updated exports for all new types |
| `workers/structured_extractor.py` | Added `confidence: 1.0` on all entities/relations + temporal versioning block |
| `workers/nl_extractor.py` | Added confidence propagation from SemanticaEngine mentions |
| `workers/knowledge_graph_writer.py` | Added `compute_partition_hash()`, `get_existing_partition_hashes()`, `clear_partition_data()` |
| `pipeline/workflow_to_kg_pipeline.py` | Rewritten as 11-phase pipeline, imports + wires all 4 new workers |

### 11-Phase Pipeline (`workflow_to_kg_pipeline.py`)

```
Phase  1: Classify partitions + incremental change detection
Phase  2: Structured extraction (schema-driven, deterministic)
       ↓  build_domain_context() — entity types, relation types, domain-scoped + cross-domain known entities
Phase  3: NL extraction (SemanticaEngine, domain-aware via ontology + structured hints)
Phase  4: Assemble (merge + structural layer + content hash stamping)
Phase  5: Entity resolution (P0 — embedding-based fuzzy dedup)
Phase  6: Relationship inference (P3 — multi-hop transitive)
Phase  7: Graph schema validation (P2 — SHACL-like pre-insertion)
Phase  8: Community detection (P1 — Louvain clustering)
       ↓  compute_graph_stats() — recomputed AFTER all transformations for accurate validation
Phase  9: Insert into Neo4j (batched, namespaced, incremental-aware)
Phase 10: Post-insertion validation (6-check comprehensive)
Phase 11: Export D3 JSON (optional)
```

All advanced phases (5–8) are opt-in. If disabled, they are skipped with zero overhead. Domain-aware NL extraction (Phase 3) is also optional — `domain_context=None` reverts to original behavior.

---

## Critical Design Learnings & Pitfalls

Hard-won lessons from building the Rheem schema config. These apply to any future client config.

### 1. `_find_column()` Ambiguity with Prefixed Column Names

**Bug**: Competitor sentiment data in Pricing CI. Both base and competitor SKU mappings used the same `entity_attributes` list: `["Rating", "Cost Pricing Sentiment", ...]`. The `_find_column("Rating")` function matched the base "Rating" column — NOT "Competitor Rating". Result: competitor SKU entities got the BASE product's sentiment scores.

**Fix**: Split into two separate attribute lists with explicit prefixes:
```python
_BASE_SENTIMENT_SKU_ATTRS = ["Rating", "Cost Pricing Sentiment", ...]
_COMPETITOR_SENTIMENT_SKU_ATTRS = ["Competitor Rating", "Competitor Cost Pricing Sentiment", ...]
```

**Rule**: When a partition has parallel column sets (base vs competitor, current vs previous, actual vs forecast), ALWAYS use separate `entity_attributes` lists with the full column name including the prefix. Never rely on `_find_column()` to disambiguate short names that match multiple columns.

### 2. Transaction-Level Fields Must NOT Be Entity Attributes

**Bug**: `spend_type` ("Direct"/"Indirect") and `rheem_global_bu` were set as `entity_attributes` on the SUPPLIER entity mapping in `edp_spend_f_curated`. But a single supplier can have BOTH Direct and Indirect spend across different transactions. The entity would capture whichever value appeared first, producing a misleading attribute.

**Fix**: Removed from `entity_attributes` (set to `[]`). Added `spend_type` to `aggregation.group_by_columns` instead, so Direct and Indirect spend create separate aggregation groups with separate edge metrics.

**Rule**: Before adding a column to `entity_attributes`, ask: "Can this value differ across rows for the same entity?" If yes → it's a transaction/fact-level field, not an entity attribute. Either:
- Add it to `group_by_columns` (creates separate aggregation groups)
- Add it to `metric_columns` on a relationship (places it on the edge)
- Exclude it (if it's not semantically useful at the aggregate level)

### 3. Cross-Partition Entity Merging Requires Identical Key Values

Deterministic ID merging via `make_entity_id(entity_type, normalized_value)` is powerful but fragile. It only works when both partitions use the **exact same value** for the entity.

**Works**: "Rheem" in Pricing CI column = "Rheem" in implicit entity value → same hash → merge.
**Works**: "Home Depot" as implicit entity across 13 partitions → all merge into one node.
**Fails**: `operating_unit_wid` in org_location vs `operating_unit_id` in spend → different values for same logical entity → no merge.

**Rule**: When designing cross-partition merging, verify the actual values match. Use `make_entity_id()` to test before deploying. If keys differ, you need either:
- A normalization layer (not yet in the framework)
- A lookup/join table step (not yet in the framework)
- To document the limitation and accept separate nodes

### 4. Aggregation Column Name Convention

When `AggregationStrategy` uses `count_column`, the aggregated output creates a new column named `{count_column}_count`. This auto-generated name must match whatever references it (e.g., `metric_columns` on a relationship rule).

**Example**: `count_column="segment1"` → generates column `"segment1_count"` → `metric_columns=["full_lead_time", "segment1_count"]` on the SUPPLIES_TO edge.

**Rule**: Always name `metric_columns` using the `{original}_count` convention, not the original column name.

### 5. Implicit Entity `entity_value` Must Be Exact

The `entity_value` on `ImplicitEntityConfig` must **exactly match** (before normalization) the value in the data or in the other partition that you're trying to merge with.

**Example**: Pricing scenario summary partition has column value `"Scenario 1: Flat 10%"`. The detail partition's implicit entity MUST use `entity_value="Scenario 1: Flat 10%"` — not `"Scenario 1"` or `"Flat 10%"`.

### 6. Per-Entity Scoping: Default `None` vs Explicit `[]`

- `entity_attributes=None` → **falls back** to partition-level `attribute_columns` (all partition attributes go on this entity)
- `entity_attributes=[]` → **explicitly empty** — NO attributes go on this entity

For multi-entity partitions, **always set `entity_attributes` explicitly on every mapping**. Using `None` on any mapping in a multi-entity partition risks attribute leaking. The safe pattern:

```python
# Good: explicit scoping on every mapping
ColumnEntityMapping("sku", "SKU", entity_attributes=["price", "cogs"])
ColumnEntityMapping("market", "MARKET", entity_attributes=[])

# Bad: None defaults to partition-level attributes, leaking price/cogs onto MARKET
ColumnEntityMapping("sku", "SKU", entity_attributes=["price", "cogs"])
ColumnEntityMapping("market", "MARKET")  # entity_attributes=None → gets ALL attribute_columns
```

### 7. Ontology Column Names vs Actual Data Column Names

Ontology metadata (`item.ontology.metadata`) often uses snake_case keys (`"base_sku"`) while actual data headers (`item.data[0]`) use Title Case (`"Base SKU"`). The `_find_column()` normalizer handles this.

But sometimes they genuinely differ: the ontology says `item_number` but the actual CSV has `item_id`. Always verify against the real data, not just the ontology definition.

### 8. Field Coverage Verification Method

For each partition, classify every column into exactly one bucket:
1. `entity_mapping.column_name` or `entity_mapping.display_column`
2. `entity_mapping.entity_attributes`
3. `relationship_rule.metric_columns`
4. `aggregation.group_by_columns / sum_columns / avg_columns / count_column`
5. Partition-level `attribute_columns`
6. **NOT CAPTURED** — must justify (internal ID, lost in aggregation by design, etc.)

Any column in bucket 6 without justification is a gap. Run this audit for every partition before deploying a new client config.

### 9. Rheem Composition Final Coverage Stats

| Domain | Data Partitions | Columns | Captured | Legitimately Excluded |
|--------|:-:|:-:|:-:|:-:|
| Pricing CI | 6 | 122 | 122 | 0 |
| Pricing Simulator | 5 | ~120 | ~120 | 0 |
| Warranty Intelligence | 5 | ~85 | ~85 | 0 |
| Supply Chain | 6 | 54 | 42 | 12 (internal IDs + aggregation-lost) |
| **Total** | **22** | **~381** | **~369** | **~12** |

21/21 entity types used. 20/20 relationship types used (including COMPETES_WITH added for Pricing CI). 30 partition schemas defined across 4 domains.

---

## Production Hardening & Symbolic AI Readiness (March 2026)

Comprehensive review and hardening pass to make the KG Builder serve as the foundation for Neural-Symbolic AI, Symbolic AI, and LLM reasoning over the knowledge graph.

### Mandatory Property Contract

**Every entity and relation in Neo4j carries these 4 properties — no exceptions:**

| Property | Structured | NL | Inferred | Structural |
|----------|-----------|-----|----------|------------|
| `namespace` | `self.namespace` | param | param | `namespace` |
| `extraction_type` | `STRUCTURED` | `NL` | `INFERRED` | `STRUCTURAL` |
| `confidence` | `1.0` | `max(mentions)` / `0.9` | `min(hop1, hop2)` | `1.0` |
| `source_partition` | comma-delimited | per-partition | via `via_entity` | N/A (structural) |

**17 creation points across 5 workers — all verified.** All use `ExtractionType` enum, never raw strings.

### Safety Guardrails

| Guardrail | What It Prevents |
|-----------|-----------------|
| Entity batch failure → abort relations | Dangling edges (relations referencing missing entities) |
| Reachability hard fail < 50% | Disconnected subgraphs unreachable by reasoning engine |
| `workflow_root_exists is True` (strict) | `None` from query failure no longer passes validation |
| `CONTAINS` exact match (split+trim) | "market" no longer deletes "market_analysis" entities |
| Self-loop guard in inferrer | Prevents meaningless A→A reflexive edges |
| Transitive cluster cohesion validation | Prevents over-merging in entity resolution |
| Hierarchy edge deduplication | One HAS_SUB_COMMUNITY edge per pair, not per entity |
| Graph stats recomputed after all phases | Validation uses accurate counts, not stale Phase 4 stats |
| `merge_artifacts` preserves `raw` dict | Stats, pre-insertion validation survive through merges |
| `_parse_numeric()` for metric columns | Handles `%`→decimal, `$`/`,` stripping, with DEBUG logging for transformations |
| `context_columns` for string edge props | `TTM_YEAR="FY2025"` stored as-is, not dropped as non-numeric |
| Domain-aware NL extraction (optional) | Domain context as hints — can be toggled off with `domain_context=None` |

### Reasoning Metadata

| Metadata | Where | Purpose |
|----------|-------|---------|
| `extraction_type` | Every entity + relation | Source-aware confidence weighting, provenance queries |
| `merged_from: [ids]` | Canonical entities after resolution | Audit trail: "which entities were merged?" |
| `via_entity`, `via_predicate` | Inferred relations | Explainability: "why was this relation inferred?" |
| `inferred: true` | Inferred relations | Quick filter for derived vs observed facts |
| `aliases: [labels]` | Resolved entities | All name variants for the canonical entity |
| `RelationshipAxiom` | Schema config | OWL-like axioms (inverse_of, symmetric, transitive) for query-time expansion |
| `content_hash` | SECTION nodes | Incremental update detection |

### Reasoning Engine Query Patterns

```cypher
-- Neural-Symbolic: confidence-weighted path scoring
MATCH p = (s)-[*]->(t)
WHERE all(r IN relationships(p) WHERE r.confidence >= 0.8)

-- Symbolic: provenance-filtered reasoning (only deterministic facts)
MATCH (n {extraction_type: "structured"})

-- LLM/GraphRAG: community-level summarization
MATCH (c {type: "COMMUNITY"})-[:SEMANTICA_REL {predicate: "HAS_SUB_COMMUNITY"}]->(sub)
RETURN c.summary

-- Audit: entity resolution trace
MATCH (n) WHERE n.merged_from IS NOT NULL
RETURN n.label, n.merged_from

-- Explainability: why was this edge inferred?
MATCH ()-[r {extraction_type: "inferred"}]->()
RETURN r.via_entity, r.via_predicate, r.confidence

-- Full traversal from root to any content entity
MATCH (w {type: "WORKFLOW"})-[:SEMANTICA_REL {predicate: "HAS_DOMAIN"}]->(d)
      -[:SEMANTICA_REL {predicate: "BELONGS_TO_DOMAIN"}]<-(s)
      -[:SEMANTICA_REL {predicate: "CONTAINS_ENTITY"}]->(e)
RETURN d.label, s.label, e.type, e.extraction_type, e.confidence
```

### Development Utilities

```bash
# Run the pipeline
python -m app.kr_engine.pipeline.knowledge_graph_builder.run_workflow_to_knowledge_graph_pipeline

# Clear the KG (for development iteration)
python -m app.kr_engine.pipeline.knowledge_graph_builder.run_workflow_to_knowledge_graph_pipeline --clear

# Dry run — count nodes without deleting
python -m app.kr_engine.pipeline.knowledge_graph_builder.run_workflow_to_knowledge_graph_pipeline --clear --dry-run
```

```python
# Programmatic clear
from app.kr_engine.pipeline.knowledge_graph_builder.run_workflow_to_knowledge_graph_pipeline import (
    clear_knowledge_graph_for_workflow,
)
await clear_knowledge_graph_for_workflow("rheem_composition", dry_run=True)
# → {"namespace": "rheem_composition", "nodes_found": 18472}

await clear_knowledge_graph_for_workflow("rheem_composition")
# → {"namespace": "rheem_composition", "nodes_deleted": 18472}
```

### Known Architectural Limitation: Incremental + Entity Resolution

**Incremental mode (`incremental=True`) is incompatible with entity resolution.** When only changed partitions are re-extracted, canonical entities spanning multiple partitions may lose data from unchanged partitions during partition-scoped clearing. Use `incremental=False` (full rebuild) when `entity_resolution.enabled=True`. This is the expected usage — incremental is for rapid development iteration without fuzzy dedup.

### Architectural Fixes (This Session)

| Fix | Impact |
|-----|--------|
| **Stale graph_stats** — recomputed after Phases 5-8, before insertion | Validation count comparisons now use accurate numbers |
| **merge_artifacts discards raw** — now preserves raw from first artifact | graph_stats, pre_insertion_validation survive through merge chains |
| **CONTAINS operator** in clear_partition_data → split+trim exact match | "market" no longer deletes "market_analysis" entities |
| **`is not False` → `is True`** in validation verdict | `None` from query failure no longer passes as valid |
| **Entity batch abort** — relation insertion skipped on any entity batch failure | Prevents dangling edges in Neo4j |
| **Metric columns** — non-numeric values logged + skipped, not silently stored as strings | Reasoning engine gets clean numeric properties |
| **`_parse_numeric()`** — handles `%`→decimal, `$`/`,` stripping | `"65%"` → `0.65`, `"$1,234"` → `1234.0`, with transformation logging |
| **`context_columns`** on RelationshipRule | String edge properties like `TTM_YEAR="FY2025"` stored as-is, not forced to float |
| **Ontology-driven NL extraction** — domain context as hints, not constraints | LLM sees entity types, relation types, known entity names from structured data |
| **Sequential Phase 2→3** — NL runs after structured | Domain context from structured results available to guide NL extraction |
| **Domain-scoped + cross-domain entity names** | Per-partition context is focused; shared entities (MANUFACTURER, RETAILER) always included |
| **Aggregation uses `_parse_numeric()`** — sum/avg columns handle `%`, `$`, `,` | Non-numeric values in aggregation logged with count of skipped rows |

### Status: Ready for Downstream

The Knowledge Graph Builder is **production-grade and architecturally verified** as the foundation for:
- **Symbolic AI Reasoning** (Datalog, rule-based inference over typed entities + relationship axioms)
- **Neural-Symbolic Reasoning** (confidence-weighted graph traversal + LLM augmentation)
- **LLM Reasoning** (GraphRAG via community summaries + structural traversal roots + domain-aware NL extraction)
- **Explainable AI** (full provenance chain from extraction → resolution → inference → insertion)
