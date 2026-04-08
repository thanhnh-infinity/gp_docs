# Cognition Agent — Architecture & Implementation Plan

## Date: 2026-04-08

**Reference:** `01_multi_agent_architecture.md`, `04_kg_reasoner_implementation_plan.md`
**Branch:** `feature/cognition_agent`
**This document:** Complete architecture, API design, tool system, data flow, and implementation roadmap for the Cognition Agent — a Claude Code-level agentic reasoning system built on top of the Growth Protocol reasoning-engine.

---

## 1. What Is the Cognition Agent?

The Cognition Agent is an **autonomous, iterative, tool-using reasoning system** that works exactly like Claude Code: it receives a question (and optionally grounding data), then enters a **Think → Act → Observe → Think → Act → Observe → ... → Answer** loop using Claude API with tool use.

Unlike the existing 3-stage pipeline (`SemanticQueryPlanner → StructuredDataRetriever → GroundedAnswer`) which plans everything upfront and executes once, the Cognition Agent **discovers what it needs as it reasons** — calling tools, inspecting results, adjusting strategy, cross-checking answers, and self-correcting on failure.

### 1.1 Why Build This?

The existing pipeline is fast and cheap for structured lookups (L0–L2 queries). But it fails on:

| Problem | Existing Pipeline | Cognition Agent |
|---------|------------------|-----------------|
| Multi-hop reasoning ("suppliers with highest risk AND longest lead time to PLANT OXNARD") | Single Cypher template, often misses joins | Multiple tool calls, cross-references results |
| Arithmetic verification ("is the margin really 23%?") | LLM guesses from context | `execute_python` verifies computation |
| Cross-domain ("how does warranty claim rate correlate with pricing scenario impact?") | Single-domain pipeline | Multiple `execute_cypher` across domains |
| Self-correction on empty results | Returns "no data found" | Tries entity search, adjusts query, retries |
| Optimization ("what's the best allocation given constraints?") | Not supported | `execute_asp` with Clingo/Clingcon |
| Ad-hoc analysis with user-provided data | Not supported | Accepts grounding data + analyzes with tools |

### 1.2 Core Principle

```
Claude Code = Claude + Tools + Loop
Cognition Agent = Claude + Domain Tools + Loop + Domain Knowledge
```

The agent is NOT a wrapper around one engine. It is a **meta-agent** that sits above the ENTIRE reasoning-engine system and can invoke ANY capability: KG queries, SQL, ASP reasoning, Python computation, entity search, web search, RAG retrieval — whatever the question demands.

---

## 2. Architecture Overview

```
                    ┌──────────────────────────────────────────────┐
                    │              API / MCP ENTRY                  │
                    │  POST /cognition-agent/solve                  │
                    │  POST /cognition-agent/solve-with-data        │
                    │  POST /cognition-agent/stream                 │
                    └──────────────────┬───────────────────────────┘
                                       │
                    ┌──────────────────▼───────────────────────────┐
                    │            QUERY CLASSIFIER                   │
                    │       (Enhanced RouterAgent)                  │
                    │                                               │
                    │  L0–L2  ──► Existing QA Pipeline (fast/cheap) │
                    │  L3–L4  ──► Cognition Agent (thorough)        │
                    │  Pipeline fail / low confidence ──► Escalate  │
                    └────────────┬──────────────┬──────────────────┘
                                 │              │
                   ┌─────────────▼──┐   ┌───────▼──────────────────┐
                   │ Existing QA    │   │    COGNITION AGENT        │
                   │ Pipeline       │   │                           │
                   │ (3-stage)      │   │  ┌─────────────────────┐  │
                   │ ~2-5s          │   │  │  SYSTEM PROMPT       │  │
                   │ ~$0.01/query   │   │  │  Domain ontology     │  │
                   └────────────────┘   │  │  Tool descriptions   │  │
                                        │  │  Reasoning rules     │  │
                                        │  └─────────┬───────────┘  │
                                        │            │              │
                                        │  ┌─────────▼───────────┐  │
                                        │  │   AGENT LOOP        │  │
                                        │  │                     │  │
                                        │  │  Claude API call    │  │
                                        │  │      │              │  │
                                        │  │  Tool calls? ──►Yes─┤  │
                                        │  │      │         │    │  │
                                        │  │      No    Execute  │  │
                                        │  │      │     tools    │  │
                                        │  │      ▼         │    │  │
                                        │  │   ANSWER  ◄────┘    │  │
                                        │  └─────────────────────┘  │
                                        │                           │
                                        │  TOOL CATEGORIES:         │
                                        │  ┌─────┐┌──────┐┌──────┐ │
                                        │  │DATA ││SYMB. ││COMP. │ │
                                        │  │     ││REASON││      │ │
                                        │  └─────┘└──────┘└──────┘ │
                                        │  ┌─────┐┌──────┐         │
                                        │  │SRCH ││INTRO.│         │
                                        │  │& RAG││SPECT │         │
                                        │  └─────┘└──────┘         │
                                        └──────────────────────────┘
```

---

## 3. Tool System — Complete Specification

### 3.1 Category 1: DATA ACCESS

These tools retrieve data from any source in the system.

#### `execute_cypher`
```
Description: Execute a Cypher query against the Neo4j Knowledge Graph.
             Use this for structured graph traversal, relationship queries,
             entity lookups, and aggregations over the knowledge graph.

Input:
  cypher: str          # Cypher query string
  params: dict         # Query parameters (optional)
  namespace: str       # KG namespace (default: rheem_composition)

Output:
  rows: List[dict]     # Query results
  row_count: int       # Number of rows returned
  execution_time_ms: int

Wraps: GPKnowledgeGraph.execute_query()
File:  app/kr_engine/knowlegde_graph/knowledge_graph.py
```

#### `search_entities`
```
Description: Search for entities in the Knowledge Graph by name.
             Uses 3-stage matching: exact → full-text (Lucene) → vector (HNSW).
             Use this BEFORE writing Cypher to verify entity names and types.

Input:
  mention: str              # Entity name to search for
  entity_type: str          # Optional: filter by type (SKU, MANUFACTURER, etc.)
  top_k: int                # Max results (default: 5)

Output:
  matches: List[{label, type, score, match_stage}]

Wraps: Neo4jEntityIndex.search_fulltext() + search_vector()
File:  app/kr_engine/knowlegde_graph/neo4j_entity_index.py
```

#### `query_partition`
```
Description: Retrieve tabular data from a specific workflow partition.
             Supports column selection, row filtering, sorting, and limiting.
             Use this for raw data access when Cypher is insufficient.

Input:
  partition_name: str       # e.g., "competitive_pricing_between_company_and_competitors"
  columns: List[str]        # Columns to return (optional, default: all)
  filters: dict             # Column-value filters (optional)
  sort_by: str              # Sort column (optional)
  limit: int                # Max rows (default: 100)
  workflow_id: str          # Workflow identifier (optional, default: current)

Output:
  headers: List[str]
  rows: List[List[Any]]
  total_rows: int

Wraps: Direct access to PipelineStructuredOutput data
File:  NEW — app/agents/cognition_agent/tools/data_access.py
```

#### `execute_sql`
```
Description: Execute a SQL query against Snowflake via the Olympus gateway.
             Use this for data that lives in Snowflake, not in the KG or partitions.

Input:
  function_id: str          # Snowflake function identifier
  parameters: dict          # Query parameters (optional)

Output:
  rows: List[dict]
  row_count: int

Wraps: OlympusConnector (async)
File:  app/utils/olympus_connector.py
```

### 3.2 Category 2: SYMBOLIC REASONING

These tools invoke the ASP/Clingo symbolic reasoning stack.

#### `execute_asp`
```
Description: Execute an Answer Set Program using Clingo (deterministic) or
             Clingcon (constraint optimization). Use this for logic problems,
             optimization, planning, scheduling, and constraint satisfaction.

Input:
  facts: List[str]          # ASP facts (e.g., ["node(a).", "edge(a,b)."])
  rules: List[str]          # ASP rules (e.g., ["path(X,Y) :- edge(X,Y)."])
  show: List[str]           # Show directives (e.g., ["path/2"])
  solver: str               # "clingo" or "clingcon" (default: "clingo")
  num_models: int           # Number of answer sets (default: 1, 0=all)
  optimization: bool        # Whether to find optimal model (default: false)

Output:
  answer_sets: List[List[str]]   # Each answer set = list of atoms
  optimal_model: List[str]       # Best model (if optimization=true)
  cost: List[int]                # Optimization cost vector
  satisfiable: bool
  execution_time_ms: int

Wraps: ASPReasonerFactory.create() → ClingoASPReasoner / ClingconASPReasoner
File:  app/reasoner/deterministic_reasoner.py
```

#### `compile_nl_to_asp`
```
Description: Compile natural language text into ASP facts using LLM-driven
             extraction. Use this when you have unstructured text that needs
             to be converted to logical facts for ASP reasoning.

Input:
  text: str                 # Natural language text
  target_predicates: List[str]  # Expected predicate names (optional hints)

Output:
  facts: List[str]          # Compiled ASP facts

Wraps: NL2ASPFactCompiler
File:  app/kr_engine/compiler/nl2asp_fact_compiler.py
```

### 3.3 Category 3: COMPUTE ENGINE

These tools perform arithmetic, aggregation, and statistical computation.

#### `execute_python`
```
Description: Execute Python code in a sandboxed environment with pandas,
             numpy, and scipy available. Use this for arithmetic verification,
             data transformation, statistical analysis, and custom computations.
             Input data can be passed as variables.

Input:
  code: str                 # Python code to execute
  variables: dict           # Variables to inject into scope (optional)
                            # e.g., {"df": [[headers], [row1], [row2], ...]}

Output:
  result: Any               # Value of last expression or explicit return
  stdout: str               # Print output
  error: str                # Error message if execution failed

Implementation: RestrictedPython or subprocess sandbox
File:  NEW — app/agents/cognition_agent/tools/compute.py
```

#### `compute_statistics`
```
Description: Compute statistical measures on a data series. Supports growth
             rates, moving averages, percentiles, correlations, and trends.

Input:
  data: List[float]         # Numeric data series
  operation: str            # "growth_rate", "moving_avg", "percentile",
                            # "correlation", "trend", "describe"
  params: dict              # Operation-specific parameters

Output:
  result: dict              # Operation-dependent result

Wraps: SignalComputation + numpy/scipy
File:  app/statistic_engine/signal_computation.py
```

### 3.4 Category 4: SEARCH & RETRIEVAL

#### `web_search`
```
Description: Search the web for external context using Perplexity AI.
             Use this as a LAST RESORT when internal data sources are
             insufficient, or for market context / industry benchmarks.

Input:
  query: str                # Search query
  focus: str                # "web" or "academic" (default: "web")

Output:
  answer: str               # Synthesized answer with citations
  sources: List[str]        # Source URLs

Wraps: ask_perplexity_async()
File:  app/llm_engine/search_tools.py
```

#### `rag_search`
```
Description: Retrieve relevant documents from the knowledge discovery
             system using RAG (Retrieval-Augmented Generation).

Input:
  query: str                # Search query
  collection: str           # Document collection (optional)
  top_k: int                # Max results (default: 5)

Output:
  documents: List[{content, metadata, score}]

Wraps: RAGRetriever
File:  app/kr_engine/knowledge_discovery/rag_retriever.py
```

### 3.5 Category 5: INTROSPECTION

These tools help the agent understand what data and capabilities are available.

#### `get_kg_schema`
```
Description: Get the Knowledge Graph schema — entity types, relationship
             types, and their properties. Use this to understand what
             entities exist and how they connect before writing Cypher.

Input:
  namespace: str            # KG namespace (default: rheem_composition)
  entity_type: str          # Optional: get details for specific type

Output:
  entity_types: List[{type, count, sample_labels}]
  relationship_types: List[{type, source, target, count}]
  properties: dict          # Type → list of property names

Wraps: CypherSchemaCompiler + live Neo4j MATCH queries
File:  app/kr_engine/knowlegde_graph/cypher_schema_compiler.py
```

#### `get_partition_metadata`
```
Description: Get metadata about a workflow partition — column descriptions,
             methodology, row count, and ontology context. Use this to
             understand what data a partition contains before querying it.

Input:
  partition_name: str       # Partition identifier

Output:
  title: str
  columns: List[{name, description}]
  row_count: int
  methodology: str          # NLP methodology description
  domain: str               # Which domain this partition belongs to
  summary: str              # Partition summary text (if available)

Wraps: Workflow ontology dict + live data inspection
File:  NEW — app/agents/cognition_agent/tools/introspection.py
```

#### `list_partitions`
```
Description: List all available data partitions, optionally filtered by domain.
             Shows partition name, domain, row count, and whether it has data
             or is NL-only (summary text only).

Input:
  domain: str               # Optional filter: "Pricing CI", "Pricing Simulator",
                            # "Warranty Intelligence", "Supply Chain"

Output:
  partitions: List[{name, domain, row_count, has_data, has_summary}]

Wraps: WorkflowKGSchemaConfig + live data
File:  NEW — app/agents/cognition_agent/tools/introspection.py
```

#### `get_domain_context`
```
Description: Get full domain knowledge context — entity types in this domain,
             relationship types, partition list, and domain-specific methodology.
             Use this at the START of complex queries to orient yourself.

Input:
  domain: str               # Domain name

Output:
  description: str          # Domain overview
  entity_types: List[str]   # Entity types used in this domain
  relationship_types: List[str]
  partitions: List[str]     # Partition names in this domain
  methodology: str          # Combined methodology text

Wraps: DomainConfig + WorkflowKGSchemaConfig + ontology
File:  NEW — app/agents/cognition_agent/tools/introspection.py
```

---

## 4. API Design

### 4.1 REST API Endpoints

#### `POST /cognition-agent/solve`

Standard query — the agent uses its tools to find the answer.

```json
// Request
{
  "question": "Which Rheem suppliers have the highest risk scores AND supply to plants with the longest average lead times?",
  "workflow_id": "pricing_simulator_competitive_warranty_intelligence_composition_workflow",
  "config": {
    "model": "claude-sonnet-4-6",           // default; "claude-opus-4-6" for hardest queries
    "max_iterations": 15,                    // max tool-call rounds (default: 15)
    "timeout_seconds": 120,                  // hard timeout (default: 120)
    "stream": false                          // true for SSE streaming
  }
}

// Response
{
  "answer": "Based on cross-referencing supplier risk data with lead time records...\n\n| Supplier | Risk Score | Avg Lead Time | Plants |\n|...",
  "confidence": 0.92,
  "tool_trace": [
    {"tool": "get_kg_schema", "input": {...}, "output_summary": "21 entity types...", "iteration": 0},
    {"tool": "execute_cypher", "input": {"cypher": "MATCH (s:SUPPLIER)..."}, "output_summary": "47 rows", "iteration": 1},
    {"tool": "execute_cypher", "input": {"cypher": "MATCH (s:SUPPLIER)-[:SUPPLIES_TO]->..."}, "output_summary": "12 rows", "iteration": 2},
    {"tool": "execute_python", "input": {"code": "import pandas as pd..."}, "output_summary": "ranked table", "iteration": 3}
  ],
  "iterations": 4,
  "total_tokens": 12847,
  "cost_usd": 0.08,
  "model_used": "claude-sonnet-4-6",
  "execution_time_ms": 8432
}
```

#### `POST /cognition-agent/solve-with-data`

Query with user-provided grounding data. The agent can analyze this data directly AND cross-reference it with internal sources.

```json
// Request
{
  "question": "Given this sales forecast, which markets should we prioritize for Scenario 4 pricing?",
  "grounding_data": {
    "format": "tabular",                     // "tabular" | "json" | "text" | "csv"
    "content": {
      "headers": ["market", "forecast_units", "growth_rate"],
      "rows": [
        ["JACKSONVILLE", 12500, 0.15],
        ["ATLANTA", 8900, 0.08],
        ["DALLAS", 15200, 0.22]
      ]
    },
    "description": "Q3 2026 sales forecast by market"  // context for the agent
  },
  "workflow_id": "pricing_simulator_competitive_warranty_intelligence_composition_workflow",
  "config": {
    "model": "claude-sonnet-4-6",
    "max_iterations": 15
  }
}

// Response: same schema as /solve
```

#### `POST /cognition-agent/stream`

Server-Sent Events (SSE) streaming — real-time progress updates as the agent reasons.

```
// Request: same as /solve or /solve-with-data

// SSE Response Stream:
event: thinking
data: {"iteration": 0, "message": "Analyzing question — need to understand KG schema first"}

event: tool_call
data: {"iteration": 0, "tool": "get_kg_schema", "input": {"namespace": "rheem_composition"}}

event: tool_result
data: {"iteration": 0, "tool": "get_kg_schema", "summary": "21 entity types, 14 relationship types"}

event: thinking
data: {"iteration": 1, "message": "Found schema. Now searching for supplier entities with risk data..."}

event: tool_call
data: {"iteration": 1, "tool": "execute_cypher", "input": {"cypher": "MATCH (s:SemanticaEntity {type: 'SUPPLIER'})..."}}

event: tool_result
data: {"iteration": 1, "tool": "execute_cypher", "summary": "47 suppliers with risk scores"}

...

event: answer
data: {"answer": "Based on...", "confidence": 0.92, "iterations": 4, "cost_usd": 0.08}

event: done
data: {}
```

### 4.2 Python SDK (Internal Programmatic Access)

```python
from app.agents.cognition_agent import CognitionAgent, CognitionConfig, GroundingData

agent = CognitionAgent()

# Simple query
result = await agent.solve(
    question="What is the average lead time for suppliers in Mexico?",
    workflow_id="pricing_simulator_competitive_warranty_intelligence_composition_workflow",
)
print(result.answer)
print(result.tool_trace)

# Query with grounding data
result = await agent.solve(
    question="Compare this forecast with actual Scenario 4 projections",
    grounding_data=GroundingData(
        format="tabular",
        content={"headers": [...], "rows": [...]},
        description="Q3 2026 sales forecast",
    ),
    config=CognitionConfig(model="claude-opus-4-6", max_iterations=20),
)

# Streaming
async for event in agent.stream(question="...", workflow_id="..."):
    if event.type == "thinking":
        print(f"[Thinking] {event.message}")
    elif event.type == "tool_call":
        print(f"[Tool] {event.tool}({event.input})")
    elif event.type == "answer":
        print(f"[Answer] {event.answer}")
```

### 4.3 Escalation from Existing Pipeline

The Cognition Agent can also be triggered as an **escalation** when the existing pipeline produces low-confidence results:

```python
# In BaseQAPipeline._generate_answer() — after GroundedAnswer produces result
if answer_confidence < COGNITION_ESCALATION_THRESHOLD:
    cognition = CognitionAgent()
    result = await cognition.solve(
        question=question,
        grounding_data=GroundingData(
            format="tabular",
            content=evidence,  # pass existing evidence as grounding
            description="Evidence retrieved by standard pipeline (low confidence)",
        ),
    )
    return result.answer
```

---

## 5. Agent Loop — Detailed Design

### 5.1 Core Loop

```python
class CognitionAgent:
    """
    Autonomous, iterative, tool-using reasoning agent.
    
    Think → Act → Observe → Think → Act → Observe → ... → Answer
    
    Works like Claude Code: Claude decides which tools to call,
    in what order, based on accumulated context from prior tool results.
    """

    def __init__(
        self,
        tool_registry: Optional[CognitionToolRegistry] = None,
    ):
        self.client = AsyncAnthropic()  # or vertex_ai / litellm
        self.tool_registry = tool_registry or CognitionToolRegistry()
        self.guardrails = CognitionGuardrails()

    async def solve(
        self,
        question: str,
        workflow_id: Optional[str] = None,
        grounding_data: Optional[GroundingData] = None,
        config: Optional[CognitionConfig] = None,
    ) -> CognitionResult:
        config = config or CognitionConfig()
        
        # 1. Build system prompt (domain-aware)
        system_prompt = CognitionSystemPrompt.build(
            workflow_id=workflow_id,
            tool_definitions=self.tool_registry.get_definitions(),
        )
        
        # 2. Build initial user message
        user_content = self._build_user_message(question, grounding_data)
        messages = [{"role": "user", "content": user_content}]
        
        # 3. Agent loop
        tool_trace: List[ToolTraceEntry] = []
        iteration = 0
        start_time = time.monotonic()
        
        while iteration < config.max_iterations:
            # Timeout check
            elapsed = time.monotonic() - start_time
            if elapsed > config.timeout_seconds:
                return self._timeout_result(messages, tool_trace, iteration)
            
            # Call Claude
            response = await self.client.messages.create(
                model=config.model,
                system=system_prompt,
                tools=self.tool_registry.get_tool_schemas(),
                messages=messages,
                max_tokens=config.max_tokens_per_turn,
            )
            
            # Terminal: Claude has the answer
            if response.stop_reason == "end_turn":
                return CognitionResult(
                    answer=self._extract_text(response),
                    tool_trace=tool_trace,
                    iterations=iteration,
                    confidence=self._assess_confidence(tool_trace),
                    model_used=config.model,
                    total_tokens=self._count_tokens(response),
                    execution_time_ms=int((time.monotonic() - start_time) * 1000),
                )
            
            # Non-terminal: Execute tool calls
            assistant_content = response.content
            tool_result_blocks = []
            
            for block in assistant_content:
                if block.type == "tool_use":
                    # Guardrail validation
                    self.guardrails.validate_tool_call(
                        tool_name=block.name,
                        tool_input=block.input,
                        iteration=iteration,
                    )
                    
                    # Execute tool
                    try:
                        result = await self.tool_registry.execute(
                            name=block.name,
                            input=block.input,
                        )
                        tool_result_blocks.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": self._serialize_result(result),
                        })
                    except Exception as e:
                        tool_result_blocks.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": f"ERROR: {type(e).__name__}: {str(e)}",
                            "is_error": True,
                        })
                        result = {"error": str(e)}
                    
                    # Record trace
                    tool_trace.append(ToolTraceEntry(
                        tool=block.name,
                        input=block.input,
                        output=result,
                        iteration=iteration,
                        timestamp_ms=int((time.monotonic() - start_time) * 1000),
                    ))
            
            # Feed results back into conversation
            messages.append({"role": "assistant", "content": assistant_content})
            messages.append({"role": "user", "content": tool_result_blocks})
            
            # Context management: summarize old tool results if approaching token limit
            messages = self._manage_context(messages, config.max_context_tokens)
            
            iteration += 1
        
        # Max iterations reached
        return self._force_final_answer(messages, tool_trace, iteration, config)
```

### 5.2 Context Management

The agent loop accumulates context with each tool call. Without management, it will exceed the context window on complex queries.

```python
def _manage_context(
    self,
    messages: List[dict],
    max_tokens: int,
) -> List[dict]:
    """
    Compress older tool results to stay within context budget.
    
    Strategy:
    - Keep system prompt intact (always)
    - Keep last 3 tool call/result pairs intact (recent context)
    - Summarize older tool results to 1-line summaries
    - Keep the original question intact (always)
    
    This mirrors Claude Code's context compression.
    """
    estimated_tokens = self._estimate_tokens(messages)
    if estimated_tokens < max_tokens * 0.8:
        return messages  # No compression needed
    
    # Compress: replace old tool results with summaries
    compressed = [messages[0]]  # Keep original question
    for msg in messages[1:-6]:  # Compress all but last 3 exchanges
        if msg["role"] == "user" and isinstance(msg["content"], list):
            # Tool result — summarize
            summaries = []
            for block in msg["content"]:
                if block.get("type") == "tool_result":
                    content = block.get("content", "")
                    summary = content[:200] + "..." if len(content) > 200 else content
                    summaries.append({**block, "content": f"[SUMMARIZED] {summary}"})
            compressed.append({"role": msg["role"], "content": summaries})
        else:
            compressed.append(msg)
    
    # Keep recent exchanges intact
    compressed.extend(messages[-6:])
    return compressed
```

### 5.3 Grounding Data Handling

```python
def _build_user_message(
    self,
    question: str,
    grounding_data: Optional[GroundingData],
) -> str:
    """Build the initial user message with optional grounding data."""
    parts = [question]
    
    if grounding_data:
        parts.append("\n\n--- GROUNDING DATA ---")
        parts.append(f"Description: {grounding_data.description}")
        
        if grounding_data.format == "tabular":
            # Format as markdown table for Claude
            content = grounding_data.content
            headers = content["headers"]
            rows = content["rows"]
            parts.append(f"\n| {' | '.join(str(h) for h in headers)} |")
            parts.append(f"| {' | '.join('---' for _ in headers)} |")
            for row in rows[:50]:  # Cap at 50 rows in prompt
                parts.append(f"| {' | '.join(str(v) for v in row)} |")
            if len(rows) > 50:
                parts.append(f"\n... ({len(rows)} total rows, first 50 shown)")
                parts.append("Use `execute_python` with the full dataset for analysis.")
        
        elif grounding_data.format == "json":
            import json
            parts.append(f"\n```json\n{json.dumps(grounding_data.content, indent=2)[:5000]}\n```")
        
        elif grounding_data.format in ("text", "csv"):
            parts.append(f"\n```\n{str(grounding_data.content)[:5000]}\n```")
        
        parts.append("--- END GROUNDING DATA ---\n")
        parts.append("You can reference this data directly in your reasoning, "
                      "pass it to `execute_python` for computation, or cross-reference "
                      "it with internal data sources using other tools.")
    
    return "\n".join(parts)
```

---

## 6. System Prompt — Domain-Aware

```python
class CognitionSystemPrompt:
    """Builds the system prompt with full domain awareness."""
    
    @staticmethod
    def build(
        workflow_id: Optional[str] = None,
        tool_definitions: Optional[List[dict]] = None,
    ) -> str:
        return f"""You are Cognition Agent — an autonomous reasoning system for enterprise intelligence.

You solve complex analytical questions by iteratively using tools: querying databases, running computations, searching knowledge graphs, executing symbolic reasoning programs, and cross-referencing multiple data sources.

## HOW YOU WORK

You operate in a Think → Act → Observe loop:
1. THINK about what information you need
2. ACT by calling one or more tools
3. OBSERVE the results
4. THINK about what you learned and what's still missing
5. Repeat until you have a complete, verified answer

## AVAILABLE DATA SOURCES

### Knowledge Graph (Neo4j)
- 17,000+ entities across 4 domains
- Entity types: SKU, MANUFACTURER, RETAILER, MARKET, REGION, SUPPLIER, PLANT,
  OPERATING_UNIT, SPEND_CATEGORY, DISTRIBUTION_CENTER, STORE, MODEL, PRODUCT_TYPE,
  PRODUCT_CATEGORY, PRODUCT_SEGMENT, PRODUCT_FAMILY, PERFORMANCE, FUEL_TYPE,
  PRICING_SCENARIO, ZIP_CODE, STATE, COUNTRY
- Relationship types: MANUFACTURED_BY, SOLD_IN_MARKET, SOLD_BY_RETAILER, IN_MARKET,
  IN_REGION, HAS_CATEGORY, HAS_MODEL, HAS_PRODUCT_SEGMENT, HAS_PRODUCT_FAMILY,
  HAS_PERFORMANCE, COMPETES_WITH, COMPETES_WITH_MANUFACTURER, HAS_FUEL_TYPE,
  SERVED_BY_DC, SUPPLIES_TO, SUPPLIES_CATEGORY, IN_STATE, PART_OF_OPERATING_UNIT,
  PRICED_UNDER_SCENARIO, WARRANTY_IMPACT_AT, IN_COUNTRY
- All entities have namespace: "rheem_composition_reasoning_pricing_warranty_supply_chain"
- All entities are labeled :SemanticaEntity

### Structured Data Partitions (46 total)
- Pricing CI: competitive pricing (7K rows), geographic pricing (39K rows), sentiment (65 rows), profit pool (981 rows), price recommendations (1.2K rows)
- Pricing Simulator: scenario summary (4 rows), 4 scenario details (1K-6K rows each)
- Warranty: store KPIs (6K+4K rows), scatter plot (2K rows), zip analysis (2K rows), warranty curves (10 models), claim/impact tables
- Supply Chain: lead time (62K rows), operating units (18 rows), org locations (233 rows), spend categories (716 rows), spend transactions (5M rows), suppliers (9K rows)

### Symbolic Reasoning (Clingo/Clingcon ASP)
- Deterministic logic programming (constraint satisfaction, planning, graph problems)
- Constraint optimization (budget allocation, scheduling)
- Probabilistic reasoning (weighted answer sets)

## CRITICAL REASONING RULES

1. VERIFY BEFORE ANSWERING: Never guess numeric values. Use `execute_python` to verify computations.
2. ENTITY NAMES FIRST: Before writing Cypher, use `search_entities` to verify exact entity labels. The KG uses normalized labels — "Home Depot" not "THD", "1002921942" not "SKU 1002921942".
3. CYPHER PATTERNS: All entities are :SemanticaEntity nodes. Always filter by namespace and type:
   `MATCH (n:SemanticaEntity {{type: "SKU", namespace: "rheem_composition_reasoning_pricing_warranty_supply_chain"}})`
   Relationships are typed: `-[:SOLD_IN_MARKET]->`, `-[:MANUFACTURED_BY]->`, etc.
4. CROSS-CHECK: If KG returns X and raw data returns Y, investigate the discrepancy.
5. EMPTY RESULTS: If a Cypher query returns empty, don't say "no data" — check entity spelling via `search_entities`, try alternative relationship paths, or fall back to `query_partition`.
6. AGGREGATION: For "how many", "total", "average" questions — prefer Cypher COUNT/SUM/AVG, then verify with `execute_python` if results seem unexpected.
7. GROUNDING DATA: When the user provides input data, analyze it AND cross-reference with internal sources. Don't just summarize the input — enrich it.
8. MULTI-DOMAIN: Questions that span domains (e.g., pricing + warranty) require multiple tool calls across different entity types and partitions.
9. EXPLAIN YOUR REASONING: Show your work. Include relevant data points, not just conclusions.
10. TOOL EFFICIENCY: Call multiple independent tools in parallel when possible. Don't make 5 sequential calls when 3 parallel + 2 sequential would suffice.
"""
```

---

## 7. Guardrails & Safety

```python
class CognitionGuardrails:
    """Safety, cost, and quality guardrails for the agent loop."""
    
    # Cost limits
    MAX_ITERATIONS = 25                      # Absolute max tool-call rounds
    MAX_CYPHER_RESULT_ROWS = 1000            # Prevent memory explosion
    MAX_PYTHON_EXECUTION_TIME_S = 30         # Sandbox timeout
    MAX_PARTITION_ROWS = 500                 # Limit raw data retrieval
    MAX_TOTAL_TOKENS = 200_000               # Context budget
    
    # Dangerous pattern detection
    BLOCKED_CYPHER_PATTERNS = [
        r"DELETE\b",                         # No mutations
        r"SET\b",                            # No mutations
        r"CREATE\b",                         # No mutations
        r"MERGE\b",                          # No mutations
        r"REMOVE\b",                         # No mutations
        r"DETACH\b",                         # No mutations
    ]
    
    BLOCKED_PYTHON_PATTERNS = [
        r"import\s+os",                      # No OS access
        r"import\s+subprocess",              # No shell access
        r"import\s+sys",                     # No sys access
        r"open\(",                           # No file I/O
        r"exec\(",                           # No dynamic exec
        r"eval\(",                           # No dynamic eval
        r"__import__",                       # No dynamic imports
    ]
    
    def validate_tool_call(self, tool_name: str, tool_input: dict, iteration: int):
        """Validate a tool call before execution. Raises GuardrailError on violation."""
        
        if iteration >= self.MAX_ITERATIONS:
            raise GuardrailError(f"Max iterations ({self.MAX_ITERATIONS}) exceeded")
        
        if tool_name == "execute_cypher":
            cypher = tool_input.get("cypher", "")
            for pattern in self.BLOCKED_CYPHER_PATTERNS:
                if re.search(pattern, cypher, re.IGNORECASE):
                    raise GuardrailError(f"Blocked Cypher pattern: {pattern}")
            # Inject LIMIT if not present
            if "LIMIT" not in cypher.upper():
                tool_input["cypher"] = cypher.rstrip().rstrip(";") + f" LIMIT {self.MAX_CYPHER_RESULT_ROWS}"
        
        if tool_name == "execute_python":
            code = tool_input.get("code", "")
            for pattern in self.BLOCKED_PYTHON_PATTERNS:
                if re.search(pattern, code):
                    raise GuardrailError(f"Blocked Python pattern: {pattern}")
        
        if tool_name == "query_partition":
            if tool_input.get("limit", 0) > self.MAX_PARTITION_ROWS:
                tool_input["limit"] = self.MAX_PARTITION_ROWS
```

---

## 8. Data Models

```python
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional
from enum import Enum


class GroundingDataFormat(str, Enum):
    TABULAR = "tabular"         # {"headers": [...], "rows": [[...], ...]}
    JSON = "json"               # Any JSON structure
    TEXT = "text"               # Free-form text
    CSV = "csv"                 # CSV string


@dataclass
class GroundingData:
    """User-provided input data for the agent to analyze."""
    format: GroundingDataFormat
    content: Any                # Format-dependent content
    description: str = ""       # Human description of what this data is


@dataclass
class CognitionConfig:
    """Configuration for a Cognition Agent invocation."""
    model: str = "claude-sonnet-4-6"
    max_iterations: int = 15
    timeout_seconds: int = 120
    max_tokens_per_turn: int = 8192
    max_context_tokens: int = 180_000
    stream: bool = False
    escalation_source: Optional[str] = None  # "qa_pipeline" if escalated


@dataclass
class ToolTraceEntry:
    """One tool call in the agent's reasoning trace."""
    tool: str
    input: Dict[str, Any]
    output: Any
    iteration: int
    timestamp_ms: int = 0
    error: Optional[str] = None


@dataclass
class CognitionResult:
    """Complete result from a Cognition Agent invocation."""
    answer: str
    tool_trace: List[ToolTraceEntry] = field(default_factory=list)
    iterations: int = 0
    confidence: float = 0.0
    model_used: str = ""
    total_tokens: int = 0
    cost_usd: float = 0.0
    execution_time_ms: int = 0
    truncated: bool = False     # True if max_iterations hit
    escalated_from: Optional[str] = None
```

---

## 9. File Structure & Python Module Design

### 9.1 Directory Layout

```
app/agents/cognition_agent/
├── __init__.py                         # Public API exports
├── cognition_agent.py                  # CognitionAgent class (agent loop)
├── cognition_config.py                 # CognitionConfig, CognitionResult, GroundingData
├── cognition_system_prompt.py          # System prompt builder (domain-aware)
├── cognition_guardrails.py             # Safety, cost, mutation guards
├── cognition_context_manager.py        # Context window compression
├── tool_registry.py                    # Tool registration + schema generation
├── tools/
│   ├── __init__.py                     # Exports all tool functions
│   ├── data_access.py                  # execute_cypher, search_entities, query_partition, execute_sql
│   ├── symbolic_reasoning.py           # execute_asp, compile_nl_to_asp
│   ├── compute.py                      # execute_python, compute_statistics
│   ├── search.py                       # web_search, rag_search
│   └── introspection.py               # get_kg_schema, get_partition_metadata, list_partitions, get_domain_context
└── tests/
    ├── test_cognition_agent.py         # Integration tests (full loop)
    ├── test_tools.py                   # Unit tests per tool
    └── test_guardrails.py              # Guardrail validation tests
```

### 9.2 File-by-File Specification

#### `__init__.py` — Public API
```python
from app.agents.cognition_agent.cognition_agent import CognitionAgent
from app.agents.cognition_agent.cognition_config import (
    CognitionConfig,
    CognitionResult,
    GroundingData,
    GroundingDataFormat,
    ToolTraceEntry,
)

__all__ = [
    "CognitionAgent",
    "CognitionConfig",
    "CognitionResult",
    "GroundingData",
    "GroundingDataFormat",
    "ToolTraceEntry",
]
```

#### `cognition_agent.py` — Main Agent Loop (~250 lines)
```python
class CognitionAgent:
    """Autonomous, iterative, tool-using reasoning agent."""
    
    # Key methods:
    async def solve(question, workflow_id, grounding_data, config) -> CognitionResult
    async def stream(question, workflow_id, grounding_data, config) -> AsyncIterator[CognitionEvent]
    def _build_user_message(question, grounding_data) -> str
    def _extract_text(response) -> str
    def _assess_confidence(tool_trace) -> float
    def _manage_context(messages, max_tokens) -> List[dict]
    def _force_final_answer(messages, tool_trace, iteration, config) -> CognitionResult
    def _timeout_result(messages, tool_trace, iteration) -> CognitionResult
    def _serialize_result(result) -> str
    def _count_tokens(response) -> int
```

#### `cognition_config.py` — Data Models (~80 lines)
```python
class GroundingDataFormat(str, Enum):    # TABULAR, JSON, TEXT, CSV
class GroundingData:                      # format, content, description
class CognitionConfig:                    # model, max_iterations, timeout, stream, etc.
class ToolTraceEntry:                     # tool, input, output, iteration, timestamp
class CognitionResult:                    # answer, tool_trace, confidence, cost, etc.
class CognitionEvent:                     # type (thinking/tool_call/tool_result/answer), data
```

#### `cognition_system_prompt.py` — System Prompt Builder (~150 lines)
```python
class CognitionSystemPrompt:
    """Builds domain-aware system prompt from workflow config + KG schema."""
    
    @staticmethod
    def build(workflow_id, tool_definitions) -> str
    
    @staticmethod
    def _build_schema_section(workflow_id) -> str       # Entity types, relationships
    
    @staticmethod
    def _build_partition_section(workflow_id) -> str     # Available partitions + row counts
    
    @staticmethod
    def _build_reasoning_rules() -> str                 # Tool selection heuristics
```

#### `cognition_guardrails.py` — Safety Layer (~100 lines)
```python
class CognitionGuardrails:
    """Safety, cost, and quality guardrails."""
    
    def validate_tool_call(tool_name, tool_input, iteration) -> None  # raises GuardrailError
    def _check_cypher_mutation(cypher) -> None
    def _check_python_safety(code) -> None
    def _enforce_limits(tool_name, tool_input) -> None  # inject LIMIT, cap rows

class GuardrailError(Exception): ...
```

#### `cognition_context_manager.py` — Context Compression (~80 lines)
```python
class CognitionContextManager:
    """Manages context window to prevent token overflow."""
    
    def manage(messages, max_tokens) -> List[dict]
    def _estimate_tokens(messages) -> int
    def _summarize_tool_result(content) -> str
```

#### `tool_registry.py` — Tool Registration (~120 lines)
```python
class CognitionToolRegistry:
    """Registry of all tools available to the Cognition Agent."""
    
    def __init__(self):
        self._tools: Dict[str, CognitionTool] = {}
        self._register_all_tools()
    
    def register(name, func, description, input_schema) -> None
    def get_tool_schemas() -> List[dict]         # Claude API tool format
    def get_definitions() -> List[dict]           # Human-readable descriptions
    async def execute(name, input) -> Any         # Dispatch + execute
    def _register_all_tools(self) -> None         # Registers all 14 tools

@dataclass
class CognitionTool:
    name: str
    func: Callable                    # async tool function
    description: str
    input_schema: dict                # JSON Schema for Claude API
```

### 9.3 Tool Files — What Each Wraps

#### `tools/data_access.py` (~150 lines)

| Function | Wraps | Existing File |
|----------|-------|---------------|
| `execute_cypher(cypher, params, namespace)` | `GPKnowledgeGraph.execute_query()` | `app/kr_engine/knowlegde_graph/knowledge_graph.py` |
| `search_entities(mention, entity_type, top_k)` | `Neo4jEntityIndex.search_fulltext()` + `search_vector()` | `app/kr_engine/knowlegde_graph/neo4j_entity_index.py` |
| `query_partition(partition_name, columns, filters, sort_by, limit)` | Direct `PipelineStructuredOutput` data access | `app/ontology/workflow_ontology.py` |
| `execute_sql(function_id, parameters)` | `OlympusConnector` async | `app/utils/olympus_connector.py` |

```python
# Example implementation:
from app.kr_engine.knowlegde_graph.knowledge_graph import get_gp_knowledge_graph

async def execute_cypher(cypher: str, params: dict = None, namespace: str = None) -> dict:
    kg = get_gp_knowledge_graph()
    results = await kg.execute_query(cypher, parameters=params or {})
    return {"rows": results, "row_count": len(results)}

async def search_entities(mention: str, entity_type: str = None, top_k: int = 5) -> dict:
    index = Neo4jEntityIndex(kg=get_gp_knowledge_graph())
    results = await index.search_fulltext(
        mention=mention,
        namespace="rheem_composition_reasoning_pricing_warranty_supply_chain",
        entity_type=entity_type,
        limit=top_k,
    )
    return {"matches": [{"label": r["label"], "type": r["type"], "score": r.get("score", 0)} for r in results]}
```

#### `tools/symbolic_reasoning.py` (~100 lines)

| Function | Wraps | Existing File |
|----------|-------|---------------|
| `execute_asp(facts, rules, show, solver, num_models, optimization)` | `ASPReasonerFactory.create()` → `ClingoASPReasoner` / `ClingconASPReasoner` | `app/reasoner/deterministic_reasoner.py` |
| `compile_nl_to_asp(text, target_predicates)` | `NL2ASPFactCompiler` | `app/kr_engine/compiler/nl2asp_fact_compiler.py` |

```python
# Example implementation:
from app.reasoner.deterministic_reasoner import ClingoASPReasoner, ClingconASPReasoner

async def execute_asp(
    facts: list, rules: list, show: list = None,
    solver: str = "clingo", num_models: int = 1, optimization: bool = False,
) -> dict:
    reasoner = ClingconASPReasoner() if solver == "clingcon" else ClingoASPReasoner()
    reasoner.add_facts(facts)
    reasoner.add_rules(rules)
    if show:
        reasoner.add_rules([f"#show {s}." for s in show])
    result = await reasoner.execute_reasoning_async(num_models=num_models)
    return {
        "answer_sets": result.models,
        "optimal_model": result.optimal_model if optimization else None,
        "cost": result.cost,
        "satisfiable": result.satisfiable,
    }
```

#### `tools/compute.py` (~120 lines)

| Function | Wraps | Existing File |
|----------|-------|---------------|
| `execute_python(code, variables)` | RestrictedPython / subprocess sandbox | NEW (sandboxed execution) |
| `compute_statistics(data, operation, params)` | `SignalComputation` + numpy/scipy | `app/statistic_engine/signal_computation.py` |

```python
# Example implementation (sandboxed Python):
import ast
import pandas as pd
import numpy as np

async def execute_python(code: str, variables: dict = None) -> dict:
    """Execute Python in a restricted sandbox with pandas/numpy available."""
    # Validate: no dangerous imports/calls (guardrails check happens before this)
    local_vars = {"pd": pd, "np": np}
    if variables:
        for k, v in variables.items():
            if isinstance(v, dict) and "headers" in v and "rows" in v:
                local_vars[k] = pd.DataFrame(v["rows"], columns=v["headers"])
            else:
                local_vars[k] = v
    
    # Capture stdout
    import io, contextlib
    stdout = io.StringIO()
    with contextlib.redirect_stdout(stdout):
        exec(compile(code, "<cognition>", "exec"), {"__builtins__": _SAFE_BUILTINS}, local_vars)
    
    return {"stdout": stdout.getvalue(), "result": local_vars.get("result")}
```

#### `tools/search.py` (~60 lines)

| Function | Wraps | Existing File |
|----------|-------|---------------|
| `web_search(query, focus)` | `ask_perplexity_async()` | `app/llm_engine/search_tools.py` |
| `rag_search(query, collection, top_k)` | `RAGRetriever` | `app/kr_engine/knowledge_discovery/rag_retriever.py` |

```python
# Example implementation:
from app.llm_engine.search_tools import ask_perplexity_async

async def web_search(query: str, focus: str = "web") -> dict:
    result = await ask_perplexity_async(query=query)
    return {"answer": result.get("answer", ""), "sources": result.get("sources", [])}
```

#### `tools/introspection.py` (~150 lines)

| Function | Wraps | Existing File |
|----------|-------|---------------|
| `get_kg_schema(namespace, entity_type)` | `CypherSchemaCompiler` + live Neo4j | `app/kr_engine/knowlegde_graph/cypher_schema_compiler.py` |
| `get_partition_metadata(partition_name)` | Workflow ontology dict | `app/ontology/specific_workflow_ontology/` |
| `list_partitions(domain)` | `WorkflowKGSchemaConfig` | `app/kr_engine/pipeline/knowledge_graph_builder/kg_models/` |
| `get_domain_context(domain)` | `DomainConfig` + ontology | `app/kr_engine/pipeline/knowledge_graph_builder/kg_models/` |

```python
# Example implementation:
async def get_kg_schema(namespace: str = None, entity_type: str = None) -> dict:
    kg = get_gp_knowledge_graph()
    ns = namespace or "rheem_composition_reasoning_pricing_warranty_supply_chain"
    
    # Get entity type counts
    type_counts = await kg.execute_query(
        "MATCH (n:SemanticaEntity {namespace: $ns}) "
        "RETURN n.type AS type, count(n) AS count ORDER BY count DESC",
        parameters={"ns": ns},
    )
    
    # Get relationship types
    rel_types = await kg.execute_query(
        "MATCH (a:SemanticaEntity {namespace: $ns})-[r]->(b:SemanticaEntity {namespace: $ns}) "
        "RETURN type(r) AS rel_type, a.type AS src, b.type AS tgt, count(r) AS count "
        "ORDER BY count DESC",
        parameters={"ns": ns},
    )
    
    return {
        "entity_types": type_counts,
        "relationship_types": rel_types,
    }
```

### 9.4 Tool Category → File Mapping Summary

```
┌──────────────────────────┬──────────────────────────┬────────────────────────────────────────────┐
│ Tool Category            │ File                     │ Tools                                      │
├──────────────────────────┼──────────────────────────┼────────────────────────────────────────────┤
│ 1. DATA ACCESS           │ tools/data_access.py     │ execute_cypher                             │
│                          │                          │ search_entities                            │
│                          │                          │ query_partition                            │
│                          │                          │ execute_sql                                │
├──────────────────────────┼──────────────────────────┼────────────────────────────────────────────┤
│ 2. SYMBOLIC REASONING    │ tools/symbolic_reasoning │ execute_asp                                │
│                          │   .py                    │ compile_nl_to_asp                          │
├──────────────────────────┼──────────────────────────┼────────────────────────────────────────────┤
│ 3. COMPUTE ENGINE        │ tools/compute.py         │ execute_python                             │
│                          │                          │ compute_statistics                         │
├──────────────────────────┼──────────────────────────┼────────────────────────────────────────────┤
│ 4. SEARCH & RETRIEVAL    │ tools/search.py          │ web_search                                 │
│                          │                          │ rag_search                                 │
├──────────────────────────┼──────────────────────────┼────────────────────────────────────────────┤
│ 5. INTROSPECTION         │ tools/introspection.py   │ get_kg_schema                              │
│                          │                          │ get_partition_metadata                      │
│                          │                          │ list_partitions                            │
│                          │                          │ get_domain_context                         │
└──────────────────────────┴──────────────────────────┴────────────────────────────────────────────┘

Total: 5 categories, 5 files, 14 tools
Each tool: ~20-40 lines (thin wrapper around existing code)
Total new code: ~800 lines across tool files
```

### 9.5 Dependency Map — What Each File Imports

```
cognition_agent.py
├── cognition_config.py          (CognitionConfig, CognitionResult, GroundingData, ToolTraceEntry)
├── cognition_system_prompt.py   (CognitionSystemPrompt)
├── cognition_guardrails.py      (CognitionGuardrails, GuardrailError)
├── cognition_context_manager.py (CognitionContextManager)
├── tool_registry.py             (CognitionToolRegistry)
│   ├── tools/data_access.py
│   │   ├── app.kr_engine.knowlegde_graph.knowledge_graph  (GPKnowledgeGraph)
│   │   ├── app.kr_engine.knowlegde_graph.neo4j_entity_index (Neo4jEntityIndex)
│   │   └── app.utils.olympus_connector                    (OlympusConnector)
│   ├── tools/symbolic_reasoning.py
│   │   ├── app.reasoner.deterministic_reasoner            (ClingoASPReasoner, ClingconASPReasoner)
│   │   └── app.kr_engine.compiler.nl2asp_fact_compiler    (NL2ASPFactCompiler)
│   ├── tools/compute.py
│   │   ├── pandas, numpy                                  (sandboxed)
│   │   └── app.statistic_engine.signal_computation        (SignalComputation)
│   ├── tools/search.py
│   │   ├── app.llm_engine.search_tools                    (ask_perplexity_async)
│   │   └── app.kr_engine.knowledge_discovery.rag_retriever (RAGRetriever)
│   └── tools/introspection.py
│       ├── app.kr_engine.knowlegde_graph.cypher_schema_compiler (CypherSchemaCompiler)
│       └── app.kr_engine.pipeline.knowledge_graph_builder.kg_models (WorkflowKGSchemaConfig)
└── anthropic (AsyncAnthropic) OR litellm
```

API registration:
```
app/services/v1/api_routers.py          # Add /cognition-agent/* endpoints
app/services/api_models.py              # Add request/response Pydantic models
```

---

## 10. Integration with Existing System

### 10.1 Tool Wrappers (Thin Adapters)

Each tool is a thin adapter that calls existing code. No duplication.

```python
# Example: execute_cypher wraps existing GPKnowledgeGraph

from app.kr_engine.knowlegde_graph.knowledge_graph import get_gp_knowledge_graph

async def execute_cypher(cypher: str, params: dict = None, namespace: str = None) -> dict:
    """Execute Cypher against Neo4j. Returns rows as list of dicts."""
    kg = get_gp_knowledge_graph()
    results = await kg.execute_query(cypher, parameters=params or {})
    return {
        "rows": results,
        "row_count": len(results),
    }
```

```python
# Example: search_entities wraps existing Neo4jEntityIndex

from app.kr_engine.knowlegde_graph.neo4j_entity_index import Neo4jEntityIndex

async def search_entities(mention: str, entity_type: str = None, top_k: int = 5) -> dict:
    """Search for entities by name using fulltext + vector."""
    index = Neo4jEntityIndex(kg=get_gp_knowledge_graph())
    results = await index.search_fulltext(
        mention=mention,
        namespace="rheem_composition_reasoning_pricing_warranty_supply_chain",
        entity_type=entity_type,
        limit=top_k,
    )
    return {
        "matches": [
            {"label": r["label"], "type": r["type"], "score": r.get("score", 0)}
            for r in results
        ]
    }
```

### 10.2 Escalation Integration

```python
# In qa_base_pipeline.py — add escalation hook

class BaseQAPipeline(ABC):
    COGNITION_ESCALATION_THRESHOLD = 0.4  # confidence below this → escalate
    
    async def _maybe_escalate_to_cognition(
        self,
        question: str,
        answer: str,
        evidence: List[PipelineStructuredOutput],
        confidence: float,
    ) -> Optional[str]:
        """Escalate to Cognition Agent if standard pipeline confidence is low."""
        if confidence >= self.COGNITION_ESCALATION_THRESHOLD:
            return None
        
        from app.agents.cognition_agent import CognitionAgent, GroundingData
        
        agent = CognitionAgent()
        result = await agent.solve(
            question=question,
            grounding_data=GroundingData(
                format="text",
                content=f"Previous pipeline answer (low confidence {confidence:.2f}):\n{answer}",
                description="Answer from standard pipeline — verify and improve",
            ),
            config=CognitionConfig(escalation_source="qa_pipeline"),
        )
        return result.answer
```

### 10.3 Router Integration

```python
# In router_agent.py — add cognition routing

class RouterAgent:
    def route(self, question: str) -> RouterDecision:
        decision = self._standard_route(question)
        
        # Flag for cognition agent eligibility
        decision.cognition_eligible = (
            decision.reasoning_level >= ReasoningLevel.L3
            or decision.task_type == TaskType.ANALYSIS
        )
        return decision
```

---

## 11. Implementation Phases

### Phase 0: Foundation (3 days)
- [ ] Create `app/agents/cognition_agent/` directory structure
- [ ] Implement `CognitionConfig`, `CognitionResult`, `GroundingData` data models
- [ ] Implement `CognitionToolRegistry` (tool registration + JSON schema generation)
- [ ] Implement `CognitionGuardrails`

### Phase 1: Core Agent Loop + Basic Tools (4 days)
- [ ] Implement `CognitionAgent.solve()` — the main agent loop
- [ ] Implement tools: `execute_cypher`, `search_entities`, `get_kg_schema`
- [ ] Implement tools: `query_partition`, `get_partition_metadata`, `list_partitions`
- [ ] Implement `CognitionSystemPrompt.build()`
- [ ] Test: single-domain KG queries end-to-end

### Phase 2: Compute + Symbolic Tools (3 days)
- [ ] Implement `execute_python` with sandboxed execution
- [ ] Implement `compute_statistics`
- [ ] Implement `execute_asp`, `compile_nl_to_asp`
- [ ] Implement `get_domain_context`
- [ ] Test: arithmetic verification, cross-domain queries

### Phase 3: Search + Grounding Data (2 days)
- [ ] Implement `web_search`, `rag_search`
- [ ] Implement `execute_sql` (Olympus connector)
- [ ] Implement grounding data handling (`solve_with_data`)
- [ ] Test: queries with user-provided data

### Phase 4: API + Streaming (3 days)
- [ ] Add REST endpoints to `api_routers.py`
- [ ] Implement SSE streaming (`CognitionAgent.stream()`)
- [ ] Add Pydantic request/response models
- [ ] Implement context window management
- [ ] Test: API integration, streaming

### Phase 5: Escalation + Optimization (2 days)
- [ ] Wire escalation from `BaseQAPipeline`
- [ ] Add cognition routing to `RouterAgent`
- [ ] Implement cost tracking and token counting
- [ ] Add caching for repeated tool calls within a session
- [ ] Test: escalation flow, cost limits

### Phase 6: Evaluation + Hardening (3 days)
- [ ] Run against 44-question Rheem benchmark
- [ ] Compare accuracy: Cognition Agent vs existing pipeline
- [ ] Tune system prompt based on failure analysis
- [ ] Add retry/recovery patterns for common failure modes
- [ ] Production readiness: logging, monitoring, error reporting

---

## 12. Example Reasoning Traces

### Example 1: Cross-Domain Supply Chain + Risk Analysis

**Question:** "Which suppliers have the highest risk scores AND supply to plants with the longest average lead times?"

```
Iteration 0 — THINK: I need supplier risk scores AND lead times. These are in different KG relationships.
  → search_entities("supplier", entity_type="SUPPLIER") → 6,088+ suppliers exist

Iteration 1 — ACT: Get top-risk suppliers
  → execute_cypher("MATCH (s:SemanticaEntity {type:'SUPPLIER', namespace:$ns})
     WHERE s.evaluation_risk_score_raw_score IS NOT NULL
     RETURN s.label, s.evaluation_risk_score_raw_score, s.vendor_name
     ORDER BY s.evaluation_risk_score_raw_score DESC LIMIT 20")
  → 20 rows

Iteration 2 — ACT: Get lead times for these suppliers
  → execute_cypher("MATCH (s:SemanticaEntity {type:'SUPPLIER'})-[r:SUPPLIES_TO]->(p:SemanticaEntity {type:'PLANT'})
     WHERE s.label IN $supplier_ids
     RETURN s.label, s.vendor_name, p.label AS plant, r.full_lead_time
     ORDER BY r.full_lead_time DESC")
  → 47 rows

Iteration 3 — ACT: Compute combined ranking
  → execute_python(code="""
     import pandas as pd
     suppliers = pd.DataFrame(risk_data)
     lead_times = pd.DataFrame(lt_data)
     merged = suppliers.merge(lead_times, on='supplier_id')
     merged['combined_score'] = merged['risk_score_norm'] + merged['lead_time_norm']
     result = merged.nlargest(10, 'combined_score')
     print(result.to_markdown())
     """)

Iteration 4 — ANSWER: "The 10 highest-risk suppliers with longest lead times are..."
```

### Example 2: Query with Grounding Data

**Question:** "Given this sales forecast, which markets should we prioritize for Scenario 4 pricing?"
**Grounding data:** Q3 2026 forecast table (market, forecast_units, growth_rate)

```
Iteration 0 — THINK: I have forecast data. I need to cross-reference with Scenario 4 KG data.
  → execute_cypher("MATCH (s:SemanticaEntity {type:'SKU'})-[r:SOLD_IN_MARKET]->(m:SemanticaEntity {type:'MARKET'}),
     (s)-[:PRICED_UNDER_SCENARIO]->(sc:SemanticaEntity {type:'PRICING_SCENARIO'})
     WHERE sc.label CONTAINS 'Scenario 4'
     AND r.source_partition CONTAINS 'scenario_4'
     RETURN m.label AS market,
       SUM(r.expected_company_grossprofit) AS total_expected_profit,
       SUM(r.current_company_grossprofit) AS total_current_profit,
       COUNT(s) AS sku_count")
  → 18 markets with profit projections

Iteration 1 — ACT: Join forecast with scenario data
  → execute_python(code="""
     forecast_df = pd.DataFrame(grounding_data)
     scenario_df = pd.DataFrame(kg_results)
     merged = forecast_df.merge(scenario_df, on='market', how='inner')
     merged['profit_uplift'] = merged['total_expected_profit'] - merged['total_current_profit']
     merged['growth_adjusted_uplift'] = merged['profit_uplift'] * (1 + merged['growth_rate'])
     result = merged.sort_values('growth_adjusted_uplift', ascending=False)
     print(result[['market','forecast_units','growth_rate','profit_uplift','growth_adjusted_uplift']].to_markdown())
     """)

Iteration 2 — ANSWER: "Based on cross-referencing your Q3 forecast with Scenario 4 projections,
  the top 3 priority markets are: DALLAS (22% growth + $X uplift), JACKSONVILLE (15% growth + $Y uplift)..."
```

### Example 3: Symbolic Reasoning (Optimization)

**Question:** "Given a $5M budget, what's the optimal allocation across our top 10 spend categories to minimize supply chain risk?"

```
Iteration 0 — THINK: This is a constraint optimization problem. I need spend category data + risk scores.
  → execute_cypher("MATCH (s:SemanticaEntity {type:'SUPPLIER'})-[r:SUPPLIES_CATEGORY]->(c:SemanticaEntity {type:'SPEND_CATEGORY'})
     RETURN c.label, c.category_description,
       SUM(r.transaction_spend) AS total_spend,
       AVG(s.evaluation_risk_score_raw_score) AS avg_risk
     ORDER BY total_spend DESC LIMIT 10")
  → 10 categories with spend and risk

Iteration 1 — ACT: Formulate as ASP optimization
  → execute_asp(
     facts=["category(cat1,'Steel',2100000,72).", "category(cat2,'Plastics',1800000,45).", ...],
     rules=[
       "1 { allocate(C, 100..5000000) } 1 :- category(C,_,_,_).",
       ":- #sum { V,C : allocate(C,V) } > 5000000.",
       "#minimize { R*V/1000,C : allocate(C,V), category(C,_,_,R) }."
     ],
     solver="clingcon",
     optimization=True)
  → Optimal allocation found

Iteration 2 — ACT: Format and verify
  → execute_python(verify allocation sums to $5M, compute risk reduction %)

Iteration 3 — ANSWER: "Optimal allocation: Steel $1.8M, Plastics $900K, ... Total risk score minimized by 23%."
```

---

## 13. Production Considerations

### 13.1 Cost Model
| Model | Input/1M tokens | Output/1M tokens | Typical query cost |
|-------|-----------------|-------------------|--------------------|
| Claude Sonnet 4.6 | $3 | $15 | $0.03–0.10 (3-8 iterations) |
| Claude Opus 4.6 | $15 | $75 | $0.15–0.50 (3-8 iterations) |

Default to Sonnet. Escalate to Opus only for L4 queries or when Sonnet's answer is low-confidence.

### 13.2 Latency Budget
| Component | Typical | Max |
|-----------|---------|-----|
| Claude API call | 1-3s | 10s |
| Neo4j Cypher execution | 50-500ms | 5s |
| Python execution | 10-100ms | 30s |
| ASP/Clingo execution | 100ms-5s | 60s |
| Web search | 2-5s | 15s |
| **Total (5-iteration query)** | **8-20s** | **120s** |

### 13.3 Observability
- **Tool trace**: Every tool call logged with input/output/timing
- **Token tracking**: Per-iteration and total token counts
- **Cost tracking**: Per-query cost estimation
- **Error classification**: Distinguish tool errors, guardrail blocks, timeouts, LLM refusals
- **Structured logging**: JSON logs compatible with Cloud Logging/Datadog

### 13.4 Caching
- **Intra-session**: Cache tool results within a single agent session (same Cypher → same result)
- **Cross-session**: Optional Redis cache for expensive tool results (Cypher, SQL)
- **System prompt**: Use Claude's prompt caching to reduce cost on repeated invocations

### 13.5 Deployment
- Runs on the same Cloud Run instance as the existing reasoning engine
- No new infrastructure — uses existing Neo4j, Redis, Olympus connections
- Claude API via existing LiteLLM / direct Anthropic client
- Memory: agent loop is stateless between requests (no persistent agent state)

---

## 14. Why the Cognition Agent Is Fundamentally Different from the Existing Pipeline

### What You Already Have (and what's missing)

The system today follows a **fixed pipeline pattern**:

```
Query → Router → SemanticQueryPlanner → StructuredDataRetriever → GroundedAnswer
```

This is powerful but **rigid**. The planner decides upfront which partitions to fetch, the retriever fetches them, and the answer engine generates one shot. If the plan is wrong, the answer is wrong. There's no self-correction loop.

**Claude Code's pattern is fundamentally different:**

```
Query → Think → Act → Observe → Think → Act → Observe → ... → Answer
```

The key difference: **the reasoning happens BETWEEN tool calls, not before them**. Claude Code doesn't plan everything upfront — it discovers what it needs as it goes.

### The Core Architectural Insight

```
                         ┌─────────────────────────────────┐
                         │        COGNITION AGENT           │
                         │   (Claude API + Tool Use Loop)   │
                         │                                  │
                         │   System Prompt:                 │
                         │   - Domain ontology              │
                         │   - Tool descriptions            │
                         │   - Reasoning guidelines         │
                         │   - Schema awareness             │
                         └──────────┬────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │          TOOL CALLS           │
                    │    (Claude decides which)      │
    ┌───────────────┼───────────────┼───────────────┼───────────────┐
    │               │               │               │               │
    ▼               ▼               ▼               ▼               ▼
┌────────┐   ┌────────────┐  ┌──────────┐  ┌────────────┐  ┌────────────┐
│  DATA  │   │  SYMBOLIC  │  │  COMPUTE │  │  SEARCH    │  │ INTROSPEC  │
│ ACCESS │   │ REASONING  │  │  ENGINE  │  │  & RAG     │  │  TION      │
│        │   │            │  │          │  │            │  │            │
│ Cypher │   │ Clingo     │  │ Python   │  │ Perplexity │  │ Schema     │
│ SQL    │   │ Clingcon   │  │ Pandas   │  │ Gemini     │  │ Ontology   │
│ KG     │   │ NeurASP    │  │ NumPy    │  │ RAG        │  │ Partitions │
│ Parti- │   │ NL->ASP    │  │ Stats    │  │ Entity     │  │ Workflow   │
│  tions │   │ Probabil.  │  │          │  │  Search    │  │  Context   │
└────────┘   └────────────┘  └──────────┘  └────────────┘  └────────────┘
    │               │               │               │               │
    └───────────────┴───────────────┴───────────────┴───────────────┘
                                    │
                         RESULTS FEED BACK TO CLAUDE
                         (next iteration of the loop)
```

### Where the Cognition Agent Sits in the System

```
                    ┌──────────────────────────────────┐
                    │          API / MCP Entry          │
                    └──────────────┬───────────────────┘
                                   │
                    ┌──────────────▼───────────────────┐
                    │         QUERY CLASSIFIER          │
                    │   (Enhanced RouterAgent)           │
                    │                                   │
                    │  L0-L2: Simple/Moderate           │
                    │    → Existing Pipeline (fast)     │
                    │                                   │
                    │  L3-L4: Complex/Strategic         │
                    │    → Cognition Agent (thorough)   │
                    │                                   │
                    │  Pipeline Failed / Low Confidence │
                    │    → Escalate to Cognition Agent  │
                    └───────┬───────────────┬──────────┘
                            │               │
              ┌─────────────▼──┐    ┌───────▼────────────┐
              │ Existing QA    │    │  COGNITION AGENT    │
              │ Pipeline       │    │                     │
              │ (3-stage)      │    │  Claude + Tools     │
              │                │    │  Iterative Loop     │
              │ Fast, Cheap    │    │  Self-Correcting    │
              │ Deterministic  │    │  Multi-Source       │
              └────────────────┘    └─────────────────────┘
```

The Cognition Agent does NOT replace the existing pipeline. It's a **new tier** — the highest tier, used when:
1. The query is L3/L4 complexity
2. The existing pipeline produces low-confidence results
3. The query requires cross-domain reasoning
4. The query needs arithmetic verification
5. The user explicitly requests deep analysis

### Claude Code Capability Mapping

| Claude Code Capability | Cognition Agent Equivalent |
|----------------------|---------------------------|
| Read files | `get_workflow_data`, `get_partition_metadata` |
| Search codebase | `search_entities`, `search_kg_schema`, `rag_search` |
| Execute bash | `execute_python`, `execute_asp` |
| Write/Edit files | (Not needed — this is a Q&A agent) |
| Multi-step reasoning | **Tool loop with context accumulation** |
| Self-correction | **Retry with different tool on failure** |
| Planning | **Claude plans which tools to call** |
| Verification | **Cross-check: Cypher result vs SQL result vs computation** |

### The System Prompt — The Secret Sauce

This is where 80% of the quality comes from. The system prompt must include:

```
REASONING GUIDELINES:
1. ALWAYS verify numeric answers with computation — never guess
2. If a Cypher query returns empty, try entity search first to verify names
3. For "how many" questions, use COUNT in Cypher, then verify with data
4. For "compare" questions, retrieve BOTH sides before answering
5. For "why" questions, gather evidence from multiple sources
6. Cross-check: if KG says X and tabular data says Y, investigate

TOOL SELECTION HEURISTICS:
- Simple entity lookup → search_entities + execute_cypher
- Aggregation questions → query_partition + execute_python
- Optimization questions → execute_asp (Clingo/Clingcon)
- Cross-domain questions → multiple execute_cypher calls
- Trend questions → query_partition + compute_statistics
- External context → web_search (last resort)
```

---

## 15. Honest Review — Does This Make Sense?

### 15.1 Does it make sense to build the Cognition Agent?

**Yes, absolutely.** Four reasons:

**1. The existing pipeline has a structural ceiling.** The 3-stage pipeline (`Plan → Retrieve → Answer`) makes ONE plan upfront. If that plan is wrong, the answer is wrong. There's no recovery loop. The Cognition Agent fixes this by reasoning BETWEEN tool calls — discovering what it needs as it goes, not guessing upfront.

**2. All the hard pieces are already built.** Neo4j + entity linker + Cypher templates + ASP/Clingo + Snowflake connector + web search + LLM integration + transformer embeddings. The Cognition Agent is NOT building new capabilities — it's a **new way to compose existing ones**. Every tool is a thin wrapper (~20 lines) around code that already works in production. The actual new code is the agent loop (~200 lines), the tool registry (~100 lines), and the system prompt (~50 lines). This is a 2-3 week project, not a 2-3 month project.

**3. Cross-domain questions are the biggest gap today.** "Which suppliers have highest risk AND supply to plants with the longest average lead times?" requires joining data from `edp_supplier_data` and `edp_lead_time`. The current pipeline can't do this because each QA pipeline is single-domain. The Cognition Agent can call `execute_cypher` multiple times across domains, `execute_python` to join results, and `compute_statistics` to rank them.

**4. The cost is reasonable for the value.** A typical Sonnet query with 5 iterations costs ~$0.03-0.10. Even Opus at 8 iterations is ~$0.15-0.50. For L3/L4 enterprise queries where accuracy matters — a question that drives a $5M pricing decision — this cost is negligible. And by routing L0-L2 queries to the existing pipeline (fast, ~$0.01), the average cost stays low.

### 15.2 Do I think it will be very strong and powerful?

**Yes — but with honest caveats.**

#### Strengths (why it will be powerful):

**Strength 1 — Tool diversity is the killer feature.** Claude Code is powerful because it has Bash, file system, search — very different tools. The Cognition Agent has Cypher, SQL, ASP/Clingo, Python, web search, RAG. That's an even MORE diverse and domain-specialized toolkit for analytical reasoning. A system that can query a knowledge graph, verify with SQL, optimize with Clingo, and compute with pandas in a single reasoning chain — that's genuinely powerful and rare in the enterprise AI space.

**Strength 2 — Self-correction changes everything.** The iterative loop means empty results don't end the conversation. The agent can `search_entities` to fix entity name spelling, try a different Cypher path, fall back to raw partition data, or even web search for external context. This single capability will dramatically improve answer quality for hard queries — the ones that matter most.

**Strength 3 — Grounding data is a competitive moat.** Accepting user-provided data AND cross-referencing with internal sources is a feature most enterprise AI systems don't have. This turns the system from "answer questions about our data" into "analyze ANY data in the context of our data" — a fundamentally more valuable proposition.

**Strength 4 — Symbolic reasoning integration.** No other agent system has Clingo/Clingcon ASP as a tool. This means the Cognition Agent can solve optimization problems (budget allocation, scheduling, constraint satisfaction) that are mathematically provable — not LLM hallucinations. This is unique.

**Strength 5 — Built on proven infrastructure.** Every tool wraps battle-tested code. The KG has 17K+ entities verified against real data. The entity linker has 3-stage matching (exact → Lucene → HNSW). The ASP reasoners have been running in production. There's no "hope it works" — the tools work; we're just composing them under an agent loop.

#### Honest Caveats (risks to manage):

**Caveat 1 — The system prompt is 80% of the quality.** The architecture is sound, but the system prompt needs aggressive iteration. The initial prompt will produce mediocre tool selections — Claude might call `execute_cypher` with wrong entity names, or skip `search_entities` when it should verify first. Plan for 2-3 rounds of prompt tuning against real queries in Phase 6. This is normal — Claude Code itself went through months of prompt iteration.

**Caveat 2 — Latency is real.** 8-20 seconds per query is fine for deep analysis but too slow for interactive chat. The routing decision (L0-L2 → existing pipeline at 2-5s, L3-L4 → Cognition Agent at 8-20s) is critical. If you send simple lookups through the agent loop, users will complain about speed. The streaming API (SSE) mitigates this — users see thinking + tool calls in real-time, which feels faster than a silent 15-second wait.

**Caveat 3 — Context window management matters.** A complex query with 10 tool calls can accumulate 50K+ tokens of context. Without compression, you'll hit limits or degrade quality (Claude attends less to early context in very long conversations). The context manager (Section 5.2) is not optional — it's required for production quality.

**Caveat 4 — Python sandbox security.** The `execute_python` tool is the highest-risk tool. The guardrails (Section 7) block obvious attacks, but a determined adversary could find bypasses. For production: use RestrictedPython or a subprocess jail with no network access, no file system, and a hard timeout. This is a known solved problem — just don't skip it.

### 15.3 Comparison with Alternatives

| Approach | Pros | Cons | Verdict |
|----------|------|------|---------|
| **Cognition Agent (this plan)** | Uses existing tools, self-correcting, multi-domain, accepts grounding data | Latency 8-20s, needs prompt tuning, API cost | **Build this** |
| **Extend existing pipeline** | Fast, cheap, deterministic | Can't self-correct, single-domain only, no compute verification | Already at ceiling |
| **LangChain/LangGraph agent** | Popular framework, community support | Extra dependency, abstraction overhead, less control over tool execution | Over-engineered for this use case |
| **Custom RAG pipeline** | Simple, well-understood | No symbolic reasoning, no compute, no self-correction | Too limited |
| **Fine-tuned model** | Fast inference, no tool overhead | Can't query live data, expensive to train, stale knowledge | Wrong approach for dynamic data |

### 15.4 Bottom Line

This is NOT "wrap KG Reasoner in a loop." This is a **meta-agent** that sits above the ENTIRE reasoning-engine system — KG, SQL, ASP, Python, web search — and orchestrates them the way Claude Code orchestrates file reads, bash commands, and searches. The existing pipeline stays for fast/cheap queries. The Cognition Agent activates for hard problems that need real reasoning.

The risk is low (thin wrappers around existing code), the effort is moderate (2-3 weeks), and the upside is high (solves an entire class of queries the current system can't handle). **Build it.**
