# OSI → GP Base Ontology Mapping

- This document describes how OSI (Open Semantic Interchange) concepts are mapped to the GP Base Ontology when generating client ontologies.
- That task is continuously evolving as we add more features to OSI and the GP Base Ontology, but this serves as a reference for the current state of the mapping.
- I am kind of treating OSI as a domain-specific modeling language for semantic models, and the GP Base Ontology as the foundational ontology that provides the core classes and properties for representing those models in a standardized way.
- The client ontology imports GP Base via `gp_ontology_gateway.rdf` (umbrella ontology), which transitively imports all GP Base modules (artifact, schema, operation).

## Class Mappings

  | OSI Type | GP Class (Created) | GP Superclass (Extends) | Description |
  |----------|-------------------|------------------------|-------------|
  | `SemanticModel` | `osi_semantic_model` | `gp_schema_design` | Top-level semantic model container |
  | `Dataset` | `osi_dataset` | `gp_dataset` | Dataset representing a table or view |
  | `Field` | `osi_field` | `gp_field` | Field definition within a dataset (+ additional `rdf:type` from `gp_semantic_link`) |
  | `Entity` | `osi_entity` | `gp_business_entity` | Business entity backed by a dataset |
  | `TimeDimension` | `osi_time_dimension` | `gp_time_entity` | Time-based dimension for temporal analysis |
  | `CategoricalDimension` | `osi_categorical_dimension` | `gp_taxonomy` | Categorical dimension for classification/grouping |
  | `Measure` | `osi_measure` | `gp_measure` | Aggregatable numeric measure |
  | `Metric` | `osi_metric` | `gp_metric` | Calculated business metric |
  | `Relationship` | `osi_relationship` | `gp_relationship` | Relationship between datasets |
  | `AIContext` (structural) | `osi_ai_context` | `gp_information_artifact` | AI/LLM context metadata for semantic elements |

## Object Properties (Structural Relationships)

  | Property Name | Domain | Range | Inverse | Description |
  |--------------|--------|-------|---------|-------------|
  | `semantic_model_contains_dataset` | `osi_semantic_model` | `osi_dataset` | `dataset_belongs_to_model` | Semantic model contains dataset |
  | `semantic_model_contains_entity` | `osi_semantic_model` | `osi_entity` | - | Semantic model contains entity |
  | `semantic_model_contains_time_dimension` | `osi_semantic_model` | `osi_time_dimension` | - | Semantic model contains time dimension |
  | `semantic_model_contains_categorical_dimension` | `osi_semantic_model` | `osi_categorical_dimension` | - | Semantic model contains categorical dimension |
  | `semantic_model_contains_measure` | `osi_semantic_model` | `osi_measure` | - | Semantic model contains measure |
  | `semantic_model_contains_metric` | `osi_semantic_model` | `osi_metric` | - | Semantic model contains metric |
  | `semantic_model_contains_relationship` | `osi_semantic_model` | `osi_relationship` | - | Semantic model contains relationship |
  | `dataset_has_field` | `osi_dataset` | `osi_field` | `field_belongs_to_dataset` | Dataset has field |
  | `entity_belongs_to_dataset` | `osi_entity` | `osi_dataset` | - | Entity is backed by dataset |
  | `time_dimension_belongs_to_dataset` | `osi_time_dimension` | `osi_dataset` | - | Time dimension belongs to dataset |
  | `time_dimension_references_field` | `osi_time_dimension` | `osi_field` | - | Time dimension references field |
  | `categorical_dimension_belongs_to_dataset` | `osi_categorical_dimension` | `osi_dataset` | - | Categorical dimension belongs to dataset |
  | `categorical_dimension_references_field` | `osi_categorical_dimension` | `osi_field` | - | Categorical dimension references field |
  | `measure_belongs_to_dataset` | `osi_measure` | `osi_dataset` | - | Measure belongs to dataset |
  | `measure_references_field` | `osi_measure` | `osi_field` | - | Measure references field |
  | `metric_uses_measure` | `osi_metric` | `osi_measure` | - | Metric uses measure |
  | `relationship_joins_left_dataset` | `osi_relationship` | `osi_dataset` | - | Relationship left side |
  | `relationship_joins_right_dataset` | `osi_relationship` | `osi_dataset` | - | Relationship right side |
  | `has_ai_context` | `owl:Thing` | `osi_ai_context` | - | Links element to its AI context metadata |

## Object Properties (Typed Attributes — converted from Data Properties)

  These properties were promoted from Data Properties (xsd:string) to Object Properties pointing to GP Base classes, enabling richer semantic querying and classification.

  | Property Name | Domain | Range (GP Class) | Description |
  |--------------|--------|-----------------|-------------|
  | `has_data_type` | `osi_field` | `gp_data_type` | Data type of the field (e.g., integer, decimal, string). **Optional**: only asserted when `gp_data_type` is explicitly provided in YAML. |
  | `has_time_granularity` | `osi_time_dimension` | `gp_time_granularity` | Temporal resolution level (day, week, month, quarter, year) |
  | `has_hierarchy_levels` | `osi_categorical_dimension` | `gp_hierarchy` | Named hierarchy for drill-down levels |
  | `has_aggregation_type` | `osi_measure` | `gp_aggregation` | Aggregation function (sum, avg, count, min, max) |
  | `has_format_string` | `osi_measure` | `gp_string_format` | Display format pattern |
  | `has_unit` | `osi_measure` | `gp_unit` | Unit of measurement (currency, weight, percentage) |
  | `has_metric_type` | `osi_metric` | `gp_metric_type` | Metric computation type (simple, derived, composite) |
  | `has_join_type` | `osi_relationship` | `gp_join` | Join type (inner, left, right, full, cross) |
  | `has_cardinality` | `osi_relationship` | `gp_cardinality` | Relationship multiplicity (one_to_one, one_to_many, many_to_many) |

## Semantic Field Typing (`gp_semantic_link`)

  Fields support multiple `rdf:type` via the `gp_semantic_link` YAML field. Each field is structurally `osi_field` but can also be semantically typed to a GP Base schema field class (subclass of `gp_field` from `gp_schema_ontology`).

  Available GP Base field types:

  | GP Field Type | Semantic Meaning | Example Fields |
  |--------------|-----------------|----------------|
  | `gp_price_field` | Monetary price | ss_sales_price, i_current_price |
  | `gp_currency_value_field` | Monetary value | ss_ext_sales_price, ss_net_profit |
  | `gp_units_sold_field` | Quantity sold | ss_quantity |
  | `gp_name_field` | Name/label of entity | c_first_name, s_store_name, i_brand |
  | `gp_label_field` | Generic label/identifier | ss_ticket_number, c_customer_id |
  | `gp_location_field` | Geographic location | s_city, s_state |
  | `gp_time_field` | Temporal value | d_date, d_year, ss_sold_date_sk |
  | `gp_product_id_field` | Product identifier | i_item_id, ss_item_sk |
  | `gp_product_name_field` | Product name/description | i_item_desc |
  | `gp_product_category_field` | Product category | i_category |
  | `gp_capacity_field` | Capacity/headcount | s_number_employees |

## Field Data Type Mapping (`gp_data_type`)

  Fields optionally declare their data type via the `gp_data_type` YAML field. This maps to the `has_data_type` object property pointing to a `gp_data_type` vocabulary individual. **If not specified, no `has_data_type` assertion is made** — the engine does not guess or default.

  | `gp_data_type` Value | GP Individual Created | Example Fields |
  |---------------------|----------------------|----------------|
  | `integer` | `gp_data_type_integer` | ss_sold_date_sk, ss_quantity, d_year, c_customer_sk |
  | `decimal` | `gp_data_type_decimal` | ss_sales_price, ss_ext_sales_price, i_current_price |
  | `string` | `gp_data_type_string` | c_customer_id, i_brand, s_store_name, s_city |
  | `date` | `gp_data_type_date` | d_date |
  | `boolean` | `gp_data_type_boolean` | (none in TPC-DS example) |
  | `timestamp` | `gp_data_type_timestamp` | (none in TPC-DS example) |

## Data Properties (Literal Attributes)

  These properties remain as Data Properties because their values are genuinely literal strings/booleans, not reusable named concepts.

  | Property Name | Domain | Range Type | Description |
  |--------------|--------|------------|-------------|
  | `has_name` | `owl:Thing` | `xsd:string` | Name of the element |
  | `has_description` | `owl:Thing` | `xsd:string` | Description of the element |
  | `has_expression` | `owl:Thing` | `xsd:string` | SQL expression (literal code) |
  | `has_version` | `osi_semantic_model` | `xsd:string` | Version of semantic model |
  | `has_fully_qualified_name` | `osi_dataset` | `xsd:string` | Fully qualified table name |
  | `is_nullable` | `osi_field` | `xsd:boolean` | Whether field can be null |
  | `has_ai_instructions` | `osi_ai_context` | `xsd:string` | AI/LLM instructions for interpreting the element |
  | `has_ai_synonym` | `osi_ai_context` | `xsd:string` | Alternative name or synonym (multi-valued) |
  | `has_ai_example` | `osi_ai_context` | `xsd:string` | Example value or usage pattern (multi-valued) |

## GP Base Class Hierarchy (with OSI extensions)
```
  gp_artifact
  ├── gp_concept
  │   ├── gp_aggregation              ←── range for has_aggregation_type
  │   ├── gp_cardinality              ←── range for has_cardinality
  │   ├── gp_hierarchy                ←── range for has_hierarchy_levels
  │   ├── gp_metric_type              ←── range for has_metric_type
  │   ├── gp_time_granularity         ←── range for has_time_granularity
  │   ├── gp_measure
  │   │   └── osi_measure             ←── OSI Measure
  │   ├── gp_metric
  │   │   └── osi_metric              ←── OSI Metric
  │   ├── gp_relationship
  │   │   ├── gp_join                 ←── range for has_join_type
  │   │   └── osi_relationship        ←── OSI Relationship
  │   ├── gp_taxonomy
  │   │   └── osi_categorical_dimension ←── OSI CategoricalDimension
  │   ├── gp_unit                     ←── range for has_unit
  │   │   ├── currency_unit
  │   │   ├── measurement_unit
  │   │   └── scale_unit
  │   └── ...
  ├── gp_entity
  │   ├── gp_business_entity
  │   │   └── osi_entity              ←── OSI Entity
  │   └── gp_time_entity
  │       └── osi_time_dimension      ←── OSI TimeDimension
  ├── gp_event
  ├── gp_information_artifact
  │   ├── gp_data_format
  │   │   └── gp_string_format        ←── range for has_format_string
  │   ├── gp_data_type                ←── range for has_data_type
  │   │   └── gp_primitive_data_type
  │   ├── gp_dataset
  │   │   └── osi_dataset             ←── OSI Dataset
  │   ├── gp_field
  │   │   └── osi_field               ←── OSI Field
  │   ├── gp_schema_design
  │   │   └── osi_semantic_model      ←── OSI SemanticModel
  │   ├── osi_ai_context              ←── OSI AI Context (structural)
  │   └── ...
  └── gp_process
```

## Relationship Diagram
```
  ┌───────────────────────────────────────────────────────────────────────────────┐
  │                           osi_semantic_model                                  │
  │                                                                               │
  │  contains_dataset    contains_entity    contains_dimension    contains_metric │
  │         │                  │                   │                    │         │
  │         ▼                  ▼                   ▼                    ▼         │
  │   ┌──────────┐      ┌──────────┐       ┌─────────────┐      ┌───────────┐     │
  │   │osi_dataset│◄────│osi_entity│       │osi_dimension│      │osi_metric │     │
  │   └──────────┘      └──────────┘       └─────────────┘      └───────────┘     │
  │         │                                     │                    │          │
  │    has_field                          references_field      uses_measure      │
  │         │                                     │                    │          │
  │         ▼                                     ▼                    ▼          │
  │   ┌──────────┐                         ┌──────────┐        ┌───────────┐      │
  │   │osi_field │◄────────────────────────│          │        │osi_measure│      │
  │   └──────────┘                         └──────────┘        └───────────┘      │
  │         │                                                        │            │
  │    has_data_type ──► [gp_data_type]                  has_aggregation_type     │
  │                                                      ──► [gp_aggregation]     │
  │                                                        has_unit               │
  │                                                      ──► [gp_unit]            │
  │                                                                               │
  │                        ┌──────────────────────┐              ┌───────────┐    │
  │                        │  osi_relationship    │─────────────►│osi_dataset│    │
  │                        │                      │  joins_left  └───────────┘    │
  │                        │  has_join_type       │  joins_right                  │
  │                        │  ──► [gp_join]       │                               │
  │                        │  has_cardinality     │                               │
  │                        │  ──► [gp_cardinality]│                               │
  │                        └──────────────────────┘                               │
  └───────────────────────────────────────────────────────────────────────────────┘
```