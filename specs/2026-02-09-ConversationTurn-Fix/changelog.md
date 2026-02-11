# ConversationTurn NameError Fix

**Date**: February 9, 2026  
**Version**: v0.2.2  
**Status**: Completed  
**Priority**: Critical

---

## Summary

Fixed a `NameError: name 'ConversationTurn' is not defined` that occurred when processing queries through the Query API orchestration pipeline. This was a runtime error caused by a TYPE_CHECKING-only import being evaluated at runtime.

---

## Problem Description

### Error Message
```
Query processing error: name 'ConversationTurn' is not defined
```

### Root Cause
In `modcus_api_query/services/orchestration/state.py`, the `ConversationTurn` model was imported only under a `TYPE_CHECKING` guard:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from modcus_common.models.conversation import ConversationTurn
```

While the type annotation on line 69 used a string forward reference:
```python
conversation_history: Optional[List["ConversationTurn"]]
```

The issue occurred because:
1. In certain Python execution contexts or versions, the `TypedDict` type annotations are evaluated at runtime
2. When `from __future__ import annotations` is used (or in certain Python versions), all annotations are evaluated at runtime
3. The `ConversationTurn` class was not available at runtime because it was only imported under `TYPE_CHECKING`

### Impact
- Query API returned 500 Internal Server Error for all query requests
- The orchestration graph could not be instantiated
- Service health check passed, but query endpoint was non-functional

---

## Solution

Changed the import from TYPE_CHECKING-only to a regular import:

**File**: `modcus_api_query/services/orchestration/state.py`

**Before** (lines 8-12):
```python
from typing import TypedDict, Optional, Dict, Any, List, TYPE_CHECKING
from datetime import datetime

if TYPE_CHECKING:
    from modcus_common.models.conversation import ConversationTurn
```

**After**:
```python
from typing import TypedDict, Optional, Dict, Any, List
from datetime import datetime
from modcus_common.models.conversation import ConversationTurn
```

---

## Technical Details

### Why TYPE_CHECKING Failed

The `TYPE_CHECKING` constant is `False` at runtime and only `True` during type checking. While string forward references (like `"ConversationTurn"`) are typically not evaluated until needed, `TypedDict` classes have special behavior:

1. `TypedDict` is processed at class definition time to create the type structure
2. Some Python versions or configurations evaluate string annotations during this processing
3. When the annotation is evaluated, the referenced class must be available in the namespace

### Best Practice Going Forward

For `TypedDict` definitions that reference model classes:
- Use regular imports (not TYPE_CHECKING-only) for any types used in annotations
- String forward references help with circular imports but don't eliminate the need for runtime availability
- Alternative: Use `from __future__ import annotations` and ensure proper lazy evaluation

---

## Testing

### Verification Steps

1. **Restart Query Service**
   ```bash
   docker compose -f docker-compose.query.yml restart modcus-api-query
   ```

2. **Test Health Endpoint**
   ```bash
   curl http://localhost:8080/v1/health
   ```
   Expected: `{"status": "healthy", "index_status": "available"}`

3. **Test Query Endpoint**
   ```bash
   curl -X POST http://localhost:8080/v1/query \
     -H "X-API-Key: your-api-key" \
     -H "Content-Type: application/json" \
     -d '{
       "query": "What is Apple revenue?",
       "mode": "company-analysis",
       "level": "novice"
     }'
   ```
   Expected: 200 OK with JSON response (not 500 error)

4. **Check Logs**
   ```bash
   docker logs modcus-api-query 2>&1 | grep -i error
   ```
   Expected: No ConversationTurn-related errors

---

## Files Modified

| File | Lines Changed | Change Type |
|------|---------------|-------------|
| `modcus_api_query/services/orchestration/state.py` | 4 | Import statement fix |

---

## Backward Compatibility

✅ **Fully backward compatible** - This is a bug fix with no API changes:
- No changes to request/response schemas
- No changes to database schema
- No configuration changes required
- Existing code that didn't hit this edge case continues to work

---

## Related Issues

This fix was discovered during v0.2.2 development while implementing:
- Query API logging system
- End-to-end query processing tests

The error manifested when the Query service attempted to process its first query after startup.

---

## References

- Python Documentation: [typing.TYPE_CHECKING](https://docs.python.org/3/library/typing.html#typing.TYPE_CHECKING)
- Python Documentation: [TypedDict](https://docs.python.org/3/library/typing.html#typing.TypedDict)
- PEP 563: [Postponed Evaluation of Annotations](https://peps.python.org/pep-0563/)
- `modcus_common/models/conversation.py` - ConversationTurn model definition

---

## Prevention

To prevent similar issues in the future:

1. **Code Review Checklist**: When reviewing TypedDict definitions, verify that:
   - All referenced types in annotations are available at runtime
   - TYPE_CHECKING imports are only used in type annotations that won't be evaluated at runtime
   - String forward references have corresponding imports

2. **Testing**: Ensure integration tests exercise the actual query pipeline, not just unit tests

3. **Static Analysis**: Consider adding a linting rule to catch TYPE_CHECKING-only imports used in TypedDict

---

**End of Changelog**
