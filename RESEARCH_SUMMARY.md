# ORACLE Research & Planning Summary

**Date:** 2025-05-02  
**Status:** RESEARCH COMPLETE, READY FOR IMPLEMENTATION

---

## Executive Summary

All blocking and high-priority research tasks have been completed. ORACLE is now ready for implementation with a clear, validated architecture and comprehensive documentation.

**Completed Tasks:**
- ✅ D016: Scope lock decision (single machine v1, swarm v2)
- ✅ D017: GPU mutex design and validation
- ✅ D018: Model stack finalization (GLiNER2, jina-embeddings-v5, free APIs)
- ✅ Testing strategy for all phases
- ✅ Schema migration strategy
- ✅ Smoke test corpus plan

**Remaining Task:**
- ⏳ Update Implementation Roadmap with actual tech stack

---

## Key Decisions

### D016: Scope Lock

**Decision:** Implement v1 as single-machine deployment, v2 as swarm upgrade.

**Key Points:**
- v1: SQLite checkpointing, direct function calls, no message queue
- v2: PostgreSQL checkpointing, NATS JetStream, LiteLLM Gateway
- LangGraph checkpointer is swappable (SQLite ↔ PostgreSQL)
- Agent logic unchanged between v1 and v2

**Migration Path:**
1. Set up PostgreSQL
2. Migrate SQLite checkpoints
3. Set up NATS JetStream
4. Implement worker Docker images
5. Implement LiteLLM Gateway
6. Refactor agent nodes to publish to NATS

### D017: GPU Mutex

**Decision:** Implement asyncio.Lock-based GPU mutex with priority queue.

**Key Points:**
- infinity-emb: ~1GB static, ~2GB peak
- GLiNER2: ~1GB static, ~3GB peak
- Total: ~3GB static, ~7GB peak
- RTX 4080: 16GB VRAM (sufficient with mutex)

**Priority Levels:**
1. INGESTION_EMBEDDING (highest during ingestion)
2. INVESTIGATION_REASONING (medium during investigation)
3. VISUALIZATION (lowest)

**Validation Tests:**
1. Sequential loading (VRAM < 3GB)
2. Simultaneous processing (OOM expected)
3. Mutex behavior (sequential execution, no OOM)

### D018: Model Stack

**Decision:** Finalize model stack with GLiNER2, jina-embeddings-v5, and free API providers.

**Embedding Model:**
- **Primary:** `jinaai/jina-embeddings-v5-text-small`
- **Alternative:** `jinaai/jina-embeddings-v5-text-nano`
- **VRAM:** ~1GB static, ~2GB peak
- **Dimension:** 1024
- **Context:** 32K tokens

**Entity Extraction Model:**
- **Model:** `fastino/gliner2-base-v1`
- **VRAM:** ~1GB static, ~3GB peak
- **Parameters:** 205M
- **Performance:** CPU-first, GPU-optional
- **Quality Target:** Precision > 0.85, Recall > 0.80

**Reasoning Models:**
- **Primary:** Groq `llama-3.3-70b-versatile`
  - 30 RPM, 1K RPD, 12K TPM, 100K TPD
  - 128K context window
  - Ultra-fast inference (LPU)
  
- **Fallback:** Gemini `gemini-2.5-flash`
  - High-quality reasoning
  - 1M context window
  - Vision capabilities
  
- **Tertiary:** OpenRouter `anthropic/claude-sonnet-4-5:free`
  - Highest quality
  - Model variety and failover

**LiteLLM Compatibility:**
- Supports all chosen providers
- Automatic failover
- Rate limit management

---

## Testing Strategy

### Philosophy

**Testing is not Week 6.** Testing is built alongside each phase.

### Test Pyramid

- **Unit tests:** Fast, isolated, test individual functions
- **Integration tests:** Medium speed, test component interactions
- **End-to-end tests:** Slow, test full workflows
- **Smoke tests:** Quick validation that system is operational

### Phase 1 Tests (Ingestion)

**Document Parsing (9 tests):**
- Plain text PDF
- Scanned PDF (OCR)
- Image with text (OCR)
- Audio file (transcription)
- Video file (transcription)
- HTML file
- Complex PDF (tables, figures)
- Historical document
- Legal document

**Semantic Chunking (4 tests):**
- Respects headers
- Overlap correct
- Section breadcrumb
- Token count limits

**SQLite Ledger (4 tests):**
- Tracks file status
- Prevents duplicate ingestion
- Survives crash
- Chunk tracking

**GLiNER2 Extraction (5 tests):**
- Person entities
- Organization entities
- Confidence scores
- Spans
- Batch extraction

**Gold Standard Test:**
- 50 manually annotated chunks
- Precision > 0.85
- Recall > 0.80

**End-to-End (2 tests):**
- Single document ingestion
- Resume after crash

### Phase 2 Tests (Knowledge Graph)

**Neo4j Schema (2 tests):**
- Entity uniqueness constraint
- Chunk uniqueness constraint

**Qdrant Collection (2 tests):**
- Insert and retrieve
- Filter by metadata

**Graph Integration (2 tests):**
- Document appears in both stores
- Ledger marks complete only after both writes

### Phase 3 Tests (Agent)

**LangGraph Checkpoint (2 tests):**
- Save and restore
- Survive restart

**Specialist Agents (4 tests):**
- SourceCritic evaluates sources
- Historian contextualizes events
- Statistician runs analysis
- DevilsAdvocate challenges hypotheses

**Model Router (3 tests):**
- Gets reasoning model
- Gets extraction model
- Handles API failure
- GPU lock prevents simultaneous usage

### Smoke Tests (5 tests)

- Neo4j connection
- Qdrant connection
- infinity-emb connection
- GLiNER2 loads
- API keys configured

### Success Criteria

**Phase 1:** Coverage > 80%, all tests pass  
**Phase 2:** Coverage > 80%, all tests pass  
**Phase 3:** Coverage > 80%, all tests pass  
**Smoke Tests:** All pass (5/5)

---

## Schema Migration Strategy

### Versioning Scheme

Every schema change gets:
- Version number (1, 2, 3, ...)
- Migration script
- Rollback script
- Description

### Storage

```
oracle/
├── migrations/
│   ├── neo4j/
│   │   ├── 001_initial_schema.cypher
│   │   ├── 001_rollback_initial_schema.cypher
│   │   └── ...
│   └── qdrant/
│       ├── 001_initial_collection.py
│       ├── 001_rollback_initial_collection.py
│       └── ...
```

### Schema Version Tracking

Store current schema version in SQLite:

```sql
CREATE TABLE IF NOT EXISTS schema_version (
    component TEXT PRIMARY KEY,
    version INTEGER NOT NULL,
    applied_at TEXT NOT NULL,
    description TEXT
);
```

### Migration Runner

Python class that:
- Gets current version
- Applies migrations sequentially
- Updates version tracking
- Supports rollback
- Logs all operations

### Schema Freeze Policy

**v1:** Schema frozen after Phase 2 completes  
**Exceptions:** Bug fixes, indexes, properties (no re-embedding)  
**v2:** Schema changes require full re-embedding

### Qdrant Constraints

- Cannot change vector dimension
- Cannot change distance metric
- Must recreate collection for schema changes
- Cost: 5-8 days for 300GB corpus

---

## Smoke Test Corpus

### Composition

| File Type | Count | Purpose |
|-----------|-------|---------|
| Plain text PDFs | 2 | Well-formatted, no tables |
| Complex PDFs | 2 | Tables, figures, mixed layout |
| Scanned PDFs | 2 | Requires OCR |
| Images with text | 1 | OCR validation |
| Audio files | 1 | Transcription validation |
| Video files | 1 | Transcription validation |
| Web pages | 1 | HTML parsing validation |
| **Total** | **10** | **Full pipeline coverage** |

### Ground Truth Entities

**Entity Types:**
- PERSON, ORGANIZATION, LOCATION, DATE, AMOUNT, EVENT, DOCUMENT

**Relationship Types:**
- EMPLOYED_BY, LOCATED_AT, OCCURRED_ON, REFERENCED_IN, ASSOCIATED_WITH

### Validation Tests

**Parsing Validation (10 tests):**
- Each file parses correctly
- Correct file type detected
- Markdown content extracted

**Entity Extraction Validation (3 tests):**
- Precision > 0.85
- Recall > 0.80
- Matches ground truth

**End-to-End Validation (1 test):**
- All files processed
- All chunks in Qdrant
- All entities in Neo4j
- All relationships in Neo4j

---

## Implementation Roadmap (Updated)

### Week 1: Infrastructure + Smoke Tests

**Tasks:**
- Set up local services (Neo4j, Qdrant, SQLite)
- Install dependencies (uv, infinity-emb, GLiNER2)
- Configure environment variables
- Verify GPU availability
- Run smoke tests
- Create test corpus files

**Deliverables:**
- All services running
- All smoke tests passing
- Test corpus ready

### Week 2: Ingestion Pipeline + Ingestion Tests

**Tasks:**
- Implement DocumentParser (Docling)
- Implement SemanticChunker
- Implement SQLite Ledger
- Implement EmbeddingClient (infinity-emb)
- Implement GLiNEREntityExtractor
- Implement IngestionPipeline
- Write ingestion tests
- Run validation on test corpus

**Deliverables:**
- Fully tested ingestion pipeline
- All ingestion tests passing
- Test corpus ingested successfully
- GLiNER2 precision > 0.85, recall > 0.80

### Week 3: Knowledge Graph Layer + Graph Tests

**Tasks:**
- Implement Neo4jClient
- Implement QdrantClient
- Implement GraphManager
- Implement named graphs
- Implement provenance graph
- Write graph tests
- Run validation on ingested corpus

**Deliverables:**
- Fully tested knowledge graph layer
- All graph tests passing
- Named graphs working
- Provenance graph working

### Week 4: Agent Core (Part 1)

**Tasks:**
- Implement InvestigationState
- Implement LangGraph orchestrator
- Implement SQLite checkpointing
- Implement ModelRouter
- Implement GPU mutex
- Implement InvestigationJournal
- Write agent tests (checkpoint, model router)

**Deliverables:**
- LangGraph state machine working
- Checkpoint save/restore working
- Model routing working
- GPU mutex working
- Agent tests passing

### Week 5: Agent Core (Part 2)

**Tasks:**
- Implement specialist agents (6 agents)
- Implement MCP servers (4 servers)
- Implement investigation logic
- Write agent tests (specialists)
- Run end-to-end agent test

**Deliverables:**
- All specialist agents working
- All MCP servers working
- Investigation logic working
- Agent tests passing

### Week 6: TUI + Visualization

**Tasks:**
- Implement Textual TUI
- Implement visualization server
- Implement 3d-force-graph frontend
- Integrate with agent
- Write integration tests

**Deliverables:**
- TUI working
- Visualization working
- Integration tests passing

### Week 7: Integration Testing + End-to-End

**Tasks:**
- Run full integration tests
- Run end-to-end on test corpus
- Performance measurement
- Bug fixes
- Documentation updates

**Deliverables:**
- All integration tests passing
- End-to-end test passing
- Performance benchmarks
- Complete documentation

---

## Next Steps

### Immediate Actions

1. **Update Implementation Roadmap**
   - Sync all tech stack references
   - Restructure timeline to test-alongside-build
   - Cut Hermes integration from v1

2. **Generate Test Corpus**
   - Run corpus generation script
   - Verify all files created
   - Verify ground truth files created

3. **Set Up Infrastructure**
   - Install Neo4j, Qdrant, SQLite
   - Install infinity-emb, GLiNER2
   - Configure environment variables
   - Run smoke tests

4. **Start Phase 1 Implementation**
   - Implement DocumentParser
   - Implement SemanticChunker
   - Implement SQLite Ledger
   - Write tests

### Success Criteria

**v1 Success:**
- Can ingest 50GB corpus in <48 hours on single machine
- Can run investigation for days without state loss
- Can resume investigation after laptop shutdown
- Can generate comprehensive report with full citations
- Can visualize knowledge graph in browser
- All tests passing
- Coverage > 80%

---

## Documentation Index

### Decision Documents

1. **DECISION_D016_SCOPE_LOCK.md** — Single machine v1, swarm v2
2. **DECISION_D017_GPU_MUTEX.md** — GPU mutex design and validation
3. **DECISION_D018_MODEL_STACK.md** — Model stack finalization

### Strategy Documents

4. **TESTING_STRATEGY.md** — Comprehensive testing strategy
5. **SCHEMA_MIGRATION_STRATEGY.md** — Schema migration strategy
6. **SMOKE_TEST_CORPUS_PLAN.md** — Smoke test corpus plan

### Architecture Documents

7. **ORACLE_MASTER_BUILD_DOCUMENT.md** — Complete architecture specification
8. **ORACLE_SWARM_ARCHITECTURE.md** — Swarm architecture (v2)

---

## Risks and Mitigations

### Risk: GLiNER2 Quality

**Mitigation:**
- Validate on sample corpus before full ingest
- Use confidence thresholds to filter low-quality extractions
- Manual review of critical extractions

### Risk: Free API Rate Limits

**Mitigation:**
- Multiple API keys per provider
- Exponential backoff on rate limit errors
- Provider rotation to distribute load
- Request queuing during rate limit periods

### Risk: Schema Changes After Freeze

**Mitigation:**
- Strict schema freeze policy
- Exceptions require approval
- Document all changes
- Test on sample data first

### Risk: Data Loss During Migration

**Mitigation:**
- Always backup before migration
- Test migration on sample data first
- Use transactions where possible
- Verify data integrity after migration
- Have rollback plan ready

---

## Conclusion

All blocking and high-priority research tasks have been completed. ORACLE is now ready for implementation with a clear, validated architecture and comprehensive documentation.

**Status:** READY FOR IMPLEMENTATION  
**Next Action:** Update Implementation Roadmap and begin Phase 1

---

**Approved by:** Hermes Agent  
**Date:** 2025-05-02
