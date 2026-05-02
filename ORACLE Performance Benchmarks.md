# ORACLE Performance Benchmarks

**Last Updated:** 2026-05-01  
**Hardware:** RTX 4080 (16GB VRAM), i9-13900HX (24 cores), 32GB RAM

---

## Overview

This document outlines expected performance characteristics for ORACLE across ingestion, querying, and visualization workloads. All benchmarks are based on the specified hardware configuration.

---

## Ingestion Performance

### Document Parsing

| Document Type | Throughput | Notes |
|---------------|------------|-------|
| Simple PDF (text-only) | ~100 docs/sec | CPU-bound, 4 workers |
| Complex PDF (tables, figures) | ~50 docs/sec | CPU-bound, 4 workers |
| Scanned PDF (OCR required) | ~20 docs/sec | OCR is CPU-intensive |
| Images (OCR) | ~30 images/sec | OCR is CPU-intensive |
| Video (transcription) | ~1 min/sec | Whisper Turbo is fast |
| Audio (transcription) | ~2 min/sec | Whisper Turbo is fast |

**Total for 300GB corpus:**
- ~1M documents estimated
- Parsing time: ~5.5 hours (simple), ~11 hours (complex)

### Embedding

| Model | Throughput | VRAM | Notes |
|-------|------------|------|-------|
| nomic-embed-text | ~2000 chunks/sec | 0.6GB | GPU-saturated |
| bge-large | ~1000 chunks/sec | 1.3GB | Higher quality, slower |
| mxbai-embed-large | ~800 chunks/sec | 1.3GB | Highest quality, slowest |

**Total for 50M chunks:**
- nomic-embed-text: ~7 hours
- bge-large: ~14 hours
- mxbai-embed-large: ~17 hours

### Entity Extraction

| Model | Throughput | VRAM | Notes |
|-------|------------|------|-------|
| qwen3:8b | ~10 chunks/sec | 5GB | LLM-bound |
| claude-haiku (API) | ~50 chunks/sec | N/A | Faster, but costs money |
| gemini-flash (API) | ~40 chunks/sec | N/A | Fast, but costs money |

**Total for 50M chunks:**
- qwen3:8b: ~58 days (overnight batch jobs)
- claude-haiku: ~12 days (faster, but costs money)

### Overall Ingestion Time

**For 300GB corpus (~1M documents, ~50M chunks):**

| Configuration | Parsing | Embedding | Extraction | Total |
|---------------|---------|-----------|------------|-------|
| Fast (API extraction) | 5.5h | 7h | 12d | ~12.5 days |
| Balanced (local extraction) | 5.5h | 7h | 58d | ~58.5 days |
| Quality (bge-large) | 11h | 14h | 58d | ~58.5 days |

**Recommendation:** Run extraction overnight in batches. Use API models for faster turnaround if budget allows.

---

## Query Performance

### Vector Search (Qdrant)

| Operation | Latency | Throughput | Notes |
|-----------|---------|------------|-------|
| Single query | 10-50ms | 10K+ QPS | Depends on collection size |
| Batch query (100) | 50-200ms | 5K+ QPS | Efficient batching |
| Filtered query | 20-100ms | 5K+ QPS | Slightly slower |

**Scaling:**
- 1M vectors: ~10-50ms latency
- 10M vectors: ~20-100ms latency
- 100M vectors: ~50-200ms latency

### Graph Traversal (Neo4j)

| Operation | Latency | Notes |
|-----------|---------|-------|
| 1-hop query | 5-20ms | Direct neighbors |
| 2-hop query | 10-100ms | Neighbors of neighbors |
| 3-hop query | 20-200ms | Three levels deep |
| 5-hop query | 50-500ms | Five levels deep |
| Path finding (shortest) | 10-100ms | Between two entities |
| Path finding (all) | 50-500ms | All paths up to N hops |

**Scaling:**
- 100K nodes: ~5-50ms for 2-hop
- 1M nodes: ~10-100ms for 2-hop
- 10M nodes: ~20-200ms for 2-hop

### GDS Algorithms

| Algorithm | Graph Size | Time | Notes |
|-----------|------------|------|-------|
| PageRank | 10K nodes | ~1s | Fast on small graphs |
| PageRank | 100K nodes | ~10s | Moderate |
| PageRank | 1M nodes | ~100s | Slower on large graphs |
| Louvain | 10K nodes | ~2s | Community detection |
| Louvain | 100K nodes | ~20s | Moderate |
| Louvain | 1M nodes | ~200s | Slower on large graphs |

**Optimization:** Use named graphs to run algorithms on targeted subsets.

---

## Investigation Performance

### Simple Question

**Example:** "Who is connected to X?"

| Step | Time | Notes |
|------|------|-------|
| Query decomposition | 1-2s | LLM call |
| Vector search | 50-100ms | Qdrant |
| Graph traversal | 10-100ms | Neo4j |
| Specialist analysis | 5-10s | LLM calls |
| Integration | 1-2s | State update |
| **Total** | **7-15s** | Fast response |

### Complex Question

**Example:** "What is the connection between delayed shipments and revenue drop?"

| Step | Time | Notes |
|------|------|-------|
| Query decomposition | 2-5s | LLM call |
| Multiple searches | 1-5s | Multiple Qdrant queries |
| Graph traversals | 100-500ms | Multiple Neo4j queries |
| Specialist analysis | 30-120s | Multiple LLM calls |
| Integration | 5-10s | State update |
| **Total** | **5-15 minutes** | Moderate response |

### Deep Investigation

**Example:** Full investigation of Epstein files

| Phase | Time | Notes |
|-------|------|-------|
| Initial planning | 5-10 minutes | Hypothesis generation |
| Corpus search | 30-60 minutes | Multiple searches |
| Graph analysis | 1-2 hours | Multiple traversals |
| Specialist work | 4-8 hours | Multiple agents |
| Synthesis | 1-2 hours | Report generation |
| **Total** | **6-13 hours** | Deep investigation |

**Note:** Deep investigations can run for days or weeks depending on scope.

---

## Visualization Performance

### 3d-force-graph Rendering

| Node Count | FPS | Notes |
|------------|-----|-------|
| 1,000 | 60 | Smooth interaction |
| 5,000 | 45-60 | Good interaction |
| 10,000 | 30-60 | Acceptable interaction |
| 25,000 | 20-40 | Limited interaction |
| 50,000 | 15-30 | Slow interaction |
| 100,000 | 5-15 | Very slow interaction |

**Recommendation:** Limit visualization to 10,000-50,000 nodes for smooth interaction.

### Graph Export

| Graph Size | Export Time | Notes |
|------------|-------------|-------|
| 1K nodes | <1s | Instant |
| 10K nodes | 1-5s | Fast |
| 100K nodes | 10-30s | Moderate |
| 1M nodes | 2-5 minutes | Slow |

**Optimization:** Use incremental loading and node sampling for large graphs.

---

## Resource Utilization

### GPU Utilization

| Operation | VRAM Usage | GPU Utilization | Notes |
|-----------|------------|-----------------|-------|
| infinity-emb (idle) | 0.6GB | 0% | Model loaded |
| infinity-emb (active) | 0.6GB | 95-100% | GPU-saturated |
| qwen3:8b (idle) | 5GB | 0% | Model loaded |
| qwen3:8b (active) | 5GB | 80-95% | High utilization |
| 3d-force-graph | 1-2GB | 60-80% | Rendering |
| Combined (time-sliced) | 5-6GB | 80-95% | Sequential usage |

**Recommendation:** Time-slice GPU between services to avoid VRAM exhaustion.

### CPU Utilization

| Operation | CPU Usage | Notes |
|-----------|-----------|-------|
| Document parsing | 80-95% | CPU-bound, 4 workers |
| Embedding (infinity-emb) | 20-30% | GPU-bound, light CPU |
| Entity extraction (qwen3:8b) | 30-40% | LLM-bound, moderate CPU |
| Graph queries (Neo4j) | 10-20% | I/O-bound |
| Vector search (Qdrant) | 10-20% | I/O-bound |

**Recommendation:** 24 cores sufficient for parallel parsing.

### RAM Utilization

| Component | RAM Usage | Notes |
|-----------|-----------|-------|
| Neo4j | 4-8GB | Depends on graph size |
| Qdrant | 2-4GB | Depends on collection size |
| Python services | 2-4GB | Depends on workload |
| OS + other | 4-8GB | System overhead |
| **Total** | **12-24GB** | Fits in 32GB |

**Recommendation:** 32GB sufficient for current design.

### Storage Utilization

| Component | Size (300GB corpus) | Notes |
|-----------|-------------------|-------|
| Raw corpus | 300GB | User-provided |
| Parsed output | 30GB | Text is smaller |
| Qdrant vectors | 50GB | 768-dim float32 |
| Neo4j database | 20-40GB | Depends on entity density |
| Checkpoints | 5GB | Investigation state |
| **Total** | **405-425GB** | Requires 500GB+ |

**Recommendation:** 500GB+ free space required.

---

## Bottlenecks

### Primary Bottlenecks

1. **GPU VRAM** — Limits concurrent model usage
   - Mitigation: Time-slice GPU, use API models

2. **Entity Extraction** — LLM-bound, slow
   - Mitigation: Run overnight, use API models

3. **Graph Algorithms** — Slow on large graphs
   - Mitigation: Use named graphs, target subsets

4. **Storage I/O** — Limits ingestion speed
   - Mitigation: Use SSD, optimize queries

### Secondary Bottlenecks

1. **Network Latency** — API calls to external services
   - Mitigation: Use local models where possible

2. **CPU Parsing** — Document parsing is CPU-bound
   - Mitigation: More workers, better hardware

3. **Memory Bandwidth** — Limits data transfer
   - Mitigation: Optimize data structures

---

## Optimization Strategies

### Ingestion Optimization

1. **Parallel Parsing**
   - Use 4-8 workers for document parsing
   - CPU-bound, scales with cores

2. **GPU Saturation**
   - infinity-emb dynamic batching
   - Keep GPU 95%+ utilized

3. **Batch Extraction**
   - Run extraction overnight
   - Process in batches of 1000 chunks

4. **Incremental Processing**
   - Resume from failures
   - Skip already-processed files

### Query Optimization

1. **Vector Search**
   - Use filtered search when possible
   - Batch queries
   - Cache frequent queries

2. **Graph Traversal**
   - Use indexes
   - Limit result sets
   - Use named graphs

3. **GDS Algorithms**
   - Run on named graphs
   - Cache results
   - Use incremental algorithms

### Visualization Optimization

1. **Node Sampling**
   - Limit to 10K-50K nodes
   - Sample by importance (PageRank)

2. **Incremental Loading**
   - Load nodes in batches
   - Stream from database

3. **Level of Detail**
   - Show less detail at zoom out
   - Show more detail at zoom in

---

## Testing Strategy

### Unit Tests

- Test each component in isolation
- Mock external dependencies
- Verify correctness

### Integration Tests

- Test component interactions
- Use test databases
- Verify end-to-end flows

### Performance Tests

- Benchmark critical paths
- Measure latency and throughput
- Identify bottlenecks

### Load Tests

- Test with large datasets
- Verify scalability
- Monitor resource usage

### Stress Tests

- Test under heavy load
- Verify error handling
- Test recovery procedures

---

## Monitoring

### Metrics to Track

1. **Ingestion Metrics**
   - Documents processed per second
   - Chunks embedded per second
   - Entities extracted per second
   - Error rates

2. **Query Metrics**
   - Query latency
   - Query throughput
   - Cache hit rates
   - Error rates

3. **Resource Metrics**
   - GPU utilization
   - CPU utilization
   - RAM usage
   - Disk I/O

4. **Investigation Metrics**
   - Investigation duration
   - Steps completed
   - Specialist agent performance
   - Report generation time

### Alerting

- Alert on high error rates
- Alert on performance degradation
- Alert on resource exhaustion
- Alert on service failures

---

## Related Topics

- [[ORACLE Architecture]] — System architecture
- [[ORACLE Risk Assessment]] — Performance risks
- [[ORACLE Implementation Roadmap]] — Implementation phases

---

**Tags:** #oracle #performance #benchmarks #optimization

**Links:**
- [[ORACLE Project Overview]]
- [[ORACLE Architecture]]
- [[ORACLE Risk Assessment]]
