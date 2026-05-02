# Decision D016: Scope Lock — Single Machine v1, Swarm v2

**Status:** APPROVED  
**Date:** 2025-05-02  
**Decision:** Implement v1 as single-machine deployment, v2 as swarm upgrade

---

## Context

The ORACLE architecture documentation simultaneously describes a single-laptop tool and a full distributed swarm system. These have different starting architectures and attempting to build both simultaneously would result in building neither cleanly.

## Decision

**ORACLE v1 will target single-machine deployment only.**  
**ORACLE v2 will upgrade to swarm when v1 is proven.**

## v1 Architecture (Single Machine)

### Components
- **Checkpointing:** SQLite via `AsyncSqliteSaver` (LangGraph)
- **Message Queue:** None (direct function calls)
- **Model Routing:** Direct API calls (no LiteLLM)
- **Workers:** None (single process)
- **Storage:** Local Neo4j + Qdrant

### Constraints
- Single investigation at a time (SQLite write lock)
- No horizontal scaling
- No distributed task distribution
- All services run on one machine

### Capabilities
- Full ingestion pipeline (Docling → GLiNER2 → infinity-emb)
- Knowledge graph construction (Neo4j + Qdrant)
- Multi-agent investigation (LangGraph with specialist agents)
- Provenance graph tracking
- TUI and visualization
- Report generation

## v2 Architecture (Swarm Upgrade)

### Components Added
- **Checkpointing:** PostgreSQL via `AsyncPostgresSaver` (LangGraph)
- **Message Queue:** NATS JetStream
- **Model Routing:** LiteLLM Gateway
- **Workers:** Docker-based worker pool
- **Storage:** Shared Neo4j + Qdrant (same as v1)

### Capabilities Added
- Multiple concurrent investigations
- Horizontal scaling across multiple machines
- Distributed task distribution
- Multi-provider model routing with failover
- Worker fault tolerance and auto-recovery

## Migration Path v1 → v2

### What Changes
1. **Checkpointing:** SQLite → PostgreSQL
   - LangGraph checkpointer interface is identical
   - No code changes to agent logic
   - Only initialization changes: `AsyncSqliteSaver` → `AsyncPostgresSaver`

2. **Task Distribution:** Direct calls → NATS
   - Agent nodes publish tasks to NATS subjects
   - Workers subscribe and process tasks
   - Results published back to response subjects

3. **Model Routing:** Direct API → LiteLLM
   - All LLM calls routed through LiteLLM
   - Automatic failover between providers
   - Rate limit management across multiple API keys

### What Stays the Same
- **Agent logic:** LangGraph state machine unchanged
- **Specialist agents:** Same code, just distributed
- **Knowledge graph:** Neo4j + Qdrant unchanged
- **Ingestion pipeline:** Same code, just distributed
- **TUI:** Same code, connects to swarm orchestrator
- **Visualization:** Same code, connects to shared Neo4j

### Validation: LangGraph Checkpointer Swappability

**Research Finding:** LangGraph checkpointing is designed to be swappable.

**Evidence:**
- All implementations inherit from `BaseCheckpointSaver`
- All implement the same core methods:
  - `get_tuple()` / `aget_tuple()` — Retrieve a checkpoint
  - `list()` / `alist()` — List checkpoints with filtering
  - `put()` / `aput()` — Save a checkpoint
  - `put_writes()` / `aput_writes()` — Save intermediate writes
- Both `AsyncSqliteSaver` and `AsyncPostgresSaver` provide async support
- The agent code only interacts with the checkpointer interface, not the implementation

**Conclusion:** The checkpointer can be swapped without any code changes to the agent logic. Only the initialization changes.

## Implementation Strategy

### v1 Implementation Order
1. Set up local services (Neo4j, Qdrant, SQLite)
2. Implement ingestion pipeline (single-process)
3. Implement knowledge graph layer
4. Implement LangGraph agent with SQLite checkpointing
5. Implement TUI and visualization
6. Test end-to-end on sample corpus

### v2 Upgrade Order (when v1 is proven)
1. Set up PostgreSQL
2. Migrate SQLite checkpoints to PostgreSQL
3. Set up NATS JetStream
4. Implement worker Docker images
5. Implement LiteLLM Gateway
6. Refactor agent nodes to publish to NATS
7. Implement worker pool manager
8. Test distributed execution

## Rationale

### Why v1 First?
1. **Complexity management:** Building swarm-first adds massive complexity before proving the core system works
2. **Faster iteration:** Single-machine allows rapid testing and debugging
3. **Lower risk:** Fewer moving parts means fewer failure modes
4. **Clear validation:** Can prove the investigation logic works before scaling

### Why v2 Later?
1. **Proven foundation:** v1 validates the agent logic, knowledge graph, and investigation patterns
2. **Clear scaling needs:** v1 will reveal where horizontal scaling is actually needed
3. **Architecture validation:** v1 proves the LangGraph + Neo4j + Qdrant stack works
4. **Resource planning:** v1 will reveal actual resource requirements for swarm

### Why This Migration Path Works
1. **LangGraph design:** Checkpointer interface is explicitly designed for swappable backends
2. **No agent logic changes:** The state machine and specialist agents don't care about the storage backend
3. **Incremental upgrade:** Can upgrade components one at a time
4. **Rollback possible:** Can revert to v1 if v2 has issues

## Risks and Mitigations

### Risk: SQLite write lock limits concurrent investigations
**Mitigation:** v1 only supports one investigation at a time. This is documented as a v1 limitation.

### Risk: PostgreSQL migration complexity
**Mitigation:** LangGraph provides migration utilities. Test migration on sample data before v2 upgrade.

### Risk: NATS learning curve
**Mitigation:** NATS has excellent documentation. Start with simple pub/sub patterns before complex workflows.

### Risk: Worker Docker image complexity
**Mitigation:** Build on v1's single-process code. The worker is just the same code in a container with NATS client.

## Success Criteria

### v1 Success
- Can ingest 50GB corpus in <48 hours on single machine
- Can run investigation for days without state loss
- Can resume investigation after laptop shutdown
- Can generate comprehensive report with full citations
- Can visualize knowledge graph in browser

### v2 Success
- Can ingest 300GB corpus in <7 days across 3 machines
- Can run 5 concurrent investigations without interference
- Can survive worker failures without data loss
- Can scale horizontally to 10+ workers
- Can route to optimal models with automatic failover

## References

- LangGraph Checkpoint Implementations: https://deepwiki.com/langchain-ai/langgraph/4.2-checkpoint-implementations
- LangGraph Persistence Docs: https://docs.langchain.com/oss/python/langgraph/persistence
- NATS JetStream Documentation: https://docs.nats.io/nats-concepts/jetstream
- LiteLLM Documentation: https://docs.litellm.ai/

---

**Approved by:** Hermes Agent  
**Next Action:** Proceed with v1 implementation planning
