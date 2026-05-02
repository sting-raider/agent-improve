# ORACLE Architecture

**Last Updated:** 2026-05-01

---

## System Architecture Overview

ORACLE is a multi-layered system designed for deep, autonomous research. The architecture combines vector search, graph databases, multi-agent orchestration, and beautiful visualization.

```
┌────────────────────────────────────────────────────────────────┐
│                         USER INTERFACE                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Textual    │  │   Browser    │  │   Hermes     │          │
│  │     TUI      │  │  Viz (WebGL) │  │   Shell      │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
└─────────┼─────────────────┼─────────────────┼──────────────────┘
          │                 │                 │
          ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      ORACLE CORE                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              LangGraph Orchestrator                      │   │
│  │  - SQLite checkpointing (pause/resume)                   │   │
│  │  - Specialist sub-agents                                 │   │
│  │  - Investigation state management                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                  │
│         ┌────────────────────┼────────────────────┐             │
│         ▼                    ▼                    ▼             │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐         │
│  │  MCP Servers │   │  Model Router│   │  Journal     │         │
│  │  - Web Search│   │  - Claude    │   │  - Rolling   │         │
│  │  - Code Exec │   │  - Gemini    │   │    markdown  │         │
│  │  - Graph Qry │   │  - Ollama    │   │              │         │
│  └──────────────┘   └──────────────┘   └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
          │                    │                    │
          ▼                    ▼                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE LAYER                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │    Neo4j     │  │    Qdrant    │  │   SQLite     │           │
│  │  - Knowledge │  │  - Vectors   │  │  - Checkpts  │           │
│  │  - Provenance│  │  - Chunks    │  │  - Ledger    │           │
│  │  - Named Gs  │  │              │  │  - State     │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  INGESTION PIPELINE                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Docling    │  │ infinity-emb │  │  Entity Ext  │           │
│  │  - Parser    │  │  - Embedder  │  │  - qwen3:8b  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. Ingestion Pipeline

**Purpose:** Convert raw documents into structured knowledge.

**Flow:**
```
Corpus Files
    ↓
Docling Parser (async workers)
    ↓
Semantic Chunker
    ↓
    ├──────────────────┐
    ↓                  ↓
infinity-emb      Entity Extractor
(GPU batched)      (qwen3:8b)
    ↓                  ↓
Qdrant           Neo4j
(vectors)        (entities/relationships)
```

**Key Features:**
- Async processing with 4 parallel workers
- Fault-tolerant with SQLite ledger
- Resumable from any point
- GPU-saturated embedding via infinity-emb

### 2. Knowledge Graph Layer

**Purpose:** Store and query structured knowledge.

**Components:**
- **Neo4j:** Graph database for entities and relationships
- **Qdrant:** Vector database for semantic search
- **Named Graphs:** Targeted subgraphs for analysis
- **Provenance Graph:** Tracking claims and inferences

**Schema:**
```
(:Entity) — nodes representing people, organizations, locations, etc.
(:Chunk) — document chunks with embeddings
(:Claim) — assertions made during investigation
(:Inference) — reasoning steps
(:Investigation) — investigation metadata
(:NamedGraph) — analytical subgraphs
```

### 3. Research Agent Core

**Purpose:** Orchestrate deep investigation with multiple specialists.

**Components:**
- **LangGraph Orchestrator:** State machine for investigation flow
- **Specialist Agents:** Domain-specific sub-agents
- **Model Router:** Routes to appropriate LLM provider
- **Investigation Journal:** Human-readable record

**Specialist Agents:**
- **SourceCritic:** Evaluates source credibility
- **Historian:** Contextualizes events in timeline
- **Statistician:** Runs numerical analysis
- **DevilsAdvocate:** Challenges hypotheses
- **PatternDetector:** Identifies connections
- **ReportWriter:** Synthesizes findings

### 4. TUI Interface

**Purpose:** Provide beautiful terminal-based interaction.

**Layout:**
```
┌──────────────────┬──────────────────────┬───────────────────────────┐
│ INVESTIGATION    │ AGENT LOG            │ CORPUS STATS              │
│                  │                      │                           │
│ Task: Epstein    │ [14:32:01] Searching │ Documents: 14,823         │
│ Status: Active   │ corpus for "Maxwell" │ Chunks: 1,247,441         │
│ Progress: 34%    │ Found 847 mentions   │ Entities: 98,231          │
│                  │                      │ Relationships: 234,871    │
│ Hypotheses: 12   │ [14:32:04] Running   │                           │
│ ✓ Confirmed: 3   │ Cypher traversal...  │ Ingestion: Complete       │
│ ✗ Disproven: 2   │                      │ Embedding: Complete       │
│ ○ Active: 7      │ [14:32:09] Pattern   │ Extraction: 87% complete  │
│                  │ detected: 6 people   │                           │
│ Graphs: 4        │ co-located on dates  │ Active Agents: 3          │
│ • assoc_net      │ matching financial   │ • PatternDetector         │
│ • timeline       │ records              │ • SourceCritic            │
│ • fin_flows      │                      │ • Historian               │
│ • contradictions │                      │                           │
├──────────────────┴──────────────────────┴───────────────────────────┤
│ > continue                                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 5. Visualization Engine

**Purpose:** Render beautiful 3D graph visualizations.

**Components:**
- **FastAPI Server:** Serves graph data via REST API
- **3d-force-graph:** WebGL rendering in browser
- **Cyberpunk Styling:** Neon glow effects, bloom, particles

**Views:**
- **Knowledge Graph:** Full entity/relationship graph
- **Reasoning Graph:** Investigation provenance chain
- **Named Graphs:** Targeted analytical subgraphs

---

## Data Flow

### Ingestion Flow

1. **File Discovery:** Walk corpus directory, check SQLite ledger
2. **Parsing:** Docling parses documents to markdown
3. **Chunking:** Semantic chunking with section awareness
4. **Embedding:** infinity-emb batches chunks for GPU embedding
5. **Extraction:** qwen3:8b extracts entities and relationships
6. **Storage:** Vectors to Qdrant, entities to Neo4j
7. **Ledger Update:** Mark file as complete in SQLite

### Investigation Flow

1. **User Question:** Natural language query
2. **Query Decomposition:** Break into atomic sub-queries
3. **Retrieval:** Vector search + graph traversal
4. **Specialist Analysis:** Domain-specific agents analyze findings
5. **Integration:** Combine findings, update state
6. **Provenance Tracking:** Record claims and inferences
7. **Journal Update:** Append to investigation journal
8. **Checkpoint:** Save state to SQLite

---

## Key Architectural Decisions

### 1. LangGraph + SQLite

**Decision:** Use LangGraph with SQLite checkpointing.

**Rationale:**
- Perfect pause/resume semantics
- No external service required
- Typed state management
- Conditional routing

**Trade-off:** Single writer only (SQLite limitation)

### 2. Neo4j + Qdrant

**Decision:** Hybrid storage with Neo4j and Qdrant.

**Rationale:**
- Neo4j for structural relationships
- Qdrant for semantic similarity
- Best of both worlds

**Trade-off:** More complex than single database

### 3. Docling

**Decision:** Use Docling for multi-modal parsing.

**Rationale:**
- Unified pipeline for all modalities
- Built-in OCR and transcription
- Consistent output format

**Trade-off:** Slower than specialized parsers

### 4. infinity-emb

**Decision:** Use infinity-emb for embedding serving.

**Rationale:**
- Dynamic batching maximizes GPU utilization
- 10-50x throughput improvement
- OpenAI-compatible API

**Trade-off:** Additional service to manage

### 5. API Models for Reasoning

**Decision:** Use Claude/Gemini API for reasoning.

**Rationale:**
- Best quality reasoning
- No VRAM contention
- Fast inference

**Trade-off:** API costs, requires internet

---

## Performance Characteristics

### Ingestion Performance

| Task | Throughput | Notes |
|------|------------|-------|
| PDF parsing | ~50 docs/sec | CPU-bound, 4 workers |
| Embedding | ~2000 chunks/sec | GPU-saturated |
| Extraction | ~10 chunks/sec | LLM-bound |

### Query Performance

| Operation | Latency | Notes |
|-----------|---------|-------|
| Vector search | 10-50ms | Qdrant |
| 2-hop traversal | 10-100ms | Neo4j |
| 5-hop traversal | 50-500ms | Neo4j |
| PageRank (named graph) | 1-10s | GDS |

### Visualization Performance

| Node Count | FPS | Notes |
|------------|-----|-------|
| 1,000 | 60 | Smooth |
| 10,000 | 30-60 | Good |
| 50,000 | 15-30 | Acceptable |
| 100,000 | 5-15 | Limited |

---

## Scalability Considerations

### Vertical Scaling

- **GPU:** RTX 4080 (16GB) sufficient for current design
- **CPU:** 24 cores excellent for parallel parsing
- **RAM:** 32GB sufficient for services
- **Storage:** 500GB+ required for 300GB corpus

### Horizontal Scaling

- **Ingestion:** Can add more parser workers
- **Embedding:** Can deploy multiple infinity-emb instances
- **Investigation:** Can run multiple investigations in parallel
- **Storage:** Can shard Qdrant collections

### Bottlenecks

1. **GPU:** VRAM limits concurrent model usage
2. **Storage:** I/O limits for large corpus ingestion
3. **LLM:** API rate limits for reasoning
4. **Network:** Latency for API calls

---

## Security Considerations

### Data Privacy

- Local processing for sensitive data
- API models send data to third parties
- Option to use fully local models

### Access Control

- SQLite file permissions
- Docker container isolation
- Network access restrictions

### Audit Trail

- Provenance graph tracks all claims
- Investigation journal records all actions
- Checkpoint history enables replay

---

## Related Topics

- [[ORACLE Technology Stack]] — Technology choices
- [[ORACLE Implementation Roadmap]] — Implementation phases
- [[ORACLE Decision Log]] — Architectural decisions
- [[Graph RAG Research]] — Background research

---

**Tags:** #oracle #architecture #system-design #graph-rag

**Links:**
- [[ORACLE Project Overview]]
- [[ORACLE Technology Stack]]
- [[ORACLE Implementation Roadmap]]
