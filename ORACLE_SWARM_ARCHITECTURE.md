# ORACLE Swarm: Distributed Intelligence Architecture

## Executive Summary

This document presents the validated architecture for ORACLE Swarm — a distributed, heterogeneous agent system capable of running investigations for months at a time. The architecture addresses three critical challenges:

1. **Multi-model orchestration** — routing requests across different LLM providers and models
2. **Horizontal scaling** — distributing work across multiple machines (your friends' GPUs)
3. **Long-running persistence** — maintaining state across months of operation

---

## Research Findings & Validations

### 1. LiteLLM Gateway — VALIDATED ✅

**What LiteLLM Actually Does:**

Based on official documentation (docs.litellm.ai), LiteLLM provides:

- **Load Balancing:** Distributes requests across multiple model instances using strategies like round-robin, least-busy, and weighted routing
- **Automatic Failover:** If a call fails after `num_retries`, it automatically falls back to another model group
- **Rate Limit Handling:** Intelligent retry with exponential backoff for 429 errors
- **Content Policy Fallbacks:** Maps content policy violation errors across providers
- **Context Window Fallbacks:** Maps context window error messages across providers
- **Health Check Routing:** Proactively removes unhealthy deployments from the routing pool
- **Cooldown Management:** Models that fail too frequently are temporarily removed from rotation

**Configuration Example:**

```yaml
model_list:
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: ${OPENAI_API_KEY}
      rpm: 500  # Rate limit for this deployment
  
  - model_name: claude-3-5-sonnet
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY}
      rpm: 500

litellm_settings:
  num_retries: 3
  request_timeout: 10
  fallbacks: [{"gpt-4o": ["claude-3-5-sonnet"]}]
  allowed_fails: 3
  cooldown_time: 30
```

**Limitations:**

- LiteLLM is a proxy server — it adds network latency (typically 10-50ms overhead)
- Requires separate deployment and management
- Key pooling is manual — you must configure keys in the config file
- No built-in cost optimization (you must implement budget routing yourself)

**Verdict:** ✅ **USE LiteLLM** — It provides the exact features needed (load balancing, failover, rate limit handling) without building from scratch.

---

### 2. PostgreSQL vs SQLite for LangGraph — PARTIALLY VALIDATED ⚠️

**What the Research Shows:**

**SQLite Limitations:**

From SQLite documentation and expert analysis:

- **Single Writer:** SQLite, even with WAL mode, supports "unlimited number of readers and a single writer at any given moment"
- **Serialized Writes:** "All write operations are effectively serialized"
- **fsync Bottleneck:** "SQLite gets stuck in the fsync operation required to ensure durability"
- **WAL Mode Improvements:** WAL mode allows readers to proceed while a writer commits, but still only one writer at a time

**PostgreSQL Advantages:**

From LangGraph documentation:

- **Row-Level Locking:** PostgreSQL uses row-level locking instead of database-level locking
- **True Concurrency:** Multiple agents can read and write their checkpoints simultaneously
- **Production Recommended:** LangGraph docs explicitly state PostgreSQL is "Ideal for using in production"
- **Point-in-Time Recovery:** PostgreSQL allows rolling back to exact points in time

**However — Critical Nuance:**

For ORACLE's use case, the concurrency issue may be **overstated**:

1. **LangGraph Checkpointing Pattern:** Each investigation has its own `thread_id`. Different investigations write to different rows/tables.
2. **Write Frequency:** Checkpoints are written at super-step boundaries, not continuously. For a typical investigation, this might be once every few seconds to minutes.
3. **SQLite WAL Performance:** With WAL mode, SQLite can handle thousands of reads per second with a single writer.

**When SQLite Actually Fails:**

- **Multiple writers to the SAME thread_id** — This doesn't happen in LangGraph (each thread has one writer)
- **Very high checkpoint frequency** — If you're checkpointing every 100ms, SQLite will bottleneck
- **Long-running write transactions** — If a checkpoint write takes >1 second, other writes block

**Verdict:** ⚠️ **CONDITIONAL** — Use PostgreSQL if:
- You expect >10 concurrent investigations with frequent checkpointing
- You need point-in-time recovery
- You want to scale horizontally (multiple machines accessing the same database)

Use SQLite if:
- You're running a single machine with <5 concurrent investigations
- Checkpoint frequency is reasonable (every few seconds)
- You want simplicity and zero infrastructure overhead

**Recommendation for ORACLE:** Start with SQLite. If you encounter locking issues (monitor for `database is locked` errors), migrate to PostgreSQL.

---

### 3. NATS vs Redis — VALIDATED ✅

**What NATS JetStream Actually Provides:**

From official NATS documentation:

- **Delivery Guarantees:** "At most once, at least once, and exactly once is available in JetStream"
- **Persistence:** "Supports memory and file persistence. Messages can be replayed by time, count, or sequence number"
- **Durable Subscriptions:** Messages survive consumer disconnects
- **Message Replay:** "Messages can be replayed by time, count, or sequence number"
- **Clustering:** "Full mesh clustering with self-healing features"
- **Horizontal Scalability:** "JetStream supports horizontal scalability with built-in mirroring"

**Redis Streams Capabilities:**

- **Durable Delivery:** Supports consumer groups with ACK semantics
- **Message Replay:** Can replay from specific offsets
- **Persistence:** AOF and RDB persistence options
- **Performance:** Extremely fast (sub-millisecond latency)

**Key Differences:**

| Feature | NATS JetStream | Redis Streams |
|---------|----------------|---------------|
| **Guaranteed Delivery** | ✅ Exactly-once | ✅ At-least-once |
| **Message Replay** | ✅ By time/count/sequence | ✅ By offset |
| **Persistence** | ✅ File-based | ✅ AOF/RDB |
| **Clustering** | ✅ Native mesh clustering | ✅ Redis Cluster |
| **Consumer Groups** | ✅ Durable subscriptions | ✅ Consumer groups |
| **Complexity** | Higher | Lower |
| **Resource Usage** | Higher | Lower |

**When NATS is Better:**

- You need **exactly-once** semantics
- You need **message replay by time** (e.g., "replay everything from 3 days ago")
- You're building a **distributed system** with multiple nodes
- You need **built-in clustering** without external dependencies

**When Redis is Better:**

- You already use Redis for other purposes
- You want **simplicity** and **low resource usage**
- You need **sub-millisecond latency**
- You're okay with **at-least-once** semantics

**Verdict:** ✅ **USE NATS JetStream** for ORACLE because:
- You need guaranteed delivery for distributed workers
- Message replay is critical for "catch-up" when workers reconnect
- Exactly-once semantics prevent duplicate processing
- Built-in clustering simplifies deployment

---

## ORACLE Swarm Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         ORACLE SWARM                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │   Hive Mind  │──────│   LiteLLM    │──────│   NATS       │  │
│  │  (LangGraph) │      │   Gateway    │      │   JetStream  │  │
│  │  + Postgres  │      │              │      │              │  │
│  └──────────────┘      └──────────────┘      └──────┬───────┘  │
│         │                     │                     │           │
│         │                     │                     │           │
│         ▼                     ▼                     ▼           │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │  Neo4j +     │      │  Qdrant      │      │  Worker Pool │  │
│  │  Qdrant      │      │  (Vector DB) │      │              │  │
│  └──────────────┘      └──────────────┘      └──────┬───────┘  │
│                                                     │           │
│                                                     │           │
│                                                     ▼           │
│                                          ┌──────────────────┐  │
│                                          │  Your Friends'   │  │
│                                          │  GPUs (Docker)   │  │
│                                          └──────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Component Details

#### 1. Hive Mind (Global Orchestrator)

**Technology:** LangGraph + PostgreSQL

**Responsibilities:**
- Decompose complex questions into atomic hypotheses
- Manage investigation state across months
- Coordinate specialist agent squads
- Track progress and completion

**State Schema:**

```python
class InvestigationState(TypedDict):
    investigation_id: str
    investigation_name: str
    original_question: str
    status: str  # initializing, researching, synthesizing, complete
    completion_pct: float
    
    # Hypotheses
    hypotheses: list[Hypothesis]
    
    # Active work
    active_tasks: list[ActiveTask]
    pending_questions: list[str]
    
    # Knowledge accumulated
    key_findings: list[str]
    contradictions: list[str]
    entities_of_interest: list[str]
    named_graphs: list[NamedGraphRef]
    
    # Context management
    journal_path: str
    journal_summary: str
    sources_consulted: list[str]
    
    # Agent communication
    messages: Annotated[list, add_messages]
    
    # Configuration
    config: dict
```

**PostgreSQL Schema:**

```sql
-- Checkpoints table (managed by LangGraph)
CREATE TABLE checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_id TEXT NOT NULL,
    parent_checkpoint_id TEXT,
    checkpoint BLOB NOT NULL,
    metadata BLOB,
    PRIMARY KEY (thread_id, checkpoint_id)
);

-- Investigations table (application-level)
CREATE TABLE investigations (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    question TEXT NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    completion_pct FLOAT DEFAULT 0.0
);

-- Hypotheses table
CREATE TABLE hypotheses (
    id TEXT PRIMARY KEY,
    investigation_id TEXT NOT NULL REFERENCES investigations(id),
    text TEXT NOT NULL,
    confidence FLOAT NOT NULL,
    status TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Named graphs tracking
CREATE TABLE named_graphs (
    id TEXT PRIMARY KEY,
    investigation_id TEXT NOT NULL REFERENCES investigations(id),
    name TEXT NOT NULL,
    graph_type TEXT NOT NULL,
    description TEXT,
    node_count INTEGER,
    relationship_count INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);
```

#### 2. LiteLLM Gateway

**Technology:** LiteLLM Proxy Server

**Configuration:**

```yaml
model_list:
  # High-reasoning models
  - model_name: oracle-reasoning-primary
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_1}
      rpm: 500
    model_info:
      id: claude-primary-1
  
  - model_name: oracle-reasoning-primary
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_2}
      rpm: 500
    model_info:
      id: claude-primary-2
  
  - model_name: oracle-reasoning-secondary
    litellm_params:
      model: openai/gpt-4o
      api_key: ${OPENAI_API_KEY_1}
      rpm: 500
    model_info:
      id: gpt4o-secondary-1
  
  - model_name: oracle-reasoning-secondary
    litellm_params:
      model: openai/gpt-4o
      api_key: ${OPENAI_API_KEY_2}
      rpm: 500
    model_info:
      id: gpt4o-secondary-2
  
  # Fast extraction models
  - model_name: oracle-extraction
    litellm_params:
      model: fastino/gliner2-base-v1
      api_base: http://localhost:11434
      rpm: 1000
  
  # Vision models
  - model_name: oracle-vision
    litellm_params:
      model: gemini/gemini-2.0-flash-exp
      api_key: ${GOOGLE_API_KEY}
      rpm: 500

router_settings:
  enable_pre_call_checks: true
  fallbacks: [
    {"oracle-reasoning-primary": ["oracle-reasoning-secondary"]},
    {"oracle-reasoning-secondary": ["oracle-extraction"]}
  ]
  content_policy_fallbacks: [
    {"oracle-reasoning-primary": ["oracle-reasoning-secondary"]}
  ]
  context_window_fallbacks: [
    {"oracle-reasoning-primary": ["oracle-reasoning-secondary"]}
  ]

litellm_settings:
  num_retries: 3
  request_timeout: 120
  allowed_fails: 3
  cooldown_time: 60
```

**Routing Strategy:**

```python
class ModelRouter:
    def get_model_for_task(self, task_type: str) -> str:
        """Route task to appropriate model."""
        routing = {
            "complex_reasoning": "oracle-reasoning-primary",
            "verification": "oracle-reasoning-secondary",
            "extraction": "oracle-extraction",
            "vision": "oracle-vision",
            "summarization": "oracle-reasoning-secondary",
        }
        return routing.get(task_type, "oracle-reasoning-primary")
    
    async def call_with_fallback(self, task_type: str, messages: list):
        """Call model with automatic fallback."""
        model = self.get_model_for_task(task_type)
        
        try:
            response = await litellm.acompletion(
                model=model,
                messages=messages,
                fallbacks=[self.get_fallback_model(model)]
            )
            return response
        except Exception as e:
            # LiteLLM handles fallbacks automatically
            raise
```

#### 3. NATS JetStream (Communication Backbone)

**Stream Configuration:**

```python
# Task dispatch stream
await js.create_stream(
    name="oracle.tasks",
    subjects=["oracle.tasks.>"],
    retention="workqueue",  # Messages deleted after ACK
    max_age=30 * 24 * 60 * 60 * 1000,  # 30 days
    storage="file",
    replicas=1
)

# Results stream
await js.create_stream(
    name="oracle.results",
    subjects=["oracle.results.>"],
    retention="limits",  # Keep last N messages
    max_msgs=100000,
    storage="file",
    replicas=1
)

# Agent communication stream
await js.create_stream(
    name="oracle.agents",
    subjects=["oracle.agents.>"],
    retention="workqueue",
    max_age=7 * 24 * 60 * 60 * 1000,  # 7 days
    storage="file",
    replicas=1
)
```

**Message Schema:**

```python
@dataclass
class TaskMessage:
    task_id: str
    task_type: str  # "extraction", "reasoning", "verification", etc.
    investigation_id: str
    payload: dict
    priority: int  # 0-10, higher is more important
    created_at: str
    timeout_seconds: int = 3600

@dataclass
class ResultMessage:
    task_id: str
    investigation_id: str
    status: str  # "success", "failure", "partial"
    result: dict
    error: str | None
    completed_at: str

@dataclass
class AgentMessage:
    agent_id: str
    investigation_id: str
    message_type: str  # "finding", "challenge", "request", etc.
    content: str
    metadata: dict
    timestamp: str
```

**Consumer Configuration:**

```python
# Worker consumer
sub = await js.subscribe(
    subject="oracle.tasks.extraction",
    stream="oracle.tasks",
    consumer_name="worker-1",
    deliver_policy="all",  # Deliver all messages
    ack_policy="explicit",  # Require explicit ACK
    max_deliver=3,  # Retry up to 3 times
)

# Process message
async def process_task(msg):
    task = TaskMessage(**json.loads(msg.data))
    
    try:
        result = await execute_task(task)
        
        # ACK success
        await msg.ack()
        
        # Publish result
        await js.publish(
            f"oracle.results.{task.investigation_id}",
            ResultMessage(
                task_id=task.task_id,
                investigation_id=task.investigation_id,
                status="success",
                result=result,
                error=None,
                completed_at=datetime.now().isoformat()
            )
        )
    except Exception as e:
        # NACK (negative ACK) for retry
        await msg.nak()
        
        # Publish failure
        await js.publish(
            f"oracle.results.{task.investigation_id}",
            ResultMessage(
                task_id=task.task_id,
                investigation_id=task.investigation_id,
                status="failure",
                result={},
                error=str(e),
                completed_at=datetime.now().isoformat()
            )
        )
```

#### 4. Worker Pool (Horizontal Scaling)

**Worker Docker Image:**

```dockerfile
FROM python:3.11-slim

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Install Ollama
RUN curl -fsSL https://ollama.com/install.sh | sh

# Pull models
# GLiNER2 is CPU-first, no pull needed
# jina-embeddings-v5 is pulled by infinity-emb

# Copy worker code
COPY oracle/worker/ /app/worker/

# Start worker
CMD ["python", "/app/worker/main.py"]
```

**Worker Configuration:**

```python
# config.yaml
nats:
  url: "nats://your-server:4222"
  jetstream: true

lite_llm:
  url: "http://your-server:4000"

neo4j:
  uri: "bolt://your-server:7687"
  username: "neo4j"
  password: "${NEO4J_PASSWORD}"

qdrant:
  host: "your-server"
  port: 6333

worker:
  id: "worker-1"
  capabilities:
    - "extraction"
    - "embedding"
    - "reasoning"
  max_concurrent_tasks: 4
```

**Worker Main Loop:**

```python
async def worker_loop():
    # Connect to NATS
    nc = await nats.connect(settings.nats.url)
    js = nc.jetstream
    
    # Subscribe to task streams based on capabilities
    for capability in settings.worker.capabilities:
        await js.subscribe(
            subject=f"oracle.tasks.{capability}",
            stream="oracle.tasks",
            consumer_name=settings.worker.id,
            deliver_policy="all",
            ack_policy="explicit",
            max_deliver=3
        )
    
    # Process messages
    async for msg in nc.messages:
        try:
            task = TaskMessage(**json.loads(msg.data))
            
            # Check if we can handle this task
            if task.task_type not in settings.worker.capabilities:
                await msg.nak()
                continue
            
            # Execute task
            result = await execute_task(task)
            
            # ACK and publish result
            await msg.ack()
            await js.publish(
                f"oracle.results.{task.investigation_id}",
                ResultMessage(...)
            )
            
        except Exception as e:
            await msg.nak()
            logger.error(f"Task failed: {e}")
```

---

## Specialist Agent Squads

### Squad Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Investigation Squad                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Historian   │  │ Statistician │  │Source Critic │     │
│  │  (Claude)    │  │  (GPT-4o)    │  │  (Gemini)    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                 │                 │              │
│         └─────────────────┼─────────────────┘              │
│                           │                                │
│                           ▼                                │
│                  ┌──────────────┐                          │
│                  │  Consensus   │                          │
│                  │  Engine      │                          │
│                  └──────────────┘                          │
│                           │                                │
│                           ▼                                │
│                  ┌──────────────┐                          │
│                  │  Report      │                          │
│                  │  Writer      │                          │
│                  └──────────────┘                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Squad Configuration

```python
SQUAD_CONFIGS = {
    "historical_analysis": {
        "agents": [
            {
                "name": "historian",
                "model": "oracle-reasoning-primary",
                "role": "Analyze historical context and timeline"
            },
            {
                "name": "archivist",
                "model": "oracle-reasoning-secondary",
                "role": "Verify source authenticity and provenance"
            }
        ],
        "consensus_threshold": 0.7,  # Require 70% agreement
        "max_iterations": 5
    },
    
    "statistical_analysis": {
        "agents": [
            {
                "name": "statistician",
                "model": "oracle-reasoning-primary",
                "role": "Perform statistical analysis and hypothesis testing"
            },
            {
                "name": "data_validator",
                "model": "oracle-reasoning-secondary",
                "role": "Validate statistical methods and assumptions"
            }
        ],
        "consensus_threshold": 0.8,
        "max_iterations": 3
    },
    
    "source_criticism": {
        "agents": [
            {
                "name": "source_critic",
                "model": "oracle-reasoning-primary",
                "role": "Evaluate source credibility and bias"
            },
            {
                "name": "fact_checker",
                "model": "oracle-vision",
                "role": "Cross-reference claims with primary sources"
            },
            {
                "name": "contrarian",
                "model": "oracle-reasoning-secondary",
                "role": "Actively challenge assumptions and findings"
            }
        ],
        "consensus_threshold": 0.6,  # Lower threshold for adversarial review
        "max_iterations": 7
    }
}
```

### Consensus Mechanism

```python
class ConsensusEngine:
    def __init__(self, threshold: float):
        self.threshold = threshold
    
    async def reach_consensus(
        self,
        squad_config: dict,
        task: str,
        context: dict
    ) -> dict:
        """Run squad and reach consensus."""
        
        results = []
        
        # Run each agent in parallel
        tasks = [
            self.run_agent(agent, task, context)
            for agent in squad_config["agents"]
        ]
        
        results = await asyncio.gather(*tasks)
        
        # Analyze results
        consensus = self.analyze_results(results)
        
        # If no consensus, iterate
        iteration = 0
        while not consensus["agreed"] and iteration < squad_config["max_iterations"]:
            iteration += 1
            
            # Create new context with disagreements
            new_context = {
                **context,
                "previous_results": results,
                "disagreements": consensus["disagreements"]
            }
            
            # Re-run agents
            tasks = [
                self.run_agent(agent, task, new_context)
                for agent in squad_config["agents"]
            ]
            
            results = await asyncio.gather(*tasks)
            consensus = self.analyze_results(results)
        
        return {
            "consensus": consensus["agreed"],
            "result": consensus["final_result"],
            "confidence": consensus["confidence"],
            "iterations": iteration + 1,
            "agent_results": results
        }
    
    def analyze_results(self, results: list) -> dict:
        """Analyze agent results for consensus."""
        
        # Extract key claims from each result
        claims = [self.extract_claims(r) for r in results]
        
        # Find common claims
        common_claims = self.find_common_claims(claims)
        
        # Calculate agreement level
        agreement = len(common_claims) / len(claims[0]) if claims else 0
        
        if agreement >= self.threshold:
            return {
                "agreed": True,
                "final_result": self.merge_results(results),
                "confidence": agreement,
                "disagreements": []
            }
        else:
            return {
                "agreed": False,
                "final_result": None,
                "confidence": agreement,
                "disagreements": self.find_disagreements(claims)
            }
```

---

## Horizontal Scaling Strategy

### Scaling Benefits by Component

| Component | Horizontal Scaling Benefit | Scaling Mechanism |
|-----------|---------------------------|-------------------|
| **Ingestion** | ⭐⭐⭐⭐⭐ HUGE | NATS task queue + worker pool |
| **Entity Extraction** | ⭐⭐⭐⭐⭐ HUGE | Distribute chunks across workers |
| **Embedding** | ⭐⭐⭐⭐⭐ HUGE | Each worker runs local infinity-emb |
| **Reasoning** | ⭐⭐ SMALL | API-bound, but key pooling helps |
| **Graph Queries** | ⭐⭐⭐ MEDIUM | Read-heavy, can distribute queries |
| **Report Generation** | ⭐⭐ SMALL | Single-threaded by nature |

### Worker Deployment

**For Your Friends:**

```bash
# 1. Clone repository
git clone https://github.com/your-repo/oracle-worker.git
cd oracle-worker

# 2. Configure
cp config.example.yaml config.yaml
# Edit config.yaml with your NATS server URL and credentials

# 3. Start worker
docker-compose up -d
```

**docker-compose.yml:**

```yaml
version: '3.8'

services:
  oracle-worker:
    build: .
    container_name: oracle-worker-${WORKER_ID}
    environment:
      - WORKER_ID=${WORKER_ID}
      - NATS_URL=${NATS_URL}
      - LITELLM_URL=${LITELLM_URL}
      - NEO4J_URI=${NEO4J_URI}
      - NEO4J_PASSWORD=${NEO4J_PASSWORD}
      - QDRANT_HOST=${QDRANT_HOST}
      - QDRANT_PORT=${QDRANT_PORT}
    volumes:
      - ./config.yaml:/app/config.yaml:ro
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

### Load Balancing

**NATS Load Balancing:**

NATS automatically load balances messages across consumers using queue groups:

```python
# All workers subscribe to the same subject with queue group
await js.subscribe(
    subject="oracle.tasks.extraction",
    stream="oracle.tasks",
    queue_group="extraction-workers",  # Load balances across workers
    consumer_name=f"worker-{worker_id}",
    deliver_policy="all",
    ack_policy="explicit"
)
```

**LiteLLM Load Balancing:**

LiteLLM automatically load balances across multiple API keys:

```yaml
model_list:
  - model_name: oracle-reasoning-primary
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_1}
      rpm: 500
    model_info:
      id: claude-primary-1
  
  - model_name: oracle-reasoning-primary
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_2}
      rpm: 500
    model_info:
      id: claude-primary-2
  
  # LiteLLM automatically round-robins between these two
```

---

## Fault Tolerance & Resilience

### Failure Scenarios & Mitigations

| Scenario | Detection | Mitigation | Recovery |
|----------|-----------|------------|----------|
| **Worker crashes** | NATS heartbeat timeout | Task re-queued after max_deliver | Worker restarts, picks up pending tasks |
| **API rate limit** | LiteLLM 429 error | Automatic fallback to backup model | Cooldown period, then retry |
| **LiteLLM down** | Connection timeout | Direct API call (bypass gateway) | LiteLLM restart |
| **NATS down** | Connection error | Local task queue (in-memory) | NATS restart, drain local queue |
| **PostgreSQL down** | Connection error | SQLite fallback (read-only) | PostgreSQL restart |
| **Neo4j down** | Connection error | Cache writes locally | Neo4j restart, replay cache |
| **Qdrant down** | Connection error | Skip embedding, continue | Qdrant restart, re-embed |

### Retry Strategy

```python
@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=2, max=60),
    retry=retry_if_exception_type((
        ConnectionError,
        TimeoutError,
        RateLimitError
    ))
async def resilient_api_call(func, *args, **kwargs):
    """Resilient API call with exponential backoff."""
    return await func(*args, **kwargs)
```

### Circuit Breaker

```python
class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open
    
    async def call(self, func, *args, **kwargs):
        """Call with circuit breaker protection."""
        
        if self.state == "open":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "half-open"
            else:
                raise CircuitBreakerOpenError()
        
        try:
            result = await func(*args, **kwargs)
            
            if self.state == "half-open":
                self.state = "closed"
                self.failures = 0
            
            return result
            
        except Exception as e:
            self.failures += 1
            self.last_failure_time = time.time()
            
            if self.failures >= self.failure_threshold:
                self.state = "open"
            
            raise
```

---

## Performance Considerations

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

### Optimization Strategies

1. **Batch Embedding:** Process chunks in batches of 32-64
2. **Async I/O:** Never block on network calls
3. **Connection Pooling:** Reuse database connections
4. **Caching:** Cache frequently accessed entities
5. **Lazy Loading:** Only load data when needed

---

## Monitoring & Observability

### Metrics to Track

**System Metrics:**
- Active investigations
- Active workers
- Task queue depth
- API call rate (per provider)
- Error rate (per component)

**Investigation Metrics:**
- Hypotheses confirmed/disproven
- Sources analyzed
- Entities discovered
- Time elapsed
- Estimated completion

**Resource Metrics:**
- CPU usage (per worker)
- GPU usage (per worker)
- Memory usage (per worker)
- Disk I/O (Neo4j, Qdrant)
- Network I/O (NATS)

### Logging Strategy

```python
import structlog

logger = structlog.get_logger()

# Log with context
logger.info(
    "task_started",
    task_id=task.task_id,
    task_type=task.task_type,
    investigation_id=task.investigation_id,
    worker_id=settings.worker.id
)

# Log errors with context
logger.error(
    "task_failed",
    task_id=task.task_id,
    error=str(e),
    traceback=traceback.format_exc(),
    worker_id=settings.worker.id
)
```

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

```bash
# Generate virtual key for each investigation
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

```python
# JWT-based authentication for workers
async def authenticate_worker(token: str) -> bool:
    """Verify worker token."""
    try:
        payload = jwt.decode(token, settings.secret_key, algorithms=["HS256"])
        return payload.get("worker_id") in settings.authorized_workers
    except jwt.InvalidTokenError:
        return False
```

---

## Deployment Checklist

### Phase 1: Infrastructure Setup

- [ ] Install PostgreSQL
- [ ] Install NATS JetStream
- [ ] Install LiteLLM
- [ ] Configure Neo4j
- [ ] Configure Qdrant
- [ ] Set up environment variables

### Phase 2: Hive Mind Deployment

- [ ] Install LangGraph checkpoint-postgres
- [ ] Create database schema
- [ ] Deploy Hive Mind service
- [ ] Test checkpointing

### Phase 3: Gateway Deployment

- [ ] Configure LiteLLM
- [ ] Add API keys
- [ ] Set up routing rules
- [ ] Test failover

### Phase 4: NATS Deployment

- [ ] Install NATS Server
- [ ] Enable JetStream
- [ ] Create streams
- [ ] Test message delivery

### Phase 5: Worker Deployment

- [ ] Build worker Docker image
- [ ] Distribute to friends
- [ ] Configure workers
- [ ] Test task processing

### Phase 6: Monitoring

- [ ] Set up Prometheus
- [ ] Configure Grafana dashboards
- [ ] Set up alerts
- [ ] Test alerting

---

## Conclusion

This architecture provides:

✅ **Multi-model orchestration** via LiteLLM with automatic failover
✅ **Horizontal scaling** via NATS JetStream and worker pool
✅ **Long-running persistence** via PostgreSQL (or SQLite for smaller deployments)
✅ **Fault tolerance** via retries, circuit breakers, and automatic recovery
✅ **Consensus mechanisms** for specialist agent squads
✅ **Monitoring** for observability and debugging

The system is designed to run investigations for months, survive failures, and scale horizontally as needed.

---

**Document Version:** 1.0  
**Last Updated:** 2025-01-02  
**Status:** VALIDATED ✅
