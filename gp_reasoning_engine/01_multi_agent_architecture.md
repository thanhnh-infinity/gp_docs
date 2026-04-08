# Multi-Agent Architecture for the Reasoning Engine

## Date: 2026-02-22

---

# PART 1: THE VISION — Why Multi-Agent Makes Sense

## What We Have Today

The Reasoning Engine is a **well-structured symbolic reasoning engine with workflow-specific orchestration**, but it is NOT yet a true multi-agent system.

### What EXISTS

| Component | Status | Detail |
|-----------|--------|--------|
| Single-agent solvers | Complete | Clingo, Clingcon, NeurASP, Probabilistic (LPMLN/Sampling) |
| Static YAML agent registry | Complete | Pre-registered agents in `agent_registry.yml` |
| 6-stage reasoning pipeline | Complete | NL → facts → knowledge retrieval → symbolic reasoning → result |
| 3 workflow-specific Q&A pipelines | Complete | Pricing CI, Pricing Simulator, Warranty Intelligence |
| Factory pattern for reasoner selection | Complete | `ASPReasonerFactory` selects Clingo/Clingcon/NeurASP/Probabilistic |
| Query planning | Complete | LLM-driven + deterministic strategies |
| LLM integration | Complete | LiteLLM (provider-agnostic) |
| OWL/RDF ontology engine | Complete | owlready2, PELLET, SPARQL |
| Knowledge graph | Complete | Neo4j async integration |
| **Knowledge Graph Builder** | **Complete** | **11-phase pipeline: structured + NL extraction → entity resolution → relationship inference → community detection → Neo4j insertion with typed relationships. Domain-aware SemanticaEngine. See `02_workflow_kg_architecture.md`** |
| OSI Engine (Client Ontology) | Complete | YAML → RDF/OWL + Pydantic + SQL |
| **Router Agent** | **Complete** | **Multi-signal complexity scoring (L0-L4), task type classification, (ReasoningLevel × TaskType) → Model routing table, cross-provider fallback chains, Gemini thinking levels** |
| **SIG-PRE API (LLM + Intelligence + Visualization)** | **Complete** | **LLM completion with auto-chunking, Perplexity/Gemini intelligence search, Claude Opus visualization generation** |

### What's MISSING

| Capability | Status | Why It Matters |
|-----------|--------|---------------|
| Parallel multi-agent execution | Missing | Solvers run alone, never together |
| Inter-agent communication | Missing | Components share data via method returns only |
| Dynamic agent composition | Missing | No runtime composition of agents |
| Agent state management | Missing | Stateless per invocation |
| Hierarchical agent delegation | Missing | Flat structure only |
| Reasoning consensus / voting | Missing | No mechanism to merge results from multiple reasoners |
| Agent failure recovery | Partial | Fallback models only, no fallback agents |
| **Symbolic reasoning over KG** | **Ready to build** | **KG is production-grade in Neo4j — need agents that traverse it for formal reasoning** |

### Current Architecture Flow

```
  User Question
       │
       ▼
  Router Agent (L0-L4 complexity scoring)
       │
       ▼
  QA Pipeline (one of 3 workflows)
       │
       ├── 1. Semantic Query Planning (LLM or deterministic)
       ├── 2. Structured Data Retrieval
       └── 3. Grounded Answer Generation (LLM)

  OR

  Reasoning Pipeline (single agent)
       │
       ├── 1. compile_facts_from_contextual_content() — LLM-powered NL→ASP
       ├── 2. retrieve_knowledge() — from KG/DB
       ├── 3. augment_facts_from_knowledge() — convert to ASP facts
       ├── 4. setup_reasoner() — via ASPReasonerFactory
       ├── 5. combine_facts() — merge all facts
       └── 6. solve_reasoning_task() — execute single reasoner

  PARALLEL (offline, not per-query):

  Knowledge Graph Builder (11-phase pipeline)
       │
       ├── Structured extraction (schema-driven, deterministic)
       ├── NL extraction (domain-aware SemanticaEngine, 3-layer)
       ├── Assembly → Entity resolution → Relationship inference
       ├── Schema validation → Community detection
       └── Neo4j insertion (typed relationships, 6-check validation)
       → Produces: ~15K entities, ~80K relationships, typed, confidence-scored
```

**The problem:** One workflow = one pipeline = one reasoner. No composition, no coordination, no hybrid reasoning. The KG is built but not yet consumed by reasoning agents.

**The opportunity:** The production-grade KG in Neo4j is ready. The next step is building agents that reason OVER it — combining KG traversal with symbolic reasoning (Clingo), ontology inference (OWL), and LLM augmentation.

## Why Multi-Agent Is the Natural Next Step

The multi-agent layer is the **natural evolution** of this architecture. We have all the reasoning primitives — what we're missing is the **orchestration layer** that composes them together.

### What a Multi-Agent System Enables

1. **Hybrid reasoning on a single question:**
   - A symbolic agent (Clingo) solves constraints
   - A neural agent interprets ambiguity
   - A knowledge graph agent provides context
   - All working **together** on the same question

2. **Hierarchical task decomposition:**
   - A planning agent decomposes a complex question into sub-questions
   - Specialist agents handle each sub-question
   - A synthesis agent merges results

3. **Verification and validation:**
   - A reasoning agent produces an answer
   - A verification agent checks it against the knowledge graph
   - A consistency agent validates against ontology constraints

4. **The unique IP:**
   No one else has a multi-agent system where the agents are **formal reasoners** (ASP solvers, ontology engines, knowledge graphs), not just LLMs calling tools. This is the differentiator.

---

# PART 2: TECHNOLOGY ASSESSMENT — Google ADK vs. Claude vs. Build Our Own

## Google ADK (Agent Development Kit)

### What It Is

Google ADK is an **open-source, code-first framework for building LLM-powered agents**. Apache 2.0 license. Available in Python, TypeScript, Go, Java.

**Core architecture:**
- `LlmAgent` — wraps an LLM model with instructions and tools
- `SequentialAgent` — runs sub-agents in order
- `ParallelAgent` — runs sub-agents concurrently
- `LoopAgent` — runs sub-agents repeatedly until condition met
- `CustomAgent` — subclass `BaseAgent` for arbitrary logic
- `Runner` — event loop orchestrator
- `Session/State` — conversation threads with scoped key-value state
- `Callbacks/Plugins` — before/after hooks on execution

**Model support:** Model-agnostic via LiteLLM wrapper (100+ providers), but optimized for Gemini.

### What ADK Does Well

| Strength | Detail |
|----------|--------|
| Agent composition | Sequential/Parallel/Loop with CustomAgent escape hatch |
| Tool system | Function tools, MCP, OpenAPI, AgentTool (wrap agent as tool) |
| A2A Protocol | Cross-service agent communication over network boundaries |
| Evaluation | Built-in trajectory + response + safety evaluation |
| Production deployment | Cloud Run, GKE, Vertex AI with OpenTelemetry |
| State management | Scoped prefixes: `temp:`, `user:`, `app:` |
| MCP bidirectional | Both consume and expose MCP tools |

### What ADK Does NOT Have

| Missing Capability | Impact on Us |
|-------------------|-------------|
| **No symbolic reasoning** | No ASP, no Clingo, no constraint solving, no logic programming |
| **No ontology support** | No OWL/RDF, no SPARQL, no description logic |
| **No knowledge graph** | No Neo4j, no graph databases, no structured knowledge |
| **No formal reasoning** | No theorem provers, no SAT solvers, no first-order logic |
| **No structured knowledge representation** | State is flat key-value, memory is conversation search |
| **No explanation/justification** | Cannot explain WHY a decision was made beyond LLM traces |
| **No query generation** | No SQL, no SPARQL, no Cypher generation |

### The Fundamental Mismatch

**ADK assumes agents are LLMs that decide what to do via next-token prediction.**

Our agents are **formal reasoners** — ASP solvers that find provably correct answer sets, ontology engines that perform description logic inference, knowledge graphs that traverse structured relationships. These are not "tools an LLM calls." They are the **primary reasoning engines**, and the LLM is a supporting component.

**If we adopt ADK, our architecture inverts:** Our ASP solvers, ontology engine, and knowledge graph become "tools that an LLM agent calls." This is the **opposite** of our design where formal reasoning drives decisions and LLMs assist. That inversion would **weaken our IP**, not strengthen it.

### Assessment: DO NOT ADOPT

**Reason:** ADK solves a different problem (LLM-agent orchestration) than ours (formal reasoning composition). The orchestration patterns it provides (sequential, parallel, loop) are conceptually simple — 50-100 lines of Python each — and can be implemented directly without adopting the entire framework with its tight coupling and Google Cloud gravity.

**One useful concept from ADK:** The A2A Protocol for exposing our engine as a service to other systems. Worth understanding, not worth adopting the framework for.

---

## Anthropic Claude Ecosystem

### claude-quickstarts

**What it is:** Educational example code showing how to build applications with the Claude API. Five quickstarts (customer support, financial analyst, computer use, browser use, coding agent) plus a minimal (~300 line) reference agent implementation.

**Assessment: ZERO VALUE for us.** Explicitly "NOT an SDK." Educational material for people starting from scratch. We are far beyond this.

### Claude Agent SDK

**What it is:** Production-grade library for building agents with Claude. Pre-built tools (Read, Write, Bash, Grep, etc.), subagent support, hooks, sessions, MCP integration.

**The same fundamental mismatch as ADK:** Assumes agents are LLMs that use tools in a loop. Our agents are formal reasoners. The SDK's agent loop is `while True: call LLM → execute tools → repeat`. Our pipeline is `NL → facts → knowledge retrieval → symbolic reasoning → structured output`. Different computational paradigms.

**Assessment: DO NOT ADOPT as architectural dependency.** Could be useful for building a chat-based front-end that delegates to our engine, but not for the reasoning layer itself.

### MCP (Model Context Protocol) — THE ONE THING WORTH ADOPTING

**What it is:** Open protocol (JSON-RPC 2.0) for connecting LLMs to external systems. Three primitives: Resources (data), Tools (functions), Prompts (templates). Donated to Linux Foundation's Agentic AI Foundation in Dec 2025. Industry standard.

**Why it matters for us:** Our repo already has an empty `app/mcp/mcp_server.py`. If we expose our reasoning engine as an MCP server:

- **Any MCP client** (Claude, ChatGPT, IDEs, custom agents) can invoke our ASP solver, query our ontology, traverse our knowledge graph
- We don't change our architecture — MCP is just an **interface layer**
- Our reasoning engine becomes the **backend brain** that any front-end agent can call
- This is how we productize without coupling to any specific LLM framework

```
  Any MCP Client              Our Reasoning Engine (MCP Server)
  (Claude, GPT, IDE)                |
       |                    ┌───────┼───────┐
       |── MCP call ──────►│  ASP Solver    │
       |                    │  Ontology      │
       |◄── MCP result ───│  Knowledge     │
       |                    │  Graph         │
                            └───────────────┘
```

**Assessment: SERIOUSLY CONSIDER implementing MCP server interface.** Low coupling, high value, standards-based, future-proof.

---

## Side-by-Side Comparison

| Capability | Our Engine | Google ADK | Claude Agent SDK | Our Engine + MCP |
|-----------|-----------|------------|-----------------|-----------------|
| ASP/Clingo solver | Core | None | None | Core + exposed via MCP |
| OWL/RDF ontology | Core | None | None | Core + exposed via MCP |
| Knowledge graph | **Production KG** (15K entities, 80K rels, typed, confidence-scored) | None | None | Core + exposed via MCP |
| **KG Builder** | **11-phase pipeline** (structured + NL, domain-aware, typed rels) | None | None | Core |
| LLM integration | LiteLLM (Vertex AI + OpenAI + 100+ providers) | LiteLLM | Claude-only | LiteLLM |
| **Router Agent** | **Complete** (L0-L4, multi-signal, cross-provider fallback) | Built-in LLM routing | Built-in | Complete |
| Deterministic reasoning | Full | None | None | Full + exposed via MCP |
| Probabilistic reasoning | Full | None | None | Full + exposed via MCP |
| NeurASP | Full | None | None | Full |
| Agent orchestration | Gap | Strong | Moderate | Gap (build our own) |
| External accessibility | API + SIG-PRE endpoints | A2A | Agent SDK | **Any MCP client** |
| Formal correctness | Provable | None | None | Provable |

---

# PART 3: THE RECOMMENDATION — Build Our Own Multi-Agent Layer

## Why Build, Not Buy

```
  DON'T ADOPT              CONSIDER               BUILD OURSELVES
  ──────────              ─────────               ────────────────
  Google ADK              MCP Server              Multi-Agent Orchestration
  Claude quickstarts      (expose our engine      (built on our formal
  Claude Agent SDK         as MCP tools)           reasoning primitives)
```

### Reasons to build our own:

1. **Our agents are fundamentally different.** An ASP solver is not "an LLM with tools." An ontology reasoner is not "a chatbot." ADK/Claude SDK model agents as LLM loops — we need agents as formal reasoning components with guaranteed properties.

2. **The orchestration patterns we need are reasoning-specific:**
   - Symbolic solver produces candidates → neural model evaluates plausibility → knowledge graph validates consistency
   - This is not "sequential agent" or "parallel agent" — this is **hybrid reasoning with formal semantics**

3. **This IS our IP.** A multi-agent orchestrator that composes formal reasoners (not LLMs) is what no one else has. If we use ADK, we're using the same framework as everyone else. If we build our own, we have something unique.

4. **The patterns are not complex to implement.** What we need:
   - Agent interface (capabilities, inputs, outputs)
   - Agent coordinator (dispatch, parallel execution, result synthesis)
   - Inter-agent communication (message passing or shared state)
   - Failure handling (fallback chains, degradation)

### Ideas to take from ADK/Anthropic (concepts, not code):

| Concept | Source | How We'd Use It |
|---------|--------|-----------------|
| Sequential/Parallel/Loop composition | ADK | Implement our own version for reasoning agents |
| A2A Protocol | ADK | For distributed deployment later |
| MCP | Anthropic/Linux Foundation | External interface layer |
| Routing pattern | Anthropic "Building Effective Agents" | Our `RouterAgent` (currently TODO) |
| Orchestrator-Workers pattern | Anthropic | Planning agent decomposes → specialist agents solve |
| Evaluator-Optimizer pattern | Anthropic | Generator + symbolic verifier in iterative loop |

---

## What Our Multi-Agent Layer Needs

### A. Agent Definition & Lifecycle

```python
# Conceptual — not actual code, just the pattern
class ReasoningAgent(ABC):
    """Base class for all reasoning agents."""
    name: str
    capabilities: list[str]           # What this agent can do
    input_schema: type[BaseModel]     # What it accepts
    output_schema: type[BaseModel]    # What it produces

    async def execute(self, input) -> output
    async def validate(self, output) -> bool
    async def explain(self, output) -> str   # Provenance/justification
```

Concrete agents:
- `SymbolicReasoningAgent` — wraps Clingo/Clingcon
- `NeuralReasoningAgent` — wraps NeurASP or LLM fact compilation
- `ProbabilisticReasoningAgent` — wraps LPMLN/Sampling
- `OntologyReasoningAgent` — wraps OWL/RDF inference (PELLET)
- `KnowledgeGraphAgent` — wraps Neo4j queries
- `FactCompilationAgent` — NL → ASP facts (LLM-powered)
- `QueryPlanningAgent` — semantic query planning
- `VerificationAgent` — checks answers against knowledge base

### B. Agent Orchestration Layer

```
  User Question
       │
       ▼
  ┌──────────────────────────────────────────┐
  │         Agent Coordinator                 │
  │                                           │
  │  1. Route question to appropriate agents  │
  │  2. Manage parallel/sequential execution  │
  │  3. Collect and synthesize results        │
  │  4. Handle failures and fallbacks         │
  └──────────────┬────────────────────────────┘
                 │
    ┌────────────┼────────────┬────────────┐
    ▼            ▼            ▼            ▼
  Symbolic    Neural      Knowledge    Verification
  Agent       Agent       Graph Agent  Agent
  (Clingo)    (NeurASP/   (Neo4j)      (Ontology
              LLM)                      consistency)
```

Composition patterns we need:

1. **Sequential Pipeline** — agent A feeds agent B feeds agent C
2. **Parallel Fan-Out** — run multiple agents concurrently, synthesize results
3. **Conditional Routing** — route to specialist agent based on question type
4. **Iterative Refinement** — agent produces result, verifier checks, refine if needed
5. **Fallback Chain** — try symbolic first, fall back to probabilistic, fall back to neural

### C. Inter-Agent Communication

Options:
- **Shared state** (simplest) — agents read/write to a shared context object
- **Message passing** (more flexible) — agents send typed messages to each other
- **Event bus** (most decoupled) — publish/subscribe for broadcast scenarios

Recommendation: Start with **shared state** (like ADK's Session), evolve to message passing when needed.

### D. Key Differentiators vs. ADK/Claude

| Property | Our Multi-Agent | ADK / Claude |
|----------|----------------|--------------|
| Agent type | Formal reasoners (ASP, OWL, KG) | LLMs with tools |
| Correctness | Provable (answer sets, OWL inference) | Best-effort (LLM generation) |
| Explanation | Formal derivation chain | Token trace |
| Composition | Reasoning-specific patterns | Generic sequential/parallel |
| Knowledge | Structured ontology + knowledge graph | Flat key-value state |
| Verification | Formal consistency checking | LLM-as-judge |

---

## ASP Example: Why Client Ontology + Multi-Agent Matters

### The Power of Formal Reasoning with Client Ontology

**GP Knowledge Base — business rule (already exists):**
```prolog
% "Flag any transaction where the price exceeds 10,000"
flag_for_review(T) :- price(T, V), V > 10000.
```

**Client data arrives as new facts:**
```prolog
ss_sales_price(order_42, 15000).
ss_sales_price(order_77, 500).
```

**WITHOUT Client Ontology:**
```prolog
% Rule looks for:       price(T, V)
% Data has:             ss_sales_price(T, V)
%
% No match. Rule never fires.
% order_42 is NOT flagged. ← SILENT FAILURE
```

**WITH Client Ontology (the bridge):**
```prolog
% Client Ontology: "ss_sales_price IS-A price"
price(T, V) :- ss_sales_price(T, V).
```

Now: `flag_for_review(order_42)` is derived. The rule fires because the Client Ontology bridges client terminology to GP's universal vocabulary.

### Multi-Agent Version

In a multi-agent system, this becomes even more powerful:

```
  Question: "Which orders need review?"
       │
       ▼
  Agent Coordinator
       │
       ├── FactCompilationAgent: converts client data → ASP facts
       │   (uses Client Ontology to map ss_sales_price → price)
       │
       ├── SymbolicAgent (Clingo): applies GP rules → flag_for_review(order_42)
       │
       ├── KnowledgeGraphAgent: enriches with context from production KG
       │   (traverses typed relationships: :MANUFACTURED_BY, :SOLD_IN_MARKET, etc.
       │    uses confidence scores, extraction_type, community summaries)
       │
       └── VerificationAgent: checks ontology consistency
           (is the flagging consistent with GP business rules?)

  Result: "Order 42 ($15,000) flagged for review.
           Customer: high credit rating, low risk.
           Rule: GP-RULE-001 (price threshold)"
```

Each agent contributes something unique. The symbolic agent provides **provably correct** results. The knowledge graph provides **context**. The verification agent provides **consistency guarantees**. No single LLM agent can do all of this.

---

# PART 4: ACTIONABLE NEXT STEPS

## Priority 1: MCP Server Interface

Fill in `app/mcp/mcp_server.py` to expose our reasoning capabilities:
- Tool: `solve_asp` — submit ASP program, get answer sets
- Tool: `query_ontology` — SPARQL queries against client/GP ontology
- Tool: `query_knowledge_graph` — Cypher queries against Neo4j
- Tool: `generate_client_ontology` — run OSI engine on YAML input
- Tool: `compile_facts` — NL → ASP fact compilation

**Impact:** Any MCP client can use our engine. Low effort, high value.

## Priority 2: Agent Interface Definition

Define the base `ReasoningAgent` protocol and implement concrete agents wrapping existing components:
- `SymbolicReasoningAgent` wrapping `ClingoASPReasoner`
- `OntologyAgent` wrapping `OwlRdfOntology`
- `KnowledgeGraphAgent` wrapping `GPKnowledgeGraph`
- `FactCompilationAgent` wrapping `NL2ASPFactCompiler`

**Impact:** Clean interfaces for composition. No behavior change yet, just better abstraction.

## Priority 3: Agent Coordinator

Build the orchestration layer:
- Sequential, parallel, conditional composition
- Shared state for inter-agent data
- Result synthesis
- Fallback chains

**Impact:** True multi-agent reasoning. Multiple agents work together on the same question.

## Priority 4: Complete the Router Agent — ✅ DONE (March 2026)

The Router Agent is now fully implemented in `app/agents/reasoning_orchestrator/engine/router_agent.py`:
- **Two-stage routing:** Fast regex exit (<1ms) for obvious cases + multi-signal complexity scoring for ambiguous questions
- **5 reasoning levels:** L0 (Direct Lookup) → L4 (Creative/Strategic)
- **6 task types:** retrieval, math, comparison, analysis, creative, program_execution
- **(ReasoningLevel × TaskType) → Model routing table** with cross-provider alternation
- **Workflow-specific complexity bonuses** (configurable per workflow)
- **Fallback chains:** Level-specific, cross-provider (Gemini ↔ Claude)
- **Gemini thinking levels:** Low/medium/high based on task complexity
- Works with `workflow_id=None` for standalone usage (outside Q&A framework)

## Priority 5: KG-Based Reasoning Agent — NEW

Now that the Knowledge Graph is production-grade in Neo4j (see `02_workflow_kg_architecture.md`), the next agent to build is a `KnowledgeGraphReasoningAgent` that:
- Traverses the typed KG (`:MANUFACTURED_BY`, `:SOLD_IN_MARKET`, etc.) for structured retrieval
- Uses confidence scores and extraction_type for source-aware reasoning
- Combines KG traversal with symbolic reasoning (Clingo) for complex analytical queries
- Leverages community summaries for GraphRAG-style global search
- Uses relationship axioms (inverse_of, symmetric, transitive) for query expansion

**Impact:** Bridges the KG we built with the symbolic reasoning engine — the core of neural-symbolic AI.

## Priority 6: Verification Agent

An agent that validates reasoning results against the knowledge graph and ontology:
- Are the derived facts consistent with the ontology?
- Do the answer sets satisfy all known constraints?
- Does the knowledge graph confirm the relationships?

**Impact:** Formal quality assurance on reasoning outputs. This is something no LLM-based agent system can provide.

---

# PART 5: THE UNIQUE ADVANTAGE

**What neither Google nor Anthropic has:**

Every agent in our system can produce:
- **Provably correct results** (ASP answer sets with formal guarantees)
- **Formally grounded knowledge** (OWL/RDF inference with description logic)
- **Structured semantic understanding** (Client Ontology linking all terminology to GP Base)

No amount of LLM prompt engineering can replicate that.

The multi-agent layer doesn't replace these capabilities — it **composes** them. A planning agent decomposes. A symbolic agent solves. A neural agent interprets. A verification agent validates. A knowledge graph agent enriches. All working together, each contributing what it does best.

**This is the IP:** A multi-agent system where agents are formal reasoners, not chatbots.

```
  Google/Anthropic approach:          Our approach:

  LLM → decides → LLM → decides      Symbolic Agent → proves
        ↓                              Neural Agent → interprets
      tools                            KG Agent → contextualizes
        ↓                              Verification Agent → validates
    "best effort"                      Coordinator → synthesizes

                                       = Provable + Contextual + Verified
```

Build on this strength. Don't dilute it by adopting frameworks designed for a fundamentally different paradigm.

---

# PART 6: WHAT TO STEAL — Adoptable Patterns from ADK & Anthropic

While we should NOT adopt these frameworks as dependencies, there are specific **implementation patterns** worth stealing and adapting for our formal reasoning agents. This section is the practical playbook.

## Decision: Don't Adopt Frameworks, Steal Patterns

```
  FROM GOOGLE ADK — steal:              FROM ANTHROPIC — steal:
  ─────────────────────────             ──────────────────────────
  1. BaseAgent interface                1. Workflow patterns (routing,
     (single abstract method)              orchestrator-workers, evaluator-optimizer)
  2. Sequential/Parallel/Loop           2. MCP tool definitions
     (trivial to reimplement)              (use FastMCP library directly)
  3. Event system with branching        3. Subagent isolation + task contract
  4. Scoped state prefixes              4. Derivation chains
  5. Callback/hook system                  (formal version of "extended thinking")
  6. AgentTool (agent-as-tool)          5. Multi-cycle coordination
  7. A2A Protocol spec                     (shared scratchpad + checkpoint/resume)
     (implement independently)          6. Hook lifecycle events
```

---

## Pattern 1: BaseAgent Interface (from ADK)

### ADK's Pattern
One abstract method (`_run_async_impl`) that yields `Event` objects. Everything else (callbacks, lifecycle, tree navigation) is built on top.

### Our Adaptation

```python
from abc import ABC, abstractmethod
from typing import AsyncGenerator, Optional
from pydantic import BaseModel
from dataclasses import dataclass, field
from enum import Enum
import uuid, time

class ReasoningEventType(Enum):
    FACTS_PRODUCED = "facts_produced"
    RULES_APPLIED = "rules_applied"
    ANSWER_SET_FOUND = "answer_set_found"
    ONTOLOGY_ALIGNED = "ontology_aligned"
    KG_QUERY_RESULT = "kg_query_result"
    LLM_RESPONSE = "llm_response"
    ERROR = "error"
    ESCALATE = "escalate"

@dataclass
class ReasoningEvent:
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    author: str = ""
    timestamp: float = field(default_factory=time.time)
    branch: str = ""                           # Isolation path (from ADK)
    event_type: ReasoningEventType = ReasoningEventType.FACTS_PRODUCED
    content: dict = field(default_factory=dict)

    # Actions (stolen from ADK)
    state_delta: dict = field(default_factory=dict)
    transfer_to_agent: Optional[str] = None
    escalate: bool = False

    # OUR ADDITIONS for formal reasoning
    provenance: Optional[dict] = None          # Which rules/facts produced this
    confidence: Optional[float] = None         # For probabilistic reasoning

class ReasoningContext:
    """Shared context passed between agents."""
    session_state: 'ReasoningState'
    ontology_service: any              # OntologyManager
    kg_service: any                    # GPKnowledgeGraph
    asp_solver: any                    # Clingo instance
    branch: str = ""

class BaseReasoningAgent(ABC):
    name: str
    description: str
    sub_agents: list['BaseReasoningAgent']
    before_callback: Optional[callable] = None
    after_callback: Optional[callable] = None

    @abstractmethod
    async def _run_impl(self, ctx: ReasoningContext) -> AsyncGenerator[ReasoningEvent, None]:
        """THE ONLY METHOD SUBCLASSES IMPLEMENT."""
        ...

    async def run(self, parent_ctx: ReasoningContext) -> AsyncGenerator[ReasoningEvent, None]:
        ctx = self._derive_context(parent_ctx)
        if self.before_callback:
            event = await self.before_callback(ctx)
            if event: yield event
        async for event in self._run_impl(ctx):
            yield event
            if event.escalate:
                return
        if self.after_callback:
            event = await self.after_callback(ctx)
            if event: yield event
```

**Key modification:** ADK events carry conversational content (LLM text). Our events carry **reasoning artifacts** (answer sets, RDF triples, ontology alignments) and **provenance** (which rules fired, which facts were used).

**Verdict: ADAPT.** ~1 day effort. Makes all our pipelines composable.

---

## Pattern 2: Sequential / Parallel / Loop Agents (from ADK)

### Reimplement — They're Trivial (50-120 lines each)

```python
import asyncio

class SequentialReasoningAgent(BaseReasoningAgent):
    """Run sub-agents in order."""
    async def _run_impl(self, ctx):
        for agent in self.sub_agents:
            async for event in agent.run(ctx):
                yield event
                if event.escalate:
                    return

class ParallelReasoningAgent(BaseReasoningAgent):
    """Run sub-agents concurrently."""
    async def _run_impl(self, ctx):
        queue = asyncio.Queue()
        async def run_branch(agent):
            branch_ctx = ctx.derive(branch=f"{ctx.branch}.{agent.name}")
            async for event in agent.run(branch_ctx):
                await queue.put(event)
        async with asyncio.TaskGroup() as tg:
            for agent in self.sub_agents:
                tg.create_task(run_branch(agent))
            while not queue.empty():
                yield await queue.get()

class LoopReasoningAgent(BaseReasoningAgent):
    """Loop until convergence — OUR KEY MODIFICATION."""
    max_iterations: int = 10
    convergence_check: Optional[callable] = None   # <-- ADK doesn't have this

    async def _run_impl(self, ctx):
        for i in range(self.max_iterations):
            prev_state = dict(ctx.session_state)
            for agent in self.sub_agents:
                async for event in agent.run(ctx):
                    yield event
                    if event.escalate:
                        return
            # FORMAL REASONING ADDITION: terminate on convergence
            if self.convergence_check and self.convergence_check(prev_state, ctx.session_state):
                return
```

**Key modification:** ADK's LoopAgent terminates on `escalation` (LLM decides it's done). Ours terminates on **convergence** — when answer sets stop changing, when ontology alignment stabilizes.

**Verdict: REIMPLEMENT.** ~1 day. Do NOT import ADK for this.

### How this maps to our existing pipeline

Our `BaseQAPipeline.solve()` is a hardcoded sequential pipeline. With this pattern:

```python
qa_pipeline = SequentialReasoningAgent(
    name="qa_pipeline",
    sub_agents=[
        ParallelReasoningAgent(
            name="planning",
            sub_agents=[
                SemanticQueryPlannerAgent(name="query_planner"),
                ReasoningProtocolDesignerAgent(name="protocol_designer"),
            ]
        ),
        StructuredDataRetrieverAgent(name="retriever"),
        GroundedAnswerAgent(name="answer_generator"),
    ]
)
```

---

## Pattern 3: Scoped State Prefixes (from ADK)

### ADK's Pattern
State is a dict with scoped prefixes: `app:`, `user:`, `temp:`. Reads check uncommitted deltas first.

### Our Adaptation — Add Reasoning-Specific Scopes

```python
class ReasoningState:
    # ADK scopes (keep these)
    APP_PREFIX = "app:"
    USER_PREFIX = "user:"
    TEMP_PREFIX = "temp:"

    # OUR ADDITIONS — reasoning-specific scopes
    ONTOLOGY_PREFIX = "onto:"
    ASP_PREFIX = "asp:"
    KG_PREFIX = "kg:"
    EVIDENCE_PREFIX = "evidence:"
    PLAN_PREFIX = "plan:"

    def __init__(self):
        self._value = {}
        self._delta = {}

    def __getitem__(self, key):
        return self._delta.get(key, self._value.get(key))

    def __setitem__(self, key, value):
        self._value[key] = value
        self._delta[key] = value

    def commit(self):
        self._delta.clear()
```

**Usage across agents:**
```python
# In SemanticQueryPlanner agent:
ctx.state["plan:selected_sections"] = ["module_1", "module_3"]

# In StructuredDataRetriever agent (reads plan state):
sections = ctx.state["plan:selected_sections"]

# In GroundedAnswer agent (reads evidence state):
evidence = ctx.state["evidence:retrieved_outputs"]
```

**Verdict: STEAL.** ~0.5 day. Directly useful for inter-agent data passing.

---

## Pattern 4: Hook/Callback System (from ADK + Anthropic)

### Combined Pattern — Reasoning-Specific Lifecycle Hooks

```python
class ReasoningHookEvent(str, Enum):
    # Pipeline lifecycle (adapted from Anthropic)
    PRE_FACT_COMPILATION = "pre_fact_compilation"
    POST_FACT_COMPILATION = "post_fact_compilation"
    PRE_KNOWLEDGE_RETRIEVAL = "pre_knowledge_retrieval"
    POST_KNOWLEDGE_RETRIEVAL = "post_knowledge_retrieval"
    PRE_SOLVE = "pre_solve"
    POST_SOLVE = "post_solve"

    # Agent lifecycle (adapted from ADK)
    AGENT_START = "agent_start"
    AGENT_STOP = "agent_stop"
    SUBAGENT_START = "subagent_start"
    SUBAGENT_STOP = "subagent_stop"

    # REASONING-SPECIFIC (our additions)
    UNSATISFIABLE = "unsatisfiable"
    CONSTRAINT_VIOLATION = "constraint_violation"
    TIMEOUT = "timeout"
    PRE_REFINEMENT = "pre_refinement"

class ReasoningHookSystem:
    def __init__(self):
        self._hooks = {}

    def register(self, event, callback, agent_pattern=None):
        self._hooks.setdefault(event, []).append((callback, agent_pattern))

    async def fire(self, event, data, context):
        for callback, pattern in self._hooks.get(event, []):
            if pattern and not fnmatch.fnmatch(context.agent_name, pattern):
                continue
            result = await callback(data, context)
            if result and result.get("decision") == "block":
                return result
        return {}
```

**Example hooks:**
```python
# Validate ASP syntax before solving
async def validate_asp_syntax(data, ctx):
    for fact in data.get("facts", []):
        if not is_valid_asp(fact):
            return {"decision": "block", "reason": f"Invalid: {fact}"}
    return {}

# Auto-relax constraints on unsatisfiability
async def auto_relax_on_unsat(data, ctx):
    return {"suggestion": "remove_weakest_constraint"}

hooks.register(ReasoningHookEvent.PRE_SOLVE, validate_asp_syntax)
hooks.register(ReasoningHookEvent.UNSATISFIABLE, auto_relax_on_unsat, "*/graph_*")
```

**Verdict: STEAL.** ~0.5 day. Excellent for observability and extensibility.

---

## Pattern 5: Evaluator-Optimizer Loop (from Anthropic)

### Anthropic's Pattern
Generator produces output → Evaluator checks → meets criteria? → YES: done / NO: refine and loop.

### Our Adaptation — Iterative Constraint Refinement

This is the **single most powerful pattern** for formal reasoning:

```python
class IterativeReasoningOptimizer:
    async def solve_with_refinement(
        self,
        agent_name: str,
        initial_facts: list[str],
        evaluation_criteria: list[str],   # ASP integrity constraints
        max_iterations: int = 5,
    ) -> ASPReasonerResponse:
        facts = initial_facts.copy()

        for iteration in range(max_iterations):
            # GENERATE — solve ASP program
            result = await self._solve(agent_name, facts)

            # EVALUATE — check answer sets against criteria
            evaluation = self._evaluate(result, evaluation_criteria)

            if evaluation["satisfies_all"]:
                return result

            # REFINE — add constraints that eliminate undesired solutions
            new_constraints = self._generate_refinement(evaluation["violations"])
            facts.extend(new_constraints)

        return result  # Best result even if not fully satisfying
```

**Why this is powerful for ASP:** The "evaluator" can itself be an ASP program. Iterative constraint tightening is a known ASP technique — Anthropic just gave it a clean architectural name.

**Verdict: HIGHLY WORTH ADAPTING.** ~1 day. Core value proposition for formal reasoning.

---

## Pattern 6: Derivation Chains (formal version of "Extended Thinking")

### Anthropic's Pattern
Claude produces `thinking` blocks showing step-by-step reasoning before the final answer.

### Our Adaptation — Formal Derivation Traces

Instead of LLM token traces, produce **provably correct derivation chains:**

```python
@dataclass
class DerivationStep:
    step_number: int
    rule_id: str                # Which rule was applied
    rule_text: str              # The ASP rule as written
    rule_explanation: str       # Human-readable (from AgentRuleAnnotation)
    input_atoms: list[str]      # Body atoms that were true
    derived_atom: str           # Head atom that was derived

@dataclass
class FormalThinkingBlock:
    """Our version of Claude's ThinkingBlock — but provably correct."""
    derivation_chain: list[DerivationStep]
    assumptions_used: dict[str, bool]
    constraints_satisfied: list[str]
    constraints_violated: list[str]
```

**This is our moat:** LLMs produce "I think X because Y" (which can hallucinate). We produce "X is derived from rule R applied to facts F1, F2" (which is formally verified).

**Verdict: HIGHLY WORTH ADAPTING.** ~1 day. Uses our existing `AgentRuleAnnotation` system.

---

## Pattern 7: MCP Server (from Anthropic/Linux Foundation)

### Concrete Implementation — Expose Our Engine as MCP Tools

Use the `FastMCP` library (`pip install "mcp[cli]"`). It's lightweight, not a framework lock-in:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("GP-ReasoningEngine")

@mcp.tool()
async def solve_asp(agent_name: str, facts: list[str] = [], max_models: int = 10) -> str:
    """Solve an ASP problem using the GP Reasoning Engine."""
    pipeline = CommonSymbolicReasoningTaskPipeline(agent_name=agent_name)
    pipeline.set_contextual_facts(facts)
    result = await pipeline.orchestrate()
    return result.model_dump_json()

@mcp.tool()
async def query_ontology(sparql_query: str) -> str:
    """Execute SPARQL query against the GP ontology."""
    # ... wrap OwlRdfOntology.sparql() ...

@mcp.tool()
async def compile_facts_from_text(query: str, agent_name: str) -> str:
    """Convert natural language to ASP facts."""
    # ... wrap NL2ASPFactCompiler ...

@mcp.tool()
async def generate_client_ontology(yaml_content: str) -> str:
    """Run the OSI Engine to generate a Client Ontology from YAML."""
    # ... wrap OSIEngine ...

@mcp.tool()
async def list_reasoning_agents() -> str:
    """List all available reasoning agents with their capabilities."""
    # ... read agent_registry.yml ...
```

**Verdict: USE FastMCP directly.** ~1-2 days. Instant compatibility with Claude Desktop, Claude Code, and any MCP client.

---

## Pattern 8: Multi-Cycle Coordination with Shared Scratchpad (from Anthropic)

### Anthropic's Pattern (from their Multi-Agent Research System)
Lead agent → saves plan to external memory → spawns parallel subagents → subagents store findings → lead synthesizes → decides if more cycles needed → repeat or finalize.

### Our Adaptation — Use Our Existing Redis/Run Store

```python
class MultiCycleReasoningCoordinator:
    def __init__(self, memory: SharedReasoningMemory):
        self.memory = memory
        self.max_cycles = 3

    async def coordinate(self, query, available_agents):
        # PLAN
        plan = self._decompose_query(query, available_agents)
        await self.memory.save_plan(plan)

        for cycle in range(self.max_cycles):
            # DISPATCH parallel subagents
            tasks = self._plan_to_tasks(plan, cycle)
            results = await self.orchestrator.dispatch_parallel(tasks)

            # SAVE to shared memory
            for task, result in zip(tasks, results):
                await self.memory.save_agent_result(task.task_id, result)

            # CHECKPOINT
            await self.memory.checkpoint(f"cycle_{cycle}", {...})

            # EVALUATE completeness
            evaluation = self._evaluate_completeness(query)
            if evaluation["complete"]:
                break

            # REFINE plan for next cycle
            plan = self._refine_plan(plan, evaluation["gaps"])

        return await self._synthesize(query)
```

**Key patterns to steal:** (1) Shared scratchpad via our existing run store, (2) Multi-cycle with gap analysis, (3) Checkpoint/resume for failure recovery.

**Verdict: ADAPT.** ~1-2 days. We already have the Redis infrastructure.

---

## Pattern 9: AgentTool — Wrap Reasoning Agent as Callable Tool (from ADK)

### The Pattern
Wrap any agent as a "tool" that an LLM orchestrator can invoke. The wrapped agent gets its own isolated session.

### Our Adaptation

```python
class ReasoningAgentTool:
    """Wrap any BaseReasoningAgent as a callable tool for LLM agents."""
    def __init__(self, agent: BaseReasoningAgent):
        self.agent = agent

    def get_tool_schema(self) -> dict:
        """Generate OpenAI-compatible function schema."""
        return {
            "type": "function",
            "function": {
                "name": self.agent.name,
                "description": self.agent.description,
                "parameters": self.agent.get_input_schema()
            }
        }

    async def invoke(self, args: dict, parent_state: ReasoningState) -> dict:
        # Create ISOLATED context (key insight from ADK)
        child_state = ReasoningState()
        for k, v in parent_state.items():
            if not k.startswith("temp:"):
                child_state[k] = v

        ctx = ReasoningContext(session_state=child_state)
        results = []
        async for event in self.agent.run(ctx):
            results.append(event)

        # Selectively merge state back to parent
        for event in results:
            if event.state_delta:
                for k, v in event.state_delta.items():
                    if k.startswith("evidence:") or k.startswith("asp:"):
                        parent_state[k] = v

        return {"events": results, "final_content": results[-1].content if results else {}}
```

**Use case:** An LLM-based orchestrator can call our ASP solver, ontology engine, or KG as "tools" — bridging the LLM world with our formal reasoning world.

**Verdict: STEAL.** ~1 day. Bridges LLM orchestration with formal reasoning.

---

## Summary: Priority Ranking of Adoptable Patterns

| # | Pattern | Source | Effort | Impact |
|---|---------|--------|--------|--------|
| 1 | **Hook System** | ADK + Anthropic | 0.5 day | Very High — observability, validation, extensibility |
| 2 | **Evaluator-Optimizer (iterative refinement)** | Anthropic | 1 day | Very High — core formal reasoning value |
| 3 | **Derivation Chains (formal thinking)** | Anthropic | 1 day | High — provably correct reasoning traces (our moat) |
| 4 | **MCP Server** | Anthropic/LF | 1-2 days | High — instant interop with any MCP client |
| 5 | **BaseAgent + Sequential/Parallel/Loop** | ADK | 1-2 days | High — makes pipelines composable |
| 6 | **Scoped State** | ADK | 0.5 day | Medium — cleaner inter-agent data passing |
| 7 | **Multi-Cycle Coordination** | Anthropic | 1-2 days | Medium — needed for complex multi-step reasoning |
| 8 | **AgentTool (agent-as-tool)** | ADK | 1 day | Medium — bridges LLM orchestration with formal reasoning |
| 9 | **A2A Protocol** | ADK | 2 days | Medium — distributed deployment (later) |

**Total estimated effort: ~10-12 days for a complete agent framework tailored to formal reasoning.**

**What NOT to steal:**
- ADK's `Runner` event loop — too coupled to LLM conversation model
- ADK's `Session` persistence — use our existing Redis
- Claude Agent SDK's `ClaudeSDKClient` — specific to LLM turns, not formal reasoning
- Claude's permission system — our agents are deterministic programs, security is at the API layer
- Any framework dependency — steal the patterns, implement them ourselves

---

# PART 4: AGENT IMPLEMENTATION AUDIT (2026-04-08)

## Full Codebase Audit Against the Multi-Agent Architecture Diagram

The architecture diagram (slide 03/06 "Multi-agent reasoning — not a pipeline, a team") defines **11 agents** organized around a central Ontology, split into Symbolic agents (left) and Neural agents (right), with two coordination modes (parallel peer communication and ontology-driven orchestration).

This audit maps every agent to its current implementation status in the codebase.

### Status Legend

| Symbol | Meaning |
|--------|---------|
| COMPLETE | Agent exists as a dedicated class with full functionality |
| PARTIAL | Capability exists as utility functions but no dedicated agent class |
| MISSING | No implementation — empty directory or comments only |

---

### Symbolic Agents (Left Side of Diagram)

#### 1. ASP Reasoner — Logic & constraint satisfaction
- **Status:** COMPLETE
- **File:** `app/reasoner/deterministic_reasoner.py`
- **Class:** `ClingoASPReasoner` (line 242)
- **Base:** `ASPReasoner` (abstract, line 36)
- **Factory:** `ASPReasonerFactory.create(agent_name)` with `solver_type: clingo`
- **Capabilities:** Multi-model solving, cost optimization, temporal reasoning, rule annotation parsing, fact loading, multi-shot grounding
- **Registry:** 20+ agents registered in `app/registry/agent_registry.yml` (graph_coloring, blocksworld, nurse_scheduling, etc.)
- **Cognition Agent tool:** `execute_asp` wraps this

#### 2. SAT Solver — Boolean satisfiability
- **Status:** MISSING
- **What exists:** Clingo can handle SAT-class problems (satisfiability checking), but there is no dedicated standalone SAT solver (e.g., MiniSat, CaDiCaL, Z3 boolean mode)
- **Where it would go:** `app/reasoner/sat_solver.py`
- **Priority:** LOW — Clingo covers most SAT use cases. A dedicated SAT solver adds value only for pure boolean satisfiability at scale (100K+ variables) where Clingo's grounding overhead is prohibitive
- **Cognition Agent tool:** Would be `solve_sat` (not yet planned)

#### 3. CP Solver — Constraint programming
- **Status:** COMPLETE
- **File:** `app/reasoner/deterministic_reasoner.py`
- **Class:** `ClingconASPReasoner` (line 492)
- **Factory:** `ASPReasonerFactory.create(agent_name)` with `solver_type: clingcon`
- **Capabilities:** Integer constraint theory over ASP, budget optimization, numerical constraint satisfaction
- **Example agents:** `numerical_constraints/budget_optimization`, `numerical_constraints/test_clingcon`
- **Cognition Agent tool:** `execute_asp` with `solver="clingcon"` wraps this

#### 4. KG Reasoner — Graph traversal & reasoning
- **Status:** COMPLETE
- **Directory:** `app/agents/reasoning_orchestrator/engine/kg_reasoner/`
- **Key files:**
  - `kg_cypher_engine.py` — Schema-aware Cypher generation + execution (Phase 0 + Phase 1)
  - `kg_query_planner.py` — Template matching, entity linking, relationship direction awareness
  - `entity_linker.py` — 3-stage entity linking (exact → Lucene fulltext → HNSW vector)
  - `kg_tier_router.py` — Routes between Tier 0 (tabular), Tier 1 (Cypher templates), Tier 2+ (future)
- **Template system:** `app/reasoner/kg_reasoner/cypher_templates/template_registry.py` — 25+ domain-agnostic templates
- **Schema compiler:** `app/kr_engine/knowlegde_graph/cypher_schema_compiler.py`
- **Phase 1 complete, Phase 2 (subgraph/iterative) not yet started**
- **Cognition Agent tool:** `execute_cypher` + `search_entities` wraps this

#### 5. Arithmetic Agent — Computation & math
- **Status:** PARTIAL (utility functions, no agent class)
- **What exists:**
  - `app/agents/reasoning_orchestrator/engine/structured_data_retriever.py` — arithmetic expression evaluation for computed fields (+, -, *, /, //, %, **)
  - `app/statistic_engine/signal_computation.py` — growth rate calculations
  - Python stdlib `math` used ad-hoc in various places
- **What's missing:** A dedicated `ArithmeticAgent` class that can:
  - Parse arithmetic expressions from natural language
  - Execute multi-step computations with variable binding
  - Verify LLM-generated numeric answers
  - Handle unit conversions, currency, percentages
- **Where it would go:** `app/reasoner/arithmetic_agent.py`
- **Priority:** HIGH — Numeric verification is critical for enterprise queries. Every pricing/margin/cost question needs arithmetic verification.
- **Cognition Agent tool:** `execute_python` + `compute_statistics` partially covers this

#### 6. Aggregate Agent — Statistics & summaries
- **Status:** PARTIAL (utility functions, no agent class)
- **What exists:**
  - `app/statistic_engine/signal_computation.py` — `calculate_relevancy_score()`, `calculate_volume_score()`, `calculate_sentiment_score()`, `extract_growth_rates()`, source distribution analysis
  - `app/agents/reasoning_orchestrator/engine/structured_data_retriever.py` — aggregation in data retrieval (group_by, sum, avg, count)
  - `AggregationStrategy` in KG Builder schema config — group_by, sum_columns, avg_columns, count_column
- **What's missing:** A dedicated `AggregateAgent` class that can:
  - Accept a dataset + aggregation specification
  - Compute group-by aggregations (SUM, AVG, COUNT, MIN, MAX, MEDIAN, PERCENTILE)
  - Compute rolling windows, time-series decomposition
  - Generate summary statistics with confidence intervals
  - Produce formatted summary tables
- **Where it would go:** `app/reasoner/aggregate_agent.py`
- **Priority:** HIGH — Aggregation queries are the most common enterprise question type ("total spend by category", "average lead time by plant")
- **Cognition Agent tool:** `compute_statistics` + `execute_python` partially covers this

---

### Neural Agents (Right Side of Diagram)

#### 7. ML Agent — Machine learning tasks
- **Status:** PARTIAL (utility functions, no agent class)
- **What exists:**
  - `app/reasoner/ml_engine/similarity.py` — TF-IDF, BM25, LSA, embedding-based similarity computation
  - `app/reasoner/transformer_engine/pre_trained/pretrained_transformer.py` — Pre-trained sentence transformers (bge-small, static-retrieval, MiniLM)
  - `app/kr_engine/pipeline/knowledge_graph_builder/workers/entity_resolver.py` — Embedding-based entity resolution using cosine similarity clustering
- **What's missing:** A dedicated `MLAgent` class that can:
  - Run classification, regression, clustering on demand
  - Feature engineering from structured data
  - Model selection and evaluation
  - Anomaly detection
- **Where it would go:** `app/reasoner/ml_engine/ml_agent.py`
- **Priority:** MEDIUM — valuable for anomaly detection (warranty claims outliers, pricing anomalies) but not blocking for current use cases
- **Cognition Agent tool:** Not yet planned as dedicated tool (covered partially by `execute_python`)

#### 8. DL Agent — Deep learning tasks
- **Status:** PARTIAL (example models, not general)
- **What exists:**
  - `app/reasoner/nn_reasoner.py` — `NeuralASPReasoner` integrating NeurASP (neural + logic)
  - `app/reasoner/models/coinflip_model.py` — `CoinFlipNet` (PyTorch example model)
  - `app/reasoner/neural_network/neural_network.py` — Basic neural network modules
  - `app/external/neurasp/` — Full NeurASP library integration
- **What's missing:** A general `DLAgent` class that can:
  - Load and run pre-trained deep learning models
  - Fine-tune models on domain data
  - Handle vision tasks (document OCR, chart understanding)
  - Time-series forecasting with neural architectures
- **Where it would go:** `app/reasoner/neural_network/dl_agent.py`
- **Priority:** LOW for now — NeurASP integration covers the neural-symbolic use case; standalone DL is not the primary reasoning pattern
- **Cognition Agent tool:** Not yet planned

#### 9. LLM Reasoner — General language reasoning
- **Status:** COMPLETE (distributed across multiple modules)
- **Key files:**
  - `app/llm_engine/llm_utils.py` — LiteLLM wrapper, provider abstraction (OpenAI, Gemini, Claude), auto-chunking, fallback chains
  - `app/llm_engine/model_ontology.py` — Model definitions (`LLMGptModels`, `LLMVertexAIGeminiModels`, `LLMVertexAIClaudeModels`)
  - `app/agents/reasoning_orchestrator/engine/router_agent.py` — Smart routing (L0-L4 × TaskType → Model)
  - `app/agents/reasoning_orchestrator/engine/semantic_query_planner.py` — LLM-driven query planning
  - `app/agents/reasoning_orchestrator/engine/grounded_answer.py` — LLM-driven answer generation with evidence grounding
  - `app/agents/reasoning_orchestrator/engine/asp_program_execution_agent.py` — LLM-driven ASP program modification
  - `app/agents/reasoning_orchestrator/engine/reasoning_protocol_designer.py` — LLM-driven reasoning protocol generation
- **Models available:** GPT-5, GPT-5-mini, GPT-4.1, Gemini 3.0 Flash, Gemini 3.1 Pro, Claude 4.6 Opus, Claude 4.6 Sonnet, Claude 4.5 Haiku
- **Routing:** 2-stage (fast regex exit + multi-signal scoring) → (Level × TaskType) → Model
- **Cognition Agent:** The Cognition Agent itself IS the next-gen LLM Reasoner — Claude with tool use in an iterative loop

#### 10. Transfer LLM — Domain adaptation
- **Status:** MISSING
- **What exists:** Empty directory at `app/reasoner/transformer_engine/transfer_learning/`
- **Intent:** Fine-tune or adapt LLMs to domain-specific vocabulary and reasoning patterns (e.g., Rheem supply chain terminology, pricing optimization language)
- **What's missing:** Everything — no code, no training pipeline, no adapter infrastructure
- **Where it would go:** `app/reasoner/transformer_engine/transfer_learning/transfer_agent.py`
- **Priority:** LOW for now — Prompt engineering + RAG + in-context learning (via system prompts) covers most domain adaptation needs. Fine-tuning becomes valuable when: (a) you have 10K+ domain Q&A pairs, (b) latency is critical (fine-tuned small model vs large prompted model), or (c) cost at scale justifies the training investment
- **Cognition Agent tool:** Not needed — domain knowledge is injected via system prompt

#### 11. Distillation LLM — High-accuracy specific tasks
- **Status:** MISSING
- **What exists:** Comments in `app/reasoner/kg_reasoner/cypher_templates/template_registry.py` mention replacing naive Cypher generation with a "Distillation LLM model"
- **Intent:** Train small, fast, high-accuracy models for specific tasks (e.g., Cypher generation, entity extraction, intent classification) by distilling from large teacher models
- **What's missing:** Everything — no distillation pipeline, no student model training, no evaluation framework
- **Where it would go:** `app/reasoner/transformer_engine/distillation/`
- **Priority:** MEDIUM-LOW — Becomes valuable when: (a) you have a specific task where a small model could replace expensive Opus/Sonnet calls, (b) latency budget is tight, (c) you have labeled training data from production queries. The natural first candidate: **Cypher generation distillation** — train a small model on (question, entity_context) → Cypher pairs collected from Phase 0 LLM outputs.
- **Cognition Agent tool:** Would replace the LLM in `kg_cypher_engine.py` for faster Cypher generation

---

### Central Ontology — Shared Semantic Model

- **Status:** COMPLETE
- **Key files:**
  - `app/ontology/agent_ontology.py` — Agent configuration ontology
  - `app/ontology/signal_ontology.py` — Signal intelligence ontology
  - `app/ontology/workflow_ontology.py` — Workflow domain ontology (`PipelineStructuredOutput`, `WorkflowOutputOntology`, `ASP_PROGRAM_CONTEXT`)
  - `app/ontology/tool_ontology.py` — `ZeusTool` enum (60+ tools)
  - `app/ontology/specific_workflow_ontology/` — Domain-specific (Rheem, Suntory, Bayer, etc.)
  - `app/registry/agent_registry.yml` — Centralized agent registry (20+ agents)
- **Coordination modes:**
  - Mode 1 (Parallel peer communication) — NOT YET IMPLEMENTED. Agents currently run alone, never in parallel.
  - Mode 2 (Ontology-driven orchestration) — PARTIALLY IMPLEMENTED. Router Agent selects models, QA pipelines orchestrate stages, but no dynamic agent composition.

---

### Summary Scorecard

```
SYMBOLIC AGENTS                          NEURAL AGENTS
┌─────────────────────┬──────────┐      ┌─────────────────────┬──────────┐
│ ASP Reasoner        │ COMPLETE │      │ ML Agent            │ PARTIAL  │
│ SAT Solver          │ MISSING  │      │ DL Agent            │ PARTIAL  │
│ CP Solver           │ COMPLETE │      │ LLM Reasoner        │ COMPLETE │
│ KG Reasoner         │ COMPLETE │      │ Transfer LLM        │ MISSING  │
│ Arithmetic Agent    │ PARTIAL  │      │ Distillation LLM    │ MISSING  │
│ Aggregate Agent     │ PARTIAL  │      │                     │          │
└─────────────────────┴──────────┘      └─────────────────────┴──────────┘

                    ┌───────────────────────────┐
                    │   CENTRAL ONTOLOGY        │
                    │   COMPLETE                │
                    │                           │
                    │   Coordination Mode 1:    │
                    │     NOT IMPLEMENTED        │
                    │   Coordination Mode 2:    │
                    │     PARTIAL               │
                    └───────────────────────────┘

SCORE: 4/11 Complete, 4/11 Partial, 3/11 Missing
```

### Priority Matrix for Remaining Work

| Priority | Agent | Why | Effort | Cognition Agent Tool |
|----------|-------|-----|--------|---------------------|
| **P0 (Now)** | KG Reasoner Phase 2 | Subgraph + iterative traversal unlocks complex graph queries | 2-3 weeks | `execute_cypher` (enhanced) |
| **P1 (Next)** | Arithmetic Agent | Numeric verification is critical for enterprise pricing/margin queries | 3-5 days | `execute_python` / `compute_statistics` |
| **P1 (Next)** | Aggregate Agent | Aggregation is the #1 enterprise question pattern | 3-5 days | `compute_statistics` (enhanced) |
| **P2 (Soon)** | Coordination Mode 1 | Parallel agent execution enables hybrid reasoning | 1-2 weeks | Agent loop handles this |
| **P2 (Soon)** | Coordination Mode 2 | Ontology-driven orchestration enables dynamic composition | 1-2 weeks | System prompt handles this |
| **P3 (Later)** | Distillation LLM | Fast Cypher generation model, reduces Opus/Sonnet cost | 2-4 weeks | Replaces LLM in kg_cypher_engine |
| **P3 (Later)** | ML Agent | Anomaly detection, classification on demand | 1-2 weeks | `execute_python` covers basics |
| **P4 (Defer)** | SAT Solver | Clingo handles most SAT; dedicated solver only for 100K+ var scale | 1 week | Low value |
| **P4 (Defer)** | DL Agent | NeurASP covers neural-symbolic; standalone DL not primary pattern | 2-3 weeks | Low value |
| **P4 (Defer)** | Transfer LLM | Prompt engineering + RAG sufficient for now; fine-tuning later | 4-8 weeks | Not needed yet |
