# ORACLE Configuration Management

This document describes the ORACLE configuration management system, which provides a flexible and powerful way to manage configuration across different environments, machines, and deployment scenarios.

## Overview

ORACLE uses a hierarchical configuration system that allows you to easily manage settings across different environments (dev, staging, production) and machines (laptop, desktop, server, workstation).

## Configuration Hierarchy

Configuration is loaded from multiple sources, with higher priority sources overriding lower priority ones:

1. **User Config** (`~/.oracle/config.yaml`) — Highest priority
2. **Machine Config** (`/etc/oracle/config.yaml`) — System-wide settings
3. **Environment Config** (`oracle/config/{env}.yaml`) — Environment-specific settings
4. **Default Config** (`oracle/config/defaults.yaml`) — Base configuration

All configs are merged with higher priority sources overriding lower priority ones.

## Configuration Files

### Default Configuration

**Location:** `oracle/config/defaults.yaml`

This file contains the base configuration for ORACLE. These values are used as defaults and can be overridden by other config files.

### Environment Configurations

**Locations:**
- `oracle/config/dev.yaml` — Development environment
- `oracle/config/staging.yaml` — Staging environment
- `oracle/config/production.yaml` — Production environment

These files contain environment-specific settings. The active environment is determined by the `ORACLE_ENV` environment variable (defaults to `dev`).

### Machine Configuration

**Location:** `/etc/oracle/config.yaml`

This file contains machine-specific settings that apply to all users on the system. This is useful for server deployments where multiple users share the same ORACLE installation.

### User Configuration

**Location:** `~/.oracle/config.yaml`

This file contains user-specific settings that override all other configurations. This is where you should make personal customizations.

## Configuration Structure

The configuration is organized into the following sections:

### Environment

```yaml
environment: dev  # dev|staging|production
machine_type: laptop  # laptop|desktop|server|workstation
```

### Directories

```yaml
directories:
  root: /oracle
  data: /oracle/data
  corpus: /oracle/corpus
  investigations: /oracle/investigations
  logs: /oracle/logs
  config: /oracle/config
  migrations: /oracle/migrations
  test_corpus: /oracle/test_corpus
  test_corpus_ground_truth: /oracle/test_corpus_ground_truth
```

### Services

```yaml
neo4j:
  uri: bolt://localhost:7687
  username: neo4j
  password: changeme_strong_password
  database: neo4j
  max_pool_size: 50
  connection_timeout: 30
  max_transaction_retry_time: 30

qdrant:
  host: localhost
  port: 6333
  grpc_port: 6334
  collection_name: oracle_corpus
  timeout: 60
  prefer_grpc: false
```

### Models

```yaml
embedding:
  host: localhost
  port: 7997
  model: jinaai/jina-embeddings-v5-text-small
  dimension: 1024
  batch_size: 32
  timeout: 120

entity_extraction:
  model: fastino/gliner2-base-v1
  device: cuda  # or "cpu"
  batch_size: 8
  confidence_threshold: 0.5
  include_spans: true
  max_entities_per_chunk: 50
```

### API Providers

```yaml
groq:
  api_key: ""
  model: llama-3.3-70b-versatile
  max_tokens: 8192
  temperature: 0.3
  timeout: 60

gemini:
  api_key: ""
  model: gemini-2.5-flash
  max_tokens: 8192
  temperature: 0.3
  timeout: 60

openrouter:
  api_key: ""
  model: anthropic/claude-sonnet-4-5:free
  max_tokens: 8192
  temperature: 0.3
  timeout: 60

model_routing:
  primary_provider: groq  # groq|gemini|openrouter
  fallback_provider: gemini
  tertiary_provider: openrouter
  max_retries: 3
  retry_delay: 1.0
  exponential_backoff: true
```

### System

```yaml
ingestion:
  workers: 4
  batch_size: 32
  chunk_size: 512
  chunk_overlap: 64
  max_queue_size: 500
  parse_timeout: 300
  embed_timeout: 120
  extract_timeout: 60

gpu:
  device: cuda  # or "cpu"
  mutex_timeout: 300
  max_vram_usage_gb: 12.0
  enable_flash_attention: true
  enable_quantization: true
  quantization_bits: 4

visualization:
  server_port: 8765
  max_nodes: 3000
  auto_open_browser: true
  bloom_strength: 0.8
  bloom_radius: 0.6
  bloom_threshold: 0.1

reporting:
  min_confidence: 0.6
  citation_style: inline  # inline|footnote|endnote
  include_appendix: true
  max_report_size_mb: 100.0

logging:
  level: INFO  # DEBUG|INFO|WARNING|ERROR
  format: json  # json|console
  file: true
  console: true
  max_file_size_mb: 100.0
  backup_count: 5

testing:
  enabled: true
  coverage_target: 0.8
  parallel_workers: 4
  timeout: 300
```

## Using the Configuration CLI

ORACLE provides a command-line interface for managing configuration:

### Show Configuration

```bash
# Show current configuration (pretty format)
python -m oracle.config_cli show

# Show as YAML
python -m oracle.config_cli show --format yaml

# Show as JSON
python -m oracle.config_cli show --format json
```

### Validate Configuration

```bash
# Validate current configuration
python -m oracle.config_cli validate
```

### Edit Configuration

```bash
# Edit user configuration
python -m oracle.config_cli edit

# Edit specific config file
python -m oracle.config_cli edit --config /path/to/config.yaml
```

### Switch Environment

```bash
# Switch to production environment
python -m oracle.config_cli switch production

# Switch to staging environment
python -m oracle.config_cli switch staging

# Switch to dev environment
python -m oracle.config_cli switch dev
```

### Initialize Configuration

```bash
# Initialize configuration (creates user config)
python -m oracle.config_cli init

# Force overwrite existing config
python -m oracle.config_cli init --force
```

### Compare Configurations

```bash
# Compare default configs
python -m oracle.config_cli diff

# Compare specific config files
python -m oracle.config_cli diff --config1 /path/to/config1.yaml --config2 /path/to/config2.yaml
```

### Export Configuration

```bash
# Export current configuration to file
python -m oracle.config_cli export my_config.yaml
```

### Import Configuration

```bash
# Import configuration from file
python -m oracle.config_cli import my_config.yaml
```

## Programmatic Usage

### Loading Configuration

```python
from oracle.config import get_config

# Get current configuration
config = get_config()

# Access configuration values
print(config.environment.value)
print(config.neo4j.uri)
print(config.embedding.model)
```

### Reloading Configuration

```python
from oracle.config import reload_config

# Reload configuration (e.g., after changing config file)
config = reload_config()
```

### Saving Configuration

```python
from oracle.config import save_config, OracleConfig

# Save configuration to file
save_config(config, "/path/to/config.yaml")
```

### Validating Configuration

```python
from oracle.config import ConfigManager

# Validate configuration
config_manager = ConfigManager()
if config_manager.validate():
    print("Configuration is valid")
else:
    print("Configuration is invalid")
```

## Environment Variables

The following environment variables can be set to control ORACLE behavior:

- `ORACLE_ENV` — Active environment (dev|staging|production, default: dev)
- `ORACLE_CONFIG` — Path to explicit config file (overrides all other sources)
- `NEO4J_PASSWORD` — Neo4j password (can be used instead of config file)
- `GROQ_API_KEY` — Groq API key
- `GOOGLE_API_KEY` — Gemini API key
- `OPENROUTER_API_KEY` — OpenRouter API key

## Best Practices

### 1. Use Environment-Specific Configs

Always use environment-specific configs for different deployment scenarios:

- **Dev:** Use `dev.yaml` for local development
- **Staging:** Use `staging.yaml` for testing environments
- **Production:** Use `production.yaml` for production deployments

### 2. Never Commit Secrets

Never commit API keys or passwords to version control. Use environment variables or user config files (which should be in `.gitignore`).

### 3. Validate Before Deploying

Always validate your configuration before deploying:

```bash
python -m oracle.config_cli validate
```

### 4. Use Machine Config for Shared Settings

Use `/etc/oracle/config.yaml` for settings that should be shared across all users on a machine (e.g., service URLs, database hosts).

### 5. Use User Config for Personal Settings

Use `~/.oracle/config.yaml` for personal settings (e.g., API keys, personal preferences).

### 6. Document Your Config Changes

When you make configuration changes, document them in your project's README or changelog.

## Troubleshooting

### Configuration Not Loading

If configuration is not loading as expected:

1. Check the environment variable: `echo $ORACLE_ENV`
2. Check the config file exists: `ls ~/.oracle/config.yaml`
3. Validate the config: `python -m oracle.config_cli validate`
4. Check for syntax errors: `python -m oracle.config_cli show --format yaml`

### API Keys Not Working

If API keys are not working:

1. Check the config file: `python -m oracle.config_cli show`
2. Verify the key is set correctly (not truncated)
3. Check environment variables: `echo $GROQ_API_KEY`
4. Test the API key manually

### Services Not Connecting

If services are not connecting:

1. Check the service URLs in the config: `python -m oracle.config_cli show`
2. Verify the services are running: `docker ps` or `systemctl status`
3. Check network connectivity: `ping <host>`
4. Check firewall rules

## Migration Guide

### Migrating from Old Config

If you're migrating from an old configuration system:

1. Export your old config: `python -m oracle.config_cli export old_config.yaml`
2. Review the exported config
3. Update it to match the new structure
4. Import it: `python -m oracle.config_cli import old_config.yaml`
5. Validate: `python -m oracle.config_cli validate`

### Switching Machines

When switching machines:

1. Export your config: `python -m oracle.config_cli export my_config.yaml`
2. Copy the config file to the new machine
3. Import it: `python -m oracle.config_cli import my_config.yaml`
4. Update machine-specific settings (e.g., service URLs)
5. Validate: `python -m oracle.config_cli validate`

## Examples

### Example 1: Local Development

```bash
# Set environment
export ORACLE_ENV=dev

# Initialize config
python -m oracle.config_cli init

# Edit config
python -m oracle.config_cli edit

# Validate
python -m oracle.config_cli validate

# Show config
python -m oracle.config_cli show
```

### Example 2: Server Deployment

```bash
# Set environment
export ORACLE_ENV=production

# Create machine config
sudo mkdir -p /etc/oracle
sudo tee /etc/oracle/config.yaml > /dev/null <<EOF
environment: production
machine_type: server
neo4j:
  uri: bolt://neo4j.internal:7687
  password: ${NEO4J_PASSWORD}
qdrant:
  host: qdrant.internal
EOF

# Validate
python -m oracle.config_cli validate
```

### Example 3: Switching Environments

```bash
# Switch to staging
python -m oracle.config_cli switch staging

# Verify
python -m oracle.config_cli show

# Run ORACLE
oracle investigate "test question"
```

## References

- Configuration API: `oracle/config.py`
- Configuration CLI: `oracle/config_cli.py`
- Default Config: `oracle/config/defaults.yaml`
- Dev Config: `oracle/config/dev.yaml`
- Staging Config: `oracle/config/staging.yaml`
- Production Config: `oracle/config/production.yaml`
