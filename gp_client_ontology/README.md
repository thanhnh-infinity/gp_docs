# OSI Semantic Interchange Engine — Summary

## 1. GP Base Ontology

Located at `app/kr/kr_ontology/gp_ontology/v0.1/` — a rich RDF/OWL foundational ontology:

- ~3,300+ classes (companies, brands, products, customers, industries, schemas)
- ~400+ properties with clear hierarchies (ownership, structural, classification relations)
- Extension patterns: subclass hierarchies, property inheritance, OWL restrictions
- Namespace: `https://reasoning-engine.growthprotocol.ai/ontology/`
- **Gateway pattern**: `gp_ontology_gateway.rdf` transitively imports all modules (artifact, schema, operation)

Key modules used by OSI Engine:
- **gp_artifact_ontology** — core class hierarchy (`gp_artifact` → `gp_information_artifact`, `gp_concept`, `gp_entity`, etc.)
- **gp_schema_ontology** — 27+ specialized field types (`gp_price_field`, `gp_name_field`, `gp_location_field`, etc.)

## 2. Existing Ontology Engine

At `app/kr_engine/knowledge_ontology/` — built on owlready2/rdflib:
- `AbstractOntology` → `OwlRdfOntology` (main implementation)
- SPARQL support, OWL reasoning, basic Pydantic generation
- `OWLRDFOntologySuiteManager` for multi-ontology orchestration

## 3. OSI (Open Semantic Interchange)

A YAML-based specification for semantic models (https://open-semantic-interchange.org/):
- **Datasets**: Logical tables with source, primary_key, unique_keys, fields
- **Fields**: Column definitions with multi-dialect SQL expressions, data types, AI context
- **Entities**: Business entities backed by datasets
- **Dimensions**: Time and categorical dimensions with hierarchies and granularity
- **Measures**: Aggregatable numeric values (sum, avg, count, etc.)
- **Metrics**: Calculated business metrics (simple, derived, ratio, cumulative, conversion)
- **Relationships**: Foreign key mappings between datasets with join type and cardinality
- **Custom Extensions**: Vendor-specific metadata (dbt, Snowflake, Salesforce, etc.)
- **AI Context**: Instructions, synonyms, and examples for neural model integration

## 4. MetricFlow/dbt Semantic Layer

Reference implementation with entity types (primary, unique, foreign, natural), dimensions, measures. Used as design reference for OSI models.

---

## Architecture: OSI Semantic Interchange Engine (As Built)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OSI Semantic Interchange Engine                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Input:                                                                     │
│  ┌──────────────────┐    ┌───────────────────────┐                          │
│  │ Client OSI       │    │  GP Base Ontology     │                          │
│  │ YAML File        │    │  (RDF/OWL, READ-ONLY) │                          │
│  │                  │    │  via Gateway import   │                          │
│  └────────┬─────────┘    └─────────────┬─────────┘                          │
│           │                            │                                    │
│           ▼                            │                                    │
│  ┌────────────────────────────────┐    │                                    │
│  │  1. OSI Parser & Validator     │    │                                    │
│  │  - Parse YAML → Pydantic       │    │                                    │
│  │  - Validate structure + refs   │    │                                    │
│  │  - Handle multi-dialect expr   │    │                                    │
│  └────────────┬───────────────────┘    │                                    │
│               │                        │                                    │
│               ▼                        │                                    │
│  ┌──────────────────────────────────┐  │                                    │
│  │  2. Ontology Mapper              │  │                                    │
│  │  - Map OSI → GP Base classes     │◄─┘                                    │
│  │  - Create class hierarchy        │                                       │
│  │  - Create individuals            │                                       │
│  │  - Wire object/data properties   │                                       │
│  │  - Apply semantic layers:        │                                       │
│  │    • gp_semantic_link (rdf:type) │                                       │
│  │    • gp_data_type (has_data_type)│                                       │
│  │    • ai_context (osi_ai_context)│                                        │
│  └────────────┬───────────────────┘                                         │
│               │                                                             │
│    ┌──────────┼──────────┐                                                  │
│    ▼          ▼          ▼                                                  │
│  ┌────────┐ ┌─────────┐ ┌─────────┐                                         │
│  │ 3a.    │ │ 3b.     │ │ 3c.     │                                         │
│  │ RDF/OWL│ │ Pydantic│ │ SQL     │                                         │
│  │ Gen    │ │ Gen     │ │ Gen     │                                         │
│  └───┬────┘ └────┬────┘ └────┬────┘                                         │
│      │           │           │                                              │
│      ▼           ▼           ▼                                              │
│  Output:                                                                    │
│  ┌──────────────────┐ ┌──────────────┐ ┌──────────────┐                     │
│  │ client_ontology  │ │ client_      │ │ SQL queries  │                     │
│  │ .rdf (+ catalog) │ │ models.py    │ │ (multi-      │                     │
│  │                  │ │ (Pydantic)   │ │  dialect)    │                     │
│  └──────────────────┘ └──────────────┘ └──────────────┘                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Class Mapping: OSI → GP Base Ontology

| OSI Type | OSI Class (Created) | GP Superclass (Extends) | Description |
|----------|-------------------|------------------------|-------------|
| `SemanticModel` | `osi_semantic_model` | `gp_schema_design` | Top-level semantic model container |
| `Dataset` | `osi_dataset` | `gp_dataset` | Dataset representing a table or view |
| `Field` | `osi_field` | `gp_field` | Field definition (+ additional `rdf:type` from `gp_semantic_link`) |
| `Entity` | `osi_entity` | `gp_business_entity` | Business entity backed by a dataset |
| `TimeDimension` | `osi_time_dimension` | `gp_time_entity` | Time-based dimension |
| `CategoricalDimension` | `osi_categorical_dimension` | `gp_taxonomy` | Categorical dimension |
| `Measure` | `osi_measure` | `gp_measure` | Aggregatable numeric measure |
| `Metric` | `osi_metric` | `gp_metric` | Calculated business metric |
| `Relationship` | `osi_relationship` | `gp_relationship` | Relationship between datasets |
| `AIContext` | `osi_ai_context` | `gp_information_artifact` | Structured AI/neural context metadata |

## Three Semantic Layers on Fields

Every field individual can have up to three layers of semantic enrichment:

```
field_store_sales_ss_sales_price
  ├── rdf:type osi_field                           ← structural type (always)
  ├── rdf:type gp_price_field                      ← gp_semantic_link (what it MEANS)
  ├── has_data_type gp_data_type_decimal            ← gp_data_type (what type it IS)
  └── has_ai_context ai_context_field_..._ss_sales  ← ai_context (how to INTERPRET it)
            ├── has_ai_synonym "unit price"
            └── has_ai_synonym "price"
```

| Layer | YAML Field | Mechanism | Purpose |
|-------|-----------|-----------|---------|
| Semantic type | `gp_semantic_link` | Additional `rdf:type` assertion | Field IS-A GP Base class (e.g., `gp_price_field`) |
| Data type | `gp_data_type` | `has_data_type` object property | Field HAS data type (e.g., `integer`, `decimal`). Optional — not asserted if unspecified |
| AI context | `ai_context` | `has_ai_context` → `osi_ai_context` individual | Instructions, synonyms, examples for neural reasoning |

## Downstream Consumers

The Client Ontology is the **central knowledge artifact** that feeds all reasoning layers:

```
                              Client Ontology (RDF/OWL)
                              imports GP Base Ontology
                                        |
          +----------------+------------+------------+----------------+
          |                |            |            |                |
    Operational       Symbolic      Neural      Knowledge       Backbone
       Layer            Layer        Layer        Graph           Layer
          |                |            |            |                |
   Pydantic Models    ASP Facts    Neural       Logical          All Reasoning
   (data validation,  (Answer Set  Context      Representation     Steps
    serialization,     Programming, (ai_context, (RDF graph,
    workflow use)       logic rules, synonyms,    SPARQL queries,
                        constraint   gp_semantic  class hierarchy,
                        solving)     _link        entity links)
                                     → field
                                     meaning)
```

| Consumer | What it uses | Purpose |
|----------|-------------|---------|
| **Pydantic Models** | Datasets, fields, data types, structure | Data validation and workflow integration |
| **ASP Facts** | Individuals, class hierarchy, properties, relationships | Symbolic reasoning — logic rules, constraint checking, inference |
| **Neural Context** | `ai_context`, `gp_semantic_link`, class hierarchy | Understanding what fields *mean* for any neural model (LLMs, embeddings, etc.) |
| **Knowledge Graph** | Full RDF graph — individuals, classes, properties, triples | Logical representation — SPARQL queries, graph traversal, class inference |
| **Reasoning Engine** | All of the above combined | Hybrid reasoning — symbolic grounds neural, knowledge graph provides structure |

## Design Philosophy

The engine is a **mechanical mapper** — it maps what the OSI YAML explicitly declares and stays silent when it doesn't know. Two external conditions must be met:

1. **OSI YAML provides the semantic annotation** (e.g., `gp_semantic_link: gp_price_field`)
2. **GP Base Ontology has the target artifact defined** (e.g., `gp_price_field` class exists in `gp_schema_ontology`)

Both are knowledge tasks, not engine problems. The engine never guesses or defaults.

## Implementation Status

| Component | Status | Location |
|-----------|--------|----------|
| OSI Pydantic Models | Done | `app/kr_engine/osi_engine/osi_models/` |
| OSI Parser | Done | `app/kr_engine/osi_engine/parser/yaml_parser.py` |
| Model Validator | Done | `app/kr_engine/osi_engine/validator/` |
| Ontology Mapper | Done | `app/kr_engine/osi_engine/mapper/ontology_mapper.py` |
| Mapping Rules | Done | `app/kr_engine/osi_engine/mapper/mapping_rules.py` |
| RDF/OWL Generator | Done | `app/kr_engine/osi_engine/generator/rdf_generator.py` |
| Pydantic Generator | Done | `app/kr_engine/osi_engine/generator/pydantic_generator.py` |
| SQL Generator | Done | `app/kr_engine/osi_engine/generator/sql_generator.py` |
| `gp_semantic_link` (field semantic typing) | Done | Additional `rdf:type` from GP Base schema classes |
| `gp_data_type` (field data typing) | Done | Optional `has_data_type` object property |
| `osi_ai_context` (structured AI context) | Done | `has_ai_context` → `osi_ai_context` class with 3 properties |
| Gateway import pattern | Done | Client ontology imports GP Base via `gp_ontology_gateway.rdf` |

## Key Files

```
app/kr_engine/osi_engine/
├── osi_models/           # Pydantic models for OSI spec
│   ├── core.py           # SemanticModel, Dataset, Field, Expression, AIContext
│   ├── entities.py       # Entity, EntityKey
│   ├── dimensions.py     # Dimension, DimensionHierarchy
│   ├── measures.py       # Measure, PercentileConfig
│   ├── metrics.py        # Metric, MetricFilter, ConversionStep
│   ├── relationships.py  # Relationship, JoinCondition
│   └── enums.py          # All enumerations
├── parser/
│   └── yaml_parser.py    # YAML → Pydantic models
├── validator/            # Structural + semantic validation
├── mapper/
│   ├── mapping_rules.py  # OSI ↔ GP Base mapping definitions
│   └── ontology_mapper.py # Semantic model → MappedOntology
├── generator/
│   ├── rdf_generator.py      # MappedOntology → RDF/OWL file
│   ├── pydantic_generator.py # SemanticModel → Python classes
│   └── sql_generator.py      # SemanticModel → SQL queries
├── example/
│   ├── tpcds_semantic_model.yaml  # TPC-DS example (5 datasets, 32 fields)
│   └── run_osi_engine.py          # End-to-end runner
└── docs/
    └── learning.md        # OSI → GP Base mapping reference
```
