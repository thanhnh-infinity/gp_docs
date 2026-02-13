# Client Data Onboarding — Gap Analysis

## Current State

The OSI Engine pipeline currently handles the full path from semantic model definition to ontology generation:

```
Client Schema --> (manual) OSI YAML --> Parser --> Mapper --> RDF Generator --> Client Ontology
                                                                                     |
                                                                              imports GP Base
```

### What the pipeline captures today

| Layer | Mechanism | Example |
|-------|-----------|---------|
| Structural type | `rdf:type osi_field` | Every field is an `osi_field` individual |
| Semantic type | `gp_semantic_link` | Field IS-A `gp_price_field` (additional `rdf:type`) |
| Data type | `gp_data_type` | Field HAS data type `integer` (optional `has_data_type` property) |
| AI context | `ai_context` | Structured `osi_ai_context` with instructions/synonyms/examples |
| Containment | Object properties | `semantic_model_contains_dataset`, `dataset_has_field`, etc. |
| Relationships | `osi_relationship` | Joins between datasets with cardinality and join type |
| Business semantics | Entities, Dimensions, Measures, Metrics | Full semantic layer on top of raw schema |

### Design philosophy

The engine is a **mechanical mapper** — it maps what the OSI YAML explicitly declares and stays silent when it doesn't know. Two external conditions must be met:

1. **OSI YAML provides the semantic annotation** (e.g., `gp_semantic_link: gp_price_field`)
2. **GP Base Ontology has the target artifact defined** (e.g., `gp_price_field` class exists)

Both are knowledge tasks, not engine problems.

---

## Gap 1: The YAML Authoring Bottleneck (Biggest Friction)

### Problem

Right now, someone must manually write the entire OSI YAML for a client:
- All datasets (tables/views)
- All fields (columns) with their properties
- All semantic annotations (`gp_semantic_link`, `gp_data_type`, `ai_context`)
- All relationships (joins)
- All entities, dimensions, measures, metrics

For a real client with 50+ tables and hundreds of columns, this is a massive manual effort. The TPC-DS example has 5 datasets with 32 fields and it's already 619 lines of YAML.

### Why it matters

This is the **adoption blocker**. If onboarding a new client takes days of manual YAML authoring, the system doesn't scale. The value proposition of semantic modeling is lost if it costs more than the value it provides.

### Possible solutions

**A. Schema auto-discovery (recommended first step)**
- Connect to client's database (or accept DDL/catalog export)
- Auto-generate skeleton YAML with:
  - All datasets (from `information_schema.tables`)
  - All fields with correct `data_type` (from `information_schema.columns`)
  - Primary keys and foreign keys (from constraints)
  - Relationships inferred from foreign keys
- The structural parts (80% of YAML) are automated
- Semantic annotations still need human judgment

**B. AI-assisted annotation**
- Given a skeleton YAML, use LLM to suggest:
  - `gp_semantic_link` based on field names and descriptions (e.g., `price` in name -> `gp_price_field`)
  - `gp_data_type` from source column types
  - `ai_context.synonyms` from field names and context
- Human reviews and approves suggestions

**C. Template library**
- Pre-built OSI YAML templates for common schemas:
  - E-commerce (orders, products, customers)
  - SaaS metrics (subscriptions, usage, billing)
  - Financial (transactions, accounts, balances)
- Client starts from closest template and customizes

### Impact: HIGH — blocks adoption at scale

---

## Gap 2: Validation After Mapping

### Problem

The engine generates an ontology but never verifies its correctness. Failures are silent:
- A bad `gp_semantic_link` value (e.g., `gp_nonexistent_field`) logs a warning and silently skips the `rdf:type` assertion
- A relationship referencing a non-existent dataset creates a dangling reference
- Fields with no semantic annotation pass through without complaint
- The generated OWL may be logically inconsistent without anyone knowing

### Why it matters

The client has no feedback on whether their semantic model is correct, complete, or useful. Silent failures lead to incomplete ontologies that silently degrade reasoning quality.

### Possible solutions

**A. Post-generation validation report**
```
Validation Report for: tpcds_retail_model
==========================================
Classes:     10 created, 10 resolved to GP Base superclasses    [OK]
Fields:      32 total
  - gp_semantic_link: 32/32 resolved                           [OK]
  - gp_data_type:     32/32 specified                           [OK]
  - ai_context:       28/32 have synonyms                       [WARN: 4 fields missing ai_context]
Relationships: 4 total, all datasets resolved                   [OK]
Consistency:   OWL DL consistent                                [OK]
```

**B. Pre-generation schema validation (current: basic)**
- The existing `validator.py` checks basic structural rules
- Could be extended to verify:
  - All `gp_semantic_link` values exist in GP Base class hierarchy
  - All `gp_data_type` values are valid
  - Referential integrity (relationships point to real datasets, measures reference real fields)

**C. Semantic completeness scoring**
- Score how "complete" the semantic model is:
  - What % of fields have `gp_semantic_link`?
  - What % of fields have `ai_context`?
  - Are all entities backed by valid datasets?
- Helps client prioritize annotation effort

### Impact: MEDIUM — solvable incrementally, important for trust

---

## Gap 3: Downstream Consumption — ALREADY ADDRESSED

### Status: RESOLVED by Reasoning Engine Architecture

The Client Ontology is **not** a dead document — it is the **central knowledge artifact** that feeds all reasoning layers in the platform. It already has four active consumers:

### Active consumers

```
                        Client Ontology (RDF/OWL)
                        imports GP Base Ontology
                                  |
                 +----------------+----------------+----------------+
                 |                |                |                |
          Operational       Symbolic           Neural          Backbone
             Layer            Layer             Layer           Layer
                 |                |                |                |
        Pydantic Models     ASP Facts        LLM Context     All Reasoning
        (data validation,   (Answer Set      (ai_context,      Steps
         serialization,      Programming,     synonyms,
         workflow use)        logic rules,     gp_semantic_link
                              constraint       → field meaning,
                              solving)         instructions)
```

**A. Pydantic model generation (Operational Layer)**
- The Pydantic generator produces typed Python classes from the ontology structure
- These models are used in data workflows for validation, serialization, and type safety
- Each dataset becomes a Pydantic class with correctly typed fields

**B. ASP fact generation (Symbolic AI Reasoning Layer)**
- Pydantic base models include `to_asp_facts()` method
- The ontology structure (classes, individuals, properties, relationships) maps to ASP facts
- An ASP solver can then reason over the facts — constraint checking, inference, rule evaluation
- This is the **symbolic grounding** that makes reasoning deterministic where it needs to be

**C. LLM reasoning enrichment (Neural Layer)**
- `ai_context` (synonyms, instructions, examples) provides natural language context
- `gp_semantic_link` tells the LLM what fields *mean*, not just what they're named
- Class hierarchy from GP Base gives the LLM structured knowledge about data relationships
- This is the backbone for NL2SQL, semantic search, and data discovery

**D. Full reasoning pipeline support (Backbone Layer)**
- The Client Ontology is the single source of truth for all reasoning steps
- Hybrid reasoning: symbolic (ASP) grounds the LLM, LLM handles ambiguity
- Every reasoning step can query the ontology for field semantics, relationships, and context

### What each consumer uses

| Consumer | What it uses from the ontology | Purpose |
|----------|-------------------------------|---------|
| **Pydantic Generator** | Datasets, fields, data types, structure | Data validation and workflow integration |
| **ASP Facts** | Individuals, class hierarchy, properties, relationships | Symbolic reasoning — logic rules, constraint checking, inference |
| **LLM Reasoning** | `ai_context` (synonyms, instructions, examples), `gp_semantic_link`, class hierarchy | Understanding what fields *mean*, not just what they're named |
| **Reasoning Engine** | All of the above combined | Hybrid reasoning — symbolic grounds the LLM, LLM handles ambiguity |

### Why this works

The ontology's three semantic layers map directly to consumer needs:
- **`gp_semantic_link`** → ASP knows field types, LLM knows field meaning
- **`gp_data_type`** → Pydantic gets correct Python types, ASP gets type constraints
- **`ai_context`** → LLM gets synonyms/instructions for natural language understanding

Without the Client Ontology, the LLM has no semantic grounding, the ASP solver has no facts, and the workflows have no typed models. It is the backbone.

### Impact: RESOLVED — the architecture already addresses this

---

## Gap 4: Schema Evolution

### Problem

Client schemas change over time:
- New tables are added
- New columns appear in existing tables
- Columns are renamed or their types change
- Tables are deprecated or removed
- Semantic annotations need updating

Currently, any change requires regenerating the entire ontology from scratch.

### Why it matters

In production, schema changes are frequent. If every change requires a full re-onboarding cycle (update YAML manually, regenerate, validate), the maintenance cost becomes prohibitive.

### Possible solutions

**A. Diff-based incremental updates**
- Compare new YAML against previous version
- Generate only the changes (add/remove/modify individuals and properties)
- Preserve existing annotations that haven't changed

**B. Schema drift detection**
- Periodically compare client's actual schema (from database catalog) against the OSI YAML
- Flag discrepancies:
  - "New column `ss_discount` found in `store_sales` — not in YAML"
  - "Column `ss_old_field` in YAML but no longer exists in database"

**C. Versioned ontologies**
- Each generation produces a versioned ontology
- Maintain history for auditing and rollback
- Track when semantic annotations were added/changed

### Impact: LOW for onboarding, MEDIUM for production — this is a maintenance concern, not a first-time onboarding blocker

---

## Gap 5: Multi-Source Clients

### Problem

Real clients have multiple data sources:
- Production database (PostgreSQL)
- Data warehouse (Snowflake/BigQuery)
- CRM system (Salesforce)
- Analytics tool (Tableau/Looker)
- External APIs

Each source might have overlapping entities (e.g., "customer" exists in CRM and warehouse). The current framework generates one ontology per semantic model, with no mechanism to link across models.

### Why it matters

If a client's customer data lives in both Salesforce and Snowflake, the reasoning engine needs to know that `salesforce.Contact` and `warehouse.customer` represent the same business entity. Without cross-model linking, each data source is an isolated silo — which defeats the purpose of a unified semantic layer.

### Possible solutions

**A. Cross-model entity resolution**
- Define equivalence mappings: "salesforce.Contact.Email = warehouse.customer.c_email_address"
- Generate `owl:sameAs` or custom linking properties between individuals across ontologies

**B. Shared entity registry**
- A central registry of business entities (Customer, Product, Order, etc.)
- Each semantic model maps its datasets to the shared registry
- The registry serves as the integration point

**C. Federation ontology**
- A meta-ontology that imports all client ontologies
- Defines cross-model relationships and equivalences
- Enables queries that span multiple data sources

### Impact: LOW for initial onboarding, HIGH for enterprise clients — most clients start with one source and expand later

---

## Assessment and Priority

```
  STATUS          GAP                          IMPACT
  --------        ---                          ------
  RESOLVED        Gap 3 (downstream consumers) Was HIGH — now addressed by reasoning engine architecture
  OPEN            Gap 1 (YAML authoring)       HIGH — blocks adoption at scale
  OPEN            Gap 2 (validation)           MEDIUM — builds trust, solvable incrementally
  OPEN            Gap 5 (multi-source)         LOW now, HIGH for enterprise
  OPEN            Gap 4 (schema evolution)     LOW — production concern, not onboarding
```

```
                        HIGH
                         |
          Gap 1          |          Gap 3
     (YAML authoring)   |    (downstream consumption)
     Blocks adoption     |    RESOLVED
                         |
  ADOPTION BLOCKER ------+------ VALUE DELIVERED
                         |
          Gap 2          |         Gap 5
      (validation)       |    (multi-source)
    Builds trust         |    Enterprise need
                         |
                        LOW
                         |
                       Gap 4
                  (schema evolution)
                  Production concern
```

### Revised recommended sequence

Since Gap 3 is resolved (the reasoning engine already consumes the ontology via Pydantic, ASP, LLM, and the full reasoning pipeline), the priority shifts:

1. **Gap 1 first (YAML authoring)** — This is now the single biggest blocker. The value pipeline works end-to-end, but getting data INTO the pipeline is too manual. Schema auto-discovery would cut 80% of the effort.

2. **Gap 2 second (validation)** — As more clients onboard, we need confidence that the mappings are correct. A validation report after generation gives the client feedback and builds trust.

3. **Gap 5 third (multi-source)** — When enterprise clients need to link data across multiple sources, this becomes critical. Can be deferred until we have real enterprise demand.

4. **Gap 4 last (schema evolution)** — Important for production stability but not a blocker. Regeneration from scratch works for now.

### The value loop (as it stands today)

```
Client Schema ---(manual, Gap 1)---> OSI YAML ---(parser)---> Mapper ---> RDF Generator
                                                                                |
                                                                         Client Ontology
                                                                         imports GP Base
                                                                                |
                                 +----------------------------------------------+
                                 |                  |                  |                  |
                          Pydantic Models      ASP Facts         LLM Context       All Reasoning
                          (workflows)          (symbolic          (neural             (hybrid
                                                reasoning)         reasoning)          reasoning)
```

The **middle** (Parser -> Mapper -> RDF) is solid. The **output** (consumption) is solid — four active consumers. The **input** (YAML authoring) is the bottleneck. Gap 1 is the priority.
