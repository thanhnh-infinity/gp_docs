# Knowledge Graph Reasoner — Implementation Plan

## Date: 2026-03-19

**Reference:** `03_kg_reasoner_north_star_plan.md` (architecture vision)
**This document:** Hands-on build guide — exact files, classes, methods, data flow, dependencies, test criteria.

---

## Code Structure

### New Files to Create

The KG Reasoner spans two layers: **reasoning primitives** (in `app/reasoner/kg_reasoner/`) and **orchestration** (in `app/agents/reasoning_orchestrator/engine/kg_reasoner/`). This mirrors the existing pattern where `ClingoASPReasoner` (primitive in `app/reasoner/`) is called by `ASPProgramExecutionAgent` (orchestrator in `engine/`).

```
app/
├── reasoner/
│   ├── kg_reasoner/                              # NEW: KG reasoning PRIMITIVES
│   │   ├── __init__.py
│   │   ├── kg_traversal.py                       # k-hop ego-network, beam search, path scoring
│   │   ├── kg_axiom_expander.py                  # inverse/symmetric/transitive query expansion
│   │   ├── kg_confidence_propagator.py           # confidence-weighted path evaluation + filtering
│   │   ├── kg_subgraph_serializer.py             # Subgraph → triple text for LLM prompt
│   │   └── cypher_templates/                     # Cypher template library (parameterized patterns)
│   │       ├── __init__.py
│   │       ├── single_hop.py                     # Single entity → relationship → target
│   │       ├── multi_hop.py                      # 2-3 hop traversal patterns
│   │       ├── aggregation.py                    # SUM, AVG, COUNT, GROUP BY patterns
│   │       └── path_finding.py                   # Shortest path, reachability, variable-length
│   │
│   └── (existing: deterministic_reasoner.py, asp reasoners, etc.)
│
├── agents/reasoning_orchestrator/
│   ├── engine/
│   │   ├── kg_reasoner/                          # NEW: KG reasoning ORCHESTRATION
│   │   │   ├── __init__.py
│   │   │   ├── kg_tier_router.py                 # KG tier classification (extends RouterAgent)
│   │   │   ├── entity_linker.py                  # NL mentions → KG entity IDs (3-stage)
│   │   │   ├── kg_query_planner.py               # Text-to-Cypher (calls templates + axiom expander)
│   │   │   ├── kg_subgraph_retriever.py          # Subgraph extraction (calls kg_traversal + serializer)
│   │   │   ├── kg_reasoning_agent.py             # ReAct agent with KG tools (Tier 3)
│   │   │   ├── kg_global_reasoner.py             # Community summary synthesis (Tier 4)
│   │   │   └── kg_answer_grounding.py            # Post-generation fact-checking
│   │   │
│   │   └── (existing files unchanged)
│   │
│   └── kg_qa_pipeline.py                         # NEW: KG-aware Q&A pipeline (extends BaseQAPipeline)
│
├── kr_engine/
│   ├── knowlegde_graph/
│   │   ├── knowledge_graph.py                    # EXISTING (add helper query methods)
│   │   ├── cypher_schema_compiler.py             # NEW: Schema → LLM-ready descriptor
│   │   └── entity_alias_index.py                 # NEW: Redis-backed alias index
│   │
│   ├── compiler/
│   │   └── kg_to_asp_compiler.py                 # NEW: KG subgraph → ASP facts
│   │
│   └── pipeline/knowledge_graph_builder/         # EXISTING (unchanged)
│
└── reasoner/                                     # (see above — kg_reasoner/ added here)
```

### Layer Separation Principle

```
┌──────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION LAYER  (app/agents/reasoning_orchestrator/engine/)    │
│  Knows about: questions, users, tiers, LLM prompts, answers         │
│  Files: kg_tier_router, entity_linker, kg_query_planner,            │
│         kg_reasoning_agent, kg_global_reasoner, kg_answer_grounding  │
│  Calls ↓                                                             │
├──────────────────────────────────────────────────────────────────────┤
│  REASONING PRIMITIVES LAYER  (app/reasoner/kg_reasoner/)            │
│  Knows about: graphs, nodes, edges, paths, Cypher, axioms           │
│  Does NOT know about: questions, users, LLMs, tiers                 │
│  Files: kg_traversal, kg_axiom_expander, kg_confidence_propagator,  │
│         kg_subgraph_serializer, cypher_templates/*                   │
├──────────────────────────────────────────────────────────────────────┤
│  KG DATA LAYER  (app/kr_engine/)                                     │
│  Knows about: Neo4j, schemas, entities, compilation                  │
│  Files: knowledge_graph.py, cypher_schema_compiler,                  │
│         entity_alias_index, kg_to_asp_compiler                       │
├──────────────────────────────────────────────────────────────────────┤
│  SYMBOLIC REASONING LAYER  (app/reasoner/ — existing)               │
│  Knows about: ASP facts, rules, answer sets, constraints             │
│  Files: ClingoASPReasoner, ClingconASPReasoner, NeurASP, LPMLN      │
└──────────────────────────────────────────────────────────────────────┘
```

### Existing Files Modified

```
app/agents/reasoning_orchestrator/engine/
    router_agent.py           # ADD: relational signal patterns + KGTier enum
    grounded_answer.py        # ADD: execute_cypher_tool (Phase 0)

app/kr_engine/knowlegde_graph/
    knowledge_graph.py        # ADD: get_schema_descriptor(), search_neighbors()

app/kr_engine/pipeline/knowledge_graph_builder/workers/
    knowledge_graph_writer.py # ADD: populate entity alias index on write
```

---

## Phase 0: Quick Win — Raw Cypher Generation (1 week)

### Goal
Working KG reasoner in days. LLM generates Cypher directly — 30-40% error rate expected. Failure data informs Phase 1.

### Step 0.1: Cypher Schema Compiler

**File:** `app/kr_engine/knowlegde_graph/cypher_schema_compiler.py`

```python
"""Compile WorkflowKGSchemaConfig into a compact schema descriptor for LLM prompts."""

from typing import Dict, List, Optional
from app.kr_engine.pipeline.knowledge_graph_builder.kg_models import (
    WorkflowKGSchemaConfig,
    StructuralEntityType,
    StructuralRelationType,
)


class CypherSchemaCompiler:
    """Compiles KG schema into a compact text descriptor for LLM system prompts.

    Produces a ~500-token schema string that tells the LLM:
    - What entity types exist and their key properties
    - What relationship types exist and their direction
    - What structural navigation paths are available
    - What RelationshipAxioms apply (inverse, symmetric, transitive)
    """

    def __init__(self, schema_config: WorkflowKGSchemaConfig):
        self.schema_config = schema_config

    def compile(self, include_samples: bool = False) -> str:
        """Compile schema into LLM-ready text descriptor."""
        ...

    def compile_selective(self, entity_types: List[str]) -> str:
        """Compile only schema elements relevant to specific entity types.
        Reduces token usage by ~60% vs full schema."""
        ...
```

**Dependencies:** `WorkflowKGSchemaConfig` (existing)
**Test:** Schema descriptor includes all 20+ predicates, entity types, and axioms. Fits in <800 tokens.

### Step 0.2: Cypher Execution Tool on GroundedAnswerEngine

**File:** `app/agents/reasoning_orchestrator/engine/grounded_answer.py` (modify)

Add a KG query tool that the LLM can invoke during answer generation:

```python
async def _execute_kg_query(
    self,
    question: str,
    schema_descriptor: str,
    namespace: str,
) -> Optional[Dict[str, Any]]:
    """Let LLM generate and execute Cypher against the KG.

    Phase 0 approach: free-form Cypher generation.
    Returns: {cypher: str, results: List[Dict], error: Optional[str]}
    """
    # 1. Build prompt: schema + question → generate Cypher
    # 2. Execute via GPKnowledgeGraph.execute_query()
    # 3. Return results as structured evidence
    # 4. Log Cypher + success/failure for Phase 1 analysis
    ...
```

**Integration point:** Called from `answer()` when RouterAgent detects KG-relevant question.
**Dependencies:** `GPKnowledgeGraph.execute_query()`, `CypherSchemaCompiler`
**Test:** "Which suppliers provide to Water Heater Division?" → generates valid Cypher → returns supplier names.

### Step 0.3: KG Question Detection in RouterAgent

**File:** `app/agents/reasoning_orchestrator/engine/router_agent.py` (modify)

Add relational signal patterns to Stage 2 scoring:

```python
# NEW patterns for KG-relevant question detection
_KG_ENTITY_PATTERNS = [
    r"\b(supplier|manufacturer|retailer|market|region|plant|dc|sku|store)\b",
    r"\b(distribution center|operating unit|spend category)\b",
]
_KG_RELATIONAL_PATTERNS = [
    r"\b(connected|related|linked|path|supplies|manufactur|competes)\b",
    r"\b(across|between)\s+\w+\s+(and|to)\b",
    r"\b(upstream|downstream|supply chain)\b",
]

# NEW field on RouterDecision
class RouterDecision:
    ...
    kg_relevant: bool = False  # True if relational signals detected
```

**Dependencies:** None (extends existing RouterAgent)
**Test:** "Which suppliers provide to Rheem plants?" → `kg_relevant=True`. "Show me top 10 SKUs by margin" → `kg_relevant=False`.

### Step 0.4: Wire Phase 0 Together

**File:** `app/agents/reasoning_orchestrator/engine/grounded_answer.py` (modify `answer()`)

```python
async def answer(self, *, question, evidence_outputs, ...):
    # Existing: route question
    decision = router.route(question)

    # NEW: if KG-relevant, try Cypher query first
    if decision.kg_relevant and self._kg_schema_descriptor:
        kg_result = await self._execute_kg_query(
            question=question,
            schema_descriptor=self._kg_schema_descriptor,
            namespace=self._kg_namespace,
        )
        if kg_result and kg_result.get("results"):
            # Inject KG results as additional evidence
            evidence_outputs = self._merge_kg_evidence(evidence_outputs, kg_result)

    # Continue with existing answer generation
    ...
```

**Test criteria for Phase 0:**
- Run 30+ real Rheem questions
- Log all generated Cypher (success + failure)
- Measure: success rate, common failure patterns, latency
- Output: failure analysis document → drives Phase 1 template design

---

## Phase 1: Foundation — Entity Linker + Template Cypher (3-4 weeks)

### Step 1.1: Entity Alias Index

**File:** `app/kr_engine/knowlegde_graph/entity_alias_index.py`

```python
"""Redis-backed alias index for fast entity linking.
Populated during KG write, queried during reasoning."""

import logging
from typing import Dict, List, Optional, Tuple

logger = logging.getLogger(__name__)


class EntityAliasIndex:
    """Maps normalized entity name variants to KG entity IDs.

    Storage: Redis hash per namespace.
    Key: entity_aliases:{namespace}
    Field: normalized_label → entity_id|entity_type

    Populated by WorkflowKnowledgeGraphWriter after insertion.
    Queried by EntityLinker during reasoning.
    """

    def __init__(self, redis_client):
        self.redis = redis_client

    async def populate_from_artifact(
        self,
        artifact: "SemanticaGraphArtifact",
        namespace: str,
    ) -> int:
        """Populate alias index from KG artifact. Returns count of aliases added."""
        # For each entity: add normalized label, lowercase, acronym variants
        ...

    async def lookup(
        self,
        mention: str,
        namespace: str,
        entity_type: Optional[str] = None,
    ) -> Optional[Tuple[str, str]]:
        """Lookup entity ID by mention text. Returns (entity_id, entity_type) or None."""
        ...

    async def invalidate(self, namespace: str) -> None:
        """Clear all aliases for a namespace (called on KG rebuild)."""
        ...
```

**Dependencies:** Redis (existing via `RedisRunStore`)
**Integration:** `WorkflowKnowledgeGraphWriter.insert_batched()` calls `populate_from_artifact()` after successful insertion.

### Step 1.2: Entity Linker

**File:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/entity_linker.py`

```python
"""Three-stage entity linker: deterministic → alias → embedding."""

from dataclasses import dataclass
from typing import List, Optional
from enum import Enum


class ResolutionMethod(str, Enum):
    DETERMINISTIC = "deterministic"  # make_entity_id() exact match
    ALIAS = "alias"                  # Redis alias index
    EMBEDDING = "embedding"          # bge-small-en-v1.5 similarity


@dataclass
class LinkedEntity:
    entity_id: str
    entity_type: str
    label: str
    confidence: float
    resolution_method: ResolutionMethod


class EntityLinker:
    """Links NL entity mentions to KG entity IDs.

    Three stages (in order, first match wins):
    1. Deterministic: compute make_entity_id() for plausible types, check Neo4j
    2. Alias: lookup in Redis alias index (normalized variants)
    3. Embedding: cosine similarity via bge-small-en-v1.5 against pre-computed embeddings

    Type disambiguation: if multiple types match, use question's relational context.
    """

    def __init__(
        self,
        alias_index: "EntityAliasIndex",
        kg: "GPKnowledgeGraph",
        namespace: str,
        schema_config: "WorkflowKGSchemaConfig",
    ):
        ...

    async def link(
        self,
        question: str,
        *,
        max_entities: int = 10,
    ) -> List[LinkedEntity]:
        """Extract and link entity mentions from a question."""
        # 1. Extract candidate mentions (NER or pattern-based)
        # 2. For each mention, try deterministic → alias → embedding
        # 3. Type disambiguation using relational context
        ...

    async def link_mention(
        self,
        mention: str,
        *,
        expected_type: Optional[str] = None,
    ) -> Optional[LinkedEntity]:
        """Link a single entity mention."""
        ...
```

**Dependencies:** `EntityAliasIndex`, `GPKnowledgeGraph`, `make_entity_id()`, embedding model
**Test:** "Rheem" → `LinkedEntity(id=gp_entity_..., type=MANUFACTURER, confidence=1.0, method=DETERMINISTIC)`. "Water heater in Jacksonville" → `LinkedEntity(type=MARKET, label=JACKSONVILLE, method=ALIAS)`.

### Step 1.3: KG Query Planner (Template-Based)

**File:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/kg_query_planner.py`

```python
"""Template-based Text-to-Cypher with LLM slot-filling."""

from dataclasses import dataclass
from typing import Any, Dict, List, Optional


@dataclass
class CypherQueryPlan:
    cypher: str
    parameters: Dict[str, Any]
    template_name: str                  # Which template was used
    linked_entities: List["LinkedEntity"]
    confidence: float                   # Plan confidence
    axiom_expansions: List[str]         # Which axioms were applied


class KGQueryPlanner:
    """Generates Cypher queries from NL questions using template slot-filling.

    Architecture:
    1. Entity linking → seed entities with types
    2. Template selection → based on (source_type, intent, target_type)
    3. LLM slot-filling → fill template parameters (NOT free-form Cypher)
    4. Axiom expansion → RelationshipAxiom (inverse, symmetric, transitive)
    5. Confidence filtering → WHERE r.confidence >= threshold

    Fallback: if no template matches, use Phase 0 free-form Cypher with
    self-correction loop (2 retries).
    """

    def __init__(
        self,
        entity_linker: EntityLinker,
        schema_compiler: "CypherSchemaCompiler",
        schema_config: "WorkflowKGSchemaConfig",
    ):
        ...

    async def plan(
        self,
        question: str,
        *,
        linked_entities: Optional[List["LinkedEntity"]] = None,
        min_confidence: float = 0.5,
    ) -> CypherQueryPlan:
        """Generate a Cypher query plan for the question."""
        # 1. Link entities if not provided
        # 2. Detect query intent (lookup, aggregation, comparison, path)
        # 3. Select template
        # 4. LLM fills slots
        # 5. Apply axiom expansion
        # 6. Add confidence filter
        ...

    async def execute_with_retry(
        self,
        plan: CypherQueryPlan,
        kg: "GPKnowledgeGraph",
        namespace: str,
        max_retries: int = 2,
    ) -> Dict[str, Any]:
        """Execute Cypher with self-correction on failure."""
        # Execute → if empty/error → LLM revises → retry
        ...
```

**Dependencies:** `EntityLinker`, `CypherSchemaCompiler`, `WorkflowKGSchemaConfig`, `GPKnowledgeGraph`
**Test:** "What markets does SKU 1002306458 sell in?" → template `single_hop_from_entity` → `MATCH (s:SemanticaEntity {type: "SKU", id: $id})-[:SOLD_IN_MARKET]->(m) RETURN m.label` → returns market names.

### Step 1.4: KG Tier Router

**File:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/kg_tier_router.py`

```python
"""Extends RouterAgent with KG tier classification."""

from enum import IntEnum
from dataclasses import dataclass
from typing import Optional


class KGTier(IntEnum):
    TABULAR = 0          # No KG involvement (existing tabular path)
    CYPHER = 1           # Text-to-Cypher (1-2 hops)
    SUBGRAPH = 2         # Subgraph retrieval + prompt grounding
    ITERATIVE = 3        # ReAct agent with KG tools
    GLOBAL = 4           # Community summary synthesis


@dataclass
class KGRoutingDecision:
    tier: KGTier
    reasoning_level: "ReasoningLevel"  # From existing RouterAgent
    task_type: "TaskType"
    linked_entities: list               # Pre-linked entities (if any)
    relational_score: float             # How "graph-like" the question is
    selected_model: str
    fallback_models: list


class KGTierRouter:
    """Classifies questions into KG tiers (0-4).

    Uses existing RouterAgent L0-L4 + relational signal patterns.
    Relational score = 0 → Tier 0 (tabular).
    Relational score > 0 + L1/L2 → Tier 1 (Cypher).
    Relational score > 0 + L2 → Tier 2 (subgraph).
    Relational score > 0 + L3 → Tier 3 (iterative).
    L4 or (L3 + no specific entities) → Tier 4 (global).
    """

    def __init__(
        self,
        router: "RouterAgent",
        entity_linker: Optional["EntityLinker"] = None,
    ):
        ...

    def classify(self, question: str) -> KGRoutingDecision:
        """Classify question into KG tier."""
        ...

    def escalate(self, current_tier: KGTier, reason: str) -> KGTier:
        """Escalate to next tier when current tier produces insufficient results.

        Escalation chain: Tier 1 → Tier 2 → Tier 3.
        Tier 0 never escalates (tabular is the fallback).
        Tier 4 never escalates (global is the ceiling).

        Reasons for escalation:
        - Cypher returned empty results (Tier 1 → 2)
        - Subgraph insufficient for multi-hop (Tier 2 → 3)
        - Answer confidence below threshold
        """
        ...
```

**Dependencies:** `RouterAgent`, `EntityLinker`
**Test:** "Show top 10 SKUs by margin" → Tier 0. "Which suppliers provide to Water Heater Division?" → Tier 1. "Compare warranty vs pricing across Southeast" → Tier 2. Tier 1 empty result → escalates to Tier 2.

### Step 1.5: KG Q&A Pipeline

**File:** `app/agents/reasoning_orchestrator/kg_qa_pipeline.py`

```python
"""KG-aware Q&A pipeline — extends BaseQAPipeline with KG reasoning tiers."""

from app.agents.reasoning_orchestrator.qa_base_pipeline import BaseQAPipeline


class KGQAPipeline(BaseQAPipeline):
    """Q&A pipeline that reasons over both tabular data AND knowledge graph.

    Extends the existing 3-stage pipeline with KG reasoning:

    Stage 1: KG Tier Classification (new)
        → Tier 0: fall through to existing tabular path
        → Tier 1-4: KG reasoning path

    Stage 2: Evidence Retrieval
        → Tabular: existing StructuredDataRetriever (Tier 0)
        → KG: Cypher / subgraph / iterative / global (Tier 1-4)

    Stage 3: Grounded Answer Generation (existing, reused)
    """

    def __init__(
        self,
        *,
        schema_config: "WorkflowKGSchemaConfig",
        namespace: str,
        **kwargs,
    ):
        super().__init__(**kwargs)
        self.kg_tier_router = KGTierRouter(...)
        self.entity_linker = EntityLinker(...)
        self.kg_query_planner = KGQueryPlanner(...)
        # Tier 2-4 components added in later phases

    async def solve(self, question: str, configuration=None):
        """Override: classify → route to tabular or KG path."""
        kg_decision = self.kg_tier_router.classify(question)

        if kg_decision.tier == KGTier.TABULAR:
            return await super().solve(question, configuration)

        if kg_decision.tier == KGTier.CYPHER:
            return await self._solve_via_cypher(question, kg_decision)

        # Tier 2-4 added in Phase 2-3
        ...

    async def _solve_via_cypher(self, question, decision):
        """Tier 1: entity link → Cypher → execute → ground answer."""
        ...
```

**Dependencies:** All Phase 1 components
**Test:** End-to-end: NL question → tier routing → Cypher generation → Neo4j execution → grounded answer.

---

## Phase 2: Subgraph + Symbolic (2-3 weeks)

### Step 2.1: Subgraph Retriever

**File:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/kg_subgraph_retriever.py`

```python
"""KG-RAG subgraph extraction with structural scoping."""

@dataclass
class RetrievedSubgraph:
    nodes: List[Dict[str, Any]]
    edges: List[Dict[str, Any]]
    structural_context: Dict[str, str]  # {entity_id: "domain/section"}
    serialized: str                      # Triple format for LLM prompt
    token_count: int


class KGSubgraphRetriever:
    """Extracts and serializes subgraphs for LLM prompt grounding.

    1. Domain scoping via WORKFLOW→DOMAIN→SECTION (reuse SemanticQueryPlanner)
    2. k-hop ego-network from seed entities
    3. Predicate filtering (follow only relevant relationships)
    4. Community-aware pruning (token budget)
    5. Serialize as relation-triples with confidence + extraction_type
    """

    async def retrieve(
        self,
        seed_entities: List["LinkedEntity"],
        *,
        max_hops: int = 2,
        predicate_filter: Optional[List[str]] = None,
        token_budget: int = 4000,
        namespace: str,
    ) -> RetrievedSubgraph:
        ...
```

### Step 2.2: KG-to-ASP Compiler

**File:** `app/kr_engine/compiler/kg_to_asp_compiler.py`

```python
"""Compile KG subgraphs into ASP facts for symbolic reasoning."""


class KGToASPCompiler:
    """Bridges the KG and symbolic reasoning.

    Compiles a RetrievedSubgraph into ASP facts:
    - entity(ID, Type, Label).
    - attribute(ID, PropertyName, Value).
    - rel(SubjectID, Predicate, ObjectID).
    - rel_confidence(SubjectID, Predicate, ObjectID, ConfidenceInt).

    Scoping: only compiles the subgraph, NOT the full 15K-entity KG.
    """

    def compile(
        self,
        subgraph: "RetrievedSubgraph",
        *,
        include_confidence: bool = True,
        include_attributes: bool = False,
    ) -> str:
        """Return ASP program text (facts only, no rules)."""
        ...

    def compile_with_rules(
        self,
        subgraph: "RetrievedSubgraph",
        rules: List[str],
        constraints: List[str],
    ) -> str:
        """Return complete ASP program: facts + rules + constraints."""
        ...
```

### Step 2.3: Wire Tier 2 into KGQAPipeline

Add `_solve_via_subgraph()` to `KGQAPipeline`:

```python
async def _solve_via_subgraph(self, question, decision):
    """Tier 2: entity link → subgraph → serialize → ground answer."""
    entities = decision.linked_entities
    subgraph = await self.subgraph_retriever.retrieve(
        seed_entities=entities, namespace=self.namespace,
    )
    # Inject serialized subgraph as evidence
    ...
```

---

## Phase 3: Advanced Reasoning (3-4 weeks)

### Step 3.1: ReAct Reasoning Agent (Tier 3)

**File:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/kg_reasoning_agent.py`

```python
"""ReAct agent for iterative KG reasoning with tool use."""

from dataclasses import dataclass, field
from typing import Any, Dict, List


@dataclass
class ReasoningStep:
    step_number: int
    thought: str           # Agent's reasoning
    tool_name: str         # Which tool was called
    tool_input: Dict       # Tool arguments
    tool_output: Any       # Tool result
    latency_ms: float


@dataclass
class ReasoningTrace:
    trace_id: str
    question: str
    tier: int
    steps: List[ReasoningStep] = field(default_factory=list)
    answer: Optional[str] = None
    total_latency_ms: float = 0
    llm_calls: int = 0
    kg_queries: int = 0
    asp_invocations: int = 0


class KGReasoningAgent:
    """ReAct agent with KG tools for Tier 3 deep reasoning.

    Tools:
    - search_neighbors: traverse KG from entity
    - execute_cypher: run arbitrary Cypher
    - retrieve_community_summary: get community-level context
    - check_constraints_asp: compile subgraph → ASP → verify constraints
    - lookup_entity: entity linking

    Budget: max 8 steps. Each step logged to ReasoningTrace.
    Termination: agent decides "I have enough evidence" or budget exhausted.
    """

    async def reason(
        self,
        question: str,
        seed_entities: List["LinkedEntity"],
        *,
        max_steps: int = 8,
        namespace: str,
    ) -> ReasoningTrace:
        ...
```

### Step 3.2: Global Reasoner (Tier 4)

**File:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/kg_global_reasoner.py`

```python
"""GraphRAG global synthesis from community summaries."""


class KGGlobalReasoner:
    """Tier 4: answer strategic/holistic questions using community hierarchy.

    1. Retrieve all community summaries (pre-built at KG build time)
    2. Map-reduce: summarize per domain → cross-domain synthesis
    3. Optional: ASP constraint checking on synthesis claims
    """

    async def synthesize(
        self,
        question: str,
        *,
        namespace: str,
        max_communities: int = 50,
    ) -> Dict[str, Any]:
        ...
```

---

## Phase 4: Production Hardening (2 weeks)

### Step 4.1: Caching Layer

Integrate 5-level cache into each component:
- L1 (entity linking) → `EntityLinker` checks cache before Neo4j
- L2 (Cypher results) → `KGQueryPlanner.execute_with_retry()` checks cache
- L3 (subgraphs) → `KGSubgraphRetriever.retrieve()` checks cache
- L4 (community summaries) → pre-populated at KG build time
- L5 (full answers) → `KGQAPipeline.solve()` checks cache

### Step 4.2: Audit Logging & Compliance

Every `ReasoningTrace` is immutably persisted for regulatory compliance:

```python
class KGReasonerAuditLogger:
    """Immutable audit logging for KG reasoning.

    Every reasoning request produces an audit record containing:
    - User identity + timestamp
    - Full question text
    - All Cypher queries executed (with parameters)
    - All graph paths traversed (entity IDs + predicates)
    - All LLM calls (model, input/output tokens, prompt/response)
    - Tier selected + escalation history
    - Final answer text
    - Confidence scores + evidence sources

    Storage: Redis (primary) + optional cloud storage (GCS/S3) for long-term.
    Format: JSON Lines, one record per reasoning request.
    Retention: configurable per namespace (default 90 days).
    """

    async def log_trace(
        self,
        trace: ReasoningTrace,
        user_id: str,
        namespace: str,
    ) -> str:
        """Persist trace immutably. Returns audit record ID."""
        ...

    async def export_audit_records(
        self,
        namespace: str,
        start_date: str,
        end_date: str,
    ) -> List[Dict]:
        """Export audit records for regulatory review."""
        ...
```

**Integration:** `KGQAPipeline.solve()` calls `audit_logger.log_trace()` after every reasoning request completes (success or failure).

### Step 4.3: Evaluation Pipeline

**File:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/evaluation.py`

```python
"""KG Reasoner evaluation framework."""

@dataclass
class GoldenQuestion:
    question: str
    expected_tier: KGTier
    expected_entities: List[str]       # Entity IDs that should be linked
    expected_cypher_pattern: str        # Regex pattern for valid Cypher
    expected_answer_contains: List[str] # Substrings in correct answer
    max_latency_ms: float


class KGReasonerEvaluator:
    """Run golden set questions and measure quality metrics."""

    async def evaluate(
        self,
        golden_set: List[GoldenQuestion],
    ) -> Dict[str, Any]:
        """Returns: {faithfulness, completeness, cypher_correctness,
                     latency_p50, latency_p95, hallucination_rate, tier_accuracy}"""
        ...
```

### Step 4.3: Post-Generation Fact Checking

**File:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/kg_answer_grounding.py`

```python
"""Validate that all entities/claims in the answer exist in retrieved evidence."""


class KGAnswerGrounding:
    """Post-generation fact-checking.

    After LLM generates answer, validate:
    1. All entity names mentioned exist in the retrieved subgraph/Cypher results
    2. All numeric claims can be traced to a specific edge property
    3. No relationship claims that don't exist in the KG

    If violations found: flag answer with caveats, or regenerate with constraints.
    """

    async def validate(
        self,
        answer: str,
        evidence: Dict[str, Any],
        linked_entities: List["LinkedEntity"],
    ) -> Dict[str, Any]:
        """Returns: {valid: bool, violations: List, confidence: float}"""
        ...
```

---

## Data Flow Diagrams

### Phase 0 Flow
```
Question → RouterAgent.route() [kg_relevant?]
    → YES: GroundedAnswerEngine._execute_kg_query()
              → CypherSchemaCompiler.compile()
              → LLM generates Cypher (free-form)
              → GPKnowledgeGraph.execute_query()
              → Results as evidence
    → NO:  Existing tabular path (unchanged)
    → GroundedAnswerEngine.answer() → Final answer
```

### Phase 1 Flow (Tier 0 + Tier 1)
```
Question → KGTierRouter.classify()
    → Tier 0: BaseQAPipeline.solve() [existing tabular]
    → Tier 1: EntityLinker.link()
              → KGQueryPlanner.plan() [template + slot-fill]
              → KGQueryPlanner.execute_with_retry()
              → GroundedAnswerEngine.answer()
              → KGAnswerGrounding.validate()
              → Final answer with provenance
```

### Phase 2 Flow (+ Tier 2)
```
    → Tier 2: EntityLinker.link()
              → KGSubgraphRetriever.retrieve() [k-hop + structural scoping]
              → Serialize as triples + confidence
              → Optional: KGToASPCompiler.compile() → ASP verification
              → GroundedAnswerEngine.answer()
              → Final answer with subgraph provenance
```

### Phase 3 Flow (+ Tier 3 + Tier 4)
```
    → Tier 3: EntityLinker.link()
              → KGReasoningAgent.reason() [ReAct loop]
                  Step 1: search_neighbors(entity, predicates)
                  Step 2: execute_cypher(aggregation query)
                  Step 3: check_constraints_asp(subgraph, hypothesis)
                  Step N: ... (max 8 steps)
              → GroundedAnswerEngine.answer()
              → Final answer with reasoning trace

    → Tier 4: KGGlobalReasoner.synthesize()
              → Retrieve community summaries (hierarchical)
              → Map-reduce synthesis
              → GroundedAnswerEngine.answer()
              → Final answer with community citations
```

---

## Dependency Graph

```
Phase 0:
  CypherSchemaCompiler (kr_engine) ──────────┐
  RouterAgent (modify: kg_relevant)          ├──→ GroundedAnswerEngine (modify: execute_cypher_tool)
  GPKnowledgeGraph (existing)  ──────────────┘

Phase 1:
  PRIMITIVES (app/reasoner/kg_reasoner/):
    CypherTemplates ──→ KGAxiomExpander ──→ KGConfidencePropagator
                                │
  DATA (app/kr_engine/):        │
    EntityAliasIndex            │
    CypherSchemaCompiler ───────┘
                                │
  ORCHESTRATION (app/agents/.../engine/kg_reasoner/):
    EntityLinker (calls AliasIndex + embedding) ──→ KGQueryPlanner (calls templates + axiom expander)
                                                         │
    KGTierRouter (calls RouterAgent + EntityLinker) ──────┘──→ KGQAPipeline

Phase 2:
  PRIMITIVES:
    KGTraversal (k-hop, beam search) ──→ KGSubgraphSerializer (triples for LLM)
                                                │
  DATA:                                         │
    KGToASPCompiler ──→ ASPReasoner (existing)  │
                                                │
  ORCHESTRATION:                                │
    KGSubgraphRetriever (calls traversal + serializer) ──→ KGQAPipeline (add Tier 2)

Phase 3:
  ORCHESTRATION:
    KGReasoningAgent (ReAct — calls ALL primitives as tools) ──→ KGQAPipeline (add Tier 3)
    KGGlobalReasoner (calls community summaries) ──→ KGQAPipeline (add Tier 4)

Phase 4:
  Redis cache integration across all layers
  KGAnswerGrounding ──→ post-answer validation
  KGReasonerEvaluator ──→ golden set testing
```

### Call Direction (Layer Boundaries)

```
Orchestration → calls → Primitives → calls → Data Layer → calls → Neo4j/Redis
     │                      │                     │
     │                      │                     └── GPKnowledgeGraph.execute_query()
     │                      │                     └── EntityAliasIndex.lookup()
     │                      │
     │                      └── KGTraversal.k_hop_ego_network()
     │                      └── KGAxiomExpander.expand()
     │                      └── KGConfidencePropagator.filter_by_threshold()
     │                      └── CypherTemplates.single_hop_from_entity()
     │
     └── KGQueryPlanner.plan() → calls primitives → calls data layer
     └── KGSubgraphRetriever.retrieve() → calls primitives → calls data layer
     └── KGReasoningAgent.reason() → calls orchestration + primitives + data
     └── EntityLinker.link() → calls data layer directly

NEVER: Primitives → calls → Orchestration (no upward dependency)
NEVER: Data Layer → calls → Primitives or Orchestration
```

---

## Test Strategy

### Phase 0
- 30+ real Rheem questions manually tested
- Log all Cypher generated → success/failure analysis
- Output: failure patterns document

### Phase 1
- Unit tests per component (entity linker, query planner, tier router)
- Integration test: NL question → Cypher → Neo4j → answer (10+ golden questions)
- Entity linking accuracy > 80% on test set
- Cypher success rate > 90% with templates

### Phase 2
- Subgraph retrieval tests: verify correct k-hop boundaries
- KG-to-ASP compilation tests: verify fact count matches subgraph
- ASP verification tests: known constraint violations detected

### Phase 3
- ReAct agent tests: verify tool selection, step budget, termination
- Global synthesis tests: community summaries produce coherent answers
- End-to-end latency tests: Tier 3 < 20s, Tier 4 < 30s

### Phase 4
- Golden set evaluation: 50+ questions across all tiers
- Cache hit rate > 50%
- Hallucination rate < 2%
- All metrics tracked in CI

---

*This implementation plan is the tactical companion to the north star plan. Follow Phase 0 → 1 → 2 → 3 → 4 in order. Each phase produces a working system — Phase 0 alone gives you a KG reasoner you can demo.*
