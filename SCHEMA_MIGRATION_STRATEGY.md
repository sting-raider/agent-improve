# ORACLE Schema Migration Strategy

**Status:** APPROVED  
**Date:** 2025-05-02  
**Purpose:** Strategy for handling schema changes in Neo4j and Qdrant

---

## Context

ORACLE's Neo4j schema and Qdrant collection configuration will evolve as development progresses. There is currently no plan for how to handle schema changes once real data is in the databases.

**Problem:** Schema changes are inevitable during development. Without a migration strategy, we risk:
- Data corruption
- Inconsistent state across environments
- Inability to rollback changes
- Lost data during schema updates

## Research Findings

### Neo4j Schema Migration

**Finding:** Neo4j does not have Alembic-style migrations natively.

**Options:**
1. **APOC procedures** — Can help with schema operations but not full migrations
2. **Custom migration scripts** — Write Cypher scripts for each schema change
3. **Neo4j Migrations library** — Third-party library for managing migrations
4. **Manual schema updates** — Ad-hoc Cypher commands (not recommended)

**Recommendation:** Custom migration scripts with version tracking.

**Rationale:**
- Full control over migration logic
- Can handle complex schema changes
- Easy to rollback
- No external dependencies
- Works with both Community and Enterprise editions

### Qdrant Collection Migration

**Finding:** Qdrant does not support in-place schema changes.

**Constraints:**
- Cannot change vector dimension after collection creation
- Cannot change distance metric after collection creation
- Must recreate collection for schema changes
- Must re-embed all data after recreation

**Cost:** 5-8 days for 300GB corpus re-embedding.

**Recommendation:** Freeze schema after Phase 2 completes.

**Rationale:**
- Re-embedding is expensive (time and compute)
- Schema should be stable before production use
- Can use versioned collections for testing

## Migration Strategy

### Versioning Scheme

Every schema change gets:
- **Version number** — Incremental integer (1, 2, 3, ...)
- **Migration script** — Cypher for Neo4j, Python for Qdrant
- **Rollback script** — Reverse the migration
- **Description** — Human-readable explanation

**Storage:**
```
oracle/
├── migrations/
│   ├── neo4j/
│   │   ├── 001_initial_schema.cypher
│   │   ├── 001_rollback_initial_schema.cypher
│   │   ├── 002_add_pagerank_property.cypher
│   │   ├── 002_rollback_add_pagerank_property.cypher
│   │   └── ...
│   └── qdrant/
│       ├── 001_initial_collection.py
│       ├── 001_rollback_initial_collection.py
│       └── ...
```

### Schema Version Tracking

Store current schema version in SQLite:

```sql
-- Schema version table
CREATE TABLE IF NOT EXISTS schema_version (
    component TEXT PRIMARY KEY,  -- 'neo4j' or 'qdrant'
    version INTEGER NOT NULL,
    applied_at TEXT NOT NULL,
    description TEXT
);

-- Initial version
INSERT INTO schema_version (component, version, applied_at, description)
VALUES ('neo4j', 1, datetime('now'), 'Initial schema with Entity and Chunk nodes');

INSERT INTO schema_version (component, version, applied_at, description)
VALUES ('qdrant', 1, datetime('now'), 'Initial collection with 1024-dim vectors');
```

### Migration Runner

```python
# oracle/migrations/runner.py

import aiosqlite
from pathlib import Path
from neo4j import AsyncGraphDatabase
from qdrant_client import AsyncQdrantClient
import structlog

log = structlog.get_logger()


class MigrationRunner:
    """Runs schema migrations for Neo4j and Qdrant."""
    
    def __init__(self, db_path: str, neo4j_uri: str, neo4j_auth: tuple, qdrant_host: str, qdrant_port: int):
        self.db_path = db_path
        self.neo4j_driver = AsyncGraphDatabase.driver(neo4j_uri, auth=neo4j_auth)
        self.qdrant_client = AsyncQdrantClient(host=qdrant_host, port=qdrant_port)
        self.migrations_dir = Path(__file__).parent
    
    async def get_current_version(self, component: str) -> int:
        """Get current schema version for a component."""
        async with aiosqlite.connect(self.db_path) as db:
            cursor = await db.execute(
                "SELECT version FROM schema_version WHERE component = ?",
                (component,)
            )
            row = await cursor.fetchone()
            return row[0] if row else 0
    
    async def set_version(self, component: str, version: int, description: str):
        """Set current schema version for a component."""
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("""
                INSERT OR REPLACE INTO schema_version (component, version, applied_at, description)
                VALUES (?, ?, datetime('now'), ?)
            """, (component, version, description))
            await db.commit()
    
    async def migrate_neo4j(self, target_version: int | None = None):
        """Run Neo4j migrations up to target version."""
        current_version = await self.get_current_version('neo4j')
        target_version = target_version or self._get_latest_version('neo4j')
        
        log.info("neo4j_migration_start", current=current_version, target=target_version)
        
        for version in range(current_version + 1, target_version + 1):
            migration_file = self.migrations_dir / 'neo4j' / f'{version:03d}_*.cypher'
            migration_files = list(self.migrations_dir.glob(str(migration_file)))
            
            if not migration_files:
                log.warning("migration_not_found", component='neo4j', version=version)
                continue
            
            migration_file = migration_files[0]
            
            log.info("applying_migration", component='neo4j', version=version, file=migration_file.name)
            
            # Read migration
            with open(migration_file, 'r') as f:
                cypher = f.read()
            
            # Apply migration
            async with self.neo4j_driver.session() as session:
                await session.run(cypher)
            
            # Update version
            description = self._extract_description(migration_file)
            await self.set_version('neo4j', version, description)
            
            log.info("migration_applied", component='neo4j', version=version)
        
        log.info("neo4j_migration_complete", final_version=target_version)
    
    async def rollback_neo4j(self, target_version: int):
        """Rollback Neo4j migrations to target version."""
        current_version = await self.get_current_version('neo4j')
        
        log.info("neo4j_rollback_start", current=current_version, target=target_version)
        
        for version in range(current_version, target_version, -1):
            rollback_file = self.migrations_dir / 'neo4j' / f'{version:03d}_rollback_*.cypher'
            rollback_files = list(self.migrations_dir.glob(str(rollback_file)))
            
            if not rollback_files:
                log.warning("rollback_not_found", component='neo4j', version=version)
                continue
            
            rollback_file = rollback_files[0]
            
            log.info("applying_rollback", component='neo4j', version=version, file=rollback_file.name)
            
            # Read rollback
            with open(rollback_file, 'r') as f:
                cypher = f.read()
            
            # Apply rollback
            async with self.neo4j_driver.session() as session:
                await session.run(cypher)
            
            # Update version
            await self.set_version('neo4j', version - 1, f"Rolled back to version {version - 1}")
            
            log.info("rollback_applied", component='neo4j', version=version)
        
        log.info("neo4j_rollback_complete", final_version=target_version)
    
    async def migrate_qdrant(self, target_version: int | None = None):
        """Run Qdrant migrations up to target version."""
        current_version = await self.get_current_version('qdrant')
        target_version = target_version or self._get_latest_version('qdrant')
        
        log.info("qdrant_migration_start", current=current_version, target=target_version)
        
        for version in range(current_version + 1, target_version + 1):
            migration_file = self.migrations_dir / 'qdrant' / f'{version:03d}_*.py'
            migration_files = list(self.migrations_dir.glob(str(migration_file)))
            
            if not migration_files:
                log.warning("migration_not_found", component='qdrant', version=version)
                continue
            
            migration_file = migration_files[0]
            
            log.info("applying_migration", component='qdrant', version=version, file=migration_file.name)
            
            # Load and run migration
            spec = {}
            with open(migration_file, 'r') as f:
                exec(f.read(), spec)
            
            migration_func = spec.get('migrate')
            if migration_func:
                await migration_func(self.qdrant_client)
            
            # Update version
            description = self._extract_description(migration_file)
            await self.set_version('qdrant', version, description)
            
            log.info("migration_applied", component='qdrant', version=version)
        
        log.info("qdrant_migration_complete", final_version=target_version)
    
    async def rollback_qdrant(self, target_version: int):
        """Rollback Qdrant migrations to target version."""
        current_version = await self.get_current_version('qdrant')
        
        log.info("qdrant_rollback_start", current=current_version, target=target_version)
        
        for version in range(current_version, target_version, -1):
            rollback_file = self.migrations_dir / 'qdrant' / f'{version:03d}_rollback_*.py'
            rollback_files = list(self.migrations_dir.glob(str(rollback_file)))
            
            if not rollback_files:
                log.warning("rollback_not_found", component='qdrant', version=version)
                continue
            
            rollback_file = rollback_files[0]
            
            log.info("applying_rollback", component='qdrant', version=version, file=rollback_file.name)
            
            # Load and run rollback
            spec = {}
            with open(rollback_file, 'r') as f:
                exec(f.read(), spec)
            
            rollback_func = spec.get('rollback')
            if rollback_func:
                await rollback_func(self.qdrant_client)
            
            # Update version
            await self.set_version('qdrant', version - 1, f"Rolled back to version {version - 1}")
            
            log.info("rollback_applied", component='qdrant', version=version)
        
        log.info("qdrant_rollback_complete", final_version=target_version)
    
    def _get_latest_version(self, component: str) -> int:
        """Get latest migration version for a component."""
        migrations_dir = self.migrations_dir / component
        if not migrations_dir.exists():
            return 0
        
        versions = []
        for file in migrations_dir.glob('*.cypher' if component == 'neo4j' else '*.py'):
            # Extract version number from filename
            parts = file.stem.split('_')
            if parts and parts[0].isdigit():
                versions.append(int(parts[0]))
        
        return max(versions) if versions else 0
    
    def _extract_description(self, file_path: Path) -> str:
        """Extract description from migration file."""
        with open(file_path, 'r') as f:
            first_line = f.readline().strip()
            if first_line.startswith('--'):
                return first_line[2:].strip()
            return file_path.stem
    
    async def close(self):
        """Close connections."""
        await self.neo4j_driver.close()
        await self.qdrant_client.close()
```

### Migration Scripts

#### Neo4j Migration 001: Initial Schema

```cypher
-- oracle/migrations/neo4j/001_initial_schema.cypher
-- Initial schema with Entity and Chunk nodes

-- Create constraints
CREATE CONSTRAINT entity_normalized IF NOT EXISTS
FOR (e:Entity) REQUIRE e.normalized IS UNIQUE;

CREATE CONSTRAINT chunk_unique IF NOT EXISTS
FOR (c:Chunk) REQUIRE (c.source_path, c.chunk_index) IS UNIQUE;

CREATE CONSTRAINT document_path IF NOT EXISTS
FOR (d:Document) REQUIRE d.file_path IS UNIQUE;

-- Create indexes
CREATE INDEX entity_type IF NOT EXISTS
FOR (e:Entity) ON (e.type);

CREATE INDEX entity_mention_count IF NOT EXISTS
FOR (e:Entity) ON (e.mention_count);

CREATE INDEX chunk_source IF NOT EXISTS
FOR (c:Chunk) ON (c.source_path);

CREATE FULLTEXT INDEX entity_text_search IF NOT EXISTS
FOR (e:Entity) ON EACH [e.normalized, e.text];

CREATE FULLTEXT INDEX chunk_text_search IF NOT EXISTS
FOR (c:Chunk) ON EACH [c.text_preview];
```

```cypher
-- oracle/migrations/neo4j/001_rollback_initial_schema.cypher
-- Rollback initial schema

-- Drop indexes
DROP INDEX entity_type IF EXISTS;
DROP INDEX entity_mention_count IF EXISTS;
DROP INDEX chunk_source IF EXISTS;
DROP FULLTEXT INDEX entity_text_search IF EXISTS;
DROP FULLTEXT INDEX chunk_text_search IF EXISTS;

-- Drop constraints
DROP CONSTRAINT entity_normalized IF EXISTS;
DROP CONSTRAINT chunk_unique IF EXISTS;
DROP CONSTRAINT document_path IF EXISTS;
```

#### Neo4j Migration 002: Add PageRank Property

```cypher
-- oracle/migrations/neo4j/002_add_pagerank_property.cypher
-- Add PageRank property to Entity nodes

-- Add property to existing nodes
MATCH (e:Entity)
WHERE e.pagerank IS NULL
SET e.pagerank = 1.0;
```

```cypher
-- oracle/migrations/neo4j/002_rollback_add_pagerank_property.cypher
-- Rollback PageRank property

-- Remove property from nodes
MATCH (e:Entity)
REMOVE e.pagerank;
```

#### Qdrant Migration 001: Initial Collection

```python
# oracle/migrations/qdrant/001_initial_collection.py
# Initial collection with 1024-dim vectors

from qdrant_client.models import VectorParams, Distance, PayloadSchemaType

async def migrate(client):
    """Create initial Qdrant collection."""
    await client.create_collection(
        collection_name="oracle_corpus",
        vectors_config=VectorParams(
            size=1024,
            distance=Distance.COSINE
        ),
        payload_schema={
            "text": PayloadSchemaType.TEXT,
            "source_path": PayloadSchemaType.TEXT,
            "file_type": PayloadSchemaType.TEXT,
            "chunk_index": PayloadSchemaType.INTEGER,
            "section_path": PayloadSchemaType.TEXT,
            "section_breadcrumb": PayloadSchemaType.TEXT,
            "token_count": PayloadSchemaType.INTEGER,
            "neo4j_chunk_id": PayloadSchemaType.TEXT,
            "ingested_at": PayloadSchemaType.TEXT,
        }
    )
```

```python
# oracle/migrations/qdrant/001_rollback_initial_collection.py
# Rollback initial collection

async def rollback(client):
    """Delete initial Qdrant collection."""
    await client.delete_collection("oracle_corpus")
```

## Migration Workflow

### Development Workflow

1. **Create migration:**
   ```bash
   # Create new migration file
   touch oracle/migrations/neo4j/003_add_new_property.cypher
   touch oracle/migrations/neo4j/003_rollback_add_new_property.cypher
   ```

2. **Write migration:**
   ```cypher
   -- oracle/migrations/neo4j/003_add_new_property.cypher
   -- Add new property to Entity nodes
   
   MATCH (e:Entity)
   WHERE e.new_property IS NULL
   SET e.new_property = "default_value";
   ```

3. **Write rollback:**
   ```cypher
   -- oracle/migrations/neo4j/003_rollback_add_new_property.cypher
   -- Rollback new property
   
   MATCH (e:Entity)
   REMOVE e.new_property;
   ```

4. **Test migration:**
   ```python
   # Run migration on test database
   runner = MigrationRunner(...)
   await runner.migrate_neo4j()
   ```

5. **Commit migration:**
   ```bash
   git add oracle/migrations/neo4j/003_*.cypher
   git commit -m "Add new property to Entity nodes"
   ```

### Production Workflow

1. **Backup database:**
   ```bash
   # Backup Neo4j
   neo4j-admin database dump neo4j --to-path=/backup
   
   # Backup Qdrant
   # (Qdrant doesn't have built-in backup, use snapshots)
   ```

2. **Run migration:**
   ```python
   # Run migration on production database
   runner = MigrationRunner(...)
   await runner.migrate_neo4j()
   ```

3. **Verify migration:**
   ```python
   # Verify schema version
   version = await runner.get_current_version('neo4j')
   assert version == 3
   ```

4. **Monitor for issues:**
   ```python
   # Check logs for errors
   # Monitor performance
   # Verify data integrity
   ```

5. **Rollback if needed:**
   ```python
   # Rollback to previous version
   await runner.rollback_neo4j(2)
   ```

## Schema Freeze Policy

### v1 Schema Freeze

**Policy:** Schema is frozen after Phase 2 completes.

**Rationale:**
- Re-embedding is expensive (5-8 days for 300GB corpus)
- Schema should be stable before production use
- Can use versioned collections for testing

**Exceptions:**
- Bug fixes that don't require re-embedding
- Adding new indexes (can be done without re-embedding)
- Adding new properties (can be done without re-embedding)

**Process for exceptions:**
1. Document the change
2. Create migration script
3. Test on sample data
4. Get approval
5. Apply to production
6. Monitor for issues

### v2 Schema Changes

**Policy:** Schema changes in v2 require full re-embedding.

**Rationale:**
- v2 may have different requirements
- Can afford re-embedding with distributed workers
- Schema should be optimized for swarm architecture

**Process:**
1. Document the change
2. Create migration script
3. Test on sample data
4. Get approval
5. Schedule maintenance window
6. Backup existing data
7. Apply migration
8. Re-embed all data
9. Verify integrity
10. Monitor for issues

## Risk Mitigation

### Risk: Data Loss During Migration

**Mitigation:**
- Always backup before migration
- Test migration on sample data first
- Use transactions where possible
- Verify data integrity after migration
- Have rollback plan ready

### Risk: Migration Failure

**Mitigation:**
- Write idempotent migrations
- Test rollback scripts
- Monitor migration progress
- Have manual intervention plan
- Document failure scenarios

### Risk: Performance Degradation

**Mitigation:**
- Benchmark migration performance
- Schedule during low-traffic periods
- Monitor system resources
- Have rollback plan ready
- Document performance impact

## Success Criteria

### Migration Success
- All migrations apply successfully
- All rollbacks work correctly
- No data loss during migration
- Schema version is tracked correctly
- Migration logs are complete

### Rollback Success
- All rollbacks work correctly
- No data loss during rollback
- Schema version is tracked correctly
- Rollback logs are complete

### Production Success
- Migrations apply without downtime
- No performance degradation
- No data corruption
- Schema is stable after freeze

---

**Approved by:** Hermes Agent  
**Next Action:** Implement migration runner and create initial migrations
