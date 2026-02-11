# LLMCall Database UUID Type Fix

**Date**: February 10, 2026  
**Version**: v0.2.2  
**Status**: Fixed  
**Priority**: Critical

---

## Summary

Fixed a database type mismatch error where the `api_key_id` column in the `llm_call` table was defined as `UUID` type in the database schema, but the `LLMCall` model defined it as `str` type. This caused `asyncpg.exceptions.DatatypeMismatchError` when attempting to insert LLM call records with API key references.

---

## Problem

### Error Message
```
column "api_key_id" is of type uuid but expression is of type character varying
HINT:  You will need to rewrite or cast the expression.
```

### Root Cause

The `ApiKey` model defines `id` as `UUID` type:

**File**: `modcus_common/models/api_key.py`
```python
from uuid import UUID

class ApiKey(SQLModel, table=True):
    id: UUID = Field(primary_key=True)
    ...
```

However, the `LLMCall` model defined `api_key_id` as `str` type:

**File**: `modcus_common/models/llm_call.py` (Before)
```python
class LLMCall(SQLModel, table=True):
    ...
    api_key_id: Optional[str] = Field(default=None, foreign_key="apikey.id", index=True)
```

This schema mismatch caused PostgreSQL to reject inserts when the SQLAlchemy ORM tried to pass a string value to a UUID column.

### Error Context

The error occurred during LLM call logging in the Query API:

```python
# When logging an LLM call with an API key reference
llm_call = LLMCall(
    id=llm_call_id,
    api_key_id=api_key.id,  # UUID object from ApiKey model
    ...
)
# Insert fails because api_key_id expects str but receives UUID
```

**Stack Trace**:
```
asyncpg.exceptions.DatatypeMismatchError: column "api_key_id" is of type uuid but expression is of type character varying
```

---

## Solution

Updated the `LLMCall` model to use `UUID` type for `api_key_id` to match the foreign key reference:

**File**: `modcus_common/models/llm_call.py`

### Changes Made

**Before**:
```python
"""LLMCall model for tracking all LLM API calls with conversation context."""

from datetime import datetime
from typing import Optional

from sqlmodel import Field, SQLModel

class LLMCall(SQLModel, table=True):
    ...
    api_key_id: Optional[str] = Field(default=None, foreign_key="apikey.id", index=True)
```

**After**:
```python
"""LLMCall model for tracking all LLM API calls with conversation context."""

from datetime import datetime
from typing import Optional
from uuid import UUID

from sqlmodel import Field, SQLModel

class LLMCall(SQLModel, table=True):
    ...
    api_key_id: Optional[UUID] = Field(default=None, foreign_key="apikey.id", index=True)
```

### Key Changes
1. Added `from uuid import UUID` import
2. Changed `api_key_id` type from `Optional[str]` to `Optional[UUID]`

---

## Impact

### Before Fix
- Query API failed with database type mismatch error
- LLM call logging was completely broken
- All queries that attempted to log LLM calls failed
- Error occurred at the database layer during INSERT operations

### After Fix
- LLM call logging works correctly
- API key references are properly stored as UUIDs
- Database foreign key constraints are satisfied
- Query processing completes successfully

---

## Testing

### Reproduction Steps (Before Fix)

1. Start Query API with database configured
2. Ensure an API key exists in the database
3. Make a query request:
   ```bash
   curl -X POST http://localhost:8080/v1/query \
     -H "X-API-Key: modcus-123456789" \
     -H "Content-Type: application/json" \
     -d '{
       "query": "What are the financial highlights?",
       "mode": "company-analysis",
       "ticker": "TEST"
     }'
   ```
4. Observe error: `column "api_key_id" is of type uuid but expression is of type character varying`

### Verification Steps (After Fix)

1. Restart Query API service
2. Make the same query request
3. Query processes successfully without database errors
4. LLM call records are properly saved with UUID references

---

## Database Schema Consistency

### Correct Pattern

When defining foreign key relationships in SQLModel:

1. **Match the referenced column type exactly**
2. **Use the same Python type** as the referenced model
3. **Import UUID from the uuid module** for UUID columns

### Example

```python
# Referenced model
class ApiKey(SQLModel, table=True):
    id: UUID = Field(primary_key=True)  # UUID type

# Referencing model
class LLMCall(SQLModel, table=True):
    # Must match the type exactly
    api_key_id: Optional[UUID] = Field(default=None, foreign_key="apikey.id")
```

---

## Files Modified

| File | Lines Changed | Description |
|------|---------------|-------------|
| `modcus_common/models/llm_call.py` | 2 | Added UUID import and changed api_key_id type |

---

## Related Files

| File | Role |
|------|------|
| `modcus_common/models/api_key.py` | Defines ApiKey with UUID primary key |
| `modcus_common/models/llm_call.py` | Fixed: LLMCall with UUID foreign key |
| `modcus_common/services/llm/llm_logger.py` | Creates LLMCall records |
| `modcus_common/services/llm/llm_service.py` | Passes API key to logger |

---

## Lessons Learned

1. **Always match foreign key types** - The referencing column must match the referenced column's type exactly
2. **Use UUID type for UUID columns** - Don't use str as a substitute for UUID
3. **Check schema consistency** - Verify model definitions match database schema
4. **Test with real database** - Unit tests with SQLite might not catch PostgreSQL-specific type issues

---

## Migration Notes

No database migration required. The column type in the database was already UUID; this fix aligns the Python model with the existing schema.

---

## References

- SQLModel Documentation: https://sqlmodel.tiangolo.com/
- PostgreSQL UUID Type: https://www.postgresql.org/docs/current/datatype-uuid.html
- Python UUID Module: https://docs.python.org/3/library/uuid.html

---

**End of Changelog**
