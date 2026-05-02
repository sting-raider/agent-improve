# ORACLE Smoke Test Corpus Plan

**Status:** APPROVED  
**Date:** 2025-05-02  
**Purpose:** Plan for smoke test corpus with known ground truth entities

---

## Context

Before any full corpus ingest, ORACLE needs a small, controlled test fixture. This corpus should contain known entities and relationships so we can verify extraction quality.

**Goal:** Validate the full ingestion pipeline on a representative sample before processing large corpora.

## Corpus Composition

### File Types

| File Type | Count | Purpose |
|-----------|-------|---------|
| Plain text PDFs | 2 | Well-formatted, no tables |
| Complex PDFs | 2 | Tables, figures, mixed layout |
| Scanned PDFs | 2 | Requires OCR |
| Images with text | 1 | OCR validation |
| Audio files | 1 | Transcription validation |
| Video files | 1 | Transcription validation |
| Web pages | 1 | HTML parsing validation |
| **Total** | **10** | **Full pipeline coverage** |

### Ground Truth Entities

Each file should contain known entities that can be verified:

**Entity Types:**
- PERSON (people names)
- ORGANIZATION (company names, institutions)
- LOCATION (cities, countries, addresses)
- DATE (dates, times, periods)
- AMOUNT (monetary values, quantities)
- EVENT (named events, incidents)
- DOCUMENT (document references)

**Relationship Types:**
- EMPLOYED_BY (person → organization)
- LOCATED_AT (organization → location)
- OCCURRED_ON (event → date)
- REFERENCED_IN (entity → document)
- ASSOCIATED_WITH (entity → entity)

## Corpus Files

### 1. Plain Text PDF 1: Wikipedia Article

**File:** `wikipedia_steve_jobs.pdf`  
**Source:** https://en.wikipedia.org/wiki/Steve_Jobs  
**Purpose:** Test plain text parsing and entity extraction

**Ground Truth Entities:**
```
PERSON: Steve Jobs, Steve Wozniak, Ronald Wayne, John Sculley, Tim Cook
ORGANIZATION: Apple Inc., NeXT, Pixar, Disney
LOCATION: Los Altos, California, Cupertino, Palo Alto
DATE: February 24, 1955, October 5, 2011, 1976, 1985, 1997
EVENT: Founding of Apple, NeXT acquisition, Pixar acquisition
AMOUNT: $150,000, $1.5 billion, $7.4 billion
```

**Ground Truth Relationships:**
```
Steve Jobs -[EMPLOYED_BY]-> Apple Inc.
Steve Jobs -[EMPLOYED_BY]-> NeXT
Steve Jobs -[EMPLOYED_BY]-> Pixar
Apple Inc. -[LOCATED_AT]-> Cupertino
Steve Jobs -[OCCURRED_ON]-> February 24, 1955
```

### 2. Plain Text PDF 2: News Article

**File:** `news_article_tech_acquisition.pdf`  
**Source:** Synthetic news article  
**Purpose:** Test plain text parsing and entity extraction

**Ground Truth Entities:**
```
PERSON: Elon Musk, Sam Altman, Satya Nadella
ORGANIZATION: OpenAI, Microsoft, Tesla, SpaceX
LOCATION: San Francisco, Seattle, Austin
DATE: January 15, 2025, March 2025
EVENT: AI partnership announcement
AMOUNT: $10 billion, $500 million
```

**Ground Truth Relationships:**
```
OpenAI -[EMPLOYED_BY]-> Sam Altman
Microsoft -[LOCATED_AT]-> Seattle
OpenAI -[ASSOCIATED_WITH]-> Microsoft
```

### 3. Complex PDF 1: Financial Report

**File:** `financial_report_quarterly.pdf`  
**Source:** Synthetic financial report  
**Purpose:** Test table parsing and structured data extraction

**Ground Truth Entities:**
```
ORGANIZATION: Acme Corp, Beta Inc, Gamma LLC
LOCATION: New York, London, Tokyo
DATE: Q1 2025, Q2 2025, Q3 2025
AMOUNT: $1.2 billion, $3.4 million, $890,000
DOCUMENT: 10-K, 10-Q, 8-K
```

**Ground Truth Relationships:**
```
Acme Corp -[LOCATED_AT]-> New York
Acme Corp -[OCCURRED_ON]-> Q1 2025
```

### 4. Complex PDF 2: Research Paper

**File:** `research_paper_ml.pdf`  
**Source:** Synthetic research paper  
**Purpose:** Test figure parsing and citation extraction

**Ground Truth Entities:**
```
PERSON: Dr. Jane Smith, Prof. John Doe
ORGANIZATION: MIT, Stanford, Google Research
LOCATION: Cambridge, Palo Alto
DATE: 2024, 2025
EVENT: NeurIPS 2024, ICML 2025
DOCUMENT: arXiv:2401.12345, DOI:10.1000/xyz123
```

**Ground Truth Relationships:**
```
Dr. Jane Smith -[EMPLOYED_BY]-> MIT
Prof. John Doe -[EMPLOYED_BY]-> Stanford
```

### 5. Scanned PDF 1: Historical Document

**File:** `historical_document_treaty.pdf`  
**Source:** Synthetic historical document  
**Purpose:** Test OCR on scanned text

**Ground Truth Entities:**
```
PERSON: President Wilson, Prime Minister Lloyd George
ORGANIZATION: League of Nations, United Nations
LOCATION: Paris, Versailles, Washington D.C.
DATE: 1919, June 28, 1919
EVENT: Treaty of Versailles signing
AMOUNT: $33 billion
```

**Ground Truth Relationships:**
```
President Wilson -[EMPLOYED_BY]-> United States
Treaty of Versailles -[OCCURRED_ON]-> June 28, 1919
```

### 6. Scanned PDF 2: Legal Document

**File:** `legal_document_contract.pdf`  
**Source:** Synthetic legal document  
**Purpose:** Test OCR on legal text

**Ground Truth Entities:**
```
PERSON: John Smith, Jane Doe
ORGANIZATION: ABC Corp, XYZ Inc
LOCATION: Delaware, California
DATE: January 1, 2025, December 31, 2025
AMOUNT: $1,000,000, $50,000
DOCUMENT: Contract, Agreement
```

**Ground Truth Relationships:**
```
John Smith -[EMPLOYED_BY]-> ABC Corp
ABC Corp -[LOCATED_AT]-> Delaware
```

### 7. Image with Text: Screenshot

**File:** `screenshot_ui.png`  
**Source:** Synthetic UI screenshot  
**Purpose:** Test OCR on images

**Ground Truth Entities:**
```
PERSON: User, Administrator
ORGANIZATION: Company Name
LOCATION: Dashboard, Settings
DATE: 2025-05-02
EVENT: Login, Logout
AMOUNT: 100, 50
```

**Ground Truth Relationships:**
```
User -[EMPLOYED_BY]-> Company Name
```

### 8. Audio File: Podcast Transcript

**File:** `podcast_episode.mp3`  
**Source:** Synthetic podcast transcript  
**Purpose:** Test audio transcription

**Ground Truth Entities:**
```
PERSON: Host, Guest
ORGANIZATION: Tech Weekly, AI Research
LOCATION: San Francisco
DATE: May 2, 2025
EVENT: Podcast recording
AMOUNT: 1 hour, 30 minutes
```

**Ground Truth Relationships:**
```
Host -[EMPLOYED_BY]-> Tech Weekly
Guest -[EMPLOYED_BY]-> AI Research
```

### 9. Video File: Tutorial

**File:** `tutorial_video.mp4`  
**Source:** Synthetic tutorial video  
**Purpose:** Test video transcription

**Ground Truth Entities:**
```
PERSON: Instructor, Student
ORGANIZATION: Code Academy, Learn Python
LOCATION: Online, Remote
DATE: 2025-05-02
EVENT: Tutorial recording
AMOUNT: 45 minutes, 30 minutes
```

**Ground Truth Relationships:**
```
Instructor -[EMPLOYED_BY]-> Code Academy
Student -[EMPLOYED_BY]-> Learn Python
```

### 10. Web Page: Blog Post

**File:** `blog_post.html`  
**Source:** Synthetic blog post  
**Purpose:** Test HTML parsing

**Ground Truth Entities:**
```
PERSON: Author, Editor
ORGANIZATION: Tech Blog, Medium
LOCATION: Internet, Web
DATE: May 2, 2025
EVENT: Blog post publication
AMOUNT: 1000 words, 5 minutes
```

**Ground Truth Relationships:**
```
Author -[EMPLOYED_BY]-> Tech Blog
```

## Ground Truth Annotation Format

### JSON Format

```json
{
  "file": "wikipedia_steve_jobs.pdf",
  "file_type": "pdf",
  "entities": [
    {
      "text": "Steve Jobs",
      "type": "PERSON",
      "start": 0,
      "end": 10,
      "confidence": 1.0
    },
    {
      "text": "Apple Inc.",
      "type": "ORGANIZATION",
      "start": 100,
      "end": 109,
      "confidence": 1.0
    }
  ],
  "relationships": [
    {
      "subject": "Steve Jobs",
      "predicate": "EMPLOYED_BY",
      "object": "Apple Inc.",
      "confidence": 1.0
    }
  ]
}
```

### Storage Location

```
oracle/
├── test_corpus/
│   ├── wikipedia_steve_jobs.pdf
│   ├── news_article_tech_acquisition.pdf
│   ├── financial_report_quarterly.pdf
│   ├── research_paper_ml.pdf
│   ├── historical_document_treaty.pdf
│   ├── legal_document_contract.pdf
│   ├── screenshot_ui.png
│   ├── podcast_episode.mp3
│   ├── tutorial_video.mp4
│   └── blog_post.html
│
└── test_corpus_ground_truth/
    ├── wikipedia_steve_jobs.json
    ├── news_article_tech_acquisition.json
    ├── financial_report_quarterly.json
    ├── research_paper_ml.json
    ├── historical_document_treaty.json
    ├── legal_document_contract.json
    ├── screenshot_ui.json
    ├── podcast_episode.json
    ├── tutorial_video.json
    └── blog_post.json
```

## Validation Tests

### Test 1: Parsing Validation

**Goal:** Verify all files parse correctly.

```python
# tests/validation/test_parsing.py

import pytest
from pathlib import Path
from oracle.ingestion.parser import DocumentParser

@pytest.fixture
def parser():
    return DocumentParser()

@pytest.mark.parametrize("file_name,expected_type", [
    ("wikipedia_steve_jobs.pdf", "pdf"),
    ("news_article_tech_acquisition.pdf", "pdf"),
    ("financial_report_quarterly.pdf", "pdf"),
    ("research_paper_ml.pdf", "pdf"),
    ("historical_document_treaty.pdf", "pdf"),
    ("legal_document_contract.pdf", "pdf"),
    ("screenshot_ui.png", "image"),
    ("podcast_episode.mp3", "audio"),
    ("tutorial_video.mp4", "video"),
    ("blog_post.html", "text"),
])
async def test_parse_file(parser, file_name, expected_type):
    """Test that each file parses correctly."""
    corpus_dir = Path("oracle/test_corpus")
    file_path = corpus_dir / file_name
    
    result = await parser.parse(file_path)
    
    assert result.file_type == expected_type
    assert len(result.markdown_content) > 0
    assert result.page_count > 0
```

### Test 2: Entity Extraction Validation

**Goal:** Verify GLiNER2 extracts expected entities.

```python
# tests/validation/test_entity_extraction.py

import pytest
import json
from pathlib import Path
from oracle.graph.entity_extractor import GLiNEREntityExtractor

@pytest.fixture
def extractor():
    return GLiNEREntityExtractor(
        model="fastino/gliner2-base-v1",
        device="cuda"
    )

@pytest.mark.parametrize("file_name", [
    "wikipedia_steve_jobs.pdf",
    "news_article_tech_acquisition.pdf",
    "financial_report_quarterly.pdf",
])
async def test_entity_extraction(extractor, file_name):
    """Test that entity extraction matches ground truth."""
    # Load ground truth
    ground_truth_path = Path("oracle/test_corpus_ground_truth") / f"{file_name.stem}.json"
    with open(ground_truth_path, 'r') as f:
        ground_truth = json.load(f)
    
    # Load parsed document
    corpus_dir = Path("oracle/test_corpus")
    file_path = corpus_dir / file_name
    
    parser = DocumentParser()
    parsed = await parser.parse(file_path)
    
    # Extract entities
    entities = extractor.extract(parsed.markdown_content, ["person", "organization", "location", "date", "amount"])
    
    # Compare with ground truth
    ground_truth_entities = ground_truth["entities"]
    
    # Calculate precision and recall
    true_positives = 0
    false_positives = 0
    false_negatives = 0
    
    for gt_entity in ground_truth_entities:
        found = False
        for pred_entity in entities:
            if pred_entity["text"] == gt_entity["text"] and pred_entity["type"] == gt_entity["type"]:
                found = True
                true_positives += 1
                break
        if not found:
            false_negatives += 1
    
    for pred_entity in entities:
        found = False
        for gt_entity in ground_truth_entities:
            if pred_entity["text"] == gt_entity["text"] and pred_entity["type"] == gt_entity["type"]:
                found = True
                break
        if not found:
            false_positives += 1
    
    precision = true_positives / (true_positives + false_positives) if (true_positives + false_positives) > 0 else 0
    recall = true_positives / (true_positives + false_negatives) if (true_positives + false_negatives) > 0 else 0
    
    assert precision > 0.85, f"Precision {precision:.2f} below threshold for {file_name}"
    assert recall > 0.80, f"Recall {recall:.2f} below threshold for {file_name}"
```

### Test 3: End-to-End Validation

**Goal:** Verify full ingestion pipeline works on test corpus.

```python
# tests/validation/test_e2e.py

import pytest
from pathlib import Path
from oracle.ingestion.pipeline import IngestionPipeline

@pytest.fixture
async def pipeline(tmp_path):
    pipeline = IngestionPipeline(
        corpus_dir=Path("oracle/test_corpus"),
        ledger_db=tmp_path / "ledger.db",
    )
    await pipeline.initialize()
    yield pipeline
    await pipeline.close()

async def test_full_ingestion(pipeline):
    """Test full ingestion on test corpus."""
    # Run ingestion
    await pipeline.run()
    
    # Verify all files processed
    corpus_dir = Path("oracle/test_corpus")
    file_count = len(list(corpus_dir.glob("*")))
    
    for file_path in corpus_dir.glob("*"):
        status = await pipeline.ledger.get_status(file_path)
        assert status == "complete", f"File {file_path.name} not complete"
    
    # Verify chunks in Qdrant
    chunks = await pipeline.qdrant.search(
        collection_name="oracle_corpus",
        query_vector=[0] * 1024,
        limit=1000
    )
    assert len(chunks) > 0, "No chunks in Qdrant"
    
    # Verify entities in Neo4j
    async with pipeline.neo4j_driver.session() as session:
        result = await session.run("MATCH (e:Entity) RETURN count(e) AS count")
        count = (await result.single())["count"]
        assert count > 0, "No entities in Neo4j"
    
    # Verify relationships in Neo4j
    async with pipeline.neo4j_driver.session() as session:
        result = await session.run("MATCH ()-[r]->() RETURN count(r) AS count")
        count = (await result.single())["count"]
        assert count > 0, "No relationships in Neo4j"
```

## Corpus Creation

### Automated Generation

```python
# scripts/generate_test_corpus.py

"""
Generate synthetic test corpus with known ground truth entities.
"""

import json
from pathlib import Path
from datetime import datetime

def generate_wikipedia_article():
    """Generate synthetic Wikipedia article about Steve Jobs."""
    content = """
    Steve Jobs
    ===========
    
    Steven Paul Jobs (February 24, 1955 – October 5, 2011) was an American business magnate, 
    industrial designer, investor, and media proprietor. He was the co-founder, chairman, 
    and CEO of Apple; the chairman and majority shareholder of Pixar; a member of The Walt 
    Disney Company's board of directors following its acquisition of Pixar; and the founder, 
    chairman, and CEO of NeXT.
    
    Jobs was born in San Francisco to a Syrian father and German-American mother. He was 
    adopted shortly after his birth. He attended Reed College in 1972 before dropping out 
    that same year. He then traveled through India seeking enlightenment before returning 
    to the United States.
    
    In 1976, Jobs co-founded Apple Computer in the family home's garage with Steve Wozniak 
    and Ronald Wayne. The two gained fame and wealth a year later with the production and 
    sale of the Apple II, one of the first highly successful mass-produced microcomputers.
    
    """
    
    ground_truth = {
        "file": "wikipedia_steve_jobs.pdf",
        "file_type": "pdf",
        "entities": [
            {"text": "Steve Jobs", "type": "PERSON", "start": 0, "end": 10, "confidence": 1.0},
            {"text": "Steven Paul Jobs", "type": "PERSON", "start": 30, "end": 46, "confidence": 1.0},
            {"text": "February 24, 1955", "type": "DATE", "start": 48, "end": 64, "confidence": 1.0},
            {"text": "October 5, 2011", "type": "DATE", "start": 67, "end": 81, "confidence": 1.0},
            {"text": "Apple", "type": "ORGANIZATION", "start": 200, "end": 205, "confidence": 1.0},
            {"text": "Pixar", "type": "ORGANIZATION", "start": 280, "end": 285, "confidence": 1.0},
            {"text": "The Walt Disney Company", "type": "ORGANIZATION", "start": 320, "end": 342, "confidence": 1.0},
            {"text": "NeXT", "type": "ORGANIZATION", "start": 450, "end": 454, "confidence": 1.0},
            {"text": "San Francisco", "type": "LOCATION", "start": 500, "end": 513, "confidence": 1.0},
            {"text": "Reed College", "type": "ORGANIZATION", "start": 600, "end": 611, "confidence": 1.0},
            {"text": "India", "type": "LOCATION", "start": 700, "end": 705, "confidence": 1.0},
            {"text": "United States", "type": "LOCATION", "start": 750, "end": 763, "confidence": 1.0},
            {"text": "1976", "type": "DATE", "start": 800, "end": 804, "confidence": 1.0},
            {"text": "Steve Wozniak", "type": "PERSON", "start": 900, "end": 912, "confidence": 1.0},
            {"text": "Ronald Wayne", "type": "PERSON", "start": 917, "end": 929, "confidence": 1.0},
            {"text": "Apple II", "type": "DOCUMENT", "start": 1000, "end": 1007, "confidence": 1.0},
        ],
        "relationships": [
            {"subject": "Steve Jobs", "predicate": "EMPLOYED_BY", "object": "Apple", "confidence": 1.0},
            {"subject": "Steve Jobs", "predicate": "EMPLOYED_BY", "object": "Pixar", "confidence": 1.0},
            {"subject": "Steve Jobs", "predicate": "EMPLOYED_BY", "object": "NeXT", "confidence": 1.0},
            {"subject": "Steve Jobs", "predicate": "OCCURRED_ON", "object": "February 24, 1955", "confidence": 1.0},
            {"subject": "Steve Jobs", "predicate": "LOCATED_AT", "object": "San Francisco", "confidence": 1.0},
        ]
    }
    
    return content, ground_truth

def generate_all_files():
    """Generate all test corpus files."""
    corpus_dir = Path("oracle/test_corpus")
    ground_truth_dir = Path("oracle/test_corpus_ground_truth")
    
    corpus_dir.mkdir(parents=True, exist_ok=True)
    ground_truth_dir.mkdir(parents=True, exist_ok=True)
    
    # Generate each file
    files = [
        ("wikipedia_steve_jobs.pdf", generate_wikipedia_article),
        # ... more files
    ]
    
    for file_name, generator in files:
        content, ground_truth = generator()
        
        # Save content (would convert to PDF, etc.)
        # ...
        
        # Save ground truth
        ground_truth_path = ground_truth_dir / f"{Path(file_name).stem}.json"
        with open(ground_truth_path, 'w') as f:
            json.dump(ground_truth, f, indent=2)

if __name__ == "__main__":
    generate_all_files()
```

## Success Criteria

### Phase 1 Success
- All 10 files parse correctly
- All 10 files chunk correctly
- All 10 files embed correctly
- All 10 files extract entities correctly
- GLiNER2 precision > 0.85 on ground truth
- GLiNER2 recall > 0.80 on ground truth
- End-to-end ingestion test passes

### Validation Success
- All parsing tests pass (10/10)
- All entity extraction tests pass (3/3)
- End-to-end validation test passes
- No data corruption
- No duplicate processing
- All entities and relationships in Neo4j
- All chunks in Qdrant

---

**Approved by:** Hermes Agent  
**Next Action:** Generate test corpus files and run validation tests
