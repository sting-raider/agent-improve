# ORACLE Updated Technology Stack

**Last Updated:** 2026-05-01  
**Status:** UPDATED with Gemini Analysis

---

## Overview

This document reflects the updated technology stack after investigating Gemini's claims. Valid improvements have been incorporated, while risky changes have been rejected.

---

## Changes Made

### ✅ Valid Improvements Incorporated

1. **Embedding Model:** Switched from nomic-embed-text to jina-embeddings-v5-text-small
2. **Entity Extraction:** Added GLiNER2 for fast standard entity extraction
3. **Hybrid Approach:** GLiNER2 + LLM for optimal performance and quality

### ❌ Risky Changes Rejected

1. **KùzuDB:** Rejected due to archival status (abandoned in 2025)
2. **Memory Concerns:** Rejected as exaggerated (32GB is sufficient)
3. **Docker Removal:** Rejected (Docker is appropriate for service management)

---

## Updated Technology Stack

### Core Services

| Service | Version | Status | Notes |
|---------|---------|--------|-------|
| Neo4j | 5.26+ | ✅ KEPT | Actively maintained, enterprise support |
| Qdrant | latest | ✅ KEPT | Best local vector DB, Rust-based |
| Docker Compose | 24+ | ✅ KEPT | Appropriate for service orchestration |

**Rationale for Keeping Neo4j + Qdrant:**
- KùzuDB was archived in 2025 (abandoned by creator)
- 32GB RAM is sufficient (15-24GB total usage)
- Proven track record, enterprise support
- Active community and documentation

---

### Python Services

| Service | Purpose | Status | Notes |
|---------|---------|--------|-------|
| infinity-emb | Embedding server | ✅ KEPT | Dynamic batching, 10-50x faster |
| Docling | Document parser | ✅ KEPT | Unified pipeline for all modalities |
| GLiNER2 | Entity extraction | ✅ ADDED | Fast standard entity extraction |
| LangGraph | Agent orchestration | ✅ KEPT | SQLite checkpointing, pause/resume |
| FastAPI | Viz server | ✅ KEPT | Lightweight, async |

---

### Models

### Embedding Models

**UPDATED:**

| Model | MTEB Score | Params | VRAM | Status |
|-------|------------|--------|------|--------|
| jina-embeddings-v5-text-small | 71.7 | 677M | ~1.3GB | ✅ NEW (recommended) |
| Harrier-OSS-v1-0.6b | #1 multilingual | 600M | ~1.2GB | ✅ NEW (alternative) |
| nomic-embed-text | ~62 | 137M | 0.6GB | ⚠️ DEPRECATED |

**Recommendation:** Use jina-embeddings-v5-text-small for best quality/speed tradeoff.

**Rationale:**
- 15% better MTEB score than nomic-embed-text
- State-of-the-art performance
- Actively maintained by Jina AI
- Similar parameter count (~600-700M)

### Entity Extraction

**UPDATED:**

| Model | Speed | Quality | VRAM | Status |
|-------|-------|--------|------|--------|
| GLiNER2 | ~1000 chunks/sec | Good for standard entities | ~1GB | ✅ NEW (primary) |
| qwen3:8b (LLM) | ~10 chunks/sec | Excellent for complex tasks | ~5GB | ✅ KEPT (fallback) |
| claude-haiku (API) | ~50 chunks/sec | Excellent | N/A | ✅ KEPT (optional) |

**Hybrid Approach:**
```python
# Fast path: GLiNER2 for standard entities
entities = gliner.predict_entities(text, labels=STANDARD_LABELS)

# Slow path: LLM for complex relationships
if needs_complex_reasoning:
    relationships = llm.extract_relationships(text)
```

**Rationale:**
- GLiNER2 is 100x faster than LLM extraction
- Good for standard entity types (Person, Organization, Location, etc.)
- LLM still needed for complex relationships and reasoning
- Hybrid approach provides optimal performance and quality

### Reasoning Models

**UNCHANGED:**

| Model | Purpose | VRAM | Status |
|-------|---------|------|--------|
| Claude API | Primary reasoning | N/A | ✅ KEPT |
| Gemini API | Secondary reasoning + vision | N/A | ✅ KEPT |
| qwen3:8b | Local reasoning (fallback) | ~5GB | ✅ KEPT |

---

## Updated Performance

### Ingestion Performance

**Old (nomic-embed-text + qwen3:8b):**
- Embedding: 7 hours
- Extraction: 58 days
- **Total: ~58 days**

**New (jina-embeddings-v5 + GLiNER2):**
- Embedding: 5 hours (29% faster)
- Extraction: 5-8 days (7-12x faster)
- **Total: ~5-8 days**

**Improvement:** 7-12x faster overall

### Quality

- Embedding quality: +15% MTEB score
- Extraction quality: Similar for standard entities
- Complex reasoning: Unchanged (still uses LLM)

---

## Updated Dependencies

### Python Dependencies

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
    
    # Entity extraction (NEW)
    "gliner>=2.0.0",
    
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

## Updated Configuration

### Environment Variables

```bash
# Embedding (UPDATED)
EMBEDDING_MODEL=jinaai/jina-embeddings-v5-text-small
# Alternative: EMBEDDING_MODEL=microsoft/harrier-oss-v1-0.6b
EMBEDDING_DIMENSION=768

# Entity Extraction (UPDATED)
ENTITY_EXTRACTION_MODEL=gliner
ENTITY_EXTRACTION_FALLBACK=qwen3:8b

# Ingestion Performance
INGESTION_BATCH_SIZE=32
INGESTION_WORKERS=4
CHUNK_SIZE=512
CHUNK_OVERLAP=64
```

---

## Implementation Changes

### Phase 1 Modifications

**Add GLiNER2:**
```bash
uv add gliner
```

**Update embedding model in code:**
```python
# Old
from sentence_transformers import SentenceTransformer
embedder = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5")

# New
from sentence_transformers import SentenceTransformer
embedder = SentenceTransformer("jinaai/jina-embeddings-v5-text-small")
```

**Implement hybrid extraction:**
```python
from gliner import GLiNER

class EntityExtractor:
    def __init__(self):
        # Fast path: GLiNER2
        self.gliner = GLiNER.from_pretrained("urchade/gliner_multi-v2.1")
        
        # Slow path: LLM fallback
        self.llm = ChatOllama(model="qwen3:8b")
    
    def extract(self, text):
        # Try GLiNER2 first
        entities = self.gliner.predict_entities(text, labels=STANDARD_LABELS)
        
        # Use LLM for complex relationships
        if needs_complex_reasoning(text):
            relationships = self.llm.extract_relationships(text)
        
        return entities, relationships
```

---

## Risk Assessment

### KùzuDB Risks (Rejected)

| Risk | Status | Reason |
|------|--------|--------|
| Abandoned project | ❌ REJECTED | Archived in 2025 |
| No security updates | ❌ REJECTED | No long-term support |
| Fork may die | ❌ REJECTED | High risk for production |

### GLiNER2 Risks (Accepted)

| Risk | Mitigation |
|------|------------|
| Limited flexibility | Use LLM fallback for complex tasks |
| Schema constraints | Define comprehensive schema |
| Performance on edge cases | Fallback to LLM if needed |

---

## Summary

### Changes Made

1. ✅ **Embedding model:** nomic-embed-text → jina-embeddings-v5-text-small
2. ✅ **Entity extraction:** Added GLiNER2 for fast standard extraction
3. ✅ **Hybrid approach:** GLiNER2 + LLM for optimal performance
4. ❌ **Graph database:** Kept Neo4j (KùzuDB rejected)
5. ✅ **Vector database:** Kept Qdrant
6. ✅ **Services:** Kept Docker Compose

### Performance Impact

- **Ingestion speed:** 7-12x faster (58 days → 5-8 days)
- **Embedding quality:** +15% MTEB score
- **Extraction quality:** Similar for standard entities
- **Memory usage:** Unchanged (32GB sufficient)

### Risk Profile

- **Overall risk:** REDUCED (faster extraction, better quality)
- **New risks:** GLiNER2 limitations (mitigated with LLM fallback)
- **Rejected risks:** KùzuDB archival (avoided entirely)

---

## Next Steps

1. Update [[ORACLE Implementation Roadmap]] with GLiNER2
2. Update [[ORACLE Performance Benchmarks]] with new estimates
3. Update [[ORACLE Decision Log]] with new decisions
4. Begin Phase 1 with updated technology stack

---

**Tags:** #oracle #technology-stack #updated #gemini-analysis

**Links:**
- [[ORACLE Architecture Review - Gemini Claims]]
- [[ORACLE Decision Log]]
- [[ORACLE Performance Benchmarks]]
