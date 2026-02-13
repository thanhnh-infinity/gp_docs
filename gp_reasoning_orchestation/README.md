# Q&A Framework V2 – Technical Report

## Overview

The Q&A Framework is a grounded reasoning system that answers user questions based on structured workflow outputs. It ensures answers are derived solely from stored evidence (not external knowledge) while maintaining traceability and accuracy.

---

## Architecture

┌─────────────────────────────────────────────────────────────────────┐
│                        Q&A Framework V2                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌─────────────────────┐    ┌─────────────────┐ │
│  │ RouterAgent  │───▶│ SemanticQueryPlanner│───▶│ StructuredData  │ │
│  │ (Model Route)│    │ (Partition Select)  │    │   Retriever     │ │
│  └──────────────┘    └─────────────────────┘    └────────┬────────┘ │
│         │                                                │          │
│         │            ┌─────────────────────┐             │          │
│         └───────────▶│ GroundedAnswerEngine│◀────────────┘          │
│                      │  (Final Answer)     │                        │
│                      └─────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘

---

## Core Components

### 1. RouterAgent (`router_agent.py`)

**Purpose:** Intelligent model selection based on question complexity

**Capabilities:**
- Workflow-specific keywords: mapping `workflow_id → keyword list`
- Hybrid routing: Flash model for simple queries, Pro model for complex reasoning
- Fallback: Default keywords when workflow not found

**Routing logic example:**

```python
complex_keywords = ["design", "calculate", "optimize", "compare", "analysis"...]
if any(keyword in question.lower() for keyword in keywords):
    <use this>
else:
    <use that>
```

### 2. SemanticQueryPlanner (`semantic_query_planner.py`)

**Purpose**: Determine which data partitions and fields are needed to answer a question

**Capabilities**:
	•	Batch mode: Single LLM call for all partitions (fewer API calls)
	•	Important partitions fast path: Skip LLM for critical partitions (e.g., summary tables)
	•	Field selection: Identify relevant columns from large datasets
	•	Record filtering: Determine how many rows are needed (all | top_n | sample)


### 3. StructuredDataRetriever (`structured_data_retriever.py`)

**Purpose**: Extract relevant data based on the semantic query plan

**Capabilities**:
	•	Guardrails: Caps mode="all" to max_rows (default 200) to prevent token explosion
	•	Field projection: Returns only selected columns
	•	Fail-safe: Configurable behavior on bad query plans


### 4. GroundedAnswerEngine (`grounded_answer.py`)

**Purpose**: Generate final answers grounded in retrieved evidence

**Capabilities**:
	•	Non-negotiable rules: No fabrication, no external knowledge, no contradiction
	•	Model fallback chain: Primary → fallback models on timeout/error
	•	Token budget: Max 200K input tokens with overflow handling
	•	Repair mechanism: Retries with context if first attempt fails


# Configuration (QAToolConfig)

| Parameter                               | Default | Description                                |
|-----------------------------------------|----------|--------------------------------------------|
| enable_reasoning_protocol               | False    | Dynamic vs static reasoning protocol       |
| use_suggestions_only                    | True     | Use predefined partitions only             |
| use_selection_batch_mode                | True     | Single LLM call for partition selection    |
| fail_open_on_bad_plan                   | False    | Strict mode on query plan failures         |
| include_ontology_in_answer              | False    | Pass ontology to final answer              |
| enable_important_partitions_fast_path   | True     | Skip LLM for important partitions          |

---

# Reliability Features

## LLM Call Protection

- **Timeout:** `request_timeout` parameter (default 30–60s)  
- **Fallback chain:**  
  `Gemini 3.0 Flash → Gemini 3.0 Pro → Gemini 2.5 Pro`  
- **Error suppression:**  
  `LiteLLMTimeoutFilter` converts verbose ERROR logs into clean WARNING logs  

---

## Token Management

- **Max input:** 200K tokens with early rejection  
- **Row capping:** `mode="all"` respects `max_rows`  
- **Field projection:** Returns only necessary columns  

---

# Model Selection Strategy

| Question Type     | Model            | Rationale                     |
|-------------------|------------------|--------------------------------|
| Simple lookup     | Gemini 3.0 Flash | Fast, cost-effective           |
| Complex reasoning | Gemini 3.0 Pro   | Better analytical capability   |
| Timeout / Error   | Fallback chain   | Fault tolerance                |

**Complex reasoning triggers include:**  
design, calculate, optimize, compare, analysis, margin, elasticity, profit pool, etc.

---

# Data Flow Example

**User:**  
“Which scenario maximizes company profit?”

**Execution flow:**

1. **RouterAgent** → detects “maximize” → uses Pro model  
2. **SemanticQueryPlanner** → selects partitions `[summary, scenario_1..4]`  
3. **StructuredDataRetriever** → summary (all rows), scenarios (top 50 rows)  
4. **GroundedAnswerEngine** → generates grounded answer using evidence  
5. **Return** → grounded markdown answer with embedded tables  