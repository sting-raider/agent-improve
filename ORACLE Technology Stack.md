# ORACLE Technology Stack

**Last Updated:** 2026-05-01

---

## Overview

ORACLE uses a carefully selected technology stack optimized for local deployment, GPU acceleration, and deep research capabilities. All choices are validated by 2025-2026 research.

---

## Core Services

### Neo4j 5.26+

**Purpose:** Graph database for entities, relationships, and provenance.

**Why Neo4j:**
- Mature ecosystem and tooling
- Graph Data Science (GDS) plugin works on Community edition
- Excellent Python driver
- Cypher query language is powerful and expressive
- Named graphs for targeted analysis

**Key Features:**
- Property graph model
- Index-free adjacency for fast traversal
- GDS algorithms (PageRank, Louvain, etc.)
- ACID compliance
- Bolt protocol for efficient communication

**Alternatives Considered:**
- Memgraph (faster, smaller community)
- ArangoDB (multi-model, less mature graph features)

**Configuration:**
- Memory: 4GB heap, 2GB pagecache
- Plugins: graph-data-science, apoc
- Authentication: enabled

---

### Qdrant

**Purpose:** Vector database for semantic search over document chunks.

**Why Qdrant:**
- Written in Rust (memory-safe, fast)
- Excellent filtered search performance
- Open source (no vendor lock-in)
- Easy Docker deployment
- Handles 50K+ QPS on modest hardware

**Key Features:**
- HNSW indexing for fast approximate search
- Filtered search support
- Payload storage with vectors
- REST and gRPC APIs
- Good Python client

**Alternatives Considered:**
- Milvus (more scalable, but complex setup)
- Weaviate (built-in graph features, but slower for pure vector search)
- Pinecone (excellent performance, but not open source)

**Configuration:**
- Collection: oracle_corpus
- Vector dimension: 768 (nomic-embed-text)
- HNSW parameters: M=16, ef_construct=100

---

### Docker Compose

**Purpose:** Orchestrate Neo4j and Qdrant services.

**Why Docker Compose:**
- Simple service definition
- Automatic dependency management
- Volume mounting for persistence
- Health checks
- Easy restart and logging

**Services:**
- neo4j: Graph database
- qdrant: Vector database

---

## Python Services

### infinity-emb

**Purpose:** High-throughput embedding server.

**Why infinity-emb:**
- Dynamic batching (GPU never idle)
- OpenAI-compatible API
- Supports TensorRT backend
- 10-50x throughput improvement over direct sentence-transformers
- MIT license

**Key Features:**
- Dynamic batching optimizes GPU utilization
- Multiple model support
- REST API
- CUDA optimization
- FlashAttention support

**Alternatives Considered:**
- sentence-transformers (direct usage, GPU idle between batches)
- vLLM (excellent for LLMs, overkill for embeddings)
- TEI (good alternative, but infinity-emb has better dynamic batching)

**Configuration:**
- Model: nomic-ai/nomic-embed-text-v1.5
- Device: cuda
- Port: 7997
- Batch size: dynamic

---

### Docling

**Purpose:** Multi-modal document parser.

**Why Docling:**
- Unified pipeline for PDF, DOCX, images, video, audio
- Built-in OCR for scanned documents
- Whisper Turbo integration for audio/video
- Table and figure preservation
- Consistent DoclingDocument output format
- IBM-backed, actively maintained

**Key Features:**
- Multi-format support
- OCR capabilities
- Audio/video transcription
- Table extraction
- Figure preservation
- Markdown output

**Alternatives Considered:**
- pymupdf4llm (faster for simple PDFs, but no audio/video)
- LiteParse (good fallback for simple documents)
- Unstructured (good for diverse formats, but less consistent output)

**Configuration:**
- OCR: enabled
- Table structure: enabled
- ASR model: Whisper Turbo

---

### LangGraph

**Purpose:** Agent orchestration with checkpointing.

**Why LangGraph:**
- Built specifically for LLM agents
- SQLite checkpointing (simple, reliable)
- Typed state management with Pydantic
- Conditional routing
- Perfect pause/resume semantics
- Active development and community

**Key Features:**
- State machine definition
- Checkpointing with multiple backends
- Conditional routing
- Sub-graph support
- Message passing
- Streaming support

**Alternatives Considered:**
- Temporal (more durable for distributed systems, but significant operational overhead)
- Prefect (better for data pipelines, less suited for agentic workflows)
- Custom state management (would require building checkpointing from scratch)

**Configuration:**
- Checkpointer: AsyncSqliteSaver
- State: InvestigationState (TypedDict)
- Nodes: 12 (initialize, plan, select, research, integrate, check, synthesize, report)

---

### FastAPI

**Purpose:** Visualization server for graph data.

**Why FastAPI:**
- Modern, fast Python web framework
- Automatic API documentation
- Type hints for validation
- Async support
- Easy deployment with uvicorn

**Key Features:**
- REST API
- Automatic OpenAPI docs
- Request validation
- Async support
- WebSocket support

**Alternatives Considered:**
- Flask (simpler, but less modern)
- Django (overkill for this use case)
- Tornado (good for async, but less popular)

**Configuration:**
- Port: 8765
- Workers: 1 (uvicorn)
- CORS: enabled for local development

---

## Models

### nomic-embed-text

**Purpose:** Local embedding model.

**Why nomic-embed-text:**
- Best quality/speed tradeoff under 200M parameters
- Excellent MTEB scores (~62)
- Very fast inference (137M params)
- Low VRAM usage (~0.6GB)
- Good for large-scale search and memory systems
- Open source

**Specifications:**
- Parameters: 137M
- Dimensions: 768
- Context length: 8192 tokens
- VRAM: ~0.6GB
- MTEB score: ~62

**Alternatives Considered:**
- bge-large (higher quality MTEB ~64, but slower and more VRAM)
- mxbai-embed-large (highest quality MTEB ~65, but slower)
- Cohere embed-v4 (best quality, but API-only and costly)

---

### qwen3:8b

**Purpose:** Local model for entity extraction and structured tasks.

**Why qwen3:8b:**
- Cost-effective (no API costs)
- Sufficient quality for extraction tasks
- Fits comfortably in 16GB VRAM (~5GB)
- Fast inference
- Privacy (data stays local)
- Can run overnight without cost concerns

**Specifications:**
- Parameters: 8B
- Context length: 32K tokens
- VRAM (Q4_K_M): ~5GB
- Quality: Good for extraction

**Alternatives Considered:**
- Claude Haiku (excellent quality, but adds cost)
- Gemini Flash (fast and cheap, but still adds cost)
- qwen3:30b (better quality, but exceeds VRAM)

---

### Claude API

**Purpose:** Primary reasoning model.

**Why Claude:**
- Best quality reasoning
- Large context windows (up to 200K tokens)
- Multimodal capabilities
- Excellent instruction following
- Reliable and fast

**Models Used:**
- claude-sonnet-4-5 (primary reasoning)
- claude-haiku-4-5 (fast tasks)
- claude-opus-4-6 (complex reasoning)

**Alternatives Considered:**
- Gemini (excellent vision, good reasoning)
- OpenAI GPT-4 (good quality, but more expensive)
- Local models (lower quality, VRAM constraints)

---

### Gemini API

**Purpose:** Secondary reasoning and vision model.

**Why Gemini:**
- Excellent vision capabilities
- Good reasoning quality
- Fast inference
- Large context windows
- Competitive pricing

**Models Used:**
- gemini-2.0-flash (fast reasoning and vision)
- gemini-2.0-flash-thinking-exp (complex reasoning)

**Alternatives Considered:**
- Claude (better reasoning, but vision is newer)
- GPT-4 Vision (good, but more expensive)

---

## Frameworks

### Textual

**Purpose:** Terminal User Interface framework.

**Why Textual:**
- Created by Will McGugan (Rich author)
- Modern, async architecture
- 16.7M colors
- Mouse support
- Web and terminal rendering
- Active community and good documentation
- De facto Python TUI standard

**Key Features:**
- Widget-based architecture
- Async event loop
- CSS styling
- Rich text rendering
- Layout system
- Animation support

**Alternatives Considered:**
- Rich (lower-level, Textual is built on it)
- Prompt Toolkit (older, less feature-rich)
- Urwid (very old, not recommended)

---

### 3d-force-graph

**Purpose:** WebGL graph visualization library.

**Why 3d-force-graph:**
- WebGL rendering (GPU-accelerated)
- 3D force-directed layout
- Handles 50,000+ nodes smoothly
- Built on Three.js
- Cyberpunk aesthetic possible with post-processing
- Good performance at scale

**Key Features:**
- 3D force-directed layout
- WebGL rendering
- Particle effects on edges
- Node sizing and coloring
- Interactive controls
- Post-processing (bloom, glow)

**Alternatives Considered:**
- Cytoscape.js (good for 2D, but 3D support is limited)
- D3.js (excellent for small graphs, but DOM-based rendering fails at scale)
- Gephi (desktop application, not web-based)
- vis-network (good for 2D, but no 3D support)

---

### MCP (Model Context Protocol)

**Purpose:** Universal standard for AI tool integration.

**Why MCP:**
- Universal standard (97M+ monthly SDK downloads)
- Backed by Anthropic, OpenAI, Google, Microsoft
- Open standard with good documentation
- STDIO and SSE transport options
- Resources, Tools, and Prompts primitives
- Growing ecosystem of servers

**Key Features:**
- Resources (read-only data sources)
- Tools (executable functions)
- Prompts (reusable templates)
- STDIO and SSE transport
- JSON-RPC protocol
- Growing ecosystem

**Alternatives Considered:**
- Custom integrations (more control, but requires building everything)
- LangChain tools (good, but not standardized across providers)
- OpenAI function calling (provider-specific, not universal)

---

## Python Dependencies

### Core Dependencies

```toml
[project]
name = "oracle"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    # Document parsing
    "docling>=2.0.0",
    "faster-whisper>=1.0.0",
    
    # Embedding
    "infinity-emb[all]>=0.0.50",
    "sentence-transformers>=3.0.0",
    
    # Vector store
    "qdrant-client>=1.9.0",
    
    # Graph database
    "neo4j>=5.0.0",
    
    # Agent orchestration
    "langgraph>=0.2.0",
    "langgraph-checkpoint-sqlite>=2.0.0",
    "langchain-core>=0.3.0",
    "langchain-anthropic>=0.3.0",
    "langchain-google-genai>=2.0.0",
    "langchain-ollama>=0.2.0",
    
    # MCP
    "fastmcp>=0.1.0",
    "mcp>=1.0.0",
    
    # TUI
    "textual>=0.80.0",
    "rich>=13.0.0",
    
    # Visualization
    "fastapi>=0.115.0",
    "uvicorn>=0.30.0",
    "websockets>=12.0",
    
    # Data & utilities
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "aiosqlite>=0.20.0",
    "aiofiles>=23.0.0",
    "httpx>=0.27.0",
    "tenacity>=8.0.0",
    "structlog>=24.0.0",
    "jinja2>=3.1.0",
    "tiktoken>=0.7.0",
    "python-dotenv>=1.0.0",
    
    # Code sandbox
    "docker>=7.0.0",
    
    # Web search
    "tavily-python>=0.3.0",
]
```

---

## Development Tools

### uv

**Purpose:** Fast Python package and project manager.

**Why uv:**
- Lightning-fast dependency resolution
- Fast virtual environment creation
- Lockfile support for reproducibility
- Written in Rust (fast and reliable)

---

### Docker

**Purpose:** Container runtime for Neo4j and Qdrant.

**Why Docker:**
- Isolated service environments
- Easy deployment and management
- Volume mounting for persistence
- Health checks
- Automatic restart

---

### Ollama

**Purpose:** Local LLM runtime.

**Why Ollama:**
- Simple local LLM deployment
- GPU acceleration
- Model management
- API-compatible interface
- Active model library

---

## Related Topics

- [[ORACLE Architecture]] — System architecture
- [[ORACLE Decision Log]] — Technology choices and rationale
- [[ORACLE Implementation Roadmap]] — Implementation phases

---

**Tags:** #oracle #technology-stack #dependencies #tools

**Links:**
- [[ORACLE Project Overview]]
- [[ORACLE Architecture]]
- [[ORACLE Decision Log]]
