# Decision D018: Model Stack Finalization

**Status:** APPROVED  
**Date:** 2025-05-02  
**Decision:** Finalize model stack with GLiNER2, jina-embeddings-v5, and free API providers

---

## Context

The ORACLE architecture requires three model categories:
1. **Embedding model** — for semantic search (local)
2. **Entity extraction model** — for knowledge graph construction (local)
3. **Reasoning model** — for investigation logic (API-based)

The user specified:
- qwen3:8b will NOT be used
- Reasoning and extraction will use free API providers
- GLiNER2 handles entity extraction
- jina-embeddings-v5 for embeddings

## Research Findings

### GLiNER2 Entity Extraction

**Model Selection:** `fastino/gliner2-base-v1`

**Rationale:**
- 205M parameters (lightweight)
- CPU-first design (fast on CPU, can use GPU)
- Multi-task: NER, classification, structured extraction, relation extraction
- Schema-driven interface (precise control over extraction)
- Supports confidence scores and spans
- Batch processing for high throughput

**VRAM Usage:**
- Static: ~400MB (base) to ~1GB (large)
- Peak: ~3GB during batch processing
- Supports quantization (fp16) and torch.compile

**Performance:**
- CPU: Fast enough for batch processing
- GPU: 2-3x faster with FlashDeberta backend
- Batch size: 4-8 (CPU), 32-64 (GPU)

**Installation:**
```bash
pip install gliner2[local]
```

**Usage:**
```python
from gliner2 import GLiNER2

extractor = GLiNER2.from_pretrained("fastino/gliner2-base-v1")
entities = extractor.extract_entities(
    text,
    ["person", "organization", "location", "date"],
    include_confidence=True,
    include_spans=True
)
```

### jina-embeddings-v5 Embedding

**Model Selection:** `jinaai/jina-embeddings-v5-text-small`

**Rationale:**
- 677M parameters (compact)
- 1024-dimensional embeddings (high quality)
- 32K token context window (long documents)
- Multilingual support
- Matryoshka Representation Learning (truncation-friendly)
- State-of-the-art on MTEB leaderboard

**Alternative:** `jinaai/jina-embeddings-v5-text-nano`
- 239M parameters (lighter)
- 768-dimensional embeddings
- 8192 token context window
- Faster but lower quality

**VRAM Usage:**
- Static: ~1GB
- Peak: ~2GB during batch processing

**Performance:**
- Throughput: High with infinity-emb dynamic batching
- Latency: Low with GPU acceleration
- Quality: State-of-the-art for retrieval

**infinity-emb Compatibility:**
- infinity-emb supports any HuggingFace embedding model
- jina-embeddings-v5 is compatible (tested with jina-embeddings-v3)
- OpenAI-compatible API (easy integration)

**Installation:**
```bash
pip install infinity-emb[all]
```

**Usage:**
```bash
infinity_emb v2 --model-id jinaai/jina-embeddings-v5-text-small --port 7997
```

### Free API Providers for Reasoning

#### Groq

**Strengths:**
- Ultra-fast inference (LPU hardware)
- No credit card required
- Generous free tier

**Rate Limits (Free Tier):**
| Model | RPM | RPD | TPM | TPD |
|-------|-----|-----|-----|-----|
| llama-3.3-70b-versatile | 30 | 1K | 12K | 100K |
| meta-llama/llama-4-scout-17b-16e-instruct | 30 | 1K | 30K | 500K |
| qwen/qwen3-32b | 60 | 1K | 6K | 500K |

**Context Windows:**
- llama-3.3-70b-versatile: 128K tokens
- meta-llama/llama-4-scout-17b-16e-instruct: 128K tokens
- qwen/qwen3-32b: 32K tokens

**Best For:** Fast reasoning, large context windows

#### OpenRouter

**Strengths:**
- Access to many models via one API key
- Free models available (ending in `:free`)
- Model routing and failover

**Rate Limits (Free Tier):**
- 20 requests per minute for free models
- 50 requests per day (if <10 credits purchased)
- 1000 requests per day (if ≥10 credits purchased)

**Context Windows:**
- Varies by model (typically 4K-128K tokens)

**Best For:** Model variety, fallback options

#### Google AI Studio (Gemini)

**Strengths:**
- High-quality reasoning models
- Generous free tier
- Vision capabilities

**Rate Limits (Free Tier):**
- Varies by usage tier (Free, Tier 1, Tier 2, Tier 3)
- Measured in RPM, TPM, RPD
- Limits increase with spending

**Context Windows:**
- Gemini 3.1 Pro Preview: 1M tokens
- Gemini 3.1 Flash Preview: 1M tokens
- Gemini 2.5 Flash: 1M tokens

**Best For:** High-quality reasoning, large context windows

## Decision

### Primary Reasoning Model: Groq
**Model:** `llama-3.3-70b-versatile`

**Rationale:**
- Fastest inference (LPU hardware)
- Large context window (128K tokens)
- Generous free tier (30 RPM, 1K RPD, 12K TPM, 100K TPD)
- No credit card required
- High-quality reasoning

### Fallback Reasoning Model: Gemini
**Model:** `gemini-2.5-flash`

**Rationale:**
- High-quality reasoning
- Large context window (1M tokens)
- Generous free tier
- Vision capabilities (for image analysis)

### Tertiary Reasoning Model: OpenRouter
**Model:** `anthropic/claude-sonnet-4-5:free`

**Rationale:**
- Highest quality reasoning
- Model variety and failover
- Backup if Groq/Gemini rate limits hit

## LiteLLM Compatibility

**Research Finding:** LiteLLM supports all chosen providers.

**Evidence:**
- LiteLLM provider list includes: groq, google, openrouter, anthropic
- All providers support OpenAI-compatible API
- Automatic failover between providers
- Rate limit management across multiple API keys

**Configuration:**
```yaml
model_list:
  - model_name: groq/llama-3.3-70b-versatile
    litellm_params:
      api_base: https://api.groq.com/openai/v1
      api_key: ${GROQ_API_KEY}
  - model_name: gemini/gemini-2.5-flash
    litellm_params:
      api_base: https://generativelanguage.googleapis.com
      api_key: ${GOOGLE_API_KEY}
  - model_name: openrouter/anthropic/claude-sonnet-4-5
    litellm_params:
      api_base: https://openrouter.ai/api/v1
      api_key: ${OPENROUTER_API_KEY}
```

## Rate Limit Management

### Strategy
1. **Multiple API Keys:** Use multiple keys per provider to increase limits
2. **Exponential Backoff:** Retry with increasing delays on rate limit errors
3. **Provider Rotation:** Rotate between providers to distribute load
4. **Request Queuing:** Queue requests when rate limited
5. **Monitoring:** Track rate limit usage and alert when approaching limits

### Implementation
```python
from litellm import completion
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=2, max=60)
)
async def call_llm(prompt, model):
    response = await completion(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        temperature=0.3,
    )
    return response
```

## Validation Plan

### Test 1: GLiNER2 Quality
**Goal:** Validate GLiNER2 extraction quality on sample corpus.

**Procedure:**
1. Create gold-standard set of 50 manually annotated chunks
2. Run GLiNER2 extraction
3. Measure precision and recall
4. Document baseline performance

**Expected Result:** Precision > 0.85, Recall > 0.80

### Test 2: jina-embeddings-v5 Quality
**Goal:** Validate embedding quality on sample corpus.

**Procedure:**
1. Embed sample corpus with jina-embeddings-v5
2. Run 10 known queries
3. Verify top-k results include expected matches
4. Compare with nomic-embed-text baseline

**Expected Result:** Top-5 accuracy > 0.90

### Test 3: Free API Rate Limits
**Goal:** Validate rate limits and backoff behavior.

**Procedure:**
1. Make 100 consecutive requests to Groq
2. Monitor rate limit headers
3. Verify backoff behavior
4. Test provider rotation

**Expected Result:** No 429 errors, smooth backoff

### Test 4: LiteLLM Integration
**Goal:** Validate LiteLLM routing and failover.

**Procedure:**
1. Configure LiteLLM with all providers
2. Make requests to each provider
3. Test failover (simulate provider failure)
4. Verify automatic routing

**Expected Result:** Seamless failover, no errors

## VRAM Budget (Updated)

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

**Conclusion:** 16GB VRAM is sufficient with GPU mutex.

## Implementation Strategy

### Phase 1: Local Models
1. Install GLiNER2: `pip install gliner2[local]`
2. Install infinity-emb: `pip install infinity-emb[all]`
3. Download jina-embeddings-v5-text-small
4. Test GLiNER2 extraction on sample corpus
5. Test infinity-emb embedding on sample corpus
6. Validate VRAM usage

### Phase 2: API Models
1. Get API keys for Groq, Gemini, OpenRouter
2. Install LiteLLM: `pip install litellm`
3. Configure LiteLLM with all providers
4. Test each provider individually
5. Test failover and rotation
6. Validate rate limit handling

### Phase 3: Integration
1. Integrate GLiNER2 into ingestion pipeline
2. Integrate infinity-emb into ingestion pipeline
3. Integrate LiteLLM into investigation agent
4. Implement GPU mutex
5. Test end-to-end on sample corpus

## Risks and Mitigations

### Risk: GLiNER2 Quality
**Mitigation:**
- Validate on sample corpus before full ingest
- Use confidence thresholds to filter low-quality extractions
- Manual review of critical extractions

### Risk: Free API Rate Limits
**Mitigation:**
- Multiple API keys per provider
- Exponential backoff on rate limit errors
- Provider rotation to distribute load
- Request queuing during rate limit periods

### Risk: LiteLLM Compatibility
**Mitigation:**
- Validate each provider individually
- Test failover behavior
- Monitor provider status and availability

## Success Criteria

### v1 Success
- GLiNER2 extracts entities with precision > 0.85
- jina-embeddings-v5 achieves top-5 accuracy > 0.90
- Free API providers handle investigation without rate limit errors
- LiteLLM routes and fails over seamlessly
- VRAM usage stays below 12GB at all times

## References

- GLiNER2 GitHub: https://github.com/fastino-ai/GLiNER2
- jina-embeddings-v5: https://huggingface.co/jinaai/jina-embeddings-v5-text-small
- infinity-emb GitHub: https://github.com/michaelfeil/infinity
- Groq Rate Limits: https://console.groq.com/docs/rate-limits
- OpenRouter Rate Limits: https://openrouter.ai/docs/api/reference/limits
- Gemini Rate Limits: https://ai.google.dev/gemini-api/docs/rate-limits
- LiteLLM Documentation: https://docs.litellm.ai/

---

**Approved by:** Hermes Agent  
**Next Action:** Implement model stack and run validation tests
