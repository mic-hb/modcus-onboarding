# LLM Logger Foreign Key Constraint Fix

**Date**: 2026-02-08  
**Version**: v0.2.1  
**Status**: Completed  
**Priority**: Critical (Bug Fix)

## Summary

Fixed a critical database error where `StoredFile` records failed to create due to foreign key constraint violations. The error occurred because `StoredFile` records were being created before their referenced `LLMCall` records existed in the database.

## The Problem

When the ingestion pipeline attempted to log LLM calls and store prompts/responses, it failed with:

```
psycopg2.errors.ForeignKeyViolation: insert or update on table "stored_file" 
violates foreign key constraint "fk_stored_file_llm_call_id"
DETAIL:  Key (llm_call_id)=(6fa38678-4997-4ef1-a79e-8d30426b35eb) is not present in table "llm_call".
```

This error occurred in `modcus_common/services/llm/llm_logger.py` when creating `StoredFile` records to persist full prompts and responses during embedding operations.

## Root Cause

The `LLMCallLogger` had a transaction ordering bug:

```python
# BEFORE (Broken - Transaction Ordering Bug)
async def log_call_async(...):
    llm_call_id = str(uuid4())  # Generate ID only
    
    # ❌ Creating StoredFiles BEFORE LLMCall exists
    prompt_file_id, response_file_id = await self._store_full_content_async(
        full_prompt, response_text, llm_call_id  # References non-existent LLMCall!
    )
    
    # LLMCall created AFTER StoredFiles (too late!)
    llm_call = LLMCall(
        id=llm_call_id,
        ...
        prompt_file_id=prompt_file_id,
        response_file_id=response_file_id,
    )
    # StoredFile FK constraint fails because LLMCall doesn't exist yet
```

The issue: `StoredFile` has a foreign key constraint on `llm_call_id`, but the `LLMCall` record was created in a **different database session** after the `StoredFile` records were already committed.

## The Fix

Reordered the transaction to create `LLMCall` **before** `StoredFile`:

```python
# AFTER (Fixed - Correct Transaction Order)
async def log_call_async(...):
    llm_call_id = str(uuid4())
    
    # 1. ✅ Create LLMCall FIRST (satisfies FK constraint)
    llm_call = LLMCall(
        id=llm_call_id,
        ...
        prompt_file_id=None,  # Will update later
        response_file_id=None,  # Will update later
    )
    
    async with self.session_factory() as session:
        session.add(llm_call)
        await session.commit()  # LLMCall now exists in DB
        await session.refresh(llm_call)
    
    # 2. ✅ Now create StoredFiles (FK constraint will pass)
    prompt_file_id, response_file_id = await self._store_full_content_async(
        full_prompt, response_text, llm_call_id  # LLMCall exists!
    )
    
    # 3. ✅ Update LLMCall with StoredFile references
    if prompt_file_id or response_file_id:
        async with self.session_factory() as session:
            llm_call = await session.get(LLMCall, llm_call_id)
            if llm_call:
                llm_call.prompt_file_id = prompt_file_id
                llm_call.response_file_id = response_file_id
                await session.commit()
```

**Files Modified**:
- `modcus_common/services/llm/llm_logger.py:263-350` (async version)
- `modcus_common/services/llm/llm_logger.py:405-505` (sync version)

## Changes Made

### 1. Async Log Call Method

**File**: `modcus_common/services/llm/llm_logger.py`

**Before**:
```python
async def log_call_async(...):
    llm_call_id = str(uuid4())
    
    # Create StoredFiles first (WRONG ORDER)
    prompt_file_id, response_file_id = await self._store_full_content_async(
        full_prompt or "", response_text, llm_call_id
    )
    
    # Create LLMCall after (too late for FK constraint)
    llm_call = LLMCall(..., prompt_file_id=prompt_file_id, ...)
    async with self.session_factory() as session:
        session.add(llm_call)
        await session.commit()
```

**After**:
```python
async def log_call_async(...):
    llm_call_id = str(uuid4())
    
    # 1. Create LLMCall FIRST
    llm_call = LLMCall(..., prompt_file_id=None, response_file_id=None)
    async with self.session_factory() as session:
        session.add(llm_call)
        await session.commit()
        await session.refresh(llm_call)
    
    # 2. Create StoredFiles (FK constraint satisfied)
    prompt_file_id, response_file_id = await self._store_full_content_async(
        full_prompt or "", response_text, llm_call_id
    )
    
    # 3. Update LLMCall with references
    if prompt_file_id or response_file_id:
        async with self.session_factory() as session:
            llm_call = await session.get(LLMCall, llm_call_id)
            if llm_call:
                llm_call.prompt_file_id = prompt_file_id
                llm_call.response_file_id = response_file_id
                await session.commit()
```

### 2. Sync Log Call Method

**File**: `modcus_common/services/llm/llm_logger.py`

Applied the same fix to the synchronous version (`log_call` method).

## Database Schema

**Tables Involved**:

### llm_call
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | VARCHAR | PRIMARY KEY | UUID generated by logger |
| `job_id` | VARCHAR | NULLABLE, FK | Related ingestion job |
| `prompt_file_id` | VARCHAR | NULLABLE | Reference to StoredFile |
| `response_file_id` | VARCHAR | NULLABLE | Reference to StoredFile |
| ... | ... | ... | ... |

### stored_file
| Column | Type | Constraints | Notes |
|--------|------|-------------|-------|
| `id` | VARCHAR | PRIMARY KEY | Auto-generated UUID |
| `llm_call_id` | VARCHAR | NULLABLE, **FK → llm_call.id** | **Must exist before insert** |
| `kind` | VARCHAR | NOT NULL | prompt/response/system |
| `file_path` | VARCHAR | NOT NULL | Path to stored content |
| ... | ... | ... | ... |

**Foreign Key Constraint**:
```sql
CONSTRAINT fk_stored_file_llm_call_id 
    FOREIGN KEY (llm_call_id) REFERENCES llm_call(id)
```

## Transaction Flow

### Before Fix (Broken)
```
1. Generate llm_call_id (UUID)
2. Create StoredFile with llm_call_id  ← ❌ FK VIOLATION!
3. Commit StoredFile                  ← Rollback happens here
4. Create LLMCall                     ← Never reached
```

### After Fix (Working)
```
1. Generate llm_call_id (UUID)
2. Create LLMCall with id=llm_call_id ← ✅ LLMCall exists
3. Commit LLMCall
4. Create StoredFile with llm_call_id ← ✅ FK constraint satisfied
5. Commit StoredFile
6. Update LLMCall with file references ← Optional
```

## Testing

### Test Scenario
1. Upload a document for ingestion
2. Pipeline reaches embedding stage
3. LLM service logs the embedding call
4. Creates LLMCall record
5. Creates StoredFile records for prompt/response
6. **Result**: ✅ Success - no FK constraint errors

### Test Results
**Before Fix**:
```
ERROR: insert or update on table "stored_file" violates foreign key constraint
DETAIL:  Key (llm_call_id)=(...) is not present in table "llm_call".
Job failed permanently
```

**After Fix**:
```
INFO: LLM call logged: id=6fa38678-4997-4ef1-a79e-8d30426b35eb, latency_ms=5923
INFO: StoredFile created: id=cd8a034d-3467-4e4e-9218-e547495b3503, kind=response
INFO: Job processing completed successfully
```

## Backward Compatibility

✅ **Fully backward compatible** - No breaking changes

- Existing database schema unchanged
- Existing code that uses LLMCallLogger works correctly
- New code follows correct transaction order
- No migration needed

## Related Issues

This fix was discovered during the ingestion pipeline work:
- **Related**: DoclingParser modern format support (2026-02-08)
- **Related**: StoredFile UUID auto-generation fix (2026-02-08)
- **Related**: Ingestion pipeline error handling improvements (2026-02-08)
- **Related**: Comprehensive logging implementation (2026-02-08)

## Best Practices

### For Foreign Key Constraints

Always ensure referenced records exist before creating records with foreign keys:

```python
# Good - Create parent first
parent = Parent(id=parent_id, ...)
session.add(parent)
session.commit()  # Parent exists in DB

child = Child(parent_id=parent_id, ...)  # FK constraint satisfied
session.add(child)
session.commit()

# Bad - Create child before parent
child = Child(parent_id=parent_id, ...)  # ❌ Parent doesn't exist!
session.add(child)
session.commit()  # FK violation error

parent = Parent(id=parent_id, ...)  # Too late
```

### For Multi-Session Transactions

When using multiple sessions (e.g., async/sync mix), ensure proper commit order:

```python
# Good - Explicit session management
async def create_records():
    # Parent in session 1
    async with session_factory() as session1:
        parent = Parent(...)
        session1.add(parent)
        await session1.commit()
    
    # Child in session 2 (parent already committed)
    async with session_factory() as session2:
        child = Child(parent_id=parent.id, ...)
        session2.add(child)
        await session2.commit()

# Bad - Mixed sessions without proper ordering
async def create_records():
    # Child in session 2
    async with session_factory() as session2:
        child = Child(parent_id=parent_id, ...)  # ❌ Parent not committed yet!
        await session2.commit()
    
    # Parent in session 1
    async with session_factory() as session1:
        parent = Parent(id=parent_id, ...)  # Too late
        await session1.commit()
```

### For LLM Call Logging

The corrected pattern ensures data integrity:

1. **Always** create `LLMCall` record first
2. **Then** create `StoredFile` records with the `llm_call_id`
3. **Finally** update `LLMCall` with `StoredFile` references (optional)
4. Handle errors gracefully - if StoredFile creation fails, LLMCall still exists for debugging

## Impact

**Services Affected**:
- `modcus_api_ingest` - Document ingestion with embedding
- `modcus_api_query` - Query processing with LLM calls
- Any service using `LLMService` for embeddings or completions

**Functionality Restored**:
- ✅ LLM call logging works correctly
- ✅ Prompt/response storage works
- ✅ Foreign key constraints satisfied
- ✅ Audit trail maintained
- ✅ Database integrity preserved

## References

- SQLModel Documentation: https://sqlmodel.tiangolo.com/
- SQLAlchemy Session Management: https://docs.sqlalchemy.org/en/20/orm/session_basics.html
- PostgreSQL Foreign Keys: https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-FK
- Related Fix: StoredFile UUID Auto-Generation (2026-02-08)
- Related Fix: DoclingParser Modern Format Support (2026-02-08)

## Checklist

- [x] Identify root cause (transaction ordering bug)
- [x] Implement fix (reorder LLMCall creation)
- [x] Apply fix to both async and sync methods
- [x] Test fix with actual document ingestion
- [x] Update documentation
- [x] Verify backward compatibility
- [x] No database migration required
- [x] Update AGENTS.md with transaction ordering best practices
