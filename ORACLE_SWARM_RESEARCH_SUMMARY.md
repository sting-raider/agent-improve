# ORACLE Swarm: Comprehensive Research Summary

## Executive Summary

This document provides a comprehensive summary of the research, design decisions, and architecture for ORACLE Swarm — a distributed, heterogeneous AI agent system capable of running long-term investigations across multiple machines. The system is designed to handle corpora up to 300GB, run investigations for months, and scale horizontally across multiple GPUs.

---

## Research Overview

### Research Areas Investigated

1. **Swarm Intelligence Patterns** — How AI agents can collaborate effectively
2. **Distributed Agent Orchestration** — Frameworks for coordinating multiple agents
3. **Multi-Model Routing** — Strategies for routing requests across different LLMs
4. **Horizontal Scaling Architectures** — How to scale AI systems across machines
5. **Agent Communication Protocols** — How agents communicate with each other
6. **Job Dispatch Systems** — How to distribute work across workers
7. **Rate Limit Management** — How to handle API rate limits across multiple keys
8. **Fault Tolerance Patterns** — How to make systems resilient to failures
9. **Consensus Mechanisms** — How agents reach agreement on findings
10. **Load Balancing Strategies** — How to distribute load across workers

### Key Findings

#### 1. LiteLLM Gateway — VALIDATED ✅

**What it provides:**
- Load balancing across multiple API keys
- Automatic failover between models
- Rate limit handling with exponential backoff
- Content policy fallbacks
- Context window fallbacks
- Health check routing

**Limitations:**
- Adds network latency (10-50ms overhead)
- Requires separate deployment
- Key pooling is manual
- No built-in cost optimization

**Verdict:** ✅ **USE LiteLLM** — Provides exact features needed without building from scratch.

#### 2. PostgreSQL vs SQLite — CONDITIONAL ⚠️

**SQLite Limitations:**
- Single writer at a time (even with WAL mode)
- Serialized writes
- fsync bottleneck for durability

**PostgreSQL Advantages:**
- Row-level locking
- True concurrency
- Production recommended by LangGraph
- Point-in-time recovery

**However — Critical Nuance:**
For ORACLE's use case, SQLite may be sufficient because:
- Each investigation has its own thread_id
- Checkpoints are written at super-step boundaries (not continuously)
- WAL mode handles thousands of reads per second with a single writer

**Verdict:** ⚠️ **CONDITIONAL** — Start with SQLite. Migrate to PostgreSQL if you encounter locking issues.

#### 3. NATS vs Redis — VALIDATED ✅

**NATS JetStream Advantages:**
- Exactly-once delivery semantics
- Message replay by time/count/sequence
- Built-in clustering
- Horizontal scalability
- Durable subscriptions

**Redis Streams Advantages:**
- At-least-once delivery
- Sub-millisecond latency
- Lower resource usage
- Simpler deployment

**Verdict:** ✅ **USE NATS JetStream** — Needed for guaranteed delivery and message replay in distributed systems.

---

## Architecture Decisions

### Decision 1: Multi-Model Orchestration

**Decision:** Use LiteLLM Gateway for multi-model routing

**Rationale:**
- Provides load balancing, failover, and rate limit handling out of the box
- Supports multiple providers (Anthropic, OpenAI, Google, Ollama)
- Automatic fallback between models
- No need to build custom routing logic

**Trade-offs:**
- Adds network latency
- Requires separate deployment
- Key pooling is manual

**Mitigation:**
- Deploy LiteLLM on same machine as Hive Mind to minimize latency
- Use connection pooling to reduce overhead
- Implement key pooling wrapper

### Decision 2: Horizontal Scaling

**Decision:** Use NATS JetStream + Worker Pool for horizontal scaling

**Rationale:**
- NATS provides guaranteed delivery and message replay
- Worker pool allows adding/removing workers dynamically
- Stateless workers simplify deployment and scaling
- Load balancing is automatic via queue groups

**Trade-offs:**
- Network overhead for distributed workers
- More complex deployment
- Requires coordination of multiple machines

**Mitigation:**
- Use message batching to reduce overhead
- Provide Docker images for easy deployment
- Implement worker auto-discovery

### Decision 3: Persistence Layer

**Decision:** Use PostgreSQL for checkpoints, SQLite for smaller deployments

**Rationale:**
- PostgreSQL provides true concurrency for multiple investigations
- SQLite is simpler for single-machine deployments
- LangGraph recommends PostgreSQL for production

**Trade-offs:**
- PostgreSQL requires more infrastructure
- SQLite has write serialization

**Mitigation:**
- Start with SQLite, migrate to PostgreSQL if needed
- Use WAL mode for SQLite to improve concurrency
- Monitor for locking issues

### Decision 4: Communication Protocol

**Decision:** Build custom protocol on top of NATS JetStream

**Rationale:**
- Standardized message formats across all agents
- Support for multiple communication patterns (request-response, pub-sub, queue)
- Built-in error handling and security
- Comprehensive monitoring

**Trade-offs:**
- Custom protocol requires maintenance
- More complex than using existing protocols

**Mitigation:**
- Document protocol thoroughly
- Provide libraries for common languages
- Implement comprehensive testing

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORACLE Swarm System                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Hive Mind (Orchestrator)                     │  │
│  │  - LangGraph + PostgreSQL                                 │  │
│  │  - Decomposes investigations into tasks                   │  │
│  │  - Manages state and progress                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              LiteLLM Gateway                              │  │
│  │  - Multi-model routing                                    │  │
│  │  - Load balancing                                         │  │
│  │  - Automatic failover                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              NATS JetStream (Message Broker)                │  │
│  │  - Task distribution                                      │  │
│  │  - Result aggregation                                     │  │
│  │  - Guaranteed delivery                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│         ┌─────────────────┼─────────────────┐                    │
│         │                 │                 │                    │
│         ▼                 ▼                 ▼                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Worker 1    │  │  Worker 2    │  │  Worker 3    │     │
│  │  (Your GPU)  │  │  (Friend 1)  │  │  (Friend 2)  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                 │                 │                    │
│         └─────────────────┼─────────────────┘                    │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Shared Storage (Databases)                     │  │
│  │  - Neo4j (knowledge graph)                                 │  │
│  │  - Qdrant (vector embeddings)                              │  │
│  │  - PostgreSQL (checkpoints)                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Technology |
|-----------|----------------|------------|
| **Hive Mind** | Orchestrates investigations, manages state | LangGraph + PostgreSQL |
| **LiteLLM Gateway** | Routes requests to optimal models | LiteLLM |
| **NATS JetStream** | Distributes tasks, aggregates results | NATS |
| **Workers** | Process tasks (parsing, extraction, embedding) | Python + Docker |
| **Neo4j** | Stores knowledge graph | Neo4j |
| **Qdrant** | Stores vector embeddings | Qdrant |
| **PostgreSQL** | Stores checkpoints and investigation state | PostgreSQL |

---

## Scaling Strategy

### What Scales Horizontally

| Component | Scaling Benefit | Scaling Mechanism |
|-----------|-----------------|-------------------|
| **Document Parsing** | ⭐⭐⭐⭐⭐ HUGE | NATS task queue + worker pool |
| **Entity Extraction** | ⭐⭐⭐⭐⭐ HUGE | Distribute chunks across workers |
| **Embedding** | ⭐⭐⭐⭐⭐ HUGE | Each worker runs local infinity-emb |
| **Graph Queries** | ⭐⭐⭐ MEDIUM | Read-heavy, can distribute queries |
| **Reasoning** | ⭐⭐ SMALL | API-bound, but key pooling helps |
| **Report Generation** | ⭐⭐ SMALL | Single-threaded by nature |

### Scaling Approaches

#### Vertical Scaling (Single Machine)

**When to use:**
- Small corpus (<50GB)
- Few concurrent investigations (<5)
- Limited budget
- Simple deployment

**Configuration:**
- Multiple workers on same machine
- Share GPU between workers
- Use local models for extraction

**Pros:**
- Simple deployment
- No network overhead
- Lower cost

**Cons:**
- Limited by single machine resources
- Single point of failure
- Harder to scale beyond hardware limits

#### Horizontal Scaling (Multiple Machines)

**When to use:**
- Large corpus (>50GB)
- Many concurrent investigations (>5)
- Need for fault tolerance
- Budget for multiple machines

**Configuration:**
- One worker per machine
- Distribute tasks via NATS
- Use friends' GPUs

**Pros:**
- Scales indefinitely
- Fault tolerance
- Better resource utilization

**Cons:**
- Network overhead
- More complex deployment
- Higher cost

#### Hybrid Scaling

**When to use:**
- Mixed workload (some CPU-bound, some GPU-bound)
- Budget constraints
- Need for both speed and capacity

**Configuration:**
- Local machine for GPU-bound tasks
- Remote machines for CPU-bound tasks
- Optimize resource utilization

**Pros:**
- Optimal resource utilization
- Cost-effective
- Flexible

**Cons:**
- More complex configuration
- Requires careful task routing

---

## Performance Characteristics

### Bottleneck Analysis

**Ingestion Pipeline:**
```
File Discovery → Parsing → Chunking → Embedding → Storage
     1ms          100ms    10ms      50ms       5ms
```

- **Bottleneck:** Parsing (Docling) and Embedding (LLM call)
- **Scaling:** Parallelize across workers

**Investigation Pipeline:**
```
Query → Graph Traversal → Reasoning → Consensus → Report
  10ms       50ms          2000ms      500ms      100ms
```

- **Bottleneck:** Reasoning (LLM call)
- **Scaling:** Limited by API rate limits, not compute

### Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| **Document Parsing** | <100ms per page | Docling is fast |
| **Entity Extraction** | <500ms per chunk | Local Ollama |
| **Embedding** | <50ms per chunk | infinity-emb with batching |
| **Graph Query** | <100ms per query | Neo4j with indexes |
| **Reasoning** | <2s per query | API model |
| **Task Distribution** | <10ms | NATS is fast |
| **Result Aggregation** | <10ms | NATS is fast |

### Optimization Strategies

1. **Batch Embedding** — Process chunks in batches of 32-64
2. **Async I/O** — Never block on network calls
3. **Connection Pooling** — Reuse database connections
4. **Caching** — Cache frequently accessed entities
5. **Lazy Loading** — Only load data when needed

---

## Cost Analysis

### Hardware Costs

| Component | Cost (Monthly) | Notes |
|-----------|----------------|-------|
| **Your Machine** | $0 | Already owned |
| **Friend 1 Machine** | $0 | Already owned |
| **Friend 2 Machine** | $0 | Already owned |
| **NATS Server** | $0 | Can run on your machine |
| **PostgreSQL** | $0 | Can run on your machine |
| **Neo4j** | $0 | Community edition |
| **Qdrant** | $0 | Open source |
| **LiteLLM Gateway** | $0 | Open source |
| **Total** | $0 | All open source |

### API Costs

| Provider | Model | Cost per 1M tokens | Notes |
|----------|-------|-------------------|-------|
| **Anthropic** | Claude 3.5 Sonnet | $15 | Primary reasoning |
| **OpenAI** | GPT-4o | $5 | Secondary reasoning |
| **Google** | Gemini 2.0 Flash | $0.075 | Fast reasoning |
| **Ollama** | Qwen3:8b | $0 | Local, free |
| **Ollama** | Qwen3:30B | $0 | Local, free |

**Estimated Monthly Costs:**
- Small investigation (10GB corpus): $50-100
- Medium investigation (50GB corpus): $200-500
- Large investigation (300GB corpus): $1000-2000

### Cost Optimization Strategies

1. **Use local models** for extraction and embedding
2. **Batch requests** to reduce API calls
3. **Cache results** to avoid duplicate processing
4. **Use cheaper models** for less critical tasks
5. **Implement budget caps** per investigation

---

## Fault Tolerance

### Failure Scenarios

| Scenario | Detection | Mitigation | Recovery |
|----------|-----------|------------|----------|
| **Worker crashes** | NATS heartbeat timeout | Task re-queued after max_deliver | Worker restarts, picks up pending tasks |
| **API rate limit** | LiteLLM 429 error | Automatic fallback to backup model | Cooldown period, then retry |
| **LiteLLM down** | Connection timeout | Direct API call (bypass gateway) | LiteLLM restart |
| **NATS down** | Connection error | Local task queue (in-memory) | NATS restart, drain local queue |
| **PostgreSQL down** | Connection error | SQLite fallback (read-only) | PostgreSQL restart |
| **Neo4j down** | Connection error | Cache writes locally | Neo4j restart, replay cache |
| **Qdrant down** | Connection error | Skip embedding, continue | Qdrant restart, re-embed |

### Resilience Patterns

1. **Retry with Exponential Backoff** — Retry failed requests with increasing delays
2. **Circuit Breaker** — Stop calling failing services temporarily
3. **Bulkhead Pattern** — Isolate failures to prevent cascading
4. **Timeout Pattern** — Fail fast on slow operations
5. **Fallback Pattern** — Use backup services when primary fails

---

## Security Considerations

### API Key Management

**Never commit API keys to git.** Use environment variables:

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=...
NEO4J_PASSWORD=...
```

**LiteLLM Virtual Keys:**

Generate virtual keys for each investigation to track costs:

```bash
curl -X POST 'http://localhost:4000/key/generate' \
  -H 'Authorization: Bearer sk-admin-key' \
  -H 'Content-Type: application/json' \
  -d '{
    "budget": {
      "max_budget": 100.0,
      "duration": "30d"
    },
    "metadata": {
      "investigation_id": "inv-abc123"
    }
  }'
```

### Worker Authentication

Use JWT-based authentication for workers:

```python
async def authenticate_worker(token: str) -> bool:
    """Verify worker token."""
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=["HS256"])
        return payload.get("worker_id") in settings.authorized_workers
    except jwt.InvalidTokenError:
        return False
```

### Message Security

1. **Sign messages** to verify authenticity
2. **Encrypt sensitive** messages
3. **Use TLS** for NATS connections
4. **Validate messages** before processing
5. **Rotate keys** regularly

---

## Monitoring & Observability

### Metrics to Track

**System Metrics:**
- Active investigations
- Active workers
- Task queue depth (per priority)
- Task completion rate
- Task failure rate

**Worker Metrics:**
- Tasks processed
- Tasks failed
- Average task duration
- CPU usage
- GPU usage
- Memory usage
- Network I/O

**Investigation Metrics:**
- Active investigations
- Total investigations
- Tasks per investigation
- Completion rate
- Average investigation duration

### Monitoring Stack

**Prometheus:**
- Collect metrics from all components
- Store time-series data
- Provide query interface

**Grafana:**
- Visualize metrics
- Create dashboards
- Set up alerts

**Logging:**
- Structured logging with context
- Log levels (DEBUG, INFO, WARNING, ERROR)
- Log aggregation

---

## Deployment Strategy

### Phase 1: Infrastructure Setup

**Components:**
- PostgreSQL
- NATS JetStream
- LiteLLM Gateway
- Neo4j
- Qdrant

**Steps:**
1. Install Docker
2. Run docker-compose up
3. Verify all services are healthy
4. Configure environment variables

### Phase 2: Hive Mind Deployment

**Components:**
- LangGraph with PostgreSQL checkpointing
- Investigation orchestrator
- Task dispatcher

**Steps:**
1. Install Python dependencies
2. Configure LiteLLM
3. Start Hive Mind service
4. Test checkpointing

### Phase 3: Worker Deployment

**Components:**
- Worker Docker image
- Worker configuration
- Worker registration

**Steps:**
1. Build worker Docker image
2. Distribute to friends
3. Configure workers
4. Start workers

### Phase 4: Monitoring Setup

**Components:**
- Prometheus
- Grafana
- Alerting

**Steps:**
1. Install Prometheus
2. Install Grafana
3. Configure metrics collection
4. Set up dashboards and alerts

---

## Testing Strategy

### Unit Tests

Test individual components in isolation:

```python
async def test_model_router():
    """Test model router."""
    router = ModelRouter(MODEL_REGISTRY, TASK_REQUIREMENTS)
    
    model = router.select_model(
        TaskType.COMPLEX_REASONING,
        input_tokens=1000,
        estimated_output_tokens=2000
    )
    
    assert model in ["claude-3-5-sonnet", "gpt-4o", "gemini-2.0-flash-thinking"]
```

### Integration Tests

Test component interactions:

```python
async def test_task_execution_flow():
    """Test task execution flow."""
    # Create task
    task = TaskCreateMessage(...)
    
    # Publish to NATS
    await js.publish("oracle.tasks.create", task.to_bytes())
    
    # Wait for result
    result = await wait_for_result(task.task_id)
    
    # Assert
    assert result.status == "success"
```

### End-to-End Tests

Test complete workflows:

```python
async def test_investigation_workflow():
    """Test complete investigation workflow."""
    # Start investigation
    investigation_id = await start_investigation("Test question")
    
    # Wait for completion
    await wait_for_completion(investigation_id)
    
    # Get report
    report = await get_report(investigation_id)
    
    # Assert
    assert report.status == "complete"
    assert len(report.findings) > 0
```

---

## Documentation

### Created Documents

1. **ORACLE_SWARM_ARCHITECTURE.md** — Complete system architecture
2. **ORACLE_MULTI_MODEL_ORCHESTRATION.md** — Multi-model routing design
3. **ORACLE_HORIZONTAL_SCALING.md** — Horizontal scaling strategy
4. **ORACLE_AGENT_COMMUNICATION_PROTOCOL.md** — Communication protocol specification
5. **ORACLE_SWARM_RESEARCH_SUMMARY.md** — This document

### Document Structure

```
oracle/
├── docs/
│   ├── ORACLE_SWARM_ARCHITECTURE.md
│   ├── ORACLE_MULTI_MODEL_ORCHESTRATION.md
│   ├── ORACLE_HORIZONTAL_SCALING.md
│   ├── ORACLE_AGENT_COMMUNICATION_PROTOCOL.md
│   └── ORACLE_SWARM_RESEARCH_SUMMARY.md
├── oracle/
│   ├── ingestion/
│   ├── graph/
│   ├── agents/
│   ├── mcp_servers/
│   ├── tui/
│   ├── visualization/
│   └── reporting/
└── scripts/
    ├── setup.sh
    ├── ingest.py
    ├── investigate.py
    └── oracle
```

---

## Next Steps

### Implementation Phases

**Phase 1: Infrastructure (Week 1)**
- Set up PostgreSQL
- Set up NATS JetStream
- Set up LiteLLM Gateway
- Set up Neo4j and Qdrant
- Verify all services are healthy

**Phase 2: Hive Mind (Week 2)**
- Implement LangGraph orchestrator
- Implement task dispatcher
- Implement result aggregator
- Test checkpointing

**Phase 3: Workers (Week 3)**
- Build worker Docker image
- Implement task handlers
- Implement result publishers
- Test worker registration

**Phase 4: Integration (Week 4)**
- Integrate Hive Mind with workers
- Test task distribution
- Test result aggregation
- Test fault tolerance

**Phase 5: Optimization (Week 5)**
- Optimize performance
- Implement caching
- Implement batching
- Tune parameters

**Phase 6: Testing (Week 6)**
- Write unit tests
- Write integration tests
- Write end-to-end tests
- Fix bugs

**Phase 7: Deployment (Week 7)**
- Deploy to production
- Set up monitoring
- Set up alerting
- Document procedures

---

## Conclusion

ORACLE Swarm is a comprehensive, distributed AI agent system designed to:

✅ **Handle large corpora** — Up to 300GB of documents
✅ **Run long investigations** — Months of operation without state loss
✅ **Scale horizontally** — Across multiple machines and GPUs
✅ **Use multiple models** — Optimal model for each task
✅ **Maintain state** — Survive shutdowns and failures
✅ **Provide insights** — Generate comprehensive reports with citations

The system is built on proven technologies (NATS, LiteLLM, LangGraph, PostgreSQL, Neo4j, Qdrant) and follows best practices for distributed systems, fault tolerance, and security.

### Key Takeaways

1. **Use existing tools** — Don't build from scratch when proven solutions exist
2. **Design for failure** — Assume things will fail and build resilience
3. **Monitor everything** — You can't improve what you don't measure
4. **Document thoroughly** — Future you will thank present you
5. **Test continuously** — Catch issues early, not in production

### Success Criteria

The system will be considered successful when it can:

- Ingest 300GB of documents in <7 days
- Run investigations for months without state loss
- Scale horizontally across 10+ workers
- Handle worker failures without data loss
- Generate comprehensive reports with full citations
- Provide real-time progress tracking

---

**Document Version:** 1.0  
**Last Updated:** 2025-01-02  
**Status:** COMPLETE ✅

---

## Appendix

### A. Technology Stack Summary

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Orchestrator** | LangGraph | Latest | State machine for investigations |
| **Persistence** | PostgreSQL | 15+ | Checkpoints and investigation state |
| **Message Broker** | NATS JetStream | Latest | Task distribution and results |
| **Model Router** | LiteLLM | Latest | Multi-model routing and failover |
| **Knowledge Graph** | Neo4j | 5.x | Entity and relationship storage |
| **Vector Store** | Qdrant | Latest | Semantic search over chunks |
| **Document Parser** | Docling | Latest | Multi-modal document parsing |
| **Embedding** | infinity-emb | Latest | High-throughput embedding server |
| **Local LLM** | Ollama | Latest | Local model inference |
| **TUI** | Textual | Latest | Terminal user interface |
| **Visualization** | 3d-force-graph | Latest | WebGL graph visualization |

### B. Configuration Files

**Environment Variables (.env):**
```bash
# Database
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=changeme
QDRANT_HOST=localhost
QDRANT_PORT=6333
POSTGRES_URI=postgresql://user:pass@localhost/oracle

# NATS
NATS_URL=nats://localhost:4222

# LiteLLM
LITELLM_URL=http://localhost:4000
LITELLM_MASTER_KEY=sk-litellm-

# API Keys
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GOOGLE_API_KEY=...

# Ollama
OLLAMA_HOST=http://localhost:11434

# Infinity Emb
INFINITY_EMB_HOST=localhost
INFINITY_EMB_PORT=7997
```

**LiteLLM Configuration (litellm_config.yaml):**
```yaml
model_list:
  - model_name: oracle-reasoning-primary
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY}
      rpm: 500

router_settings:
  fallbacks: [
    {"oracle-reasoning-primary": ["oracle-reasoning-secondary"]}
  ]

litellm_settings:
  num_retries: 3
  request_timeout: 120
  allowed_fails: 3
  cooldown_time: 60
```

### C. Quick Start Commands

```bash
# Start infrastructure
docker-compose up -d

# Start Hive Mind
python -m oracle.agents.orchestrator

# Start worker
docker-compose up -d oracle-worker

# Start TUI
oracle tui

# Ingest corpus
oracle ingest /path/to/corpus

# Start investigation
oracle investigate "Your research question"

# Check status
oracle status

# Generate report
oracle report <investigation_id>

# Visualize graph
oracle visualize
```

### D. Troubleshooting Guide

**Issue: Workers not connecting**

**Solution:**
1. Check NATS is running: `docker ps | grep nats`
2. Check NATS URL in worker config
3. Check network connectivity
4. Check worker logs

**Issue: Tasks not being processed**

**Solution:**
1. Check task queue depth
2. Check worker logs
3. Check worker capabilities
4. Check worker is not rate limited

**Issue: Slow processing**

**Solution:**
1. Check worker resource usage
2. Increase max_concurrent_tasks
3. Add more workers
4. Optimize task batching

**Issue: High API costs**

**Solution:**
1. Use local models for extraction
2. Implement budget caps
3. Cache results
4. Use cheaper models

---

**END OF DOCUMENT**
