# ORACLE Quick Start Guide

**Last Updated:** 2026-05-01  
**Prerequisites:** RTX 4080, 32GB RAM, 500GB+ SSD, Linux

---

## Overview

This guide will help you get ORACLE up and running quickly. For detailed documentation, see the [[ORACLE Project Overview]] and related notes.

---

## Prerequisites

### Hardware

- GPU: NVIDIA RTX 4080 (16GB VRAM) or better
- CPU: 8+ cores, 16+ threads
- RAM: 32GB DDR5 or better
- Storage: 500GB+ free SSD space
- OS: Linux (Ubuntu 22.04/24.04 or Arch recommended)

### Software

- Docker Engine 24+
- Docker Compose
- Python 3.11+
- CUDA 12.1+ (with driver 545+)
- Git

---

## Installation

### Step 1: Clone Repository

```bash
git clone <repository-url> oracle
cd oracle
```

### Step 2: Install System Dependencies

```bash
# Install Docker (if not already installed)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install uv (fast Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh
```

### Step 3: Configure Environment

```bash
# Copy environment template
cp .env.example .env

# Edit .env with your configuration
nano .env
```

**Required settings in .env:**

```bash
# Directories
ORACLE_ROOT=/oracle
ORACLE_DATA_DIR=/oracle/data
ORACLE_CORPUS_DIR=/oracle/corpus

# Neo4j
NEO4J_PASSWORD=your_strong_password_here

# API Keys (optional but recommended)
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=...
```

### Step 4: Install Python Dependencies

```bash
# Create virtual environment and install dependencies
uv sync

# Pull Ollama models
ollama pull qwen3:8b
ollama pull nomic-embed-text
```

### Step 5: Start Services

```bash
# Start Neo4j and Qdrant
docker compose up -d

# Wait for services to be healthy (30-60 seconds)
docker compose ps
```

### Step 6: Initialize Databases

```bash
# Initialize Neo4j schema
uv run python -c "from oracle.graph.neo4j_client import initialize_schema; initialize_schema()"

# Initialize Qdrant collection
uv run python -c "from oracle.graph.qdrant_client import initialize_collection; initialize_collection()"

# Start infinity-emb
uv run infinity_emb v2 --model-id nomic-ai/nomic-embed-text-v1.5 --port 7997 --device cuda &
```

### Step 7: Verify Installation

```bash
# Check Neo4j
curl http://localhost:7474

# Check Qdrant
curl http://localhost:6333/healthz

# Check infinity-emb
curl http://localhost:7997/health

# Check GPU
nvidia-smi
```

---

## First Investigation

### Step 1: Add Corpus

```bash
# Create corpus directory
mkdir -p /oracle/corpus

# Copy your documents
cp -r /path/to/your/documents/* /oracle/corpus/

# Start ingestion
oracle ingest /oracle/corpus
```

**Note:** Ingestion can take hours to days depending on corpus size. Monitor progress in the TUI.

### Step 2: Start Investigation

```bash
# Launch TUI
oracle tui

# In TUI, enter:
new investigation What are the key themes in this corpus?
```

### Step 3: Monitor Progress

The TUI will show:
- Investigation status
- Active agents
- Findings so far
- Corpus statistics

### Step 4: View Results

When investigation is complete:

```bash
# Generate report
oracle report <investigation_id> --output ~/reports/my_report.md

# View knowledge graph
oracle visualize <investigation_id>
```

---

## Common Commands

### TUI Commands

```bash
# Launch TUI
oracle tui

# In TUI command input:
continue                    # Resume paused investigation
pause                       # Pause after current step
status                      # Show investigation status
add corpus <path>           # Add documents to corpus
show graph [name]           # Open graph visualization
new investigation <q>       # Start new investigation
hypothesis list             # List all hypotheses
report draft                # Generate partial report
journal                     # Open investigation journal
help                        # Show help
```

### CLI Commands

```bash
# Start investigation
oracle investigate "your question here" --model claude

# Resume investigation
oracle resume <investigation_id>

# Add corpus
oracle ingest /path/to/documents

# Check status
oracle status <investigation_id>

# Visualize
oracle visualize <investigation_id>

# Generate report
oracle report <investigation_id> --output ~/reports/report.md
```

---

## Troubleshooting

### Services Won't Start

**Problem:** Docker containers fail to start.

**Solution:**
```bash
# Check Docker logs
docker compose logs neo4j
docker compose logs qdrant

# Check port conflicts
sudo lsof -i :7474
sudo lsof -i :6333

# Restart services
docker compose down
docker compose up -d
```

### GPU Not Accessible

**Problem:** Python can't access GPU.

**Solution:**
```bash
# Check CUDA installation
nvidia-smi

# Check PyTorch CUDA
uv run python -c "import torch; print(torch.cuda.is_available())"

# Reinstall with CUDA support
uv pip install torch --index-url https://download.pytorch.org/whl/cu121
```

### Out of Memory

**Problem:** Services crash due to memory exhaustion.

**Solution:**
```bash
# Check RAM usage
free -h

# Check VRAM usage
nvidia-smi

# Reduce batch sizes in .env
INGESTION_BATCH_SIZE=16
INGESTION_WORKERS=2

# Restart services
docker compose restart
```

### Slow Ingestion

**Problem:** Ingestion is slower than expected.

**Solution:**
```bash
# Check GPU utilization
nvidia-smi dmon -s u

# Check CPU utilization
htop

# Verify infinity-emb is running
curl http://localhost:7997/health

# Increase workers if CPU is underutilized
INGESTION_WORKERS=8
```

---

## Configuration

### Environment Variables

Key settings in `.env`:

```bash
# GPU Management
GPU_MUTEX_TIMEOUT=300

# Ingestion Performance
INGESTION_BATCH_SIZE=32
INGESTION_WORKERS=4
CHUNK_SIZE=512
CHUNK_OVERLAP=64

# Model Selection
DEFAULT_REASONING_MODEL=claude
OLLAMA_EXTRACTION_MODEL=qwen3:8b

# Visualization
VIZ_SERVER_PORT=8765
VIZ_MAX_NODES=3000
```

### Performance Tuning

For faster ingestion:
```bash
INGESTION_WORKERS=8
INGESTION_BATCH_SIZE=64
```

For better quality:
```bash
EMBEDDING_MODEL=BAAI/bge-large-en-v1.5
```

For local-only operation:
```bash
DEFAULT_REASONING_MODEL=ollama
OLLAMA_REASONING_MODEL=qwen3:8b
```

---

## Next Steps

1. **Read the Documentation**
   - [[ORACLE Project Overview]]
   - [[ORACLE Architecture]]
   - [[ORACLE Technology Stack]]

2. **Review the Roadmap**
   - [[ORACLE Implementation Roadmap]]

3. **Understand the Decisions**
   - [[ORACLE Decision Log]]

4. **Assess the Risks**
   - [[ORACLE Risk Assessment]]

5. **Check Performance**
   - [[ORACLE Performance Benchmarks]]

---

## Getting Help

### Documentation

- All documentation is in the Obsidian vault
- Use the `help` command in the TUI
- Check inline code comments

### Community

- GitHub Issues: Report bugs and request features
- Discord: Join the community for discussion
- Email: Support contact (if available)

### Debugging

Enable debug logging:
```bash
LOG_LEVEL=DEBUG
oracle tui
```

Check logs:
```bash
# Service logs
docker compose logs -f neo4j
docker compose logs -f qdrant

# Application logs
tail -f /oracle/logs/oracle.log
```

---

## Tips and Best Practices

### Ingestion

1. **Start Small:** Test with a small corpus first (10-100 documents)
2. **Monitor Progress:** Watch the TUI for ingestion status
3. **Check Storage:** Ensure you have enough disk space
4. **Use SSD:** Ingestion is much faster on SSD
5. **Run Overnight:** Large corpora take time, run overnight

### Investigation

1. **Be Specific:** Clear questions get better results
2. **Start Simple:** Begin with simple questions, then complex
3. **Monitor Progress:** Watch the TUI for investigation status
4. **Review Journal:** Check the investigation journal regularly
5. **Visualize:** Use graph visualization to understand findings

### Performance

1. **Use API Models:** For best reasoning quality
2. **Time-Slice GPU:** Don't run GPU-heavy tasks simultaneously
3. **Monitor Resources:** Keep an eye on GPU, CPU, RAM usage
4. **Optimize Queries:** Use filtered search and named graphs
5. **Limit Visualization:** Keep graphs under 50K nodes

---

## Uninstallation

To completely remove ORACLE:

```bash
# Stop services
docker compose down

# Remove data (WARNING: This deletes all data!)
rm -rf /oracle

# Remove Python environment
rm -rf .venv

# Remove Ollama models
ollama rm qwen3:8b
ollama rm nomic-embed-text
```

---

**Tags:** #oracle #quick-start #installation #setup

**Links:**
- [[ORACLE Project Overview]]
- [[ORACLE Architecture]]
- [[ORACLE Implementation Roadmap]]
