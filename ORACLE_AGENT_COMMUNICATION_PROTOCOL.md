# ORACLE Agent Communication Protocol

## Overview

This document specifies the communication protocol used by ORACLE agents to coordinate, collaborate, and share information. The protocol is built on top of NATS JetStream and defines message formats, patterns, and conventions for agent-to-agent communication.

---

## Protocol Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORACLE Communication Protocol                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Application Layer (Agent Logic)               │  │
│  │  - Task execution                                        │  │
│  │  - Result aggregation                                    │  │
│  │  - Consensus building                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Message Layer (Message Protocol)              │  │
│  │  - Message serialization                                 │  │
│  │  - Message validation                                   │  │
│  │  - Message routing                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Transport Layer (NATS JetStream)               │  │
│  │  - Message delivery                                      │  │
│  │  - Guaranteed delivery                                   │  │
│  │  - Message replay                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Message Types

### Core Message Types

```python
class MessageType(Enum):
    """Types of messages in the ORACLE protocol."""
    
    # Task messages
    TASK_CREATE = "task.create"
    TASK_UPDATE = "task.update"
    TASK_COMPLETE = "task.complete"
    TASK_FAIL = "task.fail"
    TASK_CANCEL = "task.cancel"
    
    # Result messages
    RESULT_SUCCESS = "result.success"
    RESULT_PARTIAL = "result.partial"
    RESULT_FAILURE = "result.failure"
    
    # Agent messages
    AGENT_REGISTER = "agent.register"
    AGENT_HEARTBEAT = "agent.heartbeat"
    AGENT_STATUS = "agent.status"
    AGENT_SHUTDOWN = "agent.shutdown"
    
    # Investigation messages
    INVESTIGATION_START = "investigation.start"
    INVESTIGATION_PAUSE = "investigation.pause"
    INVESTIGATION_RESUME = "investigation.resume"
    INVESTIGATION_COMPLETE = "investigation.complete"
    
    # Squad messages
    SQUAD_CREATE = "squad.create"
    SQUAD_JOIN = "squad.join"
    SQUAD_LEAVE = "squad.leave"
    SQUAD_CONSENSUS = "squad.consensus"
    
    # Communication messages
    MESSAGE_SEND = "message.send"
    MESSAGE_REPLY = "message.reply"
    MESSAGE_BROADCAST = "message.broadcast"
```

---

## Message Schema

### Base Message

```python
@dataclass
class BaseMessage:
    """Base message for all ORACLE communications."""
    
    message_id: str
    message_type: MessageType
    timestamp: str
    sender_id: str
    recipient_id: str | None = None  # None = broadcast
    investigation_id: str | None = None
    correlation_id: str | None = None  # For request/response
    reply_to: str | None = None  # Subject to reply to
    metadata: dict = field(default_factory=dict)
    
    def to_dict(self) -> dict:
        """Convert message to dictionary."""
        return {
            "message_id": self.message_id,
            "message_type": self.message_type.value,
            "timestamp": self.timestamp,
            "sender_id": self.sender_id,
            "recipient_id": self.recipient_id,
            "investigation_id": self.investigation_id,
            "correlation_id": self.correlation_id,
            "reply_to": self.reply_to,
            "metadata": self.metadata
        }
    
    @classmethod
    def from_dict(cls, data: dict) -> "BaseMessage":
        """Create message from dictionary."""
        return cls(
            message_id=data["message_id"],
            message_type=MessageType(data["message_type"]),
            timestamp=data["timestamp"],
            sender_id=data["sender_id"],
            recipient_id=data.get("recipient_id"),
            investigation_id=data.get("investigation_id"),
            correlation_id=data.get("correlation_id"),
            reply_to=data.get("reply_to"),
            metadata=data.get("metadata", {})
        )
    
    def to_bytes(self) -> bytes:
        """Convert message to bytes."""
        return json.dumps(self.to_dict()).encode()
    
    @classmethod
    def from_bytes(cls, data: bytes) -> "BaseMessage":
        """Create message from bytes."""
        return cls.from_dict(json.loads(data))
```

### Task Messages

```python
@dataclass
class TaskCreateMessage(BaseMessage):
    """Message to create a new task."""
    
    task_id: str
    task_type: str
    payload: dict
    priority: int = 5
    timeout_seconds: int = 3600
    retry_count: int = 0
    max_retries: int = 3
    
    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "task_id": self.task_id,
            "task_type": self.task_type,
            "payload": self.payload,
            "priority": self.priority,
            "timeout_seconds": self.timeout_seconds,
            "retry_count": self.retry_count,
            "max_retries": self.max_retries
        })
        return base


@dataclass
class TaskCompleteMessage(BaseMessage):
    """Message to indicate task completion."""
    
    task_id: str
    result: dict
    duration_ms: int
    tokens_used: int = 0
    cost: float = 0.0
    
    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "task_id": self.task_id,
            "result": self.result,
            "duration_ms": self.duration_ms,
            "tokens_used": self.tokens_used,
            "cost": self.cost
        })
        return base


@dataclass
class TaskFailMessage(BaseMessage):
    """Message to indicate task failure."""
    
    task_id: str
    error: str
    error_type: str
    retry_count: int
    should_retry: bool = True
    
    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "task_id": self.task_id,
            "error": self.error,
            "error_type": self.error_type,
            "retry_count": self.retry_count,
            "should_retry": self.should_retry
        })
        return base
```

### Agent Messages

```python
@dataclass
class AgentRegisterMessage(BaseMessage):
    """Message to register an agent."""
    
    agent_id: str
    agent_type: str
    capabilities: list[str]
    max_concurrent_tasks: int
    gpu_available: bool = False
    gpu_memory_mb: int = 0
    
    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "agent_id": self.agent_id,
            "agent_type": self.agent_type,
            "capabilities": self.capabilities,
            "max_concurrent_tasks": self.max_concurrent_tasks,
            "gpu_available": self.gpu_available,
            "gpu_memory_mb": self.gpu_memory_mb
        })
        return base


@dataclass
class AgentHeartbeatMessage(BaseMessage):
    """Message to indicate agent is alive."""
    
    active_tasks: int
    total_tasks_completed: int
    cpu_usage_percent: float
    memory_usage_mb: int
    gpu_usage_percent: float = 0.0
    
    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "active_tasks": self.active_tasks,
            "total_tasks_completed": self.total_tasks_completed,
            "cpu_usage_percent": self.cpu_usage_percent,
            "memory_usage_mb": self.memory_usage_mb,
            "gpu_usage_percent": self.gpu_usage_percent
        })
        return base
```

### Squad Messages

```python
@dataclass
class SquadCreateMessage(BaseMessage):
    """Message to create a squad."""
    
    squad_id: str
    squad_type: str
    agents: list[str]
    config: dict
    
    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "squad_id": self.squad_id,
            "squad_type": self.squad_type,
            "agents": self.agents,
            "config": self.config
        })
        return base


@dataclass
class SquadConsensusMessage(BaseMessage):
    """Message to indicate squad consensus."""
    
    squad_id: str
    consensus_reached: bool
    result: dict
    confidence: float
    iterations: int
    agent_results: list[dict]
    
    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "squad_id": self.squad_id,
            "consensus_reached": self.consensus_reached,
            "result": self.result,
            "confidence": self.confidence,
            "iterations": self.iterations,
            "agent_results": self.agent_results
        })
        return base
```

---

## Communication Patterns

### Request-Response Pattern

```python
class RequestResponsePattern:
    """Request-response pattern for synchronous communication."""
    
    def __init__(self, nc: nats.NATSConnection, timeout_ms: int = 30000):
        self.nc = nc
        self.timeout_ms = timeout_ms
        self.pending_requests = {}  # {correlation_id: Future}
    
    async def request(self, subject: str, message: BaseMessage) -> BaseMessage:
        """Send request and wait for response."""
        
        # Set correlation ID
        message.correlation_id = str(uuid.uuid4())
        message.reply_to = f"oracle.responses.{message.sender_id}"
        
        # Create future for response
        future = asyncio.Future()
        self.pending_requests[message.correlation_id] = future
        
        # Subscribe to responses
        sub = await self.nc.subscribe(
            message.reply_to,
            cb=self._on_response
        )
        
        try:
            # Send request
            await self.nc.publish(subject, message.to_bytes())
            
            # Wait for response
            response = await asyncio.wait_for(future, timeout=self.timeout_ms / 1000)
            
            return response
            
        finally:
            await sub.unsubscribe()
            del self.pending_requests[message.correlation_id]
    
    async def _on_response(self, msg):
        """Handle response message."""
        
        response = BaseMessage.from_bytes(msg.data)
        
        if response.correlation_id in self.pending_requests:
            future = self.pending_requests[response.correlation_id]
            future.set_result(response)
```

### Publish-Subscribe Pattern

```python
class PubSubPattern:
    """Publish-subscribe pattern for broadcast communication."""
    
    def __init__(self, nc: nats.NATSConnection):
        self.nc = nc
        self.subscriptions = {}  # {subject: [callback, ...]}
    
    async def publish(self, subject: str, message: BaseMessage):
        """Publish message to subject."""
        
        await self.nc.publish(subject, message.to_bytes())
    
    async def subscribe(self, subject: str, callback: Callable):
        """Subscribe to subject."""
        
        if subject not in self.subscriptions:
            self.subscriptions[subject] = []
        
        self.subscriptions[subject].append(callback)
        
        async def on_message(msg):
            message = BaseMessage.from_bytes(msg.data)
            await callback(message)
        
        await self.nc.subscribe(subject, cb=on_message)
```

### Queue Pattern

```python
class QueuePattern:
    """Queue pattern for task distribution."""
    
    def __init__(self, nc: nats.NATSConnection, js: nats.js.JetStreamContext):
        self.nc = nc
        self.js = js
    
    async def enqueue(self, stream: str, subject: str, message: BaseMessage):
        """Enqueue message to stream."""
        
        await self.js.publish(
            stream=stream,
            subject=subject,
            payload=message.to_bytes()
        )
    
    async def dequeue(
        self,
        stream: str,
        consumer_name: str,
        callback: Callable
    ):
        """Dequeue messages from stream."""
        
        async def on_message(msg):
            message = BaseMessage.from_bytes(msg.data)
            await callback(message)
            await msg.ack()
        
        await self.js.subscribe(
            stream=stream,
            consumer_name=consumer_name,
            cb=on_message
        )
```

---

## Subject Naming Convention

### Subject Structure

```
oracle.<category>.<action>.<entity>
```

### Categories

| Category | Description | Examples |
|----------|-------------|----------|
| `tasks` | Task-related messages | `tasks.create.parse_document` |
| `results` | Result messages | `results.success.investigation-123` |
| `agents` | Agent-related messages | `agents.register.worker-1` |
| `investigations` | Investigation messages | `investigations.start.epstein` |
| `squads` | Squad messages | `squads.create.historical-analysis` |
| `messages` | Agent communication | `messages.send.agent-1` |

### Examples

```
# Task creation
oracle.tasks.create.parse_document

# Task completion
oracle.tasks.complete.parse_document.task-123

# Agent registration
oracle.agents.register.worker-1

# Agent heartbeat
oracle.agents.heartbeat.worker-1

# Investigation start
oracle.investigations.start.epstein

# Squad creation
oracle.squads.create.historical-analysis

# Agent message
oracle.messages.send.agent-1.agent-2

# Result publication
oracle.results.success.investigation-123
```

---

## Message Flow Examples

### Task Execution Flow

```
1. Hive Mind creates task
   └─> Publish: oracle.tasks.create.parse_document
       └─> Message: TaskCreateMessage

2. Worker receives task
   └─> Subscribe: oracle.tasks.create.parse_document
       └─> Message: TaskCreateMessage

3. Worker processes task
   └─> Execute task
       ├─> Parse document
       ├─> Extract entities
       └─> Generate embeddings

4. Worker publishes result
   └─> Publish: oracle.results.success.investigation-123
       └─> Message: TaskCompleteMessage

5. Hive Mind receives result
   └─> Subscribe: oracle.results.success.>
       └─> Message: TaskCompleteMessage
       └─> Update investigation state
```

### Squad Consensus Flow

```
1. Hive Mind creates squad
   └─> Publish: oracle.squads.create.historical-analysis
       └─> Message: SquadCreateMessage

2. Agents join squad
   └─> Publish: oracle.squads.join.historical-analysis
       └─> Message: SquadJoinMessage

3. Hive Mind sends task to squad
   └─> Publish: oracle.messages.broadcast.squad-123
       └─> Message: MessageSendMessage

4. Agents process task
   └─> Each agent processes independently
       └─> Publish: oracle.messages.reply.squad-123
           └─> Message: MessageReplyMessage

5. Hive Mind aggregates results
   └─> Collect all agent results
       └─> Calculate consensus

6. Hive Mind publishes consensus
   └─> Publish: oracle.squads.consensus.historical-analysis
       └─> Message: SquadConsensusMessage
```

### Agent Registration Flow

```
1. Worker starts
   └─> Initialize components
       └─> Connect to NATS

2. Worker registers
   └─> Publish: oracle.agents.register.worker-1
       └─> Message: AgentRegisterMessage

3. Hive Mind receives registration
   └─> Subscribe: oracle.agents.register.>
       └─> Message: AgentRegisterMessage
       └─> Add to worker pool

4. Worker sends heartbeat
   └─> Publish: oracle.agents.heartbeat.worker-1
       └─> Message: AgentHeartbeatMessage
       └─> Every 30 seconds

5. Hive Mind monitors workers
   └─> Check heartbeat timestamps
       └─> Mark dead workers
```

---

## Error Handling

### Error Messages

```python
@dataclass
class ErrorMessage(BaseMessage):
    """Error message."""
    
    error_code: str
    error_message: str
    error_details: dict
    stack_trace: str | None = None
    
    def to_dict(self) -> dict:
        base = super().to_dict()
        base.update({
            "error_code": self.error_code,
            "error_message": self.error_message,
            "error_details": self.error_details,
            "stack_trace": self.stack_trace
        })
        return base


class ErrorCode(Enum):
    """Error codes."""
    
    # Task errors
    TASK_TIMEOUT = "TASK_TIMEOUT"
    TASK_FAILED = "TASK_FAILED"
    TASK_CANCELLED = "TASK_CANCELLED"
    
    # Agent errors
    AGENT_UNAVAILABLE = "AGENT_UNAVAILABLE"
    AGENT_OVERLOADED = "AGENT_OVERLOADED"
    AGENT_SHUTDOWN = "AGENT_SHUTDOWN"
    
    # Communication errors
    MESSAGE_INVALID = "MESSAGE_INVALID"
    MESSAGE_TIMEOUT = "MESSAGE_TIMEOUT"
    MESSAGE_DELIVERY_FAILED = "MESSAGE_DELIVERY_FAILED"
    
    # System errors
    DATABASE_ERROR = "DATABASE_ERROR"
    RATE_LIMIT_EXCEEDED = "RATE_LIMIT_EXCEEDED"
    INSUFFICIENT_RESOURCES = "INSUFFICIENT_RESOURCES"
```

### Error Handling Strategy

```python
class ErrorHandler:
    """Handles errors in message processing."""
    
    def __init__(self, nc: nats.NATSConnection):
        self.nc = nc
    
    async def handle_error(
        self,
        error: Exception,
        message: BaseMessage,
        context: dict
    ):
        """Handle an error."""
        
        # Determine error code
        error_code = self._classify_error(error)
        
        # Create error message
        error_message = ErrorMessage(
            message_id=str(uuid.uuid4()),
            message_type=MessageType.RESULT_FAILURE,
            timestamp=datetime.now().isoformat(),
            sender_id="error-handler",
            recipient_id=message.sender_id,
            investigation_id=message.investigation_id,
            correlation_id=message.correlation_id,
            error_code=error_code.value,
            error_message=str(error),
            error_details={
                "error_type": type(error).__name__,
                "context": context
            },
            stack_trace=traceback.format_exc() if isinstance(error, Exception) else None
        )
        
        # Publish error message
        await self.nc.publish(
            f"oracle.errors.{error_code.value}",
            error_message.to_bytes()
        )
        
        # Log error
        logger.error(
            "error_occurred",
            error_code=error_code.value,
            error_message=str(error),
            message_id=message.message_id,
            context=context
        )
    
    def _classify_error(self, error: Exception) -> ErrorCode:
        """Classify error into error code."""
        
        if isinstance(error, TimeoutError):
            return ErrorCode.TASK_TIMEOUT
        
        if isinstance(error, ConnectionError):
            return ErrorCode.AGENT_UNAVAILABLE
        
        if isinstance(error, RateLimitError):
            return ErrorCode.RATE_LIMIT_EXCEEDED
        
        if isinstance(error, MemoryError):
            return ErrorCode.INSUFFICIENT_RESOURCES
        
        return ErrorCode.TASK_FAILED
```

---

## Security

### Message Authentication

```python
class MessageAuthenticator:
    """Authenticates messages."""
    
    def __init__(self, secret_key: str):
        self.secret_key = secret_key
    
    def sign_message(self, message: BaseMessage) -> str:
        """Sign a message."""
        
        message_dict = message.to_dict()
        message_str = json.dumps(message_dict, sort_keys=True)
        
        signature = hmac.new(
            self.secret_key.encode(),
            message_str.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return signature
    
    def verify_message(self, message: BaseMessage, signature: str) -> bool:
        """Verify a message signature."""
        
        message_dict = message.to_dict()
        message_str = json.dumps(message_dict, sort_keys=True)
        
        expected_signature = hmac.new(
            self.secret_key.encode(),
            message_str.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return hmac.compare_digest(expected_signature, signature)
```

### Message Encryption

```python
class MessageEncryptor:
    """Encrypts and decrypts messages."""
    
    def __init__(self, encryption_key: str):
        self.cipher = Fernet(encryption_key.encode())
    
    def encrypt_message(self, message: BaseMessage) -> bytes:
        """Encrypt a message."""
        
        message_bytes = message.to_bytes()
        encrypted = self.cipher.encrypt(message_bytes)
        
        return encrypted
    
    def decrypt_message(self, encrypted: bytes) -> BaseMessage:
        """Decrypt a message."""
        
        decrypted = self.cipher.decrypt(encrypted)
        message = BaseMessage.from_bytes(decrypted)
        
        return message
```

---

## Performance Considerations

### Message Batching

```python
class MessageBatcher:
    """Batches messages for efficiency."""
    
    def __init__(self, batch_size: int = 100, timeout_ms: int = 100):
        self.batch_size = batch_size
        self.timeout_ms = timeout_ms
        self.batch = []
        self.last_flush = time.time()
    
    async def add_message(self, message: BaseMessage) -> list[BaseMessage] | None:
        """Add message to batch, return batch if ready."""
        
        self.batch.append(message)
        
        # Check if batch is ready
        if len(self.batch) >= self.batch_size:
            return self.flush()
        
        # Check timeout
        if (time.time() - self.last_flush) * 1000 >= self.timeout_ms:
            return self.flush()
        
        return None
    
    def flush(self) -> list[BaseMessage]:
        """Flush current batch."""
        
        batch = self.batch
        self.batch = []
        self.last_flush = time.time()
        return batch
```

### Message Compression

```python
class MessageCompressor:
    """Compresses and decompresses messages."""
    
    def compress_message(self, message: BaseMessage) -> bytes:
        """Compress a message."""
        
        message_bytes = message.to_bytes()
        compressed = gzip.compress(message_bytes)
        
        return compressed
    
    def decompress_message(self, compressed: bytes) -> BaseMessage:
        """Decompress a message."""
        
        decompressed = gzip.decompress(compressed)
        message = BaseMessage.from_bytes(decompressed)
        
        return message
```

---

## Monitoring

### Message Metrics

```python
class MessageMetrics:
    """Tracks message metrics."""
    
    def __init__(self):
        self.messages_sent = Counter()
        self.messages_received = Counter()
        self.message_latency = Histogram()
        self.message_size = Histogram()
    
    def record_sent(self, message_type: MessageType, size: int):
        """Record a sent message."""
        
        self.messages_sent[message_type.value] += 1
        self.message_size.observe(size)
    
    def record_received(self, message_type: MessageType, size: int, latency_ms: int):
        """Record a received message."""
        
        self.messages_received[message_type.value] += 1
        self.message_size.observe(size)
        self.message_latency.observe(latency_ms)
    
    def get_metrics(self) -> dict:
        """Get current metrics."""
        
        return {
            "messages_sent": dict(self.messages_sent),
            "messages_received": dict(self.messages_received),
            "message_latency": {
                "count": self.message_latency.count,
                "sum": self.message_latency.sum,
                "mean": self.message_latency.mean if self.message_latency.count > 0 else 0
            },
            "message_size": {
                "count": self.message_size.count,
                "sum": self.message_size.sum,
                "mean": self.message_size.mean if self.message_size.count > 0 else 0
            }
        }
```

---

## Testing

### Message Validation

```python
class MessageValidator:
    """Validates messages."""
    
    def validate(self, message: BaseMessage) -> tuple[bool, list[str]]:
        """Validate a message."""
        
        errors = []
        
        # Check required fields
        if not message.message_id:
            errors.append("message_id is required")
        
        if not message.message_type:
            errors.append("message_type is required")
        
        if not message.timestamp:
            errors.append("timestamp is required")
        
        if not message.sender_id:
            errors.append("sender_id is required")
        
        # Check timestamp format
        try:
            datetime.fromisoformat(message.timestamp)
        except ValueError:
            errors.append("timestamp is not in ISO format")
        
        # Check message type
        if not isinstance(message.message_type, MessageType):
            errors.append("message_type is not a valid MessageType")
        
        return len(errors) == 0, errors
```

### Integration Tests

```python
async def test_task_execution_flow():
    """Test task execution flow."""
    
    # Setup
    nc = await nats.connect()
    js = nc.jetstream
    
    # Create stream
    await js.create_stream(
        name="test.tasks",
        subjects=["test.tasks.>"],
        retention="workqueue"
    )
    
    # Create task
    task = TaskCreateMessage(
        message_id=str(uuid.uuid4()),
        message_type=MessageType.TASK_CREATE,
        timestamp=datetime.now().isoformat(),
        sender_id="test-sender",
        task_id="task-123",
        task_type="parse_document",
        payload={"file_path": "/test/document.pdf"}
    )
    
    # Publish task
    await js.publish(
        stream="test.tasks",
        subject="test.tasks.create",
        payload=task.to_bytes()
    )
    
    # Subscribe to results
    results = []
    
    async def on_result(msg):
        result = BaseMessage.from_bytes(msg.data)
        results.append(result)
    
    await js.subscribe(
        stream="test.results",
        consumer_name="test-consumer",
        cb=on_result
    )
    
    # Wait for result
    await asyncio.sleep(1)
    
    # Assert
    assert len(results) == 1
    assert results[0].message_type == MessageType.TASK_COMPLETE
    
    # Cleanup
    await js.delete_stream("test.tasks")
    await nc.close()
```

---

## Troubleshooting

### Common Issues

**Issue: Messages not being delivered**

**Solution:**
1. Check NATS server is running
2. Check stream exists
3. Check consumer exists
4. Check subject matches
5. Check network connectivity

**Issue: Messages timing out**

**Solution:**
1. Increase timeout
2. Check worker is processing messages
3. Check for blocking operations
4. Check resource usage

**Issue: Messages being duplicated**

**Solution:**
1. Check consumer configuration
2. Check ack_policy
3. Check max_deliver setting
4. Check for multiple consumers

**Issue: Messages being lost**

**Solution:**
1. Check retention policy
2. Check max_age setting
3. Check max_msgs setting
4. Check disk space

---

## Best Practices

### Message Design

1. **Keep messages small** — <1MB preferred
2. **Use compression** for large payloads
3. **Include correlation IDs** for request/response
4. **Use proper subject naming** — follow convention
5. **Validate messages** before processing

### Error Handling

1. **Always ACK/NACK** messages
2. **Log all errors** with context
3. **Implement retries** with exponential backoff
4. **Use circuit breakers** for failing services
5. **Monitor error rates** and alert

### Performance

1. **Batch messages** when possible
2. **Use compression** for large payloads
3. **Pool connections** to NATS
4. **Use async I/O** throughout
5. **Monitor latency** and optimize

### Security

1. **Sign messages** to verify authenticity
2. **Encrypt sensitive** messages
3. **Use TLS** for NATS connections
4. **Validate messages** before processing
5. **Rotate keys** regularly

---

## Conclusion

This communication protocol provides:

✅ **Standardized message formats** — All agents use the same schema
✅ **Multiple communication patterns** — Request-response, pub-sub, queue
✅ **Error handling** — Comprehensive error codes and handling
✅ **Security** — Authentication and encryption
✅ **Performance** — Batching, compression, pooling
✅ **Monitoring** — Comprehensive metrics
✅ **Testing** — Validation and integration tests

The protocol is designed to be simple, reliable, and performant for agent-to-agent communication in the ORACLE swarm.

---

**Document Version:** 1.0  
**Last Updated:** 2025-01-02  
**Status:** VALIDATED ✅
