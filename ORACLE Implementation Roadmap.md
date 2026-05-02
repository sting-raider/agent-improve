# ORACLE Implementation Roadmap

**Last Updated:** 2026-05-01  
**Status:** Ready to Begin  
**Estimated Duration:** 15-20 sessions

---

## Overview

This roadmap outlines the implementation phases for ORACLE, from infrastructure setup through Hermes integration. Each phase has clear objectives, deliverables, and success criteria.

---

## Phase 0: Infrastructure Setup

**Duration:** 1 session  
**Priority:** CRITICAL  
**Dependencies:** None

### Objectives

- Install and configure all required services
- Verify hardware capabilities
- Set up development environment
- Validate GPU access

### Tasks

1. **System Verification**
   - [ ] Check CUDA availability (`nvidia-smi`)
   - [ ] Verify Docker installation
   - [ ] Check available disk space (500GB+ required)
   - [ ] Verify Python version (3.11+)

2. **Tool Installation**
   - [ ] Install uv package manager
   - [ ] Install Ollama
   - [ ] Pull Ollama models (qwen3:8b, nomic-embed-text)

3. **Service Setup**
   - [ ] Create docker-compose.yml
   - [ ] Configure Neo4j environment
   - [ ] Configure Qdrant environment
   - [ ] Start services with Docker Compose

4. **Project Initialization**
   - [ ] Create project structure
   - [ ] Initialize uv project
   - [ ] Create .env file
   - [ ] Install Python dependencies

5. **Validation**
   - [ ] Verify Neo4j health
   - [ ] Verify Qdrant health
   - [ ] Test GPU access
   - [ ] Test Ollama models

### Deliverables

- Running Neo4j instance
- Running Qdrant instance
- Configured Python environment
- Verified GPU access

### Success Criteria

- All services healthy
- GPU accessible to Python
- Ollama models working
- Project structure created

---

## Phase 1: Ingestion Pipeline

**Duration:** 2-3 sessions  
**Priority:** HIGH  
**Dependencies:** Phase 0

### Objectives

- Implement document parsing
- Implement semantic chunking
- Set up embedding server
- Create async pipeline
- Test with sample corpus

### Tasks

1. **Document Parser**
   - [ ] Implement Docling wrapper
   - [ ] Add LiteParse fallback
   - [ ] Implement async parsing
   - [ ] Add error handling

2. **Semantic Chunker**
   - [ ] Implement header-based splitting
   - [ ] Implement paragraph-based splitting
   - [ ] Implement sentence-based splitting
   - [ ] Add overlap support

3. **Embedding Service**
   - [ ] Set up infinity-emb server
   - [ ] Implement async client
   - [ ] Add dynamic batching
   - [ ] Implement retry logic

4. **Entity Extractor**
   - [ ] Implement extraction prompt
   - [ ] Set up Ollama client
   - [ ] Add JSON schema validation
   - [ ] Implement async extraction

5. **Pipeline Orchestrator**
   - [ ] Implement async queues
   - [ ] Create worker pools
   - [ ] Implement SQLite ledger
   - [ ] Add progress tracking

6. **Testing**
   - [ ] Test with sample PDFs
   - [ ] Test with images
   - [ ] Test with video
   - [ ] Verify fault tolerance

### Deliverables

- Working document parser
- Working semantic chunker
- Running infinity-emb server
- Async ingestion pipeline
- SQLite ledger system

### Success Criteria

- Parse all supported formats
- Chunk documents correctly
- Embed at 1000+ chunks/sec
- Extract entities accurately
- Resume from failures

---

## Phase 2: Knowledge Graph Layer

**Duration:** 2-3 sessions  
**Priority:** HIGH  
**Dependencies:** Phase 1

### Objectives

- Initialize Neo4j schema
- Implement entity extraction
- Implement graph manager
- Create named graph system
- Test GDS algorithms

### Tasks

1. **Schema Initialization**
   - [ ] Create constraints
   - [ ] Create indexes
   - [ ] Create full-text indexes
   - [ ] Test schema

2. **Entity Extraction**
   - [ ] Define entity types
   - [ ] Define relationship types
   - [ ] Implement extraction logic
   - [ ] Add confidence scoring

3. **Graph Manager**
   - [ ] Implement named graph creation
   - [ ] Add graph deletion
   - [ ] Implement graph listing
   - [ ] Add graph metadata

4. **GDS Integration**
   - [ ] Test PageRank
   - [ ] Test Louvain
   - [ ] Implement result caching
   - [ ] Add performance monitoring

5. **Provenance Graph**
   - [ ] Define provenance schema
   - [ ] Implement claim tracking
   - [ ] Implement inference tracking
   - [ ] Add contradiction detection

6. **Testing**
   - [ ] Test with sample data
   - [ ] Verify graph queries
   - [ ] Test GDS algorithms
   - [ ] Verify provenance tracking

### Deliverables

- Neo4j schema with constraints
- Entity extraction system
- Named graph manager
- GDS algorithm integration
- Provenance graph system

### Success Criteria

- Schema constraints working
- Entities extracted correctly
- Named graphs created successfully
- GDS algorithms running
- Provenance tracking functional

---

## Phase 3: Research Agent Core

**Duration:** 3-4 sessions  
**Priority:** HIGH  
**Dependencies:** Phase 2

### Objectives

- Implement LangGraph orchestrator
- Create investigation state schema
- Implement specialist agents
- Set up model router
- Create journal system
- Test pause/resume

### Tasks

1. **State Schema**
   - [ ] Define InvestigationState
   - [ ] Define Hypothesis
   - [ ] Define ActiveTask
   - [ ] Define NamedGraphRef

2. **LangGraph Graph**
   - [ ] Define nodes
   - [ ] Define edges
   - [ ] Implement conditional routing
   - [ ] Add checkpointing

3. **Specialist Agents**
   - [ ] Implement SourceCritic
   - [ ] Implement Historian
   - [ ] Implement Statistician
   - [ ] Implement DevilsAdvocate
   - [ ] Implement PatternDetector
   - [ ] Implement ReportWriter

4. **Model Router**
   - [ ] Implement provider selection
   - [ ] Add GPU mutex
   - [ ] Implement fallback logic
   - [ ] Add cost tracking

5. **Investigation Journal**
   - [ ] Implement journal creation
   - [ ] Add entry logging
   - [ ] Implement summary generation
   - [ ] Add journal viewer

6. **Testing**
   - [ ] Test simple investigation
   - [ ] Test pause/resume
   - [ ] Test specialist agents
   - [ ] Verify checkpoint recovery

### Deliverables

- LangGraph orchestrator
- Investigation state schema
- Specialist agent implementations
- Model router with GPU management
- Investigation journal system

### Success Criteria

- Investigations run successfully
- Pause/resume works perfectly
- Specialist agents function correctly
- Model routing works
- Journal updates continuously

---

## Phase 4: TUI Interface

**Duration:** 2-3 sessions  
**Priority:** MEDIUM  
**Dependencies:** Phase 3

### Objectives

- Implement Textual app
- Create widgets (panels, logs, stats)
- Implement command handling
- Style with cyberpunk theme
- Test user workflows

### Tasks

1. **Main Application**
   - [ ] Implement OracleApp
   - [ ] Create layout
   - [ ] Add header and footer
   - [ ] Implement refresh loop

2. **Widgets**
   - [ ] Implement InvestigationPanel
   - [ ] Implement AgentLogPanel
   - [ ] Implement CorpusStatsPanel
   - [ ] Implement CommandInput
   - [ ] Implement JournalViewer

3. **Command Handling**
   - [ ] Implement continue command
   - [ ] Implement pause command
   - [ ] Implement status command
   - [ ] Implement add corpus command
   - [ ] Implement show graph command
   - [ ] Implement new investigation command

4. **Styling**
   - [ ] Create cyberpunk color scheme
   - [ ] Implement CSS stylesheet
   - [ ] Add animations
   - [ ] Style all widgets

5. **Integration**
   - [ ] Connect to LangGraph
   - [ ] Connect to databases
   - [ ] Implement status updates
   - [ ] Add error handling

6. **Testing**
   - [ ] Test all commands
   - [ ] Test widget updates
   - [ ] Test error handling
   - [ ] Verify user workflows

### Deliverables

- Working Textual TUI
- All widgets implemented
- Command system working
- Cyberpunk styling applied
- Integration with backend

### Success Criteria

- TUI launches successfully
- All commands work
- Widgets update correctly
- Styling looks good
- User workflows functional

---

## Phase 5: Visualization Engine

**Duration:** 2-3 sessions  
**Priority:** MEDIUM  
**Dependencies:** Phase 2

### Objectives

- Implement FastAPI server
- Create 3d-force-graph frontend
- Implement graph export
- Add cyberpunk styling
- Test with large graphs

### Tasks

1. **FastAPI Server**
   - [ ] Implement knowledge graph endpoint
   - [ ] Implement reasoning graph endpoint
   - [ ] Implement named graph endpoint
   - [ ] Add CORS support

2. **Graph Export**
   - [ ] Implement Neo4j to JSON conversion
   - [ ] Add node sampling
   - [ ] Add link filtering
   - [ ] Implement incremental loading

3. **Frontend - Knowledge Graph**
   - [ ] Create HTML page
   - [ ] Implement 3d-force-graph
   - [ ] Add cyberpunk styling
   - [ ] Implement bloom effects
   - [ ] Add particle flow

4. **Frontend - Reasoning Graph**
   - [ ] Create HTML page
   - [ ] Implement 3d-force-graph
   - [ ] Add claim visualization
   - [ ] Implement inference paths

5. **Frontend - Named Graphs**
   - [ ] Create HTML page
   - [ ] Implement graph switching
   - [ ] Add graph metadata
   - [ ] Implement graph controls

6. **Testing**
   - [ ] Test with sample graphs
   - [ ] Test with large graphs
   - [ ] Verify performance
   - [ ] Test browser compatibility

### Deliverables

- FastAPI visualization server
- Knowledge graph visualization
- Reasoning graph visualization
- Named graph visualization
- Cyberpunk styling applied

### Success Criteria

- Server starts successfully
- Graphs render correctly
- Performance acceptable at scale
- Styling looks good
- Browser compatibility verified

---

## Phase 6: MCP Integration

**Duration:** 1-2 sessions  
**Priority:** MEDIUM  
**Dependencies:** Phase 3

### Objectives

- Implement web search server
- Implement code execution server
- Implement graph query server
- Test tool integration

### Tasks

1. **Web Search Server**
   - [ ] Implement Tavily integration
   - [ ] Add search tools
   - [ ] Implement result parsing
   - [ ] Add error handling

2. **Code Execution Server**
   - [ ] Implement Docker sandbox
   - [ ] Add code execution tool
   - [ ] Implement output capture
   - [ ] Add timeout handling

3. **Graph Query Server**
   - [ ] Implement entity search
   - [ ] Implement connection query
   - [ ] Implement path finding
   - [ ] Add Cypher execution

4. **Integration**
   - [ ] Connect to LangGraph
   - [ ] Implement tool calling
   - [ ] Add error handling
   - [ ] Test all tools

5. **Testing**
   - [ ] Test web search
   - [ ] Test code execution
   - [ ] Test graph queries
   - [ ] Verify tool integration

### Deliverables

- Web search MCP server
- Code execution MCP server
- Graph query MCP server
- Tool integration working

### Success Criteria

- All MCP servers running
- Tools accessible to agents
- Error handling working
- Integration verified

---

## Phase 7: Report Generation

**Duration:** 1-2 sessions  
**Priority:** MEDIUM  
**Dependencies:** Phase 3

### Objectives

- Implement report compiler
- Create templates
- Test with sample investigation

### Tasks

1. **Report Compiler**
   - [ ] Implement report assembly
   - [ ] Add section generation
   - [ ] Implement citation building
   - [ ] Add formatting

2. **Templates**
   - [ ] Create main template
   - [ ] Create section templates
   - [ ] Add styling
   - [ ] Implement Jinja2 rendering

3. **Content Generation**
   - [ ] Implement executive summary
   - [ ] Add timeline reconstruction
   - [ ] Implement entity analysis
   - [ ] Add contradiction registry

4. **Testing**
   - [ ] Test with sample investigation
   - [ ] Verify report structure
   - [ ] Check citations
   - [ ] Validate formatting

### Deliverables

- Report compiler
- Report templates
- Sample report generated

### Success Criteria

- Reports generate correctly
- All sections included
- Citations accurate
- Formatting looks good

---

## Phase 8: Hermes Integration

**Duration:** 1 session  
**Priority:** LOW  
**Dependencies:** Phase 7

### Objectives

- Create CLI interface
- Implement ACP communication
- Create Hermes skill
- Test end-to-end

### Tasks

1. **CLI Interface**
   - [ ] Implement oracle CLI
   - [ ] Add investigate command
   - [ ] Add resume command
   - [ ] Add ingest command
   - [ ] Add status command
   - [ ] Add visualize command
   - [ ] Add report command

2. **ACP Communication**
   - [ ] Implement ACP server
   - [ ] Add command handling
   - [ ] Implement response formatting
   - [ ] Add error handling

3. **Hermes Skill**
   - [ ] Create skill documentation
   - [ ] Add command reference
   - [ ] Add memory notes
   - [ ] Test skill integration

4. **Testing**
   - [ ] Test CLI commands
   - [ ] Test ACP communication
   - [ ] Test Hermes integration
   - [ ] Verify end-to-end workflow

### Deliverables

- Oracle CLI
- ACP server
- Hermes skill
- End-to-end integration verified

### Success Criteria

- CLI commands work
- ACP communication functional
- Hermes skill installed
- End-to-end workflow verified

---

## Milestones

### M1: Infrastructure Ready
**Date:** End of Phase 0  
**Criteria:** All services running, GPU accessible

### M2: Ingestion Working
**Date:** End of Phase 1  
**Criteria:** Can ingest documents and create embeddings

### M3: Knowledge Graph Functional
**Date:** End of Phase 2  
**Criteria:** Can query graph and run GDS algorithms

### M4: Agent Core Working
**Date:** End of Phase 3  
**Criteria:** Can run investigations with pause/resume

### M5: TUI Complete
**Date:** End of Phase 4  
**Criteria:** Beautiful TUI with all commands working

### M6: Visualization Complete
**Date:** End of Phase 5  
**Criteria:** Can visualize graphs in browser

### M7: Tools Integrated
**Date:** End of Phase 6  
**Criteria:** All MCP servers working

### M8: Reporting Complete
**Date:** End of Phase 7  
**Criteria:** Can generate investigation reports

### M9: Hermes Integration Complete
**Date:** End of Phase 8  
**Criteria:** Full end-to-end workflow via Hermes

---

## Risk Mitigation

### Timeline Risks

- **Buffer:** Add 20% buffer to estimates
- **Parallelization:** Run phases in parallel where possible
- **Scope Control:** Prioritize critical features
- **Early Testing:** Test early and often

### Technical Risks

- **Fallback Plans:** Have alternatives for each technology
- **Prototyping:** Prototype risky components first
- **Monitoring:** Monitor progress closely
- **Flexibility:** Be ready to adjust approach

---

## Next Steps

1. Review this roadmap
2. Approve Phase 0 tasks
3. Begin infrastructure setup
4. Track progress daily
5. Update roadmap as needed

---

**Tags:** #oracle #implementation #roadmap #phases

**Links:**
- [[ORACLE Project Overview]]
- [[ORACLE Architecture]]
- [[ORACLE Technology Stack]]
