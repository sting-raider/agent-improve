# ORACLE Decision Log

**Purpose:** Track all architectural and technology decisions with rationale and alternatives considered.

**Last Updated:** 2026-05-01

---

## Core Architecture Decisions

### D001: Use LangGraph with SQLite Checkpointing

**Date:** 2026-05-01  
**Category:** Orchestration  
**Status:** ✅ Approved

**Decision:** Use LangGraph with `AsyncSqliteSaver` for investigation orchestration and persistence.

**Rationale:**
- LangGraph is built specifically for LLM agents
- SQLite checkpointing provides perfect pause/resume semantics
- No external service required (single file)
- Typed state management with Pydantic
- Conditional routing for complex workflows
- Proven pattern for local agent systems

**Alternatives Considered:**
- **Temporal:** More durable for distributed systems, but significant operational overhead for single-machine setup
- **Prefect:** Better for data pipelines, less suited for agentic workflows
- **Custom state management:** Would require building checkpointing from scratch

**Trade-offs:**
- ✅ Simple, reliable, no external dependencies
- ✅ Perfect pause/resume semantics
- ❌ Single writer only (SQLite limitation)
- ❌ Not suitable for distributed deployments

**Research Sources:**
- Temporal vs LangGraph comparison (2026)
- LangGraph documentation on checkpointing
- Community discussions on agentic AI orchestration

---

### D002: Use Neo4j with GDS Plugin

**Date:** 2026-05-01  
**Category:** Graph Database  
**Status:** ✅ Approved

**Decision:** Use Neo4j Community Edition with Graph Data Science (GDS) plugin.

**Rationale:**
- GDS plugin works on Community edition (no Enterprise license required)
- Mature ecosystem and tooling
- Excellent Python driver
- Supports PageRank, Louvain, and other algorithms
- Named graphs for targeted analysis
- Cypher query language is powerful and expressive

**Alternatives Considered:**
- **Memgraph:** Neo4j-compatible, faster for some workloads, but smaller community and less mature tooling
- **ArangoDB:** Multi-model, but graph features less mature
- **PostgreSQL + AGE:** Good for existing Postgres users, but less performant for graph queries

**Trade-offs:**
- ✅ Mature ecosystem and tooling
- ✅ GDS works on Community edition
- ✅ Excellent Python driver
- ❌ Commercial support requires Enterprise license
- ❌ Some advanced features require Enterprise

**Research Sources:**
- Neo4j GDS documentation
- Neo4j vs Memgraph comparison
- Community discussions on GDS availability

---

### D003: Use Qdrant for Vector Storage

**Date:** 2026-05-01  
**Category:** Vector Database  
**Status:** ✅ Approved

**Decision:** Use Qdrant as the vector database for semantic search.

**Rationale:**
- Written in Rust (memory-safe, fast)
- Excellent filtered search performance
- Open source (no vendor lock-in)
- Easy Docker deployment
- Good Python client
- Handles 50K+ QPS on modest hardware

**Alternatives Considered:**
- **Milvus:** Highly scalable, but complex setup and heavier resource requirements
- **Weaviate:** Built-in graph features, but slower for pure vector search
- **Pinecone:** Excellent performance, but not open source and expensive
- **ChromaDB:** Simple, but less performant at scale

**Trade-offs:**
- ✅ Fast filtered search
- ✅ Open source
- ✅ Easy deployment
- ❌ Smaller community than Pinecone
- ❌ Fewer enterprise features than Milvus

**Research Sources:**
- Vector database comparison 2025
- Qdrant vs Weaviate vs Milvus analysis
- Performance benchmarks

---

### D004: Use Docling for Document Parsing

**Date:** 2026-05-01  
**Category:** Document Processing  
**Status:** ✅ Approved

**Decision:** Use Docling as the primary document parser, with LiteParse as fallback.

**Rationale:**
- Unified pipeline for PDF, DOCX, images, video, audio
- Built-in OCR for scanned documents
- Whisper Turbo integration for audio/video
- Table and figure preservation
- Consistent DoclingDocument output format
- IBM-backed, actively maintained

**Alternatives Considered:**
- **pymupdf4llm:** Faster for simple text PDFs, but doesn't handle audio/video
- **LiteParse:** Good fallback for simple documents, but less feature-rich
- **Unstructured:** Good for diverse formats, but less consistent output

**Trade-offs:**
- ✅ Handles all modalities in one pipeline
- ✅ Consistent output format
- ✅ Built-in OCR and transcription
- ❌ Slower than pymupdf4llm for simple PDFs
- ❌ Heavier resource usage

**Research Sources:**
- Docling documentation and blog posts
- Docling vs PyMuPDF comparison
- IBM Think interview with Docling creators

---

### D005: Use infinity-emb for Embedding Serving

**Date:** 2026-05-01  
**Category:** Embedding Infrastructure  
**Status:** ✅ Approved

**Decision:** Use infinity-emb as the local embedding server.

**Rationale:**
- Dynamic batching (GPU never idle)
- OpenAI-compatible API
- Supports TensorRT backend
- 10-50x throughput improvement over direct sentence-transformers
- MIT license
- Active development

**Alternatives Considered:**
- **sentence-transformers (direct):** Simpler, but GPU sits idle between batches
- **vLLM:** Excellent for LLMs, but overkill for embeddings
- **TEI (Text Embeddings Inference):** Good alternative, but infinity-emb has better dynamic batching

**Trade-offs:**
- ✅ Dynamic batching maximizes GPU utilization
- ✅ OpenAI-compatible API
- ✅ 10-50x throughput improvement
- ❌ Additional service to manage
- ❌ Slightly more complex than direct usage

**Research Sources:**
- infinity-emb GitHub repository
- Embedding server comparison
- Performance benchmarks

---

### D006: Use Textual for TUI

**Date:** 2026-05-01  
**Category:** User Interface  
**Status:** ✅ Approved

**Decision:** Use Textual framework for the terminal user interface.

**Rationale:**
- Created by Will McGugan (Rich author)
- Modern, async architecture
- 16.7M colors
- Mouse support
- Web and terminal rendering
- Active community and good documentation
- De facto Python TUI standard

**Alternatives Considered:**
- **Rich:** Lower-level, Textual is built on it
- **Prompt Toolkit:** Older, less feature-rich
- **Urwid:** Very old, not recommended for modern apps

**Trade-offs:**
- ✅ Modern, feature-rich
- ✅ Good documentation
- ✅ Active community
- ❌ Learning curve for complex layouts
- ❌ Some widgets still in development

**Research Sources:**
- Textual documentation
- SE Radio interview with Will McGugan
- Computer.org article on Textual

---

### D007: Use 3d-force-graph for Visualization

**Date:** 2026-05-01  
**Category:** Visualization  
**Status:** ✅ Approved

**Decision:** Use 3d-force-graph for browser-based graph visualization.

**Rationale:**
- WebGL rendering (GPU-accelerated)
- 3D force-directed layout
- Handles 50,000+ nodes smoothly
- Built on Three.js
- Cyberpunk aesthetic possible with post-processing
- Good performance at scale

**Alternatives Considered:**
- **Cytoscape.js:** Good for 2D, but 3D support is limited
- **D3.js:** Excellent for small graphs, but DOM-based rendering fails at scale
- **Gephi:** Desktop application, not web-based
- **vis-network:** Good for 2D, but no 3D support

**Trade-offs:**
- ✅ WebGL rendering (GPU-accelerated)
- ✅ Handles 50,000+ nodes
- ✅ 3D visualization
- ❌ Steeper learning curve for customization
- ❌ Requires browser (not pure terminal)

**Research Sources:**
- 3d-force-graph GitHub repository
- Graph visualization library comparison
- Performance benchmarks

---

### D008: Use MCP for Tool Integration

**Date:** 2026-05-01  
**Category:** Integration  
**Status:** ✅ Approved

**Decision:** Use Model Context Protocol (MCP) for all external tool integrations.

**Rationale:**
- Universal standard (97M+ monthly SDK downloads)
- Backed by Anthropic, OpenAI, Google, Microsoft
- Open standard with good documentation
- STDIO and SSE transport options
- Resources, Tools, and Prompts primitives
- Growing ecosystem of servers

**Alternatives Considered:**
- **Custom integrations:** More control, but requires building everything from scratch
- **LangChain tools:** Good, but not standardized across providers
- **OpenAI function calling:** Provider-specific, not universal

**Trade-offs:**
- ✅ Universal standard
- ✅ Backed by major AI companies
- ✅ Growing ecosystem
- ❌ Learning curve for MCP protocol
- ❌ Some tools not yet available as MCP servers

**Research Sources:**
- MCP specification documentation
- Anthropic MCP announcement
- Enterprise adoption guide

---

### D009: Use API Models for Reasoning

**Date:** 2026-05-01  
**Category:** Model Selection  
**Status:** ✅ Approved

**Decision:** Use API models (Claude, Gemini) for reasoning tasks.

**Rationale:**
- Best quality reasoning
- No VRAM contention
- Faster than local 30B models
- Context windows up to 200K tokens
- Multimodal capabilities (especially Gemini)
- Cost is reasonable for investigation workloads

**Alternatives Considered:**
- **qwen3:30b (local):** Good quality, but exceeds 16GB VRAM and requires CPU offload
- **llama3:70b (local):** Excellent quality, but requires 40GB+ VRAM
- **qwen3:8b (local):** Fits in VRAM, but lower reasoning quality

**Trade-offs:**
- ✅ Best quality reasoning
- ✅ No VRAM contention
- ✅ Fast inference
- ✅ Large context windows
- ❌ Requires internet connection
- ❌ API costs (though reasonable)
- ❌ Data sent to third party

**Research Sources:**
- Local LLM performance benchmarks
- Model comparison tools
- Cost analysis

---

### D010: Use Local Models for Extraction

**Date:** 2026-05-01  
**Category:** Model Selection  
**Status:** ✅ Approved

**Decision:** Use local models (qwen3:8b) for entity extraction and other structured tasks.

**Rationale:**
- Cost-effective (no API costs)
- Sufficient quality for extraction tasks
- Fits comfortably in 16GB VRAM (~5GB)
- Fast inference
- Privacy (data stays local)
- Can run overnight without cost concerns

**Alternatives Considered:**
- **Claude Haiku (API):** Excellent quality, but adds cost
- **Gemini Flash (API):** Fast and cheap, but still adds cost
- **qwen3:30b (local):** Better quality, but exceeds VRAM

**Trade-offs:**
- ✅ Cost-effective
- ✅ Sufficient quality for extraction
- ✅ Fits in VRAM
- ✅ Privacy
- ❌ Lower quality than API models
- ❌ Slower than API models

**Research Sources:**
- Local LLM performance benchmarks
- Ollama model comparison
- Quality vs cost analysis

---

### D011: Use jina-embeddings-v5 for Embeddings

**Date:** 2025-05-02  
**Category:** Model Selection  
**Status:** ✅ Approved

**Decision:** Use jinaai/jina-embeddings-v5-text-small as the primary embedding model.

**Rationale:**
- 677M parameters (compact)
- 1024-dimensional embeddings (high quality)
- 32K token context window (long documents)
- Multilingual support
- State-of-the-art on MTEB leaderboard
- Matryoshka Representation Learning (truncation-friendly)

**Alternatives Considered:**
- **nomic-embed-text:** Good quality/speed under 200M params, but lower dimension (768)
- **bge-large:** Higher quality, but larger (335M params)
- **mxbai-embed-large:** Good quality, but larger

**Trade-offs:**
- ✅ High quality (state-of-the-art on MTEB)
- ✅ Long context window (32K tokens)
- ✅ Multilingual support
- ✅ Truncation-friendly
- ❌ Slightly larger than nomic-embed-text
- ❌ Newer model, less battle-tested

**Research Sources:**
- jina-embeddings-v5 documentation
- MTEB leaderboard
- Embedding model benchmarks 2025

---

### D012: Use Named Graphs for Targeted Analysis

**Date:** 2026-05-01  
**Category:** Graph Architecture  
**Status:** ✅ Approved

**Decision:** Use Neo4j named graphs for targeted analysis with GDS algorithms.

**Rationale:**
- Run algorithms on specific subsets (faster, more meaningful)
- Avoid running PageRank/Louvain on entire graph (hours vs minutes)
- Agent can create graphs autonomously when patterns detected
- Different graph types for different analyses (temporal, financial, etc.)
- Better resource utilization

**Alternatives Considered:**
- **Single monolithic graph:** Simpler, but algorithms are slow and results are coarse
- **Cypher filtering:** Flexible, but doesn't benefit from GDS optimizations
- **Separate databases:** More isolation, but complex to manage

**Trade-offs:**
- ✅ Faster algorithm execution
- ✅ More meaningful results
- ✅ Better resource utilization
- ❌ More complex management
- ❌ Additional storage overhead

**Research Sources:**
- Neo4j GDS documentation
- Named graph best practices
- Performance considerations

---

### D013: Use Provenance Graph for Reasoning Trace

**Date:** 2026-05-01  
**Category:** Knowledge Representation  
**Status:** ✅ Approved

**Decision:** Maintain a separate provenance graph tracking claims, inferences, and sources.

**Rationale:**
- Full traceability of every claim
- Can verify how conclusions were reached
- Identifies contradictions explicitly
- Enables "show your work" functionality
- Critical for trust and verification

**Alternatives Considered:**
- **Journal only:** Human-readable, but not queryable
- **Log files:** Detailed, but not structured
- **No provenance:** Simpler, but loses trust and verification

**Trade-offs:**
- ✅ Full traceability
- ✅ Queryable reasoning chains
- ✅ Explicit contradiction tracking
- ❌ Additional storage overhead
- ❌ More complex data model

**Research Sources:**
- Graph RAG best practices
- Provenance tracking in AI systems
- Trust and verification requirements

---

### D014: Use Investigation Journal for Context Compression

**Date:** 2026-05-01  
**Category:** State Management  
**Status:** ✅ Approved

**Decision:** Maintain a rolling investigation journal with AI-generated summaries for context reloading.

**Rationale:**
- Human-readable record of investigation
- Context compression for efficient reloading
- Survives laptop shutdown
- Can be opened in any markdown viewer
- Provides investigation history

**Alternatives Considered:**
- **Full state only:** Machine-readable, but not human-friendly
- **No journal:** Simpler, but loses human-readable history
- **Database only:** Queryable, but not as accessible

**Trade-offs:**
- ✅ Human-readable
- ✅ Context compression
- ✅ Accessible (markdown)
- ❌ Additional I/O overhead
- ❌ Requires summary generation

**Research Sources:**
- LangGraph checkpointing patterns
- Context window management
- Investigation best practices

---

### D015: Use SQLite for Checkpoints and Ledger

**Date:** 2026-05-01  
**Category:** Persistence  
**Status:** ✅ Approved

**Decision:** Use SQLite for LangGraph checkpoints and ingestion ledger.

**Rationale:**
- Single file (easy to backup)
- No external service required
- Reliable and well-tested
- Good performance for read-heavy workloads
- FTS5 full-text search support
- ACID compliance

**Alternatives Considered:**
- **PostgreSQL:** More features, but requires external service
- **Redis:** Faster, but not durable by default
- **File-based JSON:** Simpler, but no query capabilities

**Trade-offs:**
- ✅ Single file
- ✅ No external service
- ✅ Reliable
- ❌ Single writer only
- ❌ Not suitable for high-concurrency

**Research Sources:**
- SQLite documentation
- LangGraph SQLite checkpointer
- Database comparison for local apps

---

## Pending Decisions

### PD001: GPU Time-Slicing Strategy

**Status:** ⏳ Pending  
**Category:** Resource Management  
**Context:** RTX 4080 has 16GB VRAM, need to manage contention between services.

**Options:**
1. **Time-slice with mutex:** Services acquire GPU lock, release when done
2. **Priority-based:** Embedding gets priority during ingestion, LLM during investigation
3. **Separate processes:** Run services on different GPUs (not applicable here)
4. **API models for reasoning:** Eliminate local LLM VRAM usage

**Recommendation:** Option 4 (API models for reasoning) + Option 1 (mutex for local services)

**Decision Date:** During Phase 0 implementation

---

### PD002: Corpus Storage Strategy

**Status:** ⏳ Pending  
**Category:** Storage Management  
**Context:** 300GB corpus requires significant storage and efficient access.

**Options:**
1. **Keep original files:** Store raw corpus, parse on demand
2. **Pre-parse everything:** Store Docling output, discard originals
3. **Hybrid:** Keep originals for reference, cache parsed output
4. **Compressed storage:** Compress parsed output to save space

**Recommendation:** Option 3 (hybrid) - keep originals, cache parsed output

**Decision Date:** During Phase 1 implementation

---

### PD003: Investigation Concurrency Strategy

**Status:** ⏳ Pending  
**Category:** Orchestration  
**Context:** Multiple investigations may run simultaneously.

**Options:**
1. **Single SQLite file:** All investigations share one checkpoint DB
2. **Separate SQLite files:** One file per investigation
3. **PostgreSQL:** Single database with separate tables
4. **Queue investigations:** Run one at a time

**Recommendation:** Option 2 (separate SQLite files) - avoids SQLite single-writer limitation

**Decision Date:** During Phase 3 implementation

---

## Rejected Decisions

### R001: Rust-Based Crawler

**Date:** 2026-05-01  
**Status:** ❌ Rejected

**Original Proposal:** Use Rust for distributed web crawling.

**Reason for Rejection:**
- Bottleneck is GPU throughput (embedding), not network I/O
- Python asyncio with 4 workers saturates infinity-emb completely
- Rust complexity not justified given actual bottleneck profile
- Can revisit if crawling millions of web pages at high speed

**Alternative:** Python asyncio ingestion pipeline

---

### R002: Temporal for Orchestration

**Date:** 2026-05-01  
**Status:** ❌ Rejected

**Original Proposal:** Use Temporal for durable execution.

**Reason for Rejection:**
- Significant operational overhead for single-machine setup
- LangGraph SQLite checkpointing provides perfect pause/resume semantics
- Temporal better suited for distributed cloud workflows
- Additional service to manage

**Alternative:** LangGraph with SQLite checkpointing

---

### R003: PyMuPDF4LLM as Primary Parser

**Date:** 2026-05-01  
**Status:** ❌ Rejected

**Original Proposal:** Use pymupdf4llm as primary document parser.

**Reason for Rejection:**
- Doesn't handle audio/video
- Would require separate pipeline branches
- Docling provides unified pipeline for all modalities
- Can use pymupdf4llm as fallback for simple text PDFs

**Alternative:** Docling as primary, LiteParse as fallback

---

### D016: Scope Lock — Single Machine v1, Swarm v2

**Date:** 2025-05-02  
**Category:** Architecture  
**Status:** ✅ Approved

**Decision:** Implement v1 as single-machine deployment, v2 as swarm upgrade.

**Rationale:**
- v1: SQLite checkpointing, direct function calls, no message queue
- v2: PostgreSQL checkpointing, NATS JetStream, LiteLLM Gateway
- LangGraph checkpointer is swappable (SQLite ↔ PostgreSQL)
- Agent logic unchanged between v1 and v2

**v1 Architecture:**
- Single investigation at a time (SQLite write lock)
- No horizontal scaling
- No distributed task distribution
- All services run on one machine

**v2 Architecture:**
- Multiple concurrent investigations
- Horizontal scaling across multiple machines
- Distributed task distribution
- Multi-provider model routing with failover

**Migration Path:**
1. Set up PostgreSQL
2. Migrate SQLite checkpoints
3. Set up NATS JetStream
4. Implement worker Docker images
5. Implement LiteLLM Gateway
6. Refactor agent nodes to publish to NATS

**Trade-offs:**
- ✅ v1: Faster iteration, lower risk, clear validation
- ✅ v2: Horizontal scaling, fault tolerance, auto-recovery
- ❌ v1: Limited to single machine
- ❌ v2: More complex, requires additional infrastructure

**Research Sources:**
- LangGraph checkpoint implementations documentation
- LangGraph persistence docs
- NATS JetStream documentation
- LiteLLM documentation

---

### D017: GPU Mutex Design and Validation

**Date:** 2025-05-02  
**Category:** System Architecture  
**Status:** ✅ Approved

**Decision:** Implement asyncio.Lock-based GPU mutex with priority queue.

**Rationale:**
- infinity-emb: ~1GB static, ~2GB peak
- GLiNER2: ~1GB static, ~3GB peak
- Total: ~3GB static, ~7GB peak (fits in 16GB VRAM)
- Simultaneous batch processing could exceed VRAM

**Priority Levels:**
1. INGESTION_EMBEDDING (highest during ingestion)
2. INVESTIGATION_REASONING (medium during investigation)
3. VISUALIZATION (lowest)

**Design:**
- asyncio.Lock-based mutex
- Priority queue for GPU access
- Timeout on acquire (default 300 seconds)
- Deadlock detection and prevention

**Validation Tests:**
1. Sequential loading (VRAM < 3GB)
2. Simultaneous processing (OOM expected)
3. Mutex behavior (sequential execution, no OOM)

**Trade-offs:**
- ✅ Prevents VRAM overflow
- ✅ Fair access to GPU
- ✅ Priority-based scheduling
- ❌ Adds complexity to code
- ❌ Potential for priority inversion

**Research Sources:**
- GLiNER2 performance optimization documentation
- infinity-emb GitHub repository
- PyTorch memory management documentation

---

### D018: Model Stack Finalization

**Date:** 2025-05-02  
**Category:** Model Selection  
**Status:** ✅ Approved

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

**Trade-offs:**
- ✅ Free API providers (no cost)
- ✅ High-quality reasoning
- ✅ Local entity extraction (privacy)
- ❌ Rate limits on free tiers
- ❌ Dependency on external services

**Research Sources:**
- GLiNER2 GitHub repository
- jina-embeddings-v5 documentation
- Groq rate limits documentation
- OpenRouter rate limits documentation
- Gemini rate limits documentation
- LiteLLM documentation

---

## Decision Review Schedule

- **Weekly:** Review pending decisions during implementation
- **Monthly:** Review approved decisions for validity
- **Quarterly:** Comprehensive architecture review
- **As needed:** Revisit decisions if new information emerges

---

**Document Maintainer:** ORACLE Development Team  
**Review Cycle:** Monthly  
**Next Review:** 2026-06-01
