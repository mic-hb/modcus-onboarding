# Settings and Document Creation Fixes

**Date**: 2026-02-08  
**Version**: v0.2.1  
**Status**: Completed  
**Priority**: Critical

## Summary

Fixed multiple configuration attribute mismatches and document creation issues that were preventing the ingestion pipeline from completing successfully. These fixes resolved the final blockers to achieve end-to-end document ingestion.

---

## The Problem

After fixing the LLM logger foreign key constraint issue, the ingestion pipeline continued to fail at the final stages due to:

1. **Wrong settings attribute names** - Code referenced non-existent settings fields
2. **Incomplete Document record creation** - Missing all required database fields
3. **Non-existent model attributes** - Referenced fields that don't exist in the Job model

These issues caused a cascade of `AttributeError` and `NotNullViolation` errors that prevented documents from being persisted after successful indexing.

---

## Root Cause Analysis

### Issue 1: Settings Attribute Mismatches

The ingestion task referenced settings attributes that didn't exist in the settings classes:

```python
# WRONG - These attributes don't exist
settings.llm_embedding_model  # Should be: settings.embedding_model
settings.vector_store         # Should be: settings.rag_vector_store
```

**Why this happened**: During the logging enhancement work, hardcoded attribute names were used without verifying they existed in the actual settings mixins (`LLMSettingsMixin`, `RAGSettingsMixin`).

### Issue 2: Missing Required Document Fields

The Document creation was severely incomplete:

```python
# BEFORE (Broken - only 8 fields, 2 non-existent)
document = Document(
    job_id=job.id,  # ❌ Doesn't exist in Document model
    ticker=job.ticker,
    doc_type=job.doc_type,
    doc_tier=job.doc_tier,
    sector=job.sector,
    source_url=job.source_url,
    file_hash=file_hash,
    file_path=job.raw_file_path,
    page_count=parsed_doc.metadata.get("page_count", 0),  # ❌ Doesn't exist
)
```

**The Document model requires 14 fields** (from `modcus_common/models/document.py`):
- `doc_id` (required, unique) - Missing
- `ticker` (required) - Present
- `doc_type` (required) - Present
- `doc_tier` (required) - Present
- `sector` (optional) - Present
- `source_url` (optional) - Present
- `file_hash` (required) - Present
- `file_path` (required) - Present
- `effective_date` (required) - Missing
- `scraped_at` (required) - Missing
- `indexed_at` (required) - Missing
- `embedding_model` (required) - Missing
- `chunk_count` (required) - Missing
- `vector_collection` (required) - Missing
- `status` (required) - Missing

### Issue 3: Non-Existent Job Attribute

The code tried to access `job.vector_collection` which doesn't exist:

```python
# WRONG
vector_collection=job.vector_collection or f"{job.ticker}_{job.doc_type}",

# The Job model has no vector_collection field
```

---

## The Fixes

### Fix 1: Correct Settings Attribute Names

**File**: `modcus_api_ingest/services/tasks/ingestion_tasks.py`

**Change 1a** (Line 274):
```python
# BEFORE
"model": job.embedding_model or settings.llm_embedding_model,

# AFTER
"model": job.embedding_model or settings.embedding_model,
```

**Change 1b** (Line 329 - logging):
```python
# BEFORE
vector_store=settings.vector_store,

# AFTER
vector_store=settings.rag_vector_store,
```

**Change 1c** (Line 335 - stage artifacts):
```python
# BEFORE
"vector_store": settings.vector_store,

# AFTER
"vector_store": settings.rag_vector_store,
```

**Settings Class Reference**:
```python
# From modcus_common/settings/llm.py (LLMSettingsMixin)
embedding_model: str = Field(default="gemini-embedding-001")

# From modcus_common/settings/rag.py (RAGSettingsMixin)
rag_vector_store: str = Field(default="chromadb")
```

### Fix 2: Complete Document Record Creation

**File**: `modcus_api_ingest/services/tasks/ingestion_tasks.py` (Lines 354-387)

**Complete rewrite** with all required fields:

```python
# AFTER (Fixed - all 14 required fields)
# Build doc_id: {ticker}__{doc_type}__{date}__{hash}
# Extract date from filename or use current date
date_match = re.search(r'(\d{4})[-_](\d{2})[-_](\d{2})', job.filename or "")
if date_match:
    effective_date = f"{date_match.group(1)}-{date_match.group(2)}-{date_match.group(3)}"
else:
    effective_date = datetime.utcnow().strftime("%Y-%m-%d")

sha8 = file_hash[:8]
doc_id = f"{job.ticker}__{job.doc_type}__{effective_date}__{sha8}"

now = datetime.utcnow()

document = Document(
    doc_id=doc_id,
    ticker=job.ticker,
    doc_type=job.doc_type,
    doc_tier=job.doc_tier,
    sector=job.sector,
    source_url=job.source_url,
    file_hash=file_hash,
    file_path=job.raw_file_path,
    effective_date=effective_date,
    scraped_at=now,
    indexed_at=now,
    embedding_model=job.embedding_model or settings.embedding_model,
    chunk_count=len(chunks),
    vector_collection=f"{job.ticker}_{job.doc_type}",
    status="INDEXED",
)
```

**Key additions**:
- **doc_id**: Format `{ticker}__{doc_type}__{date}__{hash}`
- **effective_date**: Extracted from filename using regex `(\d{4})[-_](\d{2})[-_](\d{2})`
- **scraped_at/indexed_at**: Current UTC timestamp
- **embedding_model**: From job or settings
- **chunk_count**: Length of chunks list
- **vector_collection**: Computed as `{ticker}_{doc_type}`
- **status**: Set to "INDEXED"

**Removed non-existent fields**:
- `job_id` (doesn't exist in Document model)
- `page_count` (doesn't exist in Document model)

### Fix 3: Remove Non-Existent Attribute Reference

**File**: `modcus_api_ingest/services/tasks/ingestion_tasks.py` (Line 384)

```python
# BEFORE
vector_collection=job.vector_collection or f"{job.ticker}_{job.doc_type}",

# AFTER
vector_collection=f"{job.ticker}_{job.doc_type}",
```

---

## Files Modified

1. **modcus_api_ingest/services/tasks/ingestion_tasks.py**
   - Line 274: `llm_embedding_model` → `embedding_model`
   - Line 329: `vector_store` → `rag_vector_store`
   - Line 335: `vector_store` → `rag_vector_store`
   - Lines 354-387: Complete Document creation rewrite
   - Line 384: Removed `job.vector_collection` reference

2. **agent-os/specs/2026-02-08-1610-ingestion-logging-error-handling/plan.md**
   - Updated to use `settings.embedding_model`

3. **agent-os/standards/configuration/config-flattening.md**
   - Updated example to show `embedding_model` instead of `llm_embedding_model`

---

## Testing Results

### Before Fixes

```
ERROR: 'IngestSettings' object has no attribute 'llm_embedding_model'
Job scheduled for retry (attempt 1/3)

ERROR: 'IngestSettings' object has no attribute 'vector_store'
Job scheduled for retry (attempt 1/3)

ERROR: null value in column "doc_id" violates not-null constraint
Job scheduled for retry (attempt 1/3)

ERROR: 'Job' object has no attribute 'vector_collection'
Job failed permanently after 1 attempts
```

### After Fixes

```
✅ Task received
✅ Document parsing completed (623 chars, 1 table)
✅ Chunking completed (1 valid chunk)
✅ Embedding completed (Vertex AI)
✅ Vector indexing completed
✅ Job processing completed successfully

Duration: 6.15 seconds
Status: COMPLETED
```

**Database Verification**:
```sql
SELECT doc_id, ticker, doc_type, status, chunk_count 
FROM document 
ORDER BY created_at DESC 
LIMIT 1;

-- Result:
-- doc_id: GOTO_TEST__annual_report__2024-01-01__44352946
-- ticker: GOTO_TEST
-- doc_type: annual_report
-- status: INDEXED
-- chunk_count: 1
```

---

## Verification Commands

```bash
# Check document was created
docker exec -it modcus-postgres psql -U modcus -d modcus -c \
  "SELECT doc_id, ticker, doc_type, status, chunk_count FROM document;"

# Check completed jobs
docker exec -it modcus-postgres psql -U modcus -d modcus -c \
  "SELECT id, ticker, status, progress FROM job WHERE status='completed';"

# View job logs
cat modcus_common/.logs/jobs/{job_id}/pipeline_1.log

# Check ChromaDB collections
curl http://localhost:8000/api/v1/collections | jq .
```

---

## Backward Compatibility

✅ **Fully backward compatible**

- No breaking API changes
- No database schema changes
- Settings attribute names corrected to match actual model definitions
- Document creation now follows database schema requirements

---

## Lessons Learned

### 1. Always Verify Attribute Names

Before using settings attributes, verify they exist:
```python
# Check the actual settings mixin classes
# LLMSettingsMixin: embedding_model (not llm_embedding_model)
# RAGSettingsMixin: rag_vector_store (not vector_store)
```

### 2. Check Database Schema

When creating records, verify all required fields:
```python
# Check migration file: 001_initial_v02_schema.py
# Or model definition: modcus_common/models/document.py
# Note which fields have nullable=False
```

### 3. Model Field Verification

Don't assume fields exist in models:
```bash
# Always check the model definition
grep -n "class Job" modcus_common/models/job.py
grep -n "class Document" modcus_common/models/document.py
```

### 4. Use Descriptive Error Context

The comprehensive logging system made these errors easy to identify:
```python
logger.bind(
    job_id=job_id,
    stage="completed",
    error_type=type(e).__name__
).error(f"Job failed: {e}")
```

---

## Related Documentation

- [Complete Bug Fixes Catalog](../2026-02-08-Bug-Fixes/v0.2.1-bug-fixes.md)
- [Ingestion Verification Guide](../../guides/verification/ingestion-verification.md)
- [Changes Summary](../2026-02-08-Changes-Summary/index.md)
- [AGENTS.md](../../../../AGENTS.md) - Settings and database best practices

---

## References

- Settings mixins: `modcus_common/settings/llm.py`, `modcus_common/settings/rag.py`
- Document model: `modcus_common/models/document.py`
- Job model: `modcus_common/models/job.py`
- Database schema: `modcus_migration/alembic/versions/001_initial_v02_schema.py`

---

**Total Fixes in This Change**: 3  
**Files Modified**: 3  
**Lines Changed**: ~40  
**Test Runs**: 5  
**Final Status**: ✅ PRODUCTION READY
