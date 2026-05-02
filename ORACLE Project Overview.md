# ORACLE Project Overview

**Project:** ORACLE — Omnidirectional Research Agent with Corpus Logic and Epistemological Engine  
**Status:** Research Complete, Implementation Ready  
**Start Date:** 2026-05-01  
**Last Updated:** 2026-05-01

---

## What is ORACLE?

ORACLE is a locally-running, GPU-accelerated, multi-agent research system capable of:

- **Ingesting anything** — PDFs, scanned documents, images, video, audio, web pages (up to 300GB+)
- **Building understanding** — Constructing typed knowledge graphs with Neo4j GDS algorithms
- **Investigating deeply** — Multi-agent orchestration via LangGraph with pause/resume capability
- **Never forgetting** — Complete state persistence at every step (survives laptop shutdown)
- **Creating structure** — Autonomous named graph creation for different analytical views
- **Producing exhaustive output** — Structured markdown reports with full citation chains
- **Showing reasoning** — Provenance graph tracking every claim and inference

---

## Key Features

### 1. Multi-Modal Ingestion
- Unified pipeline for all document types (PDF, images, video, audio, web)
- Docling parser with OCR and Whisper Turbo transcription
- Semantic chunking with section awareness
- Fault-tolerant, resumable ingestion pipeline

### 2. Knowledge Graph Construction
- Neo4j with Graph Data Science (GDS) plugin
- Entity and relationship extraction via LLM
- Named graphs for targeted analysis (temporal, financial, associate networks)
- PageRank and Louvain community detection

### 3. Deep Investigation
- LangGraph orchestrator with SQLite checkpointing
- Specialist sub-agents (SourceCritic, Historian, Statistician, DevilsAdvocate)
- Multi-hop reasoning across vector and graph stores
- Provenance tracking for every claim

### 4. Beautiful Visualization
- 3d-force-graph WebGL visualization
- Cyberpunk aesthetic with bloom effects
- Real-time graph exploration
- Separate views for knowledge, reasoning, and named graphs

### 5. Modern TUI
- Textual-based terminal interface
- Real-time investigation monitoring
- Command-driven interaction
- Integration with Hermes

---

## Technology Stack

### Core Services
- **Neo4j 5.26+** — Graph database with GDS plugin
- **Qdrant** — Vector database for semantic search
- **Docker Compose** — Service orchestration

### Python Services
- **infinity-emb** — High-throughput embedding server
- **Docling** — Multi-modal document parser
- **LangGraph** — Agent orchestration with checkpointing
- **FastAPI** — Visualization server

### Models
- **nomic-embed-text** — Local embeddings (137M params, 0.6GB VRAM)
- **qwen3:8b** — Local extraction (8B params, 5GB VRAM)
- **Claude/Gemini API** — Reasoning (no VRAM cost)

### Frameworks
- **Textual** — TUI framework
- **3d-force-graph** — WebGL graph visualization
- **MCP** — Model Context Protocol for tool integration

---

## Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GPU | RTX 4080 (16GB VRAM) | RTX 4090 (24GB VRAM) |
| CPU | 8 cores, 16 threads | 16+ cores, 32+ threads |
| RAM | 32GB DDR5 | 64GB DDR5 |
| Storage | 500GB SSD | 1TB NVMe SSD |
| OS | Linux | Linux |

---

## Use Cases

### 1. Deep Document Analysis
- Ingest hundreds of gigabytes of documents
- Extract entities, relationships, and patterns
- Generate comprehensive reports with citations

### 2. Network Investigation
- Map connections between people, organizations, events
- Identify hidden relationships and patterns
- Visualize complex networks

### 3. Historical Research
- Reconstruct timelines from multiple sources
- Cross-reference claims across documents
- Identify contradictions and verify facts

### 4. Financial Analysis
- Track financial flows across entities
- Identify suspicious patterns
- Generate audit trails

### 5. Academic Research
- Synthesize findings across multiple papers
- Identify research gaps and connections
- Generate literature reviews

---

## Project Structure

```
oracle/
├── pyproject.toml              # Project definition
├── docker-compose.yml          # Service definitions
├── .env                        # Configuration
├── oracle/                     # Main package
│   ├── ingestion/              # Document ingestion
│   ├── graph/                  # Knowledge graph
│   ├── agents/                 # Research agents
│   ├── mcp_servers/            # MCP integrations
│   ├── tui/                    # Terminal UI
│   ├── visualization/          # Graph visualization
│   └── reporting/              # Report generation
├── scripts/                    # Operational scripts
└── tests/                      # Test suite
```

---

## Implementation Phases

1. **Phase 0:** Infrastructure Setup (1 session)
2. **Phase 1:** Ingestion Pipeline (2-3 sessions)
3. **Phase 2:** Knowledge Graph Layer (2-3 sessions)
4. **Phase 3:** Research Agent Core (3-4 sessions)
5. **Phase 4:** TUI Interface (2-3 sessions)
6. **Phase 5:** Visualization Engine (2-3 sessions)
7. **Phase 6:** MCP Integration (1-2 sessions)
8. **Phase 7:** Report Generation (1-2 sessions)
9. **Phase 8:** Hermes Integration (1 session)

**Total Estimated Time:** 15-20 sessions

---

## Documentation

- [[ORACLE Architecture]] — System architecture and design
- [[ORACLE Technology Stack]] — Technology choices and rationale
- [[ORACLE Implementation Roadmap]] — Implementation phases and milestones
- [[ORACLE Decision Log]] — Architectural decisions and alternatives
- [[ORACLE Risk Assessment]] — Risks and mitigation strategies
- [[ORACLE Performance Benchmarks]] — Performance expectations and testing

---

## Related Projects

- [[Hermes Agent]] — Outer shell and user interaction
- [[Graph RAG Research]] — Background research on Graph RAG
- [[Knowledge Graph]] — General knowledge graph concepts

---

## Status

✅ Research Complete  
✅ Architecture Validated  
✅ Technology Stack Selected  
⏳ Implementation In Progress  
⏳ Testing Pending  
⏳ Deployment Pending

---

## Next Steps

1. Review [[ORACLE Augmented Master Plan]]
2. Approve [[ORACLE Decision Log]]
3. Review [[ORACLE Risk Assessment]]
4. Begin Phase 0: Infrastructure Setup

---

**Tags:** #oracle #research-agent #graph-rag #knowledge-graph #ai

**Links:**
- [[ORACLE Architecture]]
- [[ORACLE Technology Stack]]
- [[ORACLE Implementation Roadmap]]
