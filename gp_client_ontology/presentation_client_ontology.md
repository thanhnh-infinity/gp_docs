# Client Ontology Engine — Presentation

## For: Leadership / Executive Audience (COO, CPO, CEO)

---

# SECTION 1: THE PROBLEM — Why We Need This

## Every New Client Brings New Data

When a new client comes to Growth Protocol, they bring their own data — their own databases, their own tables, their own column names. Every client's data looks different:

- **Retail client**: `store_sales`, `ss_sales_price`, `ss_quantity`, `customer_dim`
- **SaaS client**: `subscriptions`, `mrr`, `churn_rate`, `account_id`
- **Finance client**: `transactions`, `amount`, `ledger_balance`, `counterparty`

The problem is clear: **our Reasoning Engine cannot reason over data it doesn't understand**.

If a client has a column called `ss_sales_price`, the engine needs to know:
- This is a **price** (not a name, not a date, not a location)
- It's a **decimal number** (not a string)
- It means "unit price" or "price per item" (so it can interpret natural language queries)
- It belongs to the `store_sales` table, which relates to `customer` and `item` tables

Without this understanding, the Reasoning Engine is blind. It sees raw column names but has no idea what they mean.

## The Core Challenge: Bridging Client Data to GP Knowledge

Growth Protocol has already built a comprehensive **GP Base Ontology** — a foundational knowledge structure with 3,300+ classes that define what things mean in a standardized way. It knows what a "price" is, what a "customer" is, what a "product category" is, and how they relate to each other.

The challenge is: **how do we connect each client's unique data schema to this rich knowledge base?**

This is what the Client Ontology Engine solves.

---

# SECTION 2: WHAT WE BUILT — The Client Ontology Engine

## The Simple Idea

```
    Client's Raw Data Schema          GP Base Ontology
    (unique to each client)           (our universal knowledge base)
              |                                |
              |                                |
              v                                v
        ┌──────────────────────────────────────────┐
        │       Client Ontology Engine             │
        │                                          │
        │   "Understand client data and map it     │
        │    to GP's knowledge structure"          │
        └──────────────────┬───────────────────────┘
                           |
                           v
                  Client Ontology
                  (client-specific knowledge artifact
                   linked to GP Base)
```

The engine takes a **description of the client's data** and produces a **Client Ontology** — a knowledge artifact that formally captures what the client's data means, linked to GP's universal knowledge base.

## Non-Negotiable Requirements

The Client Ontology Engine must satisfy three fundamental requirements. These are not nice-to-haves — they are the design goals that define whether the engine delivers real value:

### Requirement 1: Automatically Generated from Client Data

The Client Ontology must be **automatically generated** — not hand-crafted. Given a description of the client's data, the engine must produce a complete ontology without manual RDF/OWL authoring.

- Today: automatically generated from an OSI YAML description of the client's data
- Tomorrow: automatically generated directly from the client's database (schema auto-discovery + AI-assisted annotation)

The automation is what makes this scalable. If creating a Client Ontology required manual ontology engineering for every client, it would never keep up with client onboarding.

### Requirement 2: Represent ALL Semantics from the Data Description

The Client Ontology must capture **every semantic element** declared in the data description — nothing is silently dropped or ignored:

- Every dataset (table) becomes a formal artifact
- Every field (column) becomes a typed individual with semantic annotations
- Every entity, dimension, measure, metric becomes a formal concept
- Every relationship between datasets is formally represented
- Every AI context (synonyms, instructions, examples) is preserved as structured metadata
- Every data type, aggregation type, join type, cardinality is captured

**If it's in the data description, it must be in the Client Ontology.** The ontology is a complete, faithful representation of the client's semantic model — not a summary or subset.

### Requirement 3: Link ALL Terminologies to GP Base Ontology

Every element in the Client Ontology must be **linked to the GP Base Ontology** so that everything carries full semantic meaning:

- Every client dataset IS-A `gp_dataset` (GP knows what a dataset is)
- Every client field IS-A `gp_field` — and optionally IS-A specific field type like `gp_price_field` or `gp_name_field` (GP knows what a price means)
- Every client entity IS-A `gp_business_entity` (GP knows what a business entity is)
- Every measure, metric, relationship extends a GP Base class
- Every data type, aggregation, join type, cardinality points to a GP vocabulary concept

**Nothing exists in isolation.** Every element in the Client Ontology is connected to GP's universal vocabulary. This is what makes cross-client reasoning possible — two different clients may call their price columns differently (`unit_price` vs `ss_sales_price`), but both are linked to GP's `gp_price_field`, so the platform knows they mean the same thing.

```
  Requirement 1:  AUTOMATIC       "Don't make humans build ontologies"
  Requirement 2:  COMPLETE        "Capture everything, drop nothing"
  Requirement 3:  LINKED          "Everything has semantic meaning via GP Base"
```

These three requirements together ensure that every client's data is fully understood, fully represented, and fully connected to GP's knowledge base — automatically.

## Input: What Goes In

The engine takes two inputs:

### Input 1: Client Data Description (OSI YAML)

A structured description of the client's data schema, written in OSI (Open Semantic Interchange) format. This describes:

| What | Example | Purpose |
|------|---------|---------|
| **Datasets** (tables) | `store_sales`, `customer`, `item` | What tables the client has |
| **Fields** (columns) | `ss_sales_price`, `c_first_name` | What columns exist in each table |
| **Relationships** | `store_sales` joins to `customer` | How tables connect |
| **Business concepts** | "Total Sales" metric, "Customer" entity | Business-level meaning on top of raw data |
| **Semantic annotations** | `ss_sales_price` IS-A price field | What each field *means* in GP's vocabulary |
| **AI context** | Synonyms: "unit price", "price per item" | How neural models should interpret each field |

The key innovation: the OSI YAML doesn't just describe the schema structurally — it **maps each element to GP's knowledge base**. A field isn't just a column; it's linked to a GP concept that carries full semantic meaning.

### Input 2: GP Base Ontology (Read-Only)

Our foundational knowledge base with:
- 3,300+ pre-defined classes (business entities, data types, field types, relationships)
- 400+ properties with clear hierarchies
- Full semantic definitions for concepts like "price", "customer name", "product category"

The GP Base Ontology is never modified — the Client Ontology extends it.

## Output: What Comes Out

The engine produces **three outputs**, each serving a different purpose:

### Output 1: Client Ontology (RDF/OWL) — The Knowledge Artifact

This is the primary output — a formal, machine-readable knowledge graph that:

- **Defines every data element** as a formally typed artifact (datasets, fields, entities, measures, metrics)
- **Links every element to GP Base** — so the platform knows what each field means in a universal vocabulary
- **Captures three layers of semantic enrichment** on every field:

```
  Example: field "ss_sales_price" in the store_sales table

  Layer 1 — Semantic Type:    This field IS-A "price field"
                              (linked to GP's gp_price_field concept)

  Layer 2 — Data Type:        This field IS a "decimal number"
                              (linked to GP's gp_data_type_decimal)

  Layer 3 — AI Context:       Synonyms: "unit price", "price per item"
                              Instructions for neural models on interpretation
```

- **Imports GP Base Ontology** — inheriting all of GP's knowledge structure automatically

### Output 2: Pydantic Models (Python Classes)

Typed Python data classes generated from the client's schema:
- Used in data workflows for validation, serialization, and type safety
- Each dataset becomes a Python class with correctly typed fields
- Enables programmatic access to client data with full type checking

### Output 3: SQL Queries (Multi-Dialect)

Pre-generated SQL queries adapted to the client's database dialect:
- Supports multiple SQL dialects (ANSI SQL, Snowflake, BigQuery, etc.)
- Enables data access directly from the semantic model definition

---

# SECTION 3: WHY THIS MATTERS — What It Unlocks

## The Client Ontology Is the Backbone of All Reasoning

The Client Ontology is not a document that sits on a shelf. It is the **central knowledge artifact** that powers every reasoning capability in the platform:

```
                          Client Ontology (RDF/OWL)
                          imports GP Base Ontology
                                    |
          +-------------+----------+----------+-------------+
          |              |          |          |             |
    Operational     Symbolic     Neural    Knowledge    All Reasoning
       Layer          Layer      Layer      Graph         Steps
          |              |          |          |             |
    Pydantic        ASP Facts   Neural     Logical       Hybrid
    Models          (logic      Context    Representation Reasoning
    (workflows,      rules,     (what       (SPARQL,      (symbolic +
     validation)     constraints fields     graph          neural +
                     inference)  MEAN)      traversal)     knowledge)
```

### 1. Operational Layer — Data Workflows Work

The Pydantic models enable typed data workflows. Without the Client Ontology, there are no typed models — data validation and serialization have no schema to work from.

### 2. Symbolic Reasoning Layer — Logic Rules Can Run

The ontology structure maps to ASP (Answer Set Programming) facts. The symbolic solver uses these facts for constraint checking, inference, and rule evaluation. Without the Client Ontology, the symbolic layer has no facts to reason over.

### 3. Neural Reasoning Layer — AI Models Understand the Data

The AI context (synonyms, instructions, semantic types) gives neural models the knowledge they need to understand what client data *means*. A neural model can know that `ss_sales_price` means "unit price" and is a monetary value — not because it guessed, but because the ontology formally declares it.

This is the foundation for:
- **Natural language to SQL** (NL2SQL) — asking questions in plain English
- **Semantic search** — finding relevant data by meaning, not just column names
- **Data discovery** — understanding what data is available and how it connects

### 4. Knowledge Graph Layer — Structured Queries and Inference

The Client Ontology IS a knowledge graph. It can be queried with SPARQL:
- "Find all price fields across all datasets"
- "What tables are connected to the customer table?"
- "Show me all measures that use the sales_price field"

Class hierarchy enables automatic inference — querying for "all fields" automatically returns price fields, name fields, location fields, etc.

### 5. Backbone Layer — Everything Works Together

The Reasoning Engine combines all layers. Symbolic reasoning grounds the neural model. The knowledge graph provides structure. Neural context provides meaning. Every reasoning step can query the ontology for field semantics, relationships, and context.

**Without the Client Ontology, none of these layers work.** It is the single prerequisite that unlocks the entire reasoning pipeline for each client.

## What This Feature Specifically Unlocks

| Capability | Without Client Ontology | With Client Ontology |
|-----------|------------------------|---------------------|
| **Client onboarding** | Cannot integrate client data into the platform | Client data is formally understood and linked to GP knowledge |
| **Reasoning Engine** | Blocked — no knowledge to reason over | Fully operational — all 5 reasoning layers active |
| **NL2SQL** | Cannot translate questions about client data | Neural models know what each field means |
| **Data discovery** | Raw column names with no context | Rich semantic understanding of all data elements |
| **Cross-client insights** | Each client is an isolated silo | Common GP vocabulary enables cross-client patterns |
| **Quality assurance** | No way to validate data semantics | Formal ontology enables consistency checking |

---

# SECTION 4: HOW IT WORKS — The Engine Pipeline

*(High-level, not implementation details)*

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Client Ontology Engine                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Input:                                                                 │
│  ┌──────────────────┐    ┌───────────────────────┐                      │
│  │ Client Data      │    │  GP Base Ontology     │                      │
│  │ Description      │    │  (our knowledge base, │                      │
│  │ (OSI YAML)       │    │   read-only)          │                      │
│  └────────┬─────────┘    └─────────────┬─────────┘                      │
│           │                            │                                │
│           ▼                            │                                │
│  ┌────────────────────────────────┐    │                                │
│  │  Step 1: Parse & Validate     │    │                                │
│  │  Read the data description    │    │                                │
│  │  and check for errors         │    │                                │
│  └────────────┬───────────────────┘    │                                │
│               │                        │                                │
│               ▼                        │                                │
│  ┌──────────────────────────────────┐  │                                │
│  │  Step 2: Map to GP Knowledge    │  │                                │
│  │  Connect every client element   │◄─┘                                │
│  │  to GP Base Ontology classes    │                                   │
│  │  and properties                 │                                   │
│  └────────────┬───────────────────┘                                    │
│               │                                                         │
│    ┌──────────┼──────────┐                                              │
│    ▼          ▼          ▼                                              │
│  ┌────────┐ ┌─────────┐ ┌─────────┐                                    │
│  │Client  │ │Pydantic │ │ SQL     │                                    │
│  │Ontology│ │Models   │ │Queries  │                                    │
│  │(RDF)   │ │(Python) │ │         │                                    │
│  └────────┘ └─────────┘ └─────────┘                                    │
│                                                                         │
│  Output: Three artifacts ready for the platform                        │
└─────────────────────────────────────────────────────────────────────────┘
```

The mapping step is where the core value is created. For each element in the client's data, the engine:

1. **Creates a formal representation** — the field becomes a typed artifact in the ontology
2. **Links it to GP's vocabulary** — the field is connected to GP's understanding of what it means
3. **Enriches it with AI context** — synonyms and instructions that help neural models interpret it

---

# SECTION 5: CURRENT STATE AND ROADMAP

## What's Built Today (Fully Operational)

| Component | Status |
|-----------|--------|
| Data description parser (OSI YAML) | Done |
| Structural validation | Done |
| Semantic mapping to GP Base Ontology | Done |
| Client Ontology generation (RDF/OWL) | Done |
| Pydantic model generation (Python classes) | Done |
| SQL query generation (multi-dialect) | Done |
| Semantic field typing (gp_semantic_link) | Done |
| Data type mapping (gp_data_type) | Done |
| AI context for neural reasoning | Done |
| GP Base Ontology import (gateway pattern) | Done |

**The engine works end-to-end today.** Given a client data description, it produces a complete Client Ontology linked to GP Base, along with Python models and SQL queries.

## Current Process: Semi-Manual

Today, preparing the client data description (OSI YAML) is a semi-manual process:

```
  Client's Database
       │
       ▼
  Data Engineer writes OSI YAML          ← Manual step
  (describes tables, fields, semantics)
       │
       ▼
  Client Ontology Engine                 ← Automated
  (parse → map → generate)
       │
       ▼
  Client Ontology + Models + SQL         ← Ready for platform
```

The manual step captures two types of knowledge:
- **Structural knowledge** (tables, columns, data types, relationships) — can be auto-discovered from the database
- **Semantic knowledge** (what fields mean, business entities, AI context) — requires human judgment or AI assistance

## Roadmap: Toward Full Automation

### Phase 1: Schema Auto-Discovery (Next)

Connect to the client's database and auto-generate 80% of the data description:
- Auto-discover all tables and columns
- Auto-detect data types from database catalog
- Auto-infer relationships from foreign keys
- Human only needs to add semantic annotations

**Impact**: Reduces onboarding effort from days to hours.

### Phase 2: AI-Assisted Semantic Annotation

Use neural models to suggest semantic mappings:
- Suggest what each field *means* based on its name and context (e.g., `price` in name suggests `gp_price_field`)
- Suggest data types from source column types
- Suggest synonyms and AI context from field names
- Human reviews and approves — the model suggests, the human validates

**Impact**: Reduces the remaining 20% of manual work to review-and-approve.

### Phase 3: Fully Automated Onboarding

Combine schema auto-discovery + AI-assisted annotation:
- Client connects their database
- Engine auto-discovers schema and suggests full semantic model
- Human reviews a validation report and approves
- Client Ontology is generated automatically

**Impact**: Client onboarding goes from days to minutes.

```
  TODAY                    PHASE 1              PHASE 2              PHASE 3
  ─────                    ───────              ───────              ───────

  100% Manual              20% Manual           5% Manual            Review Only
  Write full YAML          Add semantics        Review suggestions   Approve report
  by hand                  to auto-skeleton     from AI model        and go

  Days                     Hours                < 1 Hour             Minutes
```

---

# SECTION 6: THE BIG PICTURE

## Where Client Ontology Fits in the Platform

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │                     Growth Protocol Platform                             │
  │                                                                          │
  │  ┌─────────────┐     ┌──────────────────┐     ┌──────────────────────┐  │
  │  │ Client Data │────►│ Client Ontology  │────►│  Reasoning Engine    │  │
  │  │ Onboarding  │     │ Engine           │     │  (all 5 layers)      │  │
  │  │             │     │                  │     │                      │  │
  │  │ "Get data   │     │ "Understand it   │     │ "Reason over it"     │  │
  │  │  in"        │     │  and link it     │     │                      │  │
  │  │             │     │  to GP knowledge"│     │                      │  │
  │  └─────────────┘     └──────────────────┘     └──────────────────────┘  │
  │                              │                         │                │
  │                              ▼                         ▼                │
  │                     ┌────────────────┐        ┌────────────────┐        │
  │                     │ GP Base        │        │ Client Value   │        │
  │                     │ Ontology       │        │ (insights,     │        │
  │                     │ (universal     │        │  queries,      │        │
  │                     │  knowledge)    │        │  discovery)    │        │
  │                     └────────────────┘        └────────────────┘        │
  │                                                                          │
  └──────────────────────────────────────────────────────────────────────────┘
```

## The Value Chain

1. **Client brings data** → raw tables and columns, unique to each client
2. **Client Ontology Engine understands it** → maps every element to GP's universal vocabulary
3. **Reasoning Engine operates on it** → all 5 reasoning layers (operational, symbolic, neural, knowledge graph, hybrid) are activated
4. **Client gets value** → insights, natural language queries, data discovery, cross-dataset reasoning

The Client Ontology Engine is the **bridge between raw client data and platform intelligence**. Without it, the Reasoning Engine has nothing to reason over. With it, every client's data becomes part of a unified, semantically rich knowledge base that powers all platform capabilities.

## Key Takeaway

> The Client Ontology Engine solves a fundamental challenge: every client's data is different, but our Reasoning Engine needs to understand all of it in a common language. The engine automatically translates each client's unique data schema into a formal knowledge artifact linked to GP's universal vocabulary — unlocking the full power of the Reasoning Engine from day one.

---

# SLIDE GUIDE

## Suggested Slide Sequence

### Slide 1: Title
**Client Ontology Engine**
*Bridging Client Data to Platform Intelligence*

### Slide 2: The Problem
- Every client brings unique data (different tables, columns, naming)
- Our Reasoning Engine cannot reason over data it doesn't understand
- We need a way to understand each client's data in a common language
- Key visual: 3 different client schemas pointing to a question mark

### Slide 3: The Solution — One Sentence
**The Client Ontology Engine takes a description of client data and produces a knowledge artifact that formally captures what the data means, linked to GP's universal knowledge base.**

### Slide 4: Three Non-Negotiable Requirements
```
  1. AUTOMATIC       "Don't make humans build ontologies"
                     Generated from client data — not hand-crafted

  2. COMPLETE        "Capture everything, drop nothing"
                     Every dataset, field, entity, measure, metric,
                     relationship, AI context — all represented

  3. LINKED          "Everything has semantic meaning via GP Base"
                     Every element connects to GP's universal vocabulary
                     — nothing exists in isolation
```
- These define the design goals and success criteria for the engine
- Key visual: Three pillars or three checkboxes

### Slide 5: What Goes In (Input)
- **Client Data Description (OSI YAML)**: Structured description of tables, columns, relationships, business concepts, and semantic annotations
- **GP Base Ontology**: Our universal knowledge base (3,300+ classes, 400+ properties) — read-only, never modified
- Key visual: Two boxes feeding into the engine

### Slide 6: What Comes Out (Output)
Three outputs:
1. **Client Ontology (RDF/OWL)** — The knowledge artifact, linked to GP Base
2. **Python Models (Pydantic)** — Typed data classes for workflows
3. **SQL Queries** — Multi-dialect database queries
- Key visual: Engine producing three artifacts

### Slide 7: The Three Semantic Layers (Example)
Show one concrete field example:
```
  Field: ss_sales_price (from store_sales table)

  Layer 1 — What it MEANS:     This is a "price field"
  Layer 2 — What type it IS:   This is a "decimal number"
  Layer 3 — How to INTERPRET:  Synonyms: "unit price", "price per item"
```
- This is what makes client data understandable to the platform

### Slide 8: Why It Matters — Powers All Reasoning
- Client Ontology feeds 5 platform layers:
  - Operational (data workflows)
  - Symbolic (logic rules, constraints)
  - Neural (AI understanding of data meaning)
  - Knowledge Graph (structured queries, inference)
  - Hybrid Reasoning (all layers combined)
- **Without it: Reasoning Engine is blocked. With it: fully operational.**
- Key visual: Consumer diagram from Section 3

### Slide 9: What It Specifically Unlocks
| Without | With |
|---------|------|
| Cannot onboard client data | Data formally understood and linked |
| Reasoning Engine blocked | All 5 reasoning layers active |
| No NL2SQL for client data | Neural models understand field meaning |
| Raw column names only | Rich semantic data discovery |

### Slide 10: Current State
- Engine fully built and operational end-to-end
- All components implemented: parser, mapper, generators
- Semi-manual today: data engineer writes the data description, engine does the rest
- Key visual: Status table showing all green checkmarks

### Slide 11: Roadmap to Full Automation
```
  TODAY           →  PHASE 1          →  PHASE 2          →  PHASE 3
  Semi-Manual        Auto-Discovery      AI-Assisted         Fully Automated
  Days               Hours               < 1 Hour            Minutes
```
- Phase 1: Auto-discover 80% of schema from database
- Phase 2: AI suggests semantic mappings, human reviews
- Phase 3: End-to-end automated with human approval step

### Slide 12: The Big Picture
- Client Ontology Engine sits between data onboarding and the Reasoning Engine
- It is the bridge that makes raw data into platform-ready knowledge
- Every client, regardless of their data shape, becomes part of a unified semantic layer
- Key visual: Platform architecture diagram from Section 6

### Slide 13: Key Takeaway
> Every client's data is different. Our Reasoning Engine needs to understand all of it in a common language. The Client Ontology Engine is that translator — turning unique client schemas into formal knowledge linked to GP's universal vocabulary, unlocking the full power of the platform.
