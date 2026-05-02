# ORACLE Testing Strategy

**Status:** APPROVED  
**Date:** 2025-05-02  
**Purpose:** Comprehensive testing strategy for all phases of ORACLE development

---

## Philosophy

**Testing is not Week 6.** Testing is built alongside each phase. Phase 1 ships with ingestion tests. Phase 2 ships with graph tests. Phase 3 ships with agent tests. This is the only way to catch data corruption early in a system that runs for days.

**Test Pyramid:**
- **Unit tests:** Fast, isolated, test individual functions
- **Integration tests:** Medium speed, test component interactions
- **End-to-end tests:** Slow, test full workflows
- **Smoke tests:** Quick validation that system is operational

**Testing Tools:**
- **pytest** — Test framework
- **pytest-asyncio** — Async test support
- **pytest-cov** — Coverage reporting
- **pytest-mock** — Mocking utilities
- **testcontainers** — Containerized services for testing
- **fakeredis** — Redis mock for testing
- **aiosqlite** — In-memory SQLite for testing

---

## Phase 1: Ingestion Pipeline Tests

### Test Categories

#### 1.1 Document Parsing Tests

**Goal:** Verify Docling parses all supported formats correctly.

**Test Cases:**

```python
# tests/ingestion/test_parser.py

import pytest
from pathlib import Path
from oracle.ingestion.parser import DocumentParser

@pytest.fixture
def parser():
    return DocumentParser()

def test_parse_plain_pdf(parser, sample_plain_pdf):
    """Test parsing a plain text PDF."""
    result = await parser.parse(sample_plain_pdf)
    
    assert result.file_type == "pdf"
    assert len(result.markdown_content) > 0
    assert "##" in result.markdown_content  # Has headers
    assert result.page_count > 0

def test_parse_scanned_pdf(parser, sample_scanned_pdf):
    """Test parsing a scanned PDF (requires OCR)."""
    result = await parser.parse(sample_scanned_pdf)
    
    assert result.file_type == "pdf"
    assert len(result.markdown_content) > 0
    # OCR should have extracted text
    assert len(result.markdown_content.split()) > 50

def test_parse_image(parser, sample_image_with_text):
    """Test parsing an image with text (OCR)."""
    result = await parser.parse(sample_image_with_text)
    
    assert result.file_type == "image"
    assert len(result.markdown_content) > 0
    # OCR should have extracted text
    assert len(result.markdown_content.split()) > 10

def test_parse_audio(parser, sample_audio_file):
    """Test parsing an audio file (transcription)."""
    result = await parser.parse(sample_audio_file)
    
    assert result.file_type == "audio"
    assert len(result.markdown_content) > 0
    # Transcription should have extracted text
    assert len(result.markdown_content.split()) > 20

def test_parse_video(parser, sample_video_file):
    """Test parsing a video file (transcription)."""
    result = await parser.parse(sample_video_file)
    
    assert result.file_type == "video"
    assert len(result.markdown_content) > 0
    # Transcription should have extracted text
    assert len(result.markdown_content.split()) > 20

def test_parse_html(parser, sample_html_file):
    """Test parsing an HTML file."""
    result = await parser.parse(sample_html_file)
    
    assert result.file_type == "text"
    assert len(result.markdown_content) > 0
    # Should have cleaned HTML
    assert "<" not in result.markdown_content
```

**Fixtures:**
```python
# tests/conftest.py

import pytest
from pathlib import Path

@pytest.fixture
def sample_plain_pdf(tmp_path):
    """Create a sample plain text PDF."""
    pdf_path = tmp_path / "plain.pdf"
    # Create PDF with known content
    # ...
    return pdf_path

@pytest.fixture
def sample_scanned_pdf(tmp_path):
    """Create a sample scanned PDF."""
    pdf_path = tmp_path / "scanned.pdf"
    # Create PDF with image content
    # ...
    return pdf_path

@pytest.fixture
def sample_image_with_text(tmp_path):
    """Create a sample image with text."""
    image_path = tmp_path / "image.png"
    # Create image with known text
    # ...
    return image_path

@pytest.fixture
def sample_audio_file(tmp_path):
    """Create a sample audio file."""
    audio_path = tmp_path / "audio.mp3"
    # Create audio with known transcription
    # ...
    return audio_path

@pytest.fixture
def sample_video_file(tmp_path):
    """Create a sample video file."""
    video_path = tmp_path / "video.mp4"
    # Create video with known transcription
    # ...
    return video_path

@pytest.fixture
def sample_html_file(tmp_path):
    """Create a sample HTML file."""
    html_path = tmp_path / "page.html"
    html_path.write_text("""
        <html>
        <head><title>Test Page</title></head>
        <body>
            <h1>Main Heading</h1>
            <p>This is a paragraph with some text.</p>
        </body>
        </html>
    """)
    return html_path
```

#### 1.2 Semantic Chunking Tests

**Goal:** Verify chunks respect document structure and overlap.

**Test Cases:**

```python
# tests/ingestion/test_chunker.py

import pytest
from oracle.ingestion.chunker import SemanticChunker, TextChunk

@pytest.fixture
def chunker():
    return SemanticChunker(chunk_size=512, overlap=64)

def test_chunk_respects_headers(chunker):
    """Test that chunks don't split headers."""
    markdown = """
    # Chapter 1
    
    This is the first paragraph of chapter 1.
    It has multiple sentences.
    
    ## Section 1.1
    
    This is the first section.
    """
    
    chunks = chunker.chunk(markdown)
    
    # No chunk should contain a header in the middle
    for chunk in chunks:
        lines = chunk.text.split('\n')
        for i, line in enumerate(lines[1:], 1):  # Skip first line
            assert not line.startswith('#'), f"Header in middle of chunk: {line}"

def test_chunk_overlap_correct(chunker):
    """Test that chunks have correct overlap."""
    markdown = "A" * 1000  # Long text
    
    chunks = chunker.chunk(markdown)
    
    if len(chunks) > 1:
        # Check overlap between consecutive chunks
        for i in range(len(chunks) - 1):
            chunk1_end = chunks[i].text[-100:]
            chunk2_start = chunks[i+1].text[:100]
            # Should have some overlap
            assert len(set(chunk1_end) & set(chunk2_start)) > 0

def test_chunk_section_breadcrumb(chunker):
    """Test that chunks track section path."""
    markdown = """
    # Chapter 1
    
    Text here.
    
    ## Section 1.1
    
    More text.
    """
    
    chunks = chunker.chunk(markdown)
    
    # Find chunk in Section 1.1
    section_chunks = [c for c in chunks if "Section 1.1" in c.text]
    assert len(section_chunks) > 0
    
    # Should have breadcrumb
    chunk = section_chunks[0]
    assert "Chapter 1" in chunk.section_path
    assert "Section 1.1" in chunk.section_path

def test_chunk_token_count(chunker):
    """Test that chunks respect token count limits."""
    markdown = "A" * 2000  # Long text
    
    chunks = chunker.chunk(markdown)
    
    for chunk in chunks:
        # Should be close to target size
        assert chunk.token_count <= chunker.chunk_size + chunker.overlap
        assert chunk.token_count >= chunker.chunk_size - chunker.overlap
```

#### 1.3 SQLite Ledger Tests

**Goal:** Verify ledger tracks ingestion state correctly and survives crashes.

**Test Cases:**

```python
# tests/ingestion/test_ledger.py

import pytest
import aiosqlite
from pathlib import Path
from oracle.ingestion.ledger import IngestionLedger

@pytest.fixture
async def ledger(tmp_path):
    db_path = tmp_path / "test_ledger.db"
    ledger = IngestionLedger(db_path)
    await ledger.initialize()
    yield ledger
    await ledger.close()

async def test_ledger_tracks_file_status(ledger, sample_file):
    """Test that ledger tracks file status correctly."""
    file_hash = "abc123"
    
    # Add file
    await ledger.add_file(sample_file, file_hash, "pdf", 1024)
    
    # Check status
    status = await ledger.get_status(sample_file)
    assert status == "pending"
    
    # Update status
    await ledger.update_status(sample_file, "parsing")
    status = await ledger.get_status(sample_file)
    assert status == "parsing"
    
    # Update to complete
    await ledger.update_status(sample_file, "complete")
    status = await ledger.get_status(sample_file)
    assert status == "complete"

async def test_ledger_prevents_duplicate_ingestion(ledger, sample_file):
    """Test that ledger prevents duplicate ingestion."""
    file_hash = "abc123"
    
    # Add file as complete
    await ledger.add_file(sample_file, file_hash, "pdf", 1024)
    await ledger.update_status(sample_file, "complete")
    
    # Try to add again
    should_skip = await ledger.should_skip(sample_file, file_hash)
    assert should_skip is True

async def test_ledger_survives_crash(ledger, sample_file):
    """Test that ledger survives process crash."""
    file_hash = "abc123"
    
    # Add file and update status
    await ledger.add_file(sample_file, file_hash, "pdf", 1024)
    await ledger.update_status(sample_file, "parsing")
    
    # Simulate crash by closing and reopening
    await ledger.close()
    
    # Reopen
    ledger2 = IngestionLedger(ledger.db_path)
    await ledger2.initialize()
    
    # Check status persisted
    status = await ledger2.get_status(sample_file)
    assert status == "parsing"
    
    await ledger2.close()

async def test_ledger_chunk_tracking(ledger, sample_file):
    """Test that ledger tracks chunk processing."""
    file_hash = "abc123"
    
    # Add file
    await ledger.add_file(sample_file, file_hash, "pdf", 1024)
    
    # Add chunks
    await ledger.add_chunk(sample_file, 0, "qdrant_id_1", "neo4j_id_1")
    await ledger.add_chunk(sample_file, 1, "qdrant_id_2", "neo4j_id_2")
    
    # Check chunks
    chunks = await ledger.get_chunks(sample_file)
    assert len(chunks) == 2
    assert chunks[0]["chunk_index"] == 0
    assert chunks[1]["chunk_index"] == 1
```

#### 1.4 GLiNER2 Extraction Tests

**Goal:** Validate GLiNER2 extraction quality on sample corpus.

**Test Cases:**

```python
# tests/ingestion/test_entity_extraction.py

import pytest
from oracle.graph.entity_extractor import GLiNEREntityExtractor

@pytest.fixture
def extractor():
    return GLiNEREntityExtractor(
        model="fastino/gliner2-base-v1",
        device="cuda"  # or "cpu"
    )

def test_extract_person_entities(extractor):
    """Test extracting person entities."""
    text = "John Smith met with Jane Doe at the conference."
    
    entities = extractor.extract(text, ["person"])
    
    assert len(entities) > 0
    person_entities = [e for e in entities if e["type"] == "person"]
    assert len(person_entities) >= 2
    assert any("John Smith" in e["text"] for e in person_entities)
    assert any("Jane Doe" in e["text"] for e in person_entities)

def test_extract_organization_entities(extractor):
    """Test extracting organization entities."""
    text = "Google and Microsoft announced a partnership."
    
    entities = extractor.extract(text, ["organization"])
    
    assert len(entities) > 0
    org_entities = [e for e in entities if e["type"] == "organization"]
    assert len(org_entities) >= 2
    assert any("Google" in e["text"] for e in org_entities)
    assert any("Microsoft" in e["text"] for e in org_entities)

def test_extract_confidence_scores(extractor):
    """Test that extraction includes confidence scores."""
    text = "Apple Inc. is based in Cupertino."
    
    entities = extractor.extract(text, ["organization", "location"])
    
    for entity in entities:
        assert "confidence" in entity
        assert 0.0 <= entity["confidence"] <= 1.0

def test_extract_spans(extractor):
    """Test that extraction includes spans."""
    text = "John Smith works at Google."
    
    entities = extractor.extract(text, ["person", "organization"], include_spans=True)
    
    for entity in entities:
        assert "start" in entity
        assert "end" in entity
        assert entity["start"] >= 0
        assert entity["end"] <= len(text)
        assert entity["end"] > entity["start"]

def test_batch_extraction(extractor):
    """Test batch extraction for performance."""
    texts = [
        "John works at Google.",
        "Jane works at Microsoft.",
        "Bob works at Apple.",
    ]
    
    entities_list = extractor.batch_extract(texts, ["person", "organization"])
    
    assert len(entities_list) == 3
    for entities in entities_list:
        assert len(entities) > 0
```

**Gold Standard Test:**
```python
# tests/ingestion/test_entity_extraction_quality.py

import pytest
from oracle.graph.entity_extractor import GLiNEREntityExtractor

# Gold standard annotations
GOLD_STANDARD = [
    {
        "text": "John Smith met with Jane Doe at the Google headquarters in Mountain View.",
        "entities": [
            {"text": "John Smith", "type": "person"},
            {"text": "Jane Doe", "type": "person"},
            {"text": "Google", "type": "organization"},
            {"text": "Mountain View", "type": "location"},
        ]
    },
    # ... more samples
]

def test_extraction_precision_recall(extractor):
    """Test extraction precision and recall against gold standard."""
    true_positives = 0
    false_positives = 0
    false_negatives = 0
    
    for sample in GOLD_STANDARD:
        predicted = extractor.extract(sample["text"], ["person", "organization", "location"])
        gold_entities = sample["entities"]
        
        # Calculate precision/recall
        # ...
    
    precision = true_positives / (true_positives + false_positives)
    recall = true_positives / (true_positives + false_negatives)
    
    assert precision > 0.85, f"Precision {precision:.2f} below threshold"
    assert recall > 0.80, f"Recall {recall:.2f} below threshold"
```

#### 1.5 End-to-End Ingestion Tests

**Goal:** Verify full ingestion pipeline works correctly.

**Test Cases:**

```python
# tests/ingestion/test_pipeline_e2e.py

import pytest
from pathlib import Path
from oracle.ingestion.pipeline import IngestionPipeline

@pytest.fixture
async def pipeline(tmp_path):
    pipeline = IngestionPipeline(
        corpus_dir=tmp_path / "corpus",
        ledger_db=tmp_path / "ledger.db",
    )
    await pipeline.initialize()
    yield pipeline
    await pipeline.close()

async def test_ingest_single_document(pipeline, sample_pdf):
    """Test ingesting a single document end-to-end."""
    # Add document to corpus
    corpus_dir = pipeline.corpus_dir
    corpus_dir.mkdir(parents=True)
    shutil.copy(sample_pdf, corpus_dir / "test.pdf")
    
    # Run ingestion
    await pipeline.run()
    
    # Verify document was processed
    status = await pipeline.ledger.get_status(corpus_dir / "test.pdf")
    assert status == "complete"
    
    # Verify chunks in Qdrant
    chunks = await pipeline.qdrant.search(
        collection_name="oracle_corpus",
        query_vector=[0] * 1024,  # Dummy vector
        limit=10
    )
    assert len(chunks) > 0
    
    # Verify entities in Neo4j
    async with pipeline.neo4j_driver.session() as session:
        result = await session.run("MATCH (e:Entity) RETURN count(e) AS count")
        count = (await result.single())["count"]
        assert count > 0

async def test_ingest_resumes_after_crash(pipeline, sample_pdf):
    """Test that ingestion resumes after crash."""
    corpus_dir = pipeline.corpus_dir
    corpus_dir.mkdir(parents=True)
    shutil.copy(sample_pdf, corpus_dir / "test.pdf")
    
    # Start ingestion
    await pipeline.run()
    
    # Simulate crash by stopping mid-process
    # (In real test, would interrupt the process)
    
    # Resume ingestion
    await pipeline.run()
    
    # Verify no duplicate processing
    status = await pipeline.ledger.get_status(corpus_dir / "test.pdf")
    assert status == "complete"
    
    # Verify no duplicate chunks
    async with pipeline.neo4j_driver.session() as session:
        result = await session.run("""
            MATCH (c:Chunk {source_path: $path})
            RETURN count(c) AS count
        """, path=str(corpus_dir / "test.pdf"))
        count = (await result.single())["count"]
        # Should have expected number of chunks
        assert count > 0
```

---

## Phase 2: Knowledge Graph Tests

### Test Categories

#### 2.1 Neo4j Schema Tests

**Goal:** Verify Neo4j schema constraints enforce uniqueness.

**Test Cases:**

```python
# tests/graph/test_neo4j_schema.py

import pytest
from neo4j import AsyncGraphDatabase

@pytest.fixture
async def neo4j_driver():
    driver = AsyncGraphDatabase.driver(
        "bolt://localhost:7687",
        auth=("neo4j", "test_password")
    )
    yield driver
    await driver.close()

async def test_entity_uniqueness_constraint(neo4j_driver):
    """Test that entity uniqueness constraint works."""
    async with neo4j_driver.session() as session:
        # Create entity
        await session.run("""
            MERGE (e:Entity {normalized: "John Smith"})
            SET e.type = "person"
        """)
        
        # Try to create duplicate
        with pytest.raises(Exception):  # Should raise constraint violation
            await session.run("""
                CREATE (e:Entity {normalized: "John Smith"})
                SET e.type = "person"
            """)
        
        # Verify only one entity exists
        result = await session.run("""
            MATCH (e:Entity {normalized: "John Smith"})
            RETURN count(e) AS count
        """)
        count = (await result.single())["count"]
        assert count == 1

async def test_chunk_uniqueness_constraint(neo4j_driver):
    """Test that chunk uniqueness constraint works."""
    async with neo4j_driver.session() as session:
        # Create chunk
        await session.run("""
            MERGE (c:Chunk {source_path: "/test.pdf", chunk_index: 0})
            SET c.text_preview = "test"
        """)
        
        # Try to create duplicate
        with pytest.raises(Exception):  # Should raise constraint violation
            await session.run("""
                CREATE (c:Chunk {source_path: "/test.pdf", chunk_index: 0})
                SET c.text_preview = "test"
            """)
        
        # Verify only one chunk exists
        result = await session.run("""
            MATCH (c:Chunk {source_path: "/test.pdf", chunk_index: 0})
            RETURN count(c) AS count
        """)
        count = (await result.single())["count"]
        assert count == 1
```

#### 2.2 Qdrant Collection Tests

**Goal:** Verify Qdrant collection stores and retrieves vectors correctly.

**Test Cases:**

```python
# tests/graph/test_qdrant_collection.py

import pytest
from qdrant_client import AsyncQdrantClient
from qdrant_client.models import PointStruct, VectorParams, Distance

@pytest.fixture
async def qdrant_client():
    client = AsyncQdrantClient(host="localhost", port=6333)
    
    # Create test collection
    await client.create_collection(
        collection_name="test_collection",
        vectors_config=VectorParams(size=1024, distance=Distance.COSINE)
    )
    
    yield client
    
    # Cleanup
    await client.delete_collection("test_collection")

async def test_insert_and_retrieve(qdrant_client):
    """Test inserting and retrieving vectors."""
    # Insert points
    points = [
        PointStruct(
            id=1,
            vector=[0.1] * 1024,
            payload={"text": "test text 1", "source": "doc1"}
        ),
        PointStruct(
            id=2,
            vector=[0.2] * 1024,
            payload={"text": "test text 2", "source": "doc2"}
        ),
    ]
    
    await qdrant_client.upsert(
        collection_name="test_collection",
        points=points
    )
    
    # Retrieve points
    results = await qdrant_client.search(
        collection_name="test_collection",
        query_vector=[0.1] * 1024,
        limit=10
    )
    
    assert len(results) > 0
    assert results[0].id == 1  # Should be closest match
    assert results[0].payload["text"] == "test text 1"

async def test_filter_by_metadata(qdrant_client):
    """Test filtering by metadata."""
    # Insert points with different sources
    points = [
        PointStruct(
            id=1,
            vector=[0.1] * 1024,
            payload={"text": "test 1", "source": "doc1"}
        ),
        PointStruct(
            id=2,
            vector=[0.2] * 1024,
            payload={"text": "test 2", "source": "doc2"}
        ),
    ]
    
    await qdrant_client.upsert(
        collection_name="test_collection",
        points=points
    )
    
    # Filter by source
    results = await qdrant_client.search(
        collection_name="test_collection",
        query_vector=[0.1] * 1024,
        query_filter={
            "must": [
                {"key": "source", "match": {"value": "doc1"}}
            ]
        },
        limit=10
    )
    
    assert len(results) == 1
    assert results[0].payload["source"] == "doc1"
```

#### 2.3 Graph Integration Tests

**Goal:** Verify ingestion ledger + graph integration.

**Test Cases:**

```python
# tests/graph/test_graph_integration.py

import pytest
from oracle.ingestion.pipeline import IngestionPipeline

@pytest.fixture
async def pipeline(tmp_path):
    pipeline = IngestionPipeline(
        corpus_dir=tmp_path / "corpus",
        ledger_db=tmp_path / "ledger.db",
    )
    await pipeline.initialize()
    yield pipeline
    await pipeline.close()

async def test_document_appears_in_both_stores(pipeline, sample_pdf):
    """Test that document appears in both Qdrant and Neo4j."""
    corpus_dir = pipeline.corpus_dir
    corpus_dir.mkdir(parents=True)
    shutil.copy(sample_pdf, corpus_dir / "test.pdf")
    
    # Run ingestion
    await pipeline.run()
    
    # Verify in Qdrant
    qdrant_results = await pipeline.qdrant.search(
        collection_name="oracle_corpus",
        query_vector=[0] * 1024,
        query_filter={
            "must": [
                {"key": "source_path", "match": {"value": str(corpus_dir / "test.pdf")}}
            ]
        },
        limit=10
    )
    assert len(qdrant_results) > 0
    
    # Verify in Neo4j
    async with pipeline.neo4j_driver.session() as session:
        result = await session.run("""
            MATCH (c:Chunk {source_path: $path})
            RETURN count(c) AS count
        """, path=str(corpus_dir / "test.pdf"))
        count = (await result.single())["count"]
        assert count > 0
    
    # Verify counts match
    assert len(qdrant_results) == count

async def test_ledger_marks_complete_only_after_both_writes(pipeline, sample_pdf):
    """Test that ledger marks complete only after both Qdrant and Neo4j writes."""
    corpus_dir = pipeline.corpus_dir
    corpus_dir.mkdir(parents=True)
    shutil.copy(sample_pdf, corpus_dir / "test.pdf")
    
    # Run ingestion
    await pipeline.run()
    
    # Verify status is complete
    status = await pipeline.ledger.get_status(corpus_dir / "test.pdf")
    assert status == "complete"
    
    # Verify both stores have data
    qdrant_count = len(await pipeline.qdrant.search(
        collection_name="oracle_corpus",
        query_vector=[0] * 1024,
        query_filter={
            "must": [
                {"key": "source_path", "match": {"value": str(corpus_dir / "test.pdf")}}
            ]
        },
        limit=1000
    ))
    
    async with pipeline.neo4j_driver.session() as session:
        result = await session.run("""
            MATCH (c:Chunk {source_path: $path})
            RETURN count(c) AS count
        """, path=str(corpus_dir / "test.pdf"))
        neo4j_count = (await result.single())["count"]
    
    assert qdrant_count > 0
    assert neo4j_count > 0
```

---

## Phase 3: Agent Tests

### Test Categories

#### 3.1 LangGraph Checkpoint Tests

**Goal:** Verify checkpoint save and restore works correctly.

**Test Cases:**

```python
# tests/agents/test_checkpointing.py

import pytest
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver
from oracle.agents.orchestrator import InvestigationState

@pytest.fixture
async def checkpointer(tmp_path):
    db_path = tmp_path / "checkpoints.db"
    checkpointer = AsyncSqliteSaver.from_conn_string(f"sqlite:///{db_path}")
    yield checkpointer
    await checkpointer.close()

async def test_checkpoint_save_and_restore(checkpointer):
    """Test that checkpoint saves and restores state correctly."""
    thread_id = "test_thread"
    
    # Save checkpoint
    state = InvestigationState(
        investigation_id="test_inv",
        investigation_name="Test Investigation",
        original_question="Test question",
        status="researching",
        hypotheses=[],
        active_tasks=[],
        pending_questions=[],
        key_findings=[],
        contradictions=[],
        entities_of_interest=[],
        named_graphs=[],
        journal_path="/test/journal.md",
        journal_summary="Test summary",
        sources_consulted=[],
        messages=[],
        config={},
    )
    
    await checkpointer.aput(
        config={"configurable": {"thread_id": thread_id}},
        checkpoint=state,
    )
    
    # Restore checkpoint
    restored = await checkpointer.aget(
        config={"configurable": {"thread_id": thread_id}}
    )
    
    assert restored is not None
    assert restored["investigation_id"] == "test_inv"
    assert restored["status"] == "researching"

async def test_checkpoint_survives_restart(checkpointer):
    """Test that checkpoint survives process restart."""
    thread_id = "test_thread"
    
    # Save checkpoint
    state = InvestigationState(
        investigation_id="test_inv",
        investigation_name="Test Investigation",
        original_question="Test question",
        status="researching",
        hypotheses=[],
        active_tasks=[],
        pending_questions=[],
        key_findings=[],
        contradictions=[],
        entities_of_interest=[],
        named_graphs=[],
        journal_path="/test/journal.md",
        journal_summary="Test summary",
        sources_consulted=[],
        messages=[],
        config={},
    )
    
    await checkpointer.aput(
        config={"configurable": {"thread_id": thread_id}},
        checkpoint=state,
    )
    
    # Simulate restart by closing and reopening
    await checkpointer.close()
    
    # Reopen
    checkpointer2 = AsyncSqliteSaver.from_conn_string(
        f"sqlite:///{checkpointer.conn_string}"
    )
    
    # Restore checkpoint
    restored = await checkpointer2.aget(
        config={"configurable": {"thread_id": thread_id}}
    )
    
    assert restored is not None
    assert restored["investigation_id"] == "test_inv"
    
    await checkpointer2.close()
```

#### 3.2 Specialist Agent Tests

**Goal:** Verify each specialist agent produces correct output.

**Test Cases:**

```python
# tests/agents/test_specialists.py

import pytest
from oracle.agents.specialists.source_critic import SourceCritic
from oracle.agents.specialists.historian import Historian
from oracle.agents.specialists.statistician import Statistician
from oracle.agents.specialists.devils_advocate import DevilsAdvocate

@pytest.fixture
def mock_retrieval_layer():
    """Mock retrieval layer for testing."""
    class MockRetrieval:
        async def search(self, query):
            return [
                {"text": "Source 1", "source": "doc1.pdf", "confidence": 0.9},
                {"text": "Source 2", "source": "doc2.pdf", "confidence": 0.8},
            ]
    
    return MockRetrieval()

async def test_source_critic_evaluates_sources(mock_retrieval_layer):
    """Test that SourceCritic evaluates source credibility."""
    critic = SourceCritic(retrieval=mock_retrieval_layer)
    
    result = await critic.evaluate_sources("test query")
    
    assert "sources" in result
    assert len(result["sources"]) > 0
    for source in result["sources"]:
        assert "credibility" in source
        assert 0.0 <= source["credibility"] <= 1.0

async def test_historian_contextualizes_events(mock_retrieval_layer):
    """Test that Historian contextualizes events in timeline."""
    historian = Historian(retrieval=mock_retrieval_layer)
    
    result = await historian.contextualize("test event")
    
    assert "context" in result
    assert "timeline" in result
    assert len(result["timeline"]) > 0

async def test_statistician_runs_analysis(mock_retrieval_layer):
    """Test that Statistician runs numerical analysis."""
    statistician = Statistician(retrieval=mock_retrieval_layer)
    
    result = await statistician.analyze("test data")
    
    assert "analysis" in result
    assert "statistics" in result
    assert "code" in result  # Python code that was run

async def test_devils_advocate_challenges_hypotheses(mock_retrieval_layer):
    """Test that DevilsAdvocate challenges hypotheses."""
    advocate = DevilsAdvocate(retrieval=mock_retrieval_layer)
    
    hypothesis = "Test hypothesis"
    result = await advocate.challenge(hypothesis)
    
    assert "challenges" in result
    assert len(result["challenges"]) > 0
    for challenge in result["challenges"]:
        assert "argument" in challenge
        assert "evidence" in challenge
```

#### 3.3 Model Router Tests

**Goal:** Verify model routing and failover works correctly.

**Test Cases:**

```python
# tests/agents/test_model_router.py

import pytest
from oracle.agents.model_router import ModelRouter

@pytest.fixture
def router():
    return ModelRouter()

async def test_router_gets_reasoning_model(router):
    """Test that router gets reasoning model."""
    model = router.get_reasoning_model()
    
    assert model is not None
    # Should be configured provider (e.g., Groq)

async def test_router_gets_extraction_model(router):
    """Test that router gets extraction model."""
    model = router.get_extraction_model()
    
    assert model is not None
    # Should be GLiNER2

async def test_router_handles_api_failure(router):
    """Test that router handles API failure gracefully."""
    # Mock API failure
    # ...
    
    # Should fallback to next provider
    # ...
    
    assert True  # Should not raise exception

async def test_gpu_lock_prevents_simultaneous_usage(router):
    """Test that GPU lock prevents simultaneous usage."""
    import asyncio
    
    async def task1():
        async with router.with_gpu_lock(some_coro):
            pass
    
    async def task2():
        async with router.with_gpu_lock(some_coro):
            pass
    
    # Run both tasks
    await asyncio.gather(task1(), task2())
    
    # Should not exceed VRAM
    # ...
```

---

## Smoke Tests

### Purpose
Quick validation that system is operational before running full test suite.

### Test Cases

```python
# tests/smoke/test_smoke.py

import pytest
from neo4j import AsyncGraphDatabase
from qdrant_client import AsyncQdrantClient

async def test_neo4j_connection():
    """Test Neo4j is running and accessible."""
    driver = AsyncGraphDatabase.driver(
        "bolt://localhost:7687",
        auth=("neo4j", "test_password")
    )
    
    async with driver.session() as session:
        result = await session.run("RETURN 1 AS test")
        value = (await result.single())["test"]
        assert value == 1
    
    await driver.close()

async def test_qdrant_connection():
    """Test Qdrant is running and accessible."""
    client = AsyncQdrantClient(host="localhost", port=6333)
    
    collections = await client.get_collections()
    assert collections is not None

async def test_infinity_emb_connection():
    """Test infinity-emb is running and accessible."""
    import httpx
    
    async with httpx.AsyncClient() as client:
        response = await client.get("http://localhost:7997/health")
        assert response.status_code == 200

async def test_gliner2_loads():
    """Test GLiNER2 loads correctly."""
    from gliner2 import GLiNER2
    
    extractor = GLiNER2.from_pretrained("fastino/gliner2-base-v1")
    assert extractor is not None

async def test_api_keys_configured():
    """Test that API keys are configured."""
    import os
    
    assert os.getenv("GROQ_API_KEY") is not None
    assert os.getenv("GOOGLE_API_KEY") is not None
    assert os.getenv("OPENROUTER_API_KEY") is not None
```

---

## Test Execution

### Running Tests

```bash
# Run all tests
pytest

# Run specific test file
pytest tests/ingestion/test_parser.py

# Run specific test
pytest tests/ingestion/test_parser.py::test_parse_plain_pdf

# Run with coverage
pytest --cov=oracle --cov-report=html

# Run async tests
pytest -v

# Run smoke tests only
pytest tests/smoke/
```

### CI/CD Integration

```yaml
# .github/workflows/test.yml

name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      neo4j:
        image: neo4j:5.26-community
        ports:
          - 7474:7474
          - 7687:7687
        env:
          NEO4J_AUTH: neo4j/test_password
      
      qdrant:
        image: qdrant/qdrant:latest
        ports:
          - 6333:6333
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -e ".[dev]"
      
      - name: Run tests
        run: |
          pytest --cov=oracle --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

---

## Success Criteria

### Phase 1 Success
- All parsing tests pass (9/9 formats)
- All chunking tests pass (4/4 tests)
- All ledger tests pass (4/4 tests)
- GLiNER2 precision > 0.85, recall > 0.80
- End-to-end ingestion test passes
- Coverage > 80%

### Phase 2 Success
- All Neo4j schema tests pass (2/2 tests)
- All Qdrant collection tests pass (2/2 tests)
- Graph integration test passes
- Coverage > 80%

### Phase 3 Success
- All checkpoint tests pass (2/2 tests)
- All specialist agent tests pass (4/4 tests)
- Model router test passes
- Coverage > 80%

### Smoke Tests Success
- All smoke tests pass (5/5 tests)
- All services are running and accessible
- All API keys are configured

---

**Approved by:** Hermes Agent  
**Next Action:** Implement tests alongside each phase
