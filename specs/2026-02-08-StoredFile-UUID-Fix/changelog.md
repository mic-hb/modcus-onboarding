# StoredFile UUID Auto-Generation Fix

**Date**: 2026-02-08  
**Version**: v0.2.1  
**Status**: Completed  
**Priority**: Critical (Bug Fix)

## Summary

Fixed a critical database error where `StoredFile` records could not be created because the `id` field was not being auto-generated, causing `NotNullViolation` errors during LLM call logging.

## The Problem

When the ingestion pipeline attempted to store LLM prompts and responses, it failed with:

```
psycopg2.errors.NotNullViolation: null value in column "id" of relation "stored_file" violates not-null constraint
DETAIL: Failing row contains (null, response, llm_calls/.../response.txt, 69233, ...)
```

This error occurred in `modcus_common/services/llm/llm_logger.py` when creating `StoredFile` records to persist full prompts and responses.

## Root Cause

The `StoredFile` model defined the `id` field without an auto-generation mechanism:

```python
# BEFORE (Broken)
id: str = Field(primary_key=True)  # No default_factory!
```

When the LLM logger created a `StoredFile` instance without explicitly providing an `id`, SQLAlchemy tried to insert a NULL value, violating the database's NOT NULL constraint.

## The Fix

Added `default_factory` to auto-generate UUIDs:

```python
# AFTER (Fixed)
from uuid import uuid4

id: str = Field(default_factory=lambda: str(uuid4()), primary_key=True)
```

**File**: `modcus_common/models/stored_file.py`

## Changes Made

### 1. Model Update

**File**: `modcus_common/models/stored_file.py`

```python
"""StoredFile model for tracking stored file artifacts."""

from datetime import datetime
from typing import Optional
from uuid import uuid4  # ADDED

from sqlmodel import Field, SQLModel


class StoredFile(SQLModel, table=True):
    """StoredFile model for tracking all stored file artifacts."""

    __tablename__ = "stored_file"

    # Primary key - auto-generate UUID  # CHANGED
    id: str = Field(default_factory=lambda: str(uuid4()), primary_key=True)
    
    # ... rest of the model unchanged
```

### 2. Impact

**Files Affected**:
- `modcus_common/models/stored_file.py` - Added UUID auto-generation
- `modcus_common/services/llm/llm_logger.py` - No changes needed (uses model correctly)
- All services that create `StoredFile` records - Now work without explicit ID assignment

**Functionality Restored**:
- ✅ LLM prompts are stored correctly
- ✅ LLM responses are stored correctly
- ✅ File checksums are tracked
- ✅ Audit trail is maintained
- ✅ No manual ID management required

## Database Schema

**Table**: `stored_file`

| Column        | Type      | Constraints          | Notes                                     |
| ------------- | --------- | -------------------- | ----------------------------------------- |
| `id`          | VARCHAR   | PRIMARY KEY          | Auto-generated UUID (NEW)                 |
| `kind`        | VARCHAR   | NOT NULL, INDEX      | File type: raw/parsed/log/prompt/response |
| `file_path`   | VARCHAR   | NOT NULL             | Path to stored file                       |
| `size_bytes`  | INTEGER   | NOT NULL, CHECK >= 0 | File size                                 |
| `checksum`    | VARCHAR   | NULLABLE             | SHA-256 hash                              |
| `job_id`      | VARCHAR   | NULLABLE, FK, INDEX  | Related job                               |
| `llm_call_id` | VARCHAR   | NULLABLE, FK, INDEX  | Related LLM call                          |
| `artifact_id` | VARCHAR   | NULLABLE, FK, INDEX  | Related ephemeral artifact                |
| `created_at`  | TIMESTAMP | NOT NULL             | Creation timestamp                        |

## Testing

### Test Scenario
1. Upload a document for ingestion
2. Pipeline reaches embedding stage
3. LLM service logs the embedding call
4. `StoredFile` records created for prompt/response
5. **Result**: ✅ Success - no null constraint errors

### Test Results
**Before Fix**:
```
ERROR: null value in column "id" violates not-null constraint
Job failed permanently
```

**After Fix**:
```
INFO: Embedding completed: 1 embeddings generated
Job processing completed successfully
```

## Backward Compatibility

✅ **Fully backward compatible** - No breaking changes

- Existing code that explicitly sets `id` still works
- Existing database records remain valid
- New records automatically get UUIDs
- No migration needed for existing data

## Related Issues

This fix was discovered during the ingestion logging enhancement work:
- **Related**: DoclingParser modern format support
- **Related**: Ingestion pipeline error handling improvements
- **Related**: Comprehensive logging implementation

## Best Practices

### For Future Model Changes

Always provide default values or auto-generation for primary keys:

```python
# Good - Auto-generate UUID
id: str = Field(default_factory=lambda: str(uuid4()), primary_key=True)

# Good - Auto-increment integer
id: int = Field(default=None, primary_key=True)  # SQLModel/SQLAlchemy handles auto-increment

# Bad - No default, will fail if not provided
id: str = Field(primary_key=True)  # ❌ Don't do this!
```

### For UUID Generation

Use `uuid4()` for random UUIDs:
- ✅ Collision-resistant
- ✅ No central coordination needed
- ✅ Standard practice for distributed systems

## References

- SQLModel Documentation: https://sqlmodel.tiangolo.com/
- UUID Specification: https://tools.ietf.org/html/rfc4122
- PostgreSQL UUID Type: https://www.postgresql.org/docs/current/datatype-uuid.html
- Related Fix: Ingestion Logging & Error Handling Enhancement (2026-02-08)

## Checklist

- [x] Identify root cause (missing default_factory)
- [x] Implement fix (add UUID auto-generation)
- [x] Test fix with actual document ingestion
- [x] Update documentation
- [x] Verify backward compatibility
- [x] No database migration required
