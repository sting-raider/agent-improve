# Decision D017: GPU Mutex Design and Validation

**Status:** APPROVED  
**Date:** 2025-05-02  
**Decision:** Implement asyncio.Lock-based GPU mutex with priority queue

---

## Context

ORACLE v1 runs on a single machine with an RTX 4080 (16GB VRAM). Multiple GPU-bound services compete for VRAM:
- **infinity-emb** (embedding server): ~0.6-1GB for jina-embeddings-v5-text-small
- **GLiNER2** (entity extraction): ~400MB-1GB for gliner2-base-v1
- **Visualization** (3d-force-graph): ~1-2GB (only active during visualization)

The ingestion pipeline runs infinity-emb continuously. The investigation agent runs GLiNER2 intermittently. Both cannot run simultaneously without exceeding VRAM.

## Research Findings

### infinity-emb VRAM Behavior
**Finding:** infinity-emb loads the model into VRAM and keeps it loaded while the server is running.

**Evidence:**
- infinity-emb is a long-running server process
- Models are loaded at startup and kept in memory
- Dynamic batching keeps the GPU saturated
- No automatic VRAM release when idle

### GLiNER2 VRAM Behavior
**Finding:** GLiNER2 loads the model into VRAM and keeps it loaded while the extractor object exists.

**Evidence:**
- GLiNER2 uses PyTorch and loads model weights into GPU memory
- Static model weights: ~400MB (base) to ~1GB (large)
- Encoding memory: O(batch_size * seq_len^2)
- Span extraction memory: O(batch_size * seq_len * max_width)
- Uses `torch.inference_mode()` to disable gradient tracking
- Supports quantization (fp16) to reduce memory footprint

### VRAM Coexistence
**Finding:** infinity-emb and GLiNER2 can coexist in VRAM if not called simultaneously, but not if both are actively processing.

**Calculation:**
- infinity-emb: ~1GB (jina-embeddings-v5-text-small, 677M params)
- GLiNER2: ~1GB (gliner2-base-v1, 205M params)
- Total: ~2GB
- RTX 4080: 16GB VRAM
- **Conclusion:** Both can fit in VRAM simultaneously

**However:** During active processing, memory spikes occur:
- infinity-emb batch processing: additional memory for batch tensors
- GLiNER2 batch processing: additional memory for encoding and span extraction
- **Risk:** Simultaneous batch processing could exceed VRAM

## Decision

**Implement an asyncio.Lock-based GPU mutex with priority queue.**

### Design

```python
import asyncio
from enum import Enum
from typing import Optional

class GPUPriority(Enum):
    """Priority levels for GPU access."""
    INGESTION_EMBEDDING = 1  # Highest priority during ingestion
    INVESTIGATION_REASONING = 2  # Medium priority during investigation
    VISUALIZATION = 3  # Lowest priority

class GPUMutex:
    """GPU mutex with priority queue."""
    
    def __init__(self):
        self._lock = asyncio.Lock()
        self._queue = asyncio.PriorityQueue()
        self._current_holder: Optional[str] = None
        self._current_priority: Optional[GPUPriority] = None
    
    async def acquire(self, holder_id: str, priority: GPUPriority, timeout: float = 300.0):
        """
        Acquire GPU access with priority.
        
        Args:
            holder_id: Unique identifier for the holder
            priority: Priority level
            timeout: Maximum time to wait (seconds)
        
        Returns:
            True if acquired, False if timeout
        """
        try:
            # Add to priority queue
            await asyncio.wait_for(
                self._queue.put((priority.value, holder_id)),
                timeout=timeout
            )
            
            # Wait for turn
            while True:
                async with self._lock:
                    if self._current_holder is None:
                        # Our turn
                        self._current_holder = holder_id
                        self._current_priority = priority
                        return True
                    elif self._current_holder == holder_id:
                        # Already holding
                        return True
                    else:
                        # Someone else holding, wait
                        pass
                
                # Wait a bit and check again
                await asyncio.sleep(0.1)
                
        except asyncio.TimeoutError:
            # Remove from queue if timeout
            try:
                self._queue.get_nowait()
            except asyncio.QueueEmpty:
                pass
            return False
    
    async def release(self, holder_id: str):
        """Release GPU access."""
        async with self._lock:
            if self._current_holder == holder_id:
                self._current_holder = None
                self._current_priority = None
                # Signal next holder
                if not self._queue.empty():
                    _, next_holder = self._queue.get_nowait()
                    self._current_holder = next_holder
```

### Usage Pattern

```python
# Global GPU mutex
gpu_mutex = GPUMutex()

# During ingestion
async def embed_chunk(chunk):
    async with gpu_mutex.acquire("ingestion", GPUPriority.INGESTION_EMBEDDING):
        # infinity-emb call
        embeddings = await infinity_emb.embed([chunk.text])
    return embeddings

# During investigation
async def extract_entities(chunk):
    async with gpu_mutex.acquire("investigation", GPUPriority.INVESTIGATION_REASONING):
        # GLiNER2 call
        entities = gliner2.extract_entities(chunk.text, ["person", "org"])
    return entities
```

### Priority Queue Behavior

1. **Ingestion phase:** All embedding requests get highest priority
2. **Investigation phase:** Reasoning requests get medium priority
3. **Visualization phase:** Visualization gets lowest priority
4. **Timeout:** Default 300 seconds (5 minutes) to prevent deadlock

## Validation Plan

### Test 1: Sequential Loading
**Goal:** Verify both models can load sequentially without exceeding VRAM.

```python
import torch

# Test 1: Load infinity-emb
print("Loading infinity-emb...")
# infinity_emb.start()
vram_after_infinity = torch.cuda.memory_allocated() / 1024**3
print(f"VRAM after infinity-emb: {vram_after_infinity:.2f} GB")

# Test 2: Load GLiNER2
print("Loading GLiNER2...")
extractor = GLiNER2.from_pretrained("fastino/gliner2-base-v1", map_location="cuda")
vram_after_gliner = torch.cuda.memory_allocated() / 1024**3
print(f"VRAM after GLiNER2: {vram_after_gliner:.2f} GB")

# Test 3: Unload infinity-emb
# infinity_emb.stop()
vram_after_unload = torch.cuda.memory_allocated() / 1024**3
print(f"VRAM after unload: {vram_after_unload:.2f} GB")
```

**Expected Result:** VRAM < 3GB at all times

### Test 2: Simultaneous Processing
**Goal:** Verify simultaneous batch processing exceeds VRAM.

```python
# Test: Run both simultaneously
async def test_simultaneous():
    # infinity-emb batch
    embeddings_task = asyncio.create_task(
        infinity_emb.embed_batch([chunk.text for chunk in chunks[:32]])
    )
    
    # GLiNER2 batch
    entities_task = asyncio.create_task(
        gliner2.batch_extract_entities([chunk.text for chunk in chunks[:8]], ["person", "org"])
    )
    
    # Wait for both
    embeddings, entities = await asyncio.gather(embeddings_task, entities_task)
    
    return embeddings, entities
```

**Expected Result:** OOM error or severe slowdown

### Test 3: Mutex Behavior
**Goal:** Verify mutex prevents simultaneous processing.

```python
async def test_mutex():
    # Task 1: Embedding
    task1 = asyncio.create_task(
        embed_with_mutex(chunk1)
    )
    
    # Task 2: Entity extraction (starts immediately but waits)
    task2 = asyncio.create_task(
        extract_with_mutex(chunk2)
    )
    
    # Wait for both
    result1, result2 = await asyncio.gather(task1, task2)
    
    return result1, result2

async def embed_with_mutex(chunk):
    async with gpu_mutex.acquire("embed", GPUPriority.INGESTION_EMBEDDING):
        await asyncio.sleep(0.1)  # Simulate work
        return infinity_emb.embed([chunk.text])

async def extract_with_mutex(chunk):
    async with gpu_mutex.acquire("extract", GPUPriority.INVESTIGATION_REASONING):
        await asyncio.sleep(0.1)  # Simulate work
        return gliner2.extract_entities(chunk.text, ["person", "org"])
```

**Expected Result:** Tasks run sequentially, no OOM error

## Implementation Strategy

### Phase 1: Basic Mutex
1. Implement GPUMutex class
2. Add mutex to embedding pipeline
3. Add mutex to entity extraction pipeline
4. Test sequential processing

### Phase 2: Priority Queue
1. Implement priority queue
2. Add priority levels
3. Test priority behavior
4. Add timeout handling

### Phase 3: Integration
1. Integrate mutex into ingestion orchestrator
2. Integrate mutex into investigation agent
3. Add monitoring (who holds GPU, how long)
4. Add deadlock detection

## VRAM Budget

### Model VRAM Requirements
| Service | Model | VRAM (Static) | VRAM (Peak) |
|---------|-------|---------------|--------------|
| infinity-emb | jina-embeddings-v5-text-small | ~1GB | ~2GB (batch) |
| GLiNER2 | gliner2-base-v1 | ~1GB | ~3GB (batch) |
| Visualization | 3d-force-graph | ~1GB | ~2GB |

### Total VRAM Budget
- **Static:** ~3GB (all models loaded)
- **Peak:** ~7GB (all models processing)
- **Available:** 16GB
- **Headroom:** ~9GB

**Conclusion:** 16GB VRAM is sufficient with mutex.

## Risks and Mitigations

### Risk: Deadlock
**Mitigation:** 
- Timeout on acquire (default 300 seconds)
- Deadlock detection (monitor holder duration)
- Force release after timeout

### Risk: Priority Inversion
**Mitigation:**
- Priority queue ensures higher priority tasks go first
- Ingestion gets highest priority during ingestion phase
- Investigation gets medium priority during investigation phase

### Risk: VRAM Fragmentation
**Mitigation:**
- Call `torch.cuda.empty_cache()` periodically
- Use fixed batch sizes to avoid unpredictable memory spikes
- Monitor VRAM usage and adjust batch sizes dynamically

## Success Criteria

### v1 Success
- Can run ingestion for 24+ hours without OOM
- Can run investigation for 24+ hours without OOM
- Can switch between ingestion and investigation seamlessly
- VRAM usage stays below 12GB at all times
- No deadlocks or priority inversions

## References

- GLiNER2 Performance Optimization: https://deepwiki.com/fastino-ai/GLiNER2/6.2-performance-optimization
- infinity-emb GitHub: https://github.com/michaelfeil/infinity
- PyTorch Memory Management: https://pytorch.org/docs/stable/notes/cuda.html

---

**Approved by:** Hermes Agent  
**Next Action:** Implement GPU mutex and run validation tests
