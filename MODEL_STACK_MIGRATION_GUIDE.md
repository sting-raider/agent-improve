# ORACLE Model Stack Migration Guide

**Version:** 1.0  
**Date:** 2025-05-02  
**Purpose:** Guide for migrating from the old model stack to the new approved model stack (D018)

---

## Executive Summary

This guide provides step-by-step instructions for migrating ORACLE from the old model stack to the new approved model stack. The migration is designed to be non-breaking, with clear rollback procedures.

### Migration Overview

| Component | Old Model | New Model | Migration Complexity |
|-----------|-----------|-----------|----------------------|
| **Embedding** | nomic-embed-text-v1.5 | jina-embeddings-v5-text-small | Medium |
| **Extraction** | qwen3:8b (Ollama) | GLiNER2 (fastino/gliner2-base-v1) | Low |
| **Reasoning** | qwen3:30b (Ollama) | Groq/Gemini/OpenRouter APIs | Low |

---

## Table of Contents

1. [Pre-Migration Checklist](#1-pre-migration-checklist)
2. [Migration Strategy](#2-migration-strategy)
3. [Step-by-Step Migration](#3-step-by-step-migration)
4. [Configuration Changes](#4-configuration-changes)
5. [Testing Procedures](#5-testing-procedures)
6. [Rollback Procedures](#6-rollback-procedures)
7. [Post-Migration Tasks](#7-post-migration-tasks)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Pre-Migration Checklist

### 1.1 System Requirements

- [ ] Python 3.11+ installed
- [ ] CUDA 12.1+ installed (for GPU acceleration)
- [ ] RTX 4080 or equivalent GPU
- [ ] 32GB RAM
- [ ] 500GB+ free disk space
- [ ] Docker and Docker Compose installed

### 1.2 Backup Current State

```bash
# Backup configuration files
cp oracle/config/defaults.yaml oracle/config/defaults.yaml.backup
cp oracle/config/production.yaml oracle/config/production.yaml.backup

# Backup databases
docker exec oracle_neo4j neo4j-admin dump --to=/backup/neo4j-backup
docker exec oracle_qdrant cp -r /qdrant/storage /backup/qdrant-backup

# Backup SQLite checkpoints
cp -r /oracle/data/sqlite /backup/sqlite-backup
```

### 1.3 Verify Current Installation

```bash
# Check current models
python -m oracle.config_cli show | grep -A 10 "models:"

# Check GPU availability
nvidia-smi

# Check services running
docker ps
```

### 1.4 Document Current State

```bash
# Record current configuration
python -m oracle.config_cli show --format yaml > /backup/current-config.yaml

# Record current corpus stats
python -m oracle status --summary > /backup/current-status.txt
```

---

## 2. Migration Strategy

### 2.1 Migration Approach

**Strategy:** Incremental migration with parallel testing

1. **Phase 1:** Update configuration files (non-breaking)
2. **Phase 2:** Install new models (parallel to old)
3. **Phase 3:** Test new models in isolation
4. **Phase 4:** Switch to new models (can be rolled back)
5. **Phase 5:** Remove old models (after validation)

### 2.2 Migration Timeline

| Phase | Duration | Risk Level | Rollback Time |
|-------|----------|------------|---------------|
| Phase 1 | 5 minutes | Low | 1 minute |
| Phase 2 | 10 minutes | Low | 5 minutes |
| Phase 3 | 30 minutes | Medium | 10 minutes |
| Phase 4 | 5 minutes | Low | 1 minute |
| Phase 5 | 10 minutes | Low | N/A |

**Total Migration Time:** ~1 hour  
**Total Rollback Time:** ~17 minutes

### 2.3 Migration Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| New model installation fails | Low | Medium | Keep old models installed |
| Performance degradation | Low | High | Benchmark before migration |
| API rate limits | Medium | Medium | Use multiple API providers |
| Configuration errors | Low | High | Validate configuration before applying |

---

## 3. Step-by-Step Migration

### Phase 1: Update Configuration Files

**Duration:** 5 minutes  
**Risk:** Low  
**Rollback:** 1 minute

#### Step 1.1: Update defaults.yaml

```bash
# Edit the defaults.yaml file
python -m oracle.config_cli edit
```

**Changes to make:**

```yaml
# OLD
models:
  embedding:
    model: "nomic-ai/nomic-embed-text-v1.5"
    dimension: 768
    context_length: 8192
  
  extraction:
    model: "qwen3:8b"
    provider: "ollama"
    device: "cuda"
  
  reasoning:
    primary:
      model: "qwen3:30b"
      provider: "ollama"
      device: "cuda"

# NEW
models:
  embedding:
    model: "jinaai/jina-embeddings-v5-text-small"
    dimension: 1024
    context_length: 32768
  
  extraction:
    model: "fastino/gliner2-base-v1"
    provider: "local"
    device: "cpu"
  
  reasoning:
    primary:
      model: "llama-3.3-70b-versatile"
      provider: "groq"
      device: "api"
    fallback:
      model: "gemini-2.5-flash"
      provider: "google"
      device: "api"
    tertiary:
      model: "claude-sonnet-4.5:free"
      provider: "openrouter"
      device: "api"
```

#### Step 1.2: Validate Configuration

```bash
# Validate the new configuration
python -m oracle.config_cli validate
```

**Expected output:** `Configuration is valid ✅`

#### Step 1.3: Review Changes

```bash
# Show the diff between old and new configuration
python -m oracle.config_cli diff
```

**Rollback if needed:**

```bash
# Restore old configuration
cp oracle/config/defaults.yaml.backup oracle/config/defaults.yaml
```

---

### Phase 2: Install New Models

**Duration:** 10 minutes  
**Risk:** Low  
**Rollback:** 5 minutes

#### Step 2.1: Install GLiNER2

```bash
# GLiNER2 is CPU-first, no GPU needed
pip install gliner

# Verify installation
python -c "import gliner; print('GLiNER2 installed successfully')"
```

#### Step 2.2: Update infinity-emb Model

```bash
# Stop infinity-emb server
pkill -f infinity_emb

# Pull new model
infinity_emb v2 --model-id jinaai/jina-embeddings-v5-text-small --port 7997 --device cuda &

# Verify installation
curl http://localhost:7997/health
```

#### Step 2.3: Configure API Keys

```bash
# Add API keys to environment
export GROQ_API_KEY="your-groq-api-key"
export GOOGLE_API_KEY="your-google-api-key"
export OPENROUTER_API_KEY="your-openrouter-api-key"

# Or add to .env file
echo "GROQ_API_KEY=your-groq-api-key" >> .env
echo "GOOGLE_API_KEY=your-google-api-key" >> .env
echo "OPENROUTER_API_KEY=your-openrouter-api-key" >> .env
```

#### Step 2.4: Test API Connectivity

```bash
# Test Groq API
curl -X POST https://api.groq.com/openai/v1/models \
  -H "Authorization: Bearer $GROQ_API_KEY"

# Test Google API
curl -X POST https://generativelanguage.googleapis.com/v1beta/models \
  -H "x-goog-api-key: $GOOGLE_API_KEY"

# Test OpenRouter API
curl -X POST https://openrouter.ai/api/v1/models \
  -H "Authorization: Bearer $OPENROUTER_API_KEY"
```

**Rollback if needed:**

```bash
# Uninstall GLiNER2
pip uninstall gliner -y

# Restore old infinity-emb model
pkill -f infinity_emb
infinity_emb v2 --model-id nomic-ai/nomic-embed-text-v1.5 --port 7997 --device cuda &
```

---

### Phase 3: Test New Models in Isolation

**Duration:** 30 minutes  
**Risk:** Medium  
**Rollback:** 10 minutes

#### Step 3.1: Test Embedding Model

```python
# test_embedding.py
from infinity_emb import InfinityEmb
import asyncio

async def test_embedding():
    # Initialize with new model
    infinity = InfinityEmb(
        model_id="jinaai/jina-embeddings-v5-text-small",
        device="cuda"
    )
    
    # Test embedding
    texts = ["This is a test sentence.", "Another test sentence."]
    embeddings = await infinity.embed(texts)
    
    # Verify output
    assert len(embeddings) == 2
    assert len(embeddings[0]) == 1024  # New dimension
    print(f"✅ Embedding test passed: {len(embeddings[0])} dimensions")
    
    # Test with long context (32K tokens)
    long_text = " ".join(["test"] * 40000)  # ~32K tokens
    long_embedding = await infinity.embed([long_text])
    print(f"✅ Long context test passed")

asyncio.run(test_embedding())
```

```bash
# Run test
python test_embedding.py
```

#### Step 3.2: Test Extraction Model

```python
# test_extraction.py
from gliner import GLiNER
import json

def test_extraction():
    # Initialize GLiNER2
    model = GLiNER.from_pretrained("fastino/gliner2-base-v1")
    
    # Test extraction
    text = "Apple Inc. was founded by Steve Jobs in Cupertino, California in 1976."
    labels = ["PERSON", "ORGANIZATION", "LOCATION", "DATE"]
    
    entities = model.predict_entities(text, labels)
    
    # Verify output
    assert len(entities) > 0
    print(f"✅ Extraction test passed: {len(entities)} entities found")
    print(f"Entities: {json.dumps(entities, indent=2)}")

test_extraction()
```

```bash
# Run test
python test_extraction.py
```

#### Step 3.3: Test Reasoning Models

```python
# test_reasoning.py
import os
from openai import OpenAI

def test_groq():
    client = OpenAI(
        base_url="https://api.groq.com/openai/v1",
        api_key=os.environ["GROQ_API_KEY"]
    )
    
    response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[{"role": "user", "content": "What is 2+2?"}],
        max_tokens=10
    )
    
    assert response.choices[0].message.content
    print(f"✅ Groq test passed: {response.choices[0].message.content}")

def test_gemini():
    import google.generativeai as genai
    
    genai.configure(api_key=os.environ["GOOGLE_API_KEY"])
    model = genai.GenerativeModel("gemini-2.5-flash")
    
    response = model.generate_content("What is 2+2?")
    
    assert response.text
    print(f"✅ Gemini test passed: {response.text}")

def test_openrouter():
    client = OpenAI(
        base_url="https://openrouter.ai/api/v1",
        api_key=os.environ["OPENROUTER_API_KEY"]
    )
    
    response = client.chat.completions.create(
        model="anthropic/claude-sonnet-4.5:free",
        messages=[{"role": "user", "content": "What is 2+2?"}],
        max_tokens=10
    )
    
    assert response.choices[0].message.content
    print(f"✅ OpenRouter test passed: {response.choices[0].message.content}")

test_groq()
test_gemini()
test_openrouter()
```

```bash
# Run tests
python test_reasoning.py
```

#### Step 3.4: Benchmark Performance

```python
# benchmark.py
import time
from infinity_emb import InfinityEmb
from gliner import GLiNER
import asyncio

async def benchmark():
    # Benchmark embedding
    infinity = InfinityEmb(
        model_id="jinaai/jina-embeddings-v5-text-small",
        device="cuda"
    )
    
    texts = ["test"] * 100
    start = time.time()
    embeddings = await infinity.embed(texts)
    duration = time.time() - start
    
    print(f"✅ Embedding benchmark: {len(texts)/duration:.2f} texts/sec")
    
    # Benchmark extraction
    model = GLiNER.from_pretrained("fastino/gliner2-base-v1")
    text = "Apple Inc. was founded by Steve Jobs in Cupertino, California in 1976."
    labels = ["PERSON", "ORGANIZATION", "LOCATION", "DATE"]
    
    start = time.time()
    for _ in range(10):
        entities = model.predict_entities(text, labels)
    duration = time.time() - start
    
    print(f"✅ Extraction benchmark: {10/duration:.2f} extractions/sec")

asyncio.run(benchmark())
```

```bash
# Run benchmark
python benchmark.py
```

**Rollback if needed:**

```bash
# Restore old models
pkill -f infinity_emb
infinity_emb v2 --model-id nomic-ai/nomic-embed-text-v1.5 --port 7997 --device cuda &

# Remove test files
rm test_embedding.py test_extraction.py test_reasoning.py benchmark.py
```

---

### Phase 4: Switch to New Models

**Duration:** 5 minutes  
**Risk:** Low  
**Rollback:** 1 minute

#### Step 4.1: Restart Services

```bash
# Stop all services
docker compose down

# Start services with new configuration
docker compose up -d

# Wait for services to be healthy
sleep 30
```

#### Step 4.2: Verify Services

```bash
# Check Neo4j
docker exec oracle_neo4j cypher-shell -u neo4j -p "$NEO4J_PASSWORD" "RETURN 1"

# Check Qdrant
curl http://localhost:6333/healthz

# Check infinity-emb
curl http://localhost:7997/health
```

#### Step 4.3: Run Smoke Test

```bash
# Run smoke test corpus
python -m oracle ingest /path/to/smoke-test-corpus --test

# Verify results
python -m oracle status --summary
```

**Rollback if needed:**

```bash
# Stop services
docker compose down

# Restore old configuration
cp oracle/config/defaults.yaml.backup oracle/config/defaults.yaml

# Restart services
docker compose up -d
```

---

### Phase 5: Remove Old Models

**Duration:** 10 minutes  
**Risk:** Low  
**Rollback:** N/A

#### Step 5.1: Remove Old Ollama Models

```bash
# Remove old models from Ollama
ollama rm qwen3:8b
ollama rm qwen3:30b
ollama rm nomic-embed-text

# Optional: Remove Ollama entirely if no longer needed
# ollama uninstall
```

#### Step 5.2: Clean Up Old Files

```bash
# Remove backup files (after validation)
rm oracle/config/defaults.yaml.backup
rm oracle/config/production.yaml.backup

# Remove test files
rm test_embedding.py test_extraction.py test_reasoning.py benchmark.py
```

#### Step 5.3: Update Documentation

```bash
# Update all documentation references
# (This is done automatically by the migration script)
```

---

## 4. Configuration Changes

### 4.1 Complete Configuration File

```yaml
# oracle/config/defaults.yaml

# Model Configuration
models:
  # Embedding Model
  embedding:
    model: "jinaai/jina-embeddings-v5-text-small"
    dimension: 1024
    context_length: 32768
    provider: "infinity-emb"
    device: "cuda"
    batch_size: 32
  
  # Extraction Model
  extraction:
    model: "fastino/gliner2-base-v1"
    provider: "local"
    device: "cpu"
    batch_size: 16
  
  # Reasoning Models
  reasoning:
    primary:
      model: "llama-3.3-70b-versatile"
      provider: "groq"
      device: "api"
      max_tokens: 8192
      temperature: 0.3
    fallback:
      model: "gemini-2.5-flash"
      provider: "google"
      device: "api"
      max_tokens: 8192
      temperature: 0.3
    tertiary:
      model: "claude-sonnet-4.5:free"
      provider: "openrouter"
      device: "api"
      max_tokens: 8192
      temperature: 0.3

# GPU Configuration
gpu:
  mutex_timeout: 300
  priority_levels:
    INGESTION_EMBEDDING: 10
    INVESTIGATION_REASONING: 7
    VISUALIZATION: 5

# API Configuration
api_keys:
  groq: "${GROQ_API_KEY}"
  google: "${GOOGLE_API_KEY}"
  openrouter: "${OPENROUTER_API_KEY}"

# Performance Configuration
performance:
  embedding:
    batch_size: 32
    max_queue_size: 500
    timeout: 120
  
  extraction:
    batch_size: 16
    max_queue_size: 500
    timeout: 60
  
  reasoning:
    max_concurrent_requests: 4
    timeout: 300
    retry_attempts: 3
```

### 4.2 Environment Variables

```bash
# .env

# API Keys
GROQ_API_KEY=your-groq-api-key
GOOGLE_API_KEY=your-google-api-key
OPENROUTER_API_KEY=your-openrouter-api-key

# Database Credentials
NEO4J_URI=bolt://localhost:7687
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your-neo4j-password

QDRANT_HOST=localhost
QDRANT_PORT=6333

# Service Configuration
INFINITY_EMB_HOST=localhost
INFINITY_EMB_PORT=7997
```

---

## 5. Testing Procedures

### 5.1 Unit Tests

```bash
# Run all unit tests
pytest tests/unit/test_models.py -v

# Expected output: All tests pass ✅
```

### 5.2 Integration Tests

```bash
# Run integration tests
pytest tests/integration/test_ingestion.py -v

# Expected output: All tests pass ✅
```

### 5.3 End-to-End Tests

```bash
# Run end-to-end test
python -m oracle test --smoke

# Expected output: All tests pass ✅
```

### 5.4 Performance Tests

```bash
# Run performance benchmarks
python -m oracle benchmark --full

# Expected output: Performance meets or exceeds baseline ✅
```

---

## 6. Rollback Procedures

### 6.1 Immediate Rollback

**When to use:** Critical failure during migration

```bash
# Stop all services
docker compose down

# Restore old configuration
cp oracle/config/defaults.yaml.backup oracle/config/defaults.yaml
cp oracle/config/production.yaml.backup oracle/config/production.yaml

# Restore old models
pkill -f infinity_emb
infinity_emb v2 --model-id nomic-ai/nomic-embed-text-v1.5 --port 7997 --device cuda &

# Restart services
docker compose up -d

# Verify rollback
python -m oracle status --summary
```

### 6.2 Partial Rollback

**When to use:** Only one component fails

```bash
# Example: Rollback only embedding model
pkill -f infinity_emb
infinity_emb v2 --model-id nomic-ai/nomic-embed-text-v1.5 --port 7997 --device cuda &

# Update configuration to use old model
python -m oracle.config_cli edit
# Change embedding.model back to "nomic-ai/nomic-embed-text-v1.5"
```

### 6.3 Full System Restore

**When to use:** Complete system failure

```bash
# Restore databases
docker exec oracle_neo4j neo4j-admin load --from=/backup/neo4j-backup
docker exec oracle_qdrant cp -r /backup/qdrant-backup /qdrant/storage
cp -r /backup/sqlite-backup/* /oracle/data/sqlite/

# Restore configuration
cp /backup/current-config.yaml oracle/config/defaults.yaml

# Restart services
docker compose down
docker compose up -d

# Verify restore
python -m oracle status --summary
```

---

## 7. Post-Migration Tasks

### 7.1 Validation

```bash
# Validate all components
python -m oracle validate --full

# Expected output: All components valid ✅
```

### 7.2 Performance Monitoring

```bash
# Monitor performance for 24 hours
python -m oracle monitor --duration 24h

# Check for any performance degradation
python -m oracle benchmark --compare
```

### 7.3 Update Documentation

```bash
# Update all documentation with new model stack
# (This is done automatically by the migration script)
```

### 7.4 Update Team

```bash
# Notify team of successful migration
# Send email with migration summary and performance metrics
```

---

## 8. Troubleshooting

### 8.1 Common Issues

#### Issue: GLiNER2 Installation Fails

**Symptoms:** `pip install gliner` fails

**Solution:**
```bash
# Try installing with specific version
pip install gliner==0.2.6

# Or install from source
git clone https://github.com/urchade/GLiNER.git
cd GLiNER
pip install -e .
```

#### Issue: infinity-emb Model Pull Fails

**Symptoms:** `infinity_emb v2` fails to pull model

**Solution:**
```bash
# Check internet connection
ping huggingface.co

# Try manual pull
huggingface-cli download jinaai/jina-embeddings-v5-text-small

# Or use different mirror
export HF_ENDPOINT=https://hf-mirror.com
infinity_emb v2 --model-id jinaai/jina-embeddings-v5-text-small --port 7997 --device cuda &
```

#### Issue: API Rate Limits

**Symptoms:** API calls fail with rate limit errors

**Solution:**
```bash
# Configure multiple API providers
# Update configuration to use fallback providers

# Implement rate limiting
python -m oracle config set reasoning.rate_limit 10
```

#### Issue: VRAM Exhaustion

**Symptoms:** CUDA out of memory errors

**Solution:**
```bash
# Reduce batch sizes
python -m oracle config set embedding.batch_size 16
python -m oracle config set extraction.batch_size 8

# Or use CPU for extraction
python -m oracle config set extraction.device cpu
```

### 8.2 Getting Help

If you encounter issues not covered in this guide:

1. Check the logs: `docker logs oracle_neo4j`, `docker logs oracle_qdrant`
2. Check the documentation: `~/Documents/hermes-projects/oracle/docs/`
3. Check the decision log: `DECISION_D018_MODEL_STACK.md`
4. Open an issue on GitHub

---

## Migration Checklist

### Pre-Migration

- [ ] Backup configuration files
- [ ] Backup databases
- [ ] Document current state
- [ ] Verify system requirements
- [ ] Review migration strategy

### Migration

- [ ] Update configuration files
- [ ] Validate configuration
- [ ] Install GLiNER2
- [ ] Update infinity-emb model
- [ ] Configure API keys
- [ ] Test embedding model
- [ ] Test extraction model
- [ ] Test reasoning models
- [ ] Benchmark performance
- [ ] Restart services
- [ ] Verify services
- [ ] Run smoke test
- [ ] Remove old models
- [ ] Clean up old files

### Post-Migration

- [ ] Validate all components
- [ ] Monitor performance
- [ ] Update documentation
- [ ] Notify team
- [ ] Document lessons learned

---

## Success Criteria

Migration is considered successful when:

- ✅ All configuration files updated without errors
- ✅ All new models installed and tested
- ✅ All services running and healthy
- ✅ Smoke test passes
- ✅ Performance meets or exceeds baseline
- ✅ No errors in logs for 24 hours
- ✅ Team notified and documentation updated

---

## Appendix A: Migration Script

```bash
#!/bin/bash
# migrate.sh - Automated migration script

set -euo pipefail

echo "=== ORACLE Model Stack Migration ==="

# Pre-migration checks
echo "Step 1: Pre-migration checks..."
bash pre_migration_checks.sh

# Backup
echo "Step 2: Backing up current state..."
bash backup.sh

# Update configuration
echo "Step 3: Updating configuration..."
python -m oracle.config_cli edit

# Install new models
echo "Step 4: Installing new models..."
bash install_models.sh

# Test new models
echo "Step 5: Testing new models..."
bash test_models.sh

# Switch to new models
echo "Step 6: Switching to new models..."
bash switch_models.sh

# Validate
echo "Step 7: Validating migration..."
python -m oracle validate --full

# Cleanup
echo "Step 8: Cleaning up..."
bash cleanup.sh

echo "=== Migration Complete ==="
```

---

## Appendix B: Rollback Script

```bash
#!/bin/bash
# rollback.sh - Automated rollback script

set -euo pipefail

echo "=== ORACLE Model Stack Rollback ==="

# Stop services
echo "Step 1: Stopping services..."
docker compose down

# Restore configuration
echo "Step 2: Restoring configuration..."
cp oracle/config/defaults.yaml.backup oracle/config/defaults.yaml
cp oracle/config/production.yaml.backup oracle/config/production.yaml

# Restore models
echo "Step 3: Restoring models..."
bash restore_models.sh

# Restart services
echo "Step 4: Restarting services..."
docker compose up -d

# Verify
echo "Step 5: Verifying rollback..."
python -m oracle status --summary

echo "=== Rollback Complete ==="
```

---

**Document Version:** 1.0  
**Last Updated:** 2025-05-02  
**Status:** READY FOR USE ✅
