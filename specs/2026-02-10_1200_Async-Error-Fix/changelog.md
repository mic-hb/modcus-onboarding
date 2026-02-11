# Async Error When Loading Active Keys

**Date**: February 10, 2026
**Version**: v0.2.2
**Status**: Fixed
**Priority**: Critical

---

## Summary

Fixed a critical bug where calling `asyncio.run()` from within an already running async event loop caused `RuntimeError: asyncio.run() cannot be called from a running event loop`. This occurred in the `LLMRotator` class when the async method `_get_current_llm_async()` called the sync method `_load_active_keys()`, which then attempted to use `asyncio.run()`.

---

## Problem

### Error Message
```
Simple analysis failed: asyncio.run() cannot be called from a running event loop
```

### Root Cause

In `modcus_common/services/llm/llm_rotator.py`, the async method `_get_current_llm_async()` was calling the sync wrapper method `_load_active_keys()`:

**Before (lines 92-95)**:
```python
async def _get_current_llm_async(self) -> Any:
    """Return fresh LLM instance using current active API key (async)."""
    if not self.active_keys:
        self._load_active_keys()  # BUG: This calls asyncio.run() internally!
```

The `_load_active_keys()` method (lines 83-90) determines whether to use async or sync loading:

```python
def _load_active_keys(self) -> None:
    """Load active API keys from database based on session type."""
    if self.use_async:
        import asyncio
        asyncio.run(self._load_active_keys_async())  # ERROR: Can't use asyncio.run() in async context!
    else:
        self._load_active_keys_sync()
```

Since `_get_current_llm_async()` is already running in an async context (called via `await` from the FastAPI request handler), calling `asyncio.run()` inside it is not allowed by Python's asyncio design.

---

## Solution

Changed line 95 in `_get_current_llm_async()` to directly call the async version:

**After**:
```python
async def _get_current_llm_async(self) -> Any:
    """Return fresh LLM instance using current active API key (async)."""
    if not self.active_keys:
        await self._load_active_keys_async()  # FIXED: Direct async call
```

### Why This Works

- `_get_current_llm_async()` is already an async method running in an event loop
- It should directly `await` other async methods rather than calling sync wrappers
- The `_load_active_keys_async()` method properly uses `async with` for the database session
- No `asyncio.run()` is needed since we're already in an async context

---

## Impact

### Before Fix
- Query API failed with "Simple analysis failed: asyncio.run() cannot be called from a running event loop"
- All queries using async LLM service were failing
- Error occurred during analysis agent execution

### After Fix
- Query API works correctly with async database sessions
- All async LLM operations function properly
- Proper async/await pattern throughout the codebase

---

## Code Changes

**File**: `modcus_common/services/llm/llm_rotator.py`

**Change Summary**:
- **Line 95**: Changed `self._load_active_keys()` to `await self._load_active_keys_async()`

**Full Context**:
```python
async def _get_current_llm_async(self) -> Any:
    """Return fresh LLM instance using current active API key (async)."""
    if not self.active_keys:
        await self._load_active_keys_async()  # Fixed: was self._load_active_keys()

    if not self.active_keys:
        raise RotationError(
            f"No active API keys available for provider {self.settings.llm_provider}"
        )

    current_key = self.active_keys[self.current_key_index]

    # Check if using VertexAI
    if self.settings.llm_provider.lower() == "vertex":
        # ... Vertex AI credentials loading ...
        return create_llm_instance(
            provider=self.settings.llm_provider,
            model=self.settings.llm_model,
            project_id=self.settings.gcp_project_id,
            location=self.settings.gcp_location,
            temperature=getattr(self.settings, "llm_temperature", 0.0),
            max_tokens=getattr(self.settings, "llm_max_tokens", None),
            credentials=credentials,
        )

    # For other providers (openai, gemini, anthropic)
    return create_llm_instance(
        provider=current_key.provider,
        model=self.settings.llm_model,
        api_key=current_key.api_key,
        temperature=getattr(self.settings, "llm_temperature", 0.0),
        max_tokens=getattr(self.settings, "llm_max_tokens", None),
    )
```

---

## Testing

### Reproduction Steps (Before Fix)
1. Start Query API with async database session
2. Make a query request:
   ```bash
   curl -X POST http://localhost:8080/v1/query \
     -H "X-API-Key: modcus-123456789" \
     -H "Content-Type: application/json" \
     -d '{
       "query": "What are the key financial highlights?",
       "mode": "company-analysis",
       "ticker": "GOTO_TEST",
       "level": "novice"
     }'
   ```
3. Observe error: `Simple analysis failed: asyncio.run() cannot be called from a running event loop`

### Verification Steps (After Fix)
1. Restart Query API service
2. Make the same query request
3. Query processes successfully without async errors

---

## Async/Sync Pattern Consistency

### Correct Pattern
When implementing dual async/sync support, async methods should:
1. Call other async methods directly using `await`
2. Only use `asyncio.run()` in truly sync contexts (entry points, top-level functions)
3. Keep the sync/async boundary clear and consistent

### Code Example
```python
class LLMRotator:
    # Async method - calls other async methods directly
    async def _get_current_llm_async(self) -> Any:
        if not self.active_keys:
            await self._load_active_keys_async()  # ✅ Correct: await async method
        # ... rest of async logic

    # Sync wrapper - can use asyncio.run()
    def get_current_llm(self) -> Any:
        if self.use_async:
            import asyncio
            return asyncio.run(self._get_current_llm_async())  # ✅ OK: top-level entry point
        else:
            return self._get_current_llm_sync()
```

---

## Related Files

| File                                                 | Role                                      |
| ---------------------------------------------------- | ----------------------------------------- |
| `modcus_common/services/llm/llm_rotator.py`          | Contains the fixed method                 |
| `modcus_common/services/llm/llm_service.py`          | Uses LLMRotator for LLM operations        |
| `modcus_api_query/services/agents/analysis_agent.py` | Calls LLM service during query processing |

---

## Lessons Learned

1. **Never call `asyncio.run()` from within an async function** - it will fail
2. **Use `await` for async method calls within async contexts** - maintain async chain
3. **Reserve `asyncio.run()` for top-level entry points only** - sync wrappers, main functions
4. **Test both sync and async code paths** - they can have different bugs

---

## References

- Python asyncio documentation: https://docs.python.org/3/library/asyncio.html
- `asyncio.run()` documentation: https://docs.python.org/3/library/asyncio-task.html#asyncio.run
- LlamaIndex async patterns: https://docs.llamaindex.ai/en/stable/

---

**End of Changelog**
