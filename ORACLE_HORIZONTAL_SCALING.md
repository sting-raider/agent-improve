# ORACLE Horizontal Scaling Strategy

## Overview

This document details how ORACLE scales horizontally across multiple machines, enabling distributed processing of large corpora and long-running investigations. The strategy leverages NATS JetStream for task distribution, LiteLLM for model routing, and a worker pool architecture for parallel execution.

**Note:** This document describes the v2 swarm architecture. For the v1 single-machine architecture, see [DECISION_D016_SCOPE_LOCK.md](DECISION_D016_SCOPE_LOCK.md).

---

## Scaling Philosophy

### Core Principles

1. **Task-Based Distribution** — Work is broken into discrete tasks that can be processed independently
2. **Stateless Workers** — Workers don't maintain state; all state is in shared databases
3. **Fault Tolerance** — Workers can fail without affecting the overall system
4. **Elastic Scaling** — Workers can be added or removed dynamically
5. **Load Balancing** — Work is automatically distributed across available workers

### What Scales Horizontally

| Component | Horizontal Scaling Benefit | Scaling Mechanism |
|-----------|---------------------------|-------------------|
| **Document Parsing** | ⭐⭐⭐⭐⭐ HUGE | NATS task queue + worker pool |
| **Entity Extraction** | ⭐⭐⭐⭐⭐ HUGE | Distribute chunks across workers |
| **Embedding** | ⭐⭐⭐⭐⭐ HUGE | Each worker runs local infinity-emb |
| **Graph Queries** | ⭐⭐⭐ MEDIUM | Read-heavy, can distribute queries |
| **Reasoning** | ⭐⭐ SMALL | API-bound, but key pooling helps |
| **Report Generation** | ⭐⭐ SMALL | Single-threaded by nature |

### What Doesn't Scale Horizontally

- **Neo4j** — Single database instance (can cluster, but complex)
- **Qdrant** — Single database instance (can cluster, but complex)
- **PostgreSQL** — Single database instance (can cluster, but complex)
- **LiteLLM Gateway** — Single instance (can load balance, but not distribute)

---

## Architecture

### System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORACLE Swarm Architecture                     │
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
│  │              │  │              │  │              │     │
│  │  - Docling   │  │  - Docling   │  │  - Docling   │     │
│  │  - infinity  │  │  - infinity  │  │  - infinity  │     │
│  │  - GLiNER2   │  │  - GLiNER2   │  │  - GLiNER2   │     │
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

### Data Flow

```
1. Investigation Started
   └─> Hive Mind decomposes into tasks
       └─> Tasks published to NATS

2. Task Distribution
   └─> NATS distributes tasks to workers
       └─> Workers pull tasks from queue

3. Task Execution
   └─> Worker processes task
       ├─> Parse document (Docling)
       ├─> Extract entities (GLiNER2)
       ├─> Generate embeddings (infinity-emb with jina-embeddings-v5)
       └─> Write to databases

4. Result Aggregation
   └─> Worker publishes result to NATS
       └─> Hive Mind consumes result
           └─> Updates investigation state

5. Progress Tracking
   └─> Hive Mind writes checkpoint to PostgreSQL
       └─> Investigation can be resumed from any point
```

---

## Worker Architecture

### Worker Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        Worker Process                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              NATS Consumer                                  │  │
│  │  - Subscribes to task streams                              │  │
│  │  - Pulls tasks from queue                                  │  │
│  │  - ACKs/NACKs messages                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Task Executor                                  │  │
│  │  - Routes task to appropriate handler                     │  │
│  │  - Manages task lifecycle                                 │  │
│  │  - Handles errors and retries                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│         ┌─────────────────┼─────────────────┐                    │
│         │                 │                 │                    │
│         ▼                 ▼                 ▼                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Document    │  │  Entity      │  │  Embedding   │     │
│  │  Parser      │  │  Extractor   │  │  Generator   │     │
│  │  (Docling)   │  │  (GLiNER2)   │  │  (infinity)  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                 │                 │                    │
│         └─────────────────┼─────────────────┘                    │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Database Writer                                │  │
│  │  - Writes to Neo4j                                        │  │
│  │  - Writes to Qdrant                                       │  │
│  │  - Handles write errors                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Result Publisher                               │  │
│  │  - Publishes result to NATS                                │  │
│  │  - Includes task metadata                                 │  │
│  │  - Handles publish errors                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Worker Configuration

```python
# config.yaml
worker:
  id: "worker-1"
  capabilities:
    - "document_parsing"
    - "entity_extraction"
    - "embedding"
    - "reasoning"
  max_concurrent_tasks: 4
  task_timeout: 3600  # 1 hour

nats:
  url: "nats://your-server:4222"
  jetstream: true
  streams:
    tasks: "oracle.tasks"
    results: "oracle.results"

lite_llm:
  url: "http://your-server:4000"

neo4j:
  uri: "bolt://your-server:7687"
  username: "neo4j"
  password: "${NEO4J_PASSWORD}"

qdrant:
  host: "your-server"
  port: 6333

infinity_emb:
  host: "localhost"
  port: 7997
  model: "jinaai/jina-embeddings-v5-text-small"

gliner2:
  model: "fastino/gliner2-base-v1"
  device: "cpu"  # GLiNER2 is CPU-first
```

### Worker Docker Image

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy worker code
COPY oracle/worker/ /app/worker/

# Set working directory
WORKDIR /app/worker

# Start worker
CMD ["python", "main.py"]
```

### Worker Deployment

```yaml
# docker-compose.yml
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
      - NEO4J_PASSWORD=***
      - QDRANT_HOST=${QDRANT_HOST}
      - QDRANT_PORT=${QDRANT_PORT}
    volumes:
      - ./config.yaml:/app/config.yaml:ro
      - /path/to/corpus:/corpus:ro  # Read-only corpus access
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      - oracle-network

networks:
  oracle-network:
    external: true
```

---

## Task Distribution

### Task Types

```python
class TaskType(Enum):
    """Types of tasks that can be distributed."""
    
    # Ingestion tasks
    PARSE_DOCUMENT = "parse_document"
    EXTRACT_ENTITIES = "extract_entities"
    GENERATE_EMBEDDINGS = "generate_embeddings"
    
    # Investigation tasks
    CORPUS_SEARCH = "corpus_search"
    GRAPH_TRAVERSAL = "graph_traversal"
    WEB_RESEARCH = "web_research"
    CODE_ANALYSIS = "code_analysis"
    
    # Specialist tasks
    SOURCE_CRITICISM = "source_criticism"
    HISTORICAL_ANALYSIS = "historical_analysis"
    STATISTICAL_ANALYSIS = "statistical_analysis"
    PATTERN_DETECTION = "pattern_detection"
```

### Task Schema

```python
@dataclass
class Task:
    """A task to be processed by a worker."""
    
    task_id: str
    task_type: TaskType
    investigation_id: str
    payload: dict
    priority: int  # 0-10, higher is more important
    created_at: str
    timeout_seconds: int = 3600
    retry_count: int = 0
    max_retries: int = 3
    
    def to_message(self) -> bytes:
        """Convert to NATS message."""
        return json.dumps({
            "task_id": self.task_id,
            "task_type": self.task_type.value,
            "investigation_id": self.investigation_id,
            "payload": self.payload,
            "priority": self.priority,
            "created_at": self.created_at,
            "timeout_seconds": self.timeout_seconds,
            "retry_count": self.retry_count,
            "max_retries": self.max_retries
        }).encode()
    
    @classmethod
    def from_message(cls, data: bytes) -> "Task":
        """Create from NATS message."""
        data_dict = json.loads(data)
        return cls(
            task_id=data_dict["task_id"],
            task_type=TaskType(data_dict["task_type"]),
            investigation_id=data_dict["investigation_id"],
            payload=data_dict["payload"],
            priority=data_dict["priority"],
            created_at=data_dict["created_at"],
            timeout_seconds=data_dict.get("timeout_seconds", 3600),
            retry_count=data_dict.get("retry_count", 0),
            max_retries=data_dict.get("max_retries", 3)
        )
```

### Task Priority

```python
class TaskPriority:
    """Task priority levels."""
    
    CRITICAL = 10  # User-initiated, time-sensitive
    HIGH = 7       # Important investigation tasks
    NORMAL = 5    # Standard processing
    LOW = 3       # Background processing
    BACKGROUND = 1  # Non-critical, can wait

# Priority routing
PRIORITY_STREAMS = {
    TaskPriority.CRITICAL: "oracle.tasks.critical",
    TaskPriority.HIGH: "oracle.tasks.high",
    TaskPriority.NORMAL: "oracle.tasks.normal",
    TaskPriority.LOW: "oracle.tasks.low",
    TaskPriority.BACKGROUND: "oracle.tasks.background"
}
```

### NATS Stream Configuration

```python
async def setup_nats_streams(js):
    """Set up NATS JetStream streams."""
    
    # Task streams (one per priority level)
    for priority, stream_name in PRIORITY_STREAMS.items():
        await js.create_stream(
            name=stream_name,
            subjects=[f"oracle.tasks.{priority.value}.>"],
            retention="workqueue",  # Messages deleted after ACK
            max_age=30 * 24 * 60 * 60 * 1000,  # 30 days
            storage="file",
            replicas=1,
            max_msgs_per_subject=1000000
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

---

## Load Balancing

### NATS Load Balancing

NATS automatically load balances messages across consumers using queue groups:

```python
# All workers subscribe to the same subject with queue group
await js.subscribe(
    subject="oracle.tasks.normal.parse_document",
    stream="oracle.tasks.normal",
    queue_group="parse-document-workers",  # Load balances across workers
    consumer_name=f"worker-{worker_id}",
    deliver_policy="all",
    ack_policy="explicit",
    max_deliver=3
)
```

**How it works:**
1. Multiple workers subscribe to the same subject with the same queue group
2. NATS distributes messages round-robin across workers
3. Each message is delivered to exactly one worker
4. If a worker fails, messages are re-queued and delivered to another worker

### Worker Selection

```python
class WorkerSelector:
    """Selects workers for tasks based on capabilities."""
    
    def __init__(self):
        self.workers = {}  # {worker_id: WorkerInfo}
    
    def register_worker(self, worker_id: str, capabilities: list[str]):
        """Register a worker."""
        
        self.workers[worker_id] = WorkerInfo(
            worker_id=worker_id,
            capabilities=capabilities,
            last_seen=datetime.now()
        )
    
    def select_workers(
        self,
        task_type: TaskType,
        count: int = 1
    ) -> list[str]:
        """Select workers for a task."""
        
        # Get required capabilities for task type
        required_capabilities = TASK_CAPABILITIES.get(task_type, [])
        
        # Filter workers with required capabilities
        capable_workers = [
            worker_id
            for worker_id, worker_info in self.workers.items()
            if all(cap in worker_info.capabilities for cap in required_capabilities)
        ]
        
        if not capable_workers:
            raise NoCapableWorkersError(f"No workers capable of {task_type}")
        
        # Select workers (round-robin)
        selected = []
        for i in range(count):
            worker_id = capable_workers[i % len(capable_workers)]
            selected.append(worker_id)
        
        return selected
    
    def get_worker_status(self) -> dict:
        """Get status of all workers."""
        
        return {
            worker_id: {
                "capabilities": worker_info.capabilities,
                "last_seen": worker_info.last_seen.isoformat(),
                "active": (datetime.now() - worker_info.last_seen).total_seconds() < 60
            }
            for worker_id, worker_info in self.workers.items()
        }


@dataclass
class WorkerInfo:
    worker_id: str
    capabilities: list[str]
    last_seen: datetime
    active_tasks: int = 0
    total_tasks_completed: int = 0


TASK_CAPABILITIES = {
    TaskType.PARSE_DOCUMENT: ["document_parsing"],
    TaskType.EXTRACT_ENTITIES: ["entity_extraction"],
    TaskType.GENERATE_EMBEDDINGS: ["embedding"],
    TaskType.CORPUS_SEARCH: ["reasoning"],
    TaskType.GRAPH_TRAVERSAL: ["reasoning"],
    TaskType.WEB_RESEARCH: ["reasoning"],
    TaskType.CODE_ANALYSIS: ["reasoning"],
    TaskType.SOURCE_CRITICISM: ["reasoning"],
    TaskType.HISTORICAL_ANALYSIS: ["reasoning"],
    TaskType.STATISTICAL_ANALYSIS: ["reasoning"],
    TaskType.PATTERN_DETECTION: ["reasoning"]
}
```

---

## Scaling Strategies

### Vertical Scaling (Single Machine)

**When to use:**
- Small corpus (<50GB)
- Few concurrent investigations (<5)
- Limited budget
- Simple deployment

**Configuration:**
```yaml
# Single machine with multiple workers
workers:
  - id: "worker-1"
    capabilities: ["document_parsing", "entity_extraction", "embedding"]
    max_concurrent_tasks: 8
    gpu_fraction: 0.5  # Share GPU between workers
  
  - id: "worker-2"
    capabilities: ["reasoning"]
    max_concurrent_tasks: 4
    gpu_fraction: 0.5
```

**Pros:**
- Simple deployment
- No network overhead
- Lower cost

**Cons:**
- Limited by single machine resources
- Single point of failure
- Harder to scale beyond hardware limits

### Horizontal Scaling (Multiple Machines)

**When to use:**
- Large corpus (>50GB)
- Many concurrent investigations (>5)
- Need for fault tolerance
- Budget for multiple machines

**Configuration:**
```yaml
# Multiple machines, each with one worker
workers:
  - id: "worker-1"
    host: "your-machine"
    capabilities: ["document_parsing", "entity_extraction", "embedding"]
    max_concurrent_tasks: 4
  
  - id: "worker-2"
    host: "friend-1-machine"
    capabilities: ["document_parsing", "entity_extraction", "embedding"]
    max_concurrent_tasks: 4
  
  - id: "worker-3"
    host: "friend-2-machine"
    capabilities: ["reasoning"]
    max_concurrent_tasks: 4
```

**Pros:**
- Scales indefinitely
- Fault tolerance
- Better resource utilization

**Cons:**
- Network overhead
- More complex deployment
- Higher cost

### Hybrid Scaling

**When to use:**
- Mixed workload (some CPU-bound, some GPU-bound)
- Budget constraints
- Need for both speed and capacity

**Configuration:**
```yaml
# Local machine for GPU-bound tasks
workers:
  - id: "local-gpu-worker"
    host: "your-machine"
    capabilities: ["embedding", "reasoning"]
    max_concurrent_tasks: 2
    gpu_fraction: 1.0  # Full GPU

# Remote machines for CPU-bound tasks
  - id: "remote-cpu-worker-1"
    host: "friend-1-machine"
    capabilities: ["document_parsing", "entity_extraction"]
    max_concurrent_tasks: 8
    gpu_fraction: 0.0  # No GPU needed
  
  - id: "remote-cpu-worker-2"
    host: "friend-2-machine"
    capabilities: ["document_parsing", "entity_extraction"]
    max_concurrent_tasks: 8
    gpu_fraction: 0.0
```

**Pros:**
- Optimal resource utilization
- Cost-effective
- Flexible

**Cons:**
- More complex configuration
- Requires careful task routing

---

## Performance Optimization

### Batching

```python
class BatchProcessor:
    """Processes tasks in batches for efficiency."""
    
    def __init__(self, batch_size: int = 32, timeout_ms: int = 100):
        self.batch_size = batch_size
        self.timeout_ms = timeout_ms
        self.batch = []
        self.last_flush = time.time()
    
    async def add_task(self, task: Task) -> list[Task] | None:
        """Add task to batch, return batch if ready."""
        
        self.batch.append(task)
        
        # Check if batch is ready
        if len(self.batch) >= self.batch_size:
            return self.flush()
        
        # Check timeout
        if (time.time() - self.last_flush) * 1000 >= self.timeout_ms:
            return self.flush()
        
        return None
    
    def flush(self) -> list[Task]:
        """Flush current batch."""
        
        batch = self.batch
        self.batch = []
        self.last_flush = time.time()
        return batch
```

### Parallel Processing

```python
async def process_tasks_parallel(tasks: list[Task], max_concurrent: int = 4):
    """Process tasks in parallel with concurrency limit."""
    
    semaphore = asyncio.Semaphore(max_concurrent)
    
    async def process_with_limit(task: Task):
        async with semaphore:
            return await process_task(task)
    
    results = await asyncio.gather(
        *[process_with_limit(task) for task in tasks],
        return_exceptions=True
    )
    
    return results
```

### Connection Pooling

```python
class ConnectionPool:
    """Manages database connections."""
    
    def __init__(self, max_connections: int = 10):
        self.max_connections = max_connections
        self.neo4j_pool = []
        self.qdrant_pool = []
        self.neo4j_semaphore = asyncio.Semaphore(max_connections)
        self.qdrant_semaphore = asyncio.Semaphore(max_connections)
    
    async def get_neo4j_connection(self):
        """Get a Neo4j connection from the pool."""
        
        await self.neo4j_semaphore.acquire()
        
        if self.neo4j_pool:
            return self.neo4j_pool.pop()
        else:
            return AsyncGraphDatabase.driver(
                settings.NEO4J_URI,
                auth=(settings.NEO4J_USERNAME, settings.NEO4J_PASSWORD)
            )
    
    async def release_neo4j_connection(self, connection):
        """Release a Neo4j connection back to the pool."""
        
        if len(self.neo4j_pool) < self.max_connections:
            self.neo4j_pool.append(connection)
        else:
            await connection.close()
        
        self.neo4j_semaphore.release()
```

---

## Fault Tolerance

### Worker Failure Detection

```python
class WorkerMonitor:
    """Monitors worker health."""
    
    def __init__(self):
        self.worker_heartbeats = {}  # {worker_id: last_heartbeat}
        self.timeout_seconds = 60
    
    def record_heartbeat(self, worker_id: str):
        """Record a worker heartbeat."""
        
        self.worker_heartbeats[worker_id] = time.time()
    
    def is_worker_alive(self, worker_id: str) -> bool:
        """Check if worker is alive."""
        
        last_heartbeat = self.worker_heartbeats.get(worker_id)
        if not last_heartbeat:
            return False
        
        return (time.time() - last_heartbeat) < self.timeout_seconds
    
    def get_dead_workers(self) -> list[str]:
        """Get list of dead workers."""
        
        return [
            worker_id
            for worker_id, last_heartbeat in self.worker_heartbeats.items()
            if (time.time() - last_heartbeat) >= self.timeout_seconds
        ]
```

### Task Recovery

```python
class TaskRecovery:
    """Recovers tasks from failed workers."""
    
    async def recover_tasks(self, dead_worker_id: str):
        """Recover tasks from a dead worker."""
        
        # Get NATS connection
        nc = await nats.connect(settings.nats.url)
        js = nc.jetstream
        
        # Get stream info
        stream = await js.get_stream("oracle.tasks")
        
        # Get consumer info
        consumer = await js.get_consumer(
            "oracle.tasks",
            f"worker-{dead_worker_id}"
        )
        
        # Get pending messages
        pending = await js.stream_info("oracle.tasks")
        
        # Re-queue messages
        for msg_info in pending:
            # Publish to new stream
            await js.publish(
                f"oracle.tasks.recovered.{msg_info.subject}",
                msg_info.data
            )
        
        # Delete old consumer
        await js.delete_consumer("oracle.tasks", f"worker-{dead_worker_id}")
```

### Circuit Breaker

```python
class CircuitBreaker:
    """Circuit breaker for failing workers."""
    
    def __init__(self, failure_threshold: int = 5, timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = {}  # {worker_id: failure_count}
        self.last_failure_time = {}  # {worker_id: timestamp}
        self.state = {}  # {worker_id: "closed" | "open" | "half-open"}
    
    def record_failure(self, worker_id: str):
        """Record a failure for a worker."""
        
        self.failures[worker_id] = self.failures.get(worker_id, 0) + 1
        self.last_failure_time[worker_id] = time.time()
        
        if self.failures[worker_id] >= self.failure_threshold:
            self.state[worker_id] = "open"
    
    def is_available(self, worker_id: str) -> bool:
        """Check if worker is available."""
        
        state = self.state.get(worker_id, "closed")
        
        if state == "closed":
            return True
        
        if state == "open":
            # Check if timeout has passed
            last_failure = self.last_failure_time.get(worker_id, 0)
            if time.time() - last_failure > self.timeout:
                self.state[worker_id] = "half-open"
                return True
            return False
        
        if state == "half-open":
            return True
        
        return False
    
    def record_success(self, worker_id: str):
        """Record a success for a worker."""
        
        if self.state.get(worker_id) == "half-open":
            self.state[worker_id] = "closed"
            self.failures[worker_id] = 0
```

---

## Monitoring

### Metrics to Track

**System Metrics:**
- Active workers
- Total workers
- Dead workers
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

### Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

# System metrics
active_workers = Gauge('oracle_active_workers', 'Number of active workers')
total_workers = Gauge('oracle_total_workers', 'Number of total workers')
task_queue_depth = Gauge('oracle_task_queue_depth', 'Task queue depth', ['priority'])
task_completion_rate = Gauge('oracle_task_completion_rate', 'Task completion rate')

# Worker metrics
tasks_processed = Counter('oracle_tasks_processed_total', 'Total tasks processed', ['worker_id', 'task_type'])
tasks_failed = Counter('oracle_tasks_failed_total', 'Total tasks failed', ['worker_id', 'task_type'])
task_duration = Histogram('oracle_task_duration_seconds', 'Task duration', ['worker_id', 'task_type'])
worker_cpu_usage = Gauge('oracle_worker_cpu_usage', 'Worker CPU usage', ['worker_id'])
worker_gpu_usage = Gauge('oracle_worker_gpu_usage', 'Worker GPU usage', ['worker_id'])
worker_memory_usage = Gauge('oracle_worker_memory_usage', 'Worker memory usage', ['worker_id'])

# Investigation metrics
active_investigations = Gauge('oracle_active_investigations', 'Number of active investigations')
total_investigations = Gauge('oracle_total_investigations', 'Number of total investigations')
investigation_tasks = Gauge('oracle_investigation_tasks', 'Tasks per investigation', ['investigation_id'])
investigation_completion_rate = Gauge('oracle_investigation_completion_rate', 'Investigation completion rate', ['investigation_id'])
```

### Grafana Dashboards

**System Overview Dashboard:**
- Active workers over time
- Task queue depth over time
- Task completion rate over time
- Task failure rate over time

**Worker Performance Dashboard:**
- Tasks processed per worker
- Task duration per worker
- CPU/GPU/Memory usage per worker
- Error rate per worker

**Investigation Progress Dashboard:**
- Active investigations
- Tasks per investigation
- Completion rate per investigation
- Estimated time remaining

---

## Deployment Guide

### For Your Machine (Primary)

```bash
# 1. Clone repository
git clone https://github.com/your-repo/oracle-worker.git
cd oracle-worker

# 2. Configure
cp config.example.yaml config.yaml
# Edit config.yaml with your settings

# 3. Start worker
docker-compose up -d

# 4. Verify worker is connected
docker logs -f oracle-worker-worker-1
```

### For Your Friends' Machines

```bash
# 1. Send them the repository
git clone https://github.com/your-repo/oracle-worker.git
cd oracle-worker

# 2. Configure
cp config.example.yaml config.yaml
# Edit config.yaml with your NATS server URL

# 3. Start worker
docker-compose up -d

# 4. Verify worker is connected
docker logs -f oracle-worker-worker-1
```

### Scaling Up

```bash
# Add more workers to your machine
docker-compose up -d --scale oracle-worker=4

# Add workers to friends' machines
# They just run docker-compose up -d
```

### Scaling Down

```bash
# Remove workers from your machine
docker-compose down

# Friends can stop their workers
docker-compose down
```

---

## Troubleshooting

### Common Issues

**Issue: Workers not connecting to NATS**

**Solution:**
1. Check NATS server is running: `docker ps | grep nats`
2. Check NATS URL in config.yaml
3. Check network connectivity: `ping your-server`
4. Check firewall rules

**Issue: Tasks not being processed**

**Solution:**
1. Check task queue depth: `nats stream info oracle.tasks`
2. Check worker logs: `docker logs oracle-worker-worker-1`
3. Check worker capabilities match task requirements
4. Check worker is not rate limited

**Issue: Workers failing frequently**

**Solution:**
1. Check worker logs for errors
2. Check database connectivity
3. Check API key configuration
4. Check resource usage (CPU/GPU/Memory)

**Issue: Slow processing**

**Solution:**
1. Check worker resource usage
2. Increase max_concurrent_tasks
3. Add more workers
4. Optimize task batching

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
| **Groq** | Llama 3.3 70B Versatile | $0.59 | Primary reasoning (fast) |
| **Google** | Gemini 2.5 Flash | $0.075 | Secondary reasoning |
| **OpenRouter** | Claude Sonnet 4.5 (Free) | $0 | Tertiary reasoning |
| **GLiNER2** | fastino/gliner2-base-v1 | $0 | Local, free (CPU) |
| **Jina Embeddings** | jina-embeddings-v5-text-small | $0 | Local, free (GPU) |

**Estimated Monthly Costs:**
- Small investigation (10GB corpus): $10-30
- Medium investigation (50GB corpus): $50-100
- Large investigation (300GB corpus): $200-400

### Optimization Strategies

1. **Use local models** for extraction (GLiNER2) and embedding (jina-embeddings-v5)
2. **Batch requests** to reduce API calls
3. **Cache results** to avoid duplicate processing
4. **Use free API providers** (Groq, OpenRouter free tier)
5. **Implement budget caps** per investigation

---

## Conclusion

This horizontal scaling strategy provides:

✅ **Elastic scaling** — Add/remove workers dynamically
✅ **Fault tolerance** — Workers can fail without affecting system
✅ **Load balancing** — Automatic distribution across workers
✅ **Performance optimization** — Batching, parallel processing, connection pooling
✅ **Monitoring** — Comprehensive metrics and dashboards
✅ **Cost-effective** — Leverage friends' GPUs, use local models

The system is designed to scale from a single machine to dozens of workers, handling corpora from 10GB to 300GB+.

---

**Document Version:** 2.0  
**Last Updated:** 2025-05-02  
**Status:** UPDATED ✅ (Model stack aligned with D018)
