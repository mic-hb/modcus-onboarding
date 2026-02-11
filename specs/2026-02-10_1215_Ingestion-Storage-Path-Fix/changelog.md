# Ingestion Storage Path Mismatch Fix

**Date**: February 10, 2026
**Version**: v0.2.2
**Status**: Fixed
**Priority**: High

---

## Summary

Fixed a storage path mismatch issue where uploaded files couldn't be found by the Celery worker during ingestion processing. The Ingest API was saving files to a different location than where the Celery worker was looking, causing all ingestion jobs to fail at the parsing stage with "file not found" errors.

---

## Problem

### Error Message
```
Failed to read file /app/storage/uploads/{job_id}_{filename}: [Errno 2] No such file or directory
```

### Root Cause

In containerized development environments with bind mounts, the Ingest API and Celery worker had different views of the filesystem:

**The Path Mismatch**:

1. **Ingest API** (in dev container):
   - Code checked if `/app/storage` exists
   - In dev containers, `/app/storage` was NOT mounted (only mounted in production)
   - API fell back to local development path: `/workspace/modcus_common/.storage/uploads/`

2. **Celery Worker** (in dev container):
   - Read the stored `raw_file_path` from the database: `/app/storage/uploads/{job_id}_{filename}`
   - The file didn't exist at that path in the worker's filesystem
   - Job failed with "Unknown format" (actually "file not found")

**Code Before Fix** (`modcus_api_ingest/api/routes_ingestion.py`, lines 75-85):
```python
# Use /app/storage/uploads when in Docker, fallback to local for dev
storage_dir = Path("/app/storage/uploads")
if not storage_dir.parent.exists():
    # Local development fallback
    storage_dir = (
        Path(__file__).resolve().parents[2]
        / "modcus_common"
        / ".storage"
        / "uploads"
    )
os.makedirs(storage_dir, exist_ok=True)
```

**Issues**:
1. The existence check was checking if `/app/storage` exists, not if it's actually usable
2. In dev containers, `/app/storage` might exist but not be the shared volume
3. Different services could fall back to different paths based on their filesystem view
4. The path stored in `raw_file_path` wasn't consistent between API and worker

---

## Solution

Introduced a `STORAGE_PATH` environment variable that both services use to determine the storage location:

### 1. Updated Storage Path Logic

**File**: `modcus_common/services/llm/llm_rotator.py`
**File**: `modcus_api_ingest/api/routes_ingestion.py`

**New Implementation**:
```python
# Determine storage directory from environment or fallback
if os.environ.get("STORAGE_PATH"):
    storage_dir = Path(os.environ["STORAGE_PATH"]) / "uploads"
elif Path("/app/storage").exists():
    storage_dir = Path("/app/storage/uploads")
else:
    # Local development fallback
    storage_dir = (
        Path(__file__).resolve().parents[2]
        / "modcus_common"
        / ".storage"
        / "uploads"
    )
os.makedirs(storage_dir, exist_ok=True)
```

**Key Changes**:
1. Check `STORAGE_PATH` environment variable first
2. Only use `/app/storage` if `STORAGE_PATH` is not set AND `/app/storage` exists
3. Consistent fallback for local development

### 2. Updated Docker Compose Configuration

**File**: `docker-compose.dev.yml`

Added `STORAGE_PATH` environment variable to both services:

```yaml
# modcus-api-ingest service
environment:
  HOME: /workspace
  PYTHONPATH: /workspace
  SERVICE_NAME: ingest
  STORAGE_PATH: /workspace/modcus_common/.storage  # NEW

# celery-worker service
environment:
  HOME: /workspace
  PYTHONPATH: /workspace
  SERVICE_NAME: celery
  STORAGE_PATH: /workspace/modcus_common/.storage  # NEW
  SQLALCHEMY_POOL_CLASS: "null"
  CHROMADB_PERSIST_DIR: /workspace/modcus_common/.rag_storage/chroma_db
```

**Volume Mount** (consistent for both):
```yaml
volumes:
  - ./modcus_common/.storage:/workspace/modcus_common/.storage
```

---

## How It Works Now

### In Dev Containers
1. Both services have `STORAGE_PATH=/workspace/modcus_common/.storage`
2. Files are saved to: `/workspace/modcus_common/.storage/uploads/`
3. Worker reads from: `/workspace/modcus_common/.storage/uploads/`
4. **Same path, same mount, no mismatch!**

### In Production
1. `STORAGE_PATH` is not set
2. Both services use: `/app/storage/uploads/`
3. Files saved to: `/app/storage/uploads/`
4. Worker reads from: `/app/storage/uploads/`
5. **Consistent with Docker volume mounts**

### Local Development (non-Docker)
1. `STORAGE_PATH` is not set
2. `/app/storage` doesn't exist
3. Falls back to: `{project_root}/modcus_common/.storage/uploads/`
4. **Works for local Python execution**

---

## Testing

### Before Fix
```bash
# Upload a file
curl -X POST 'http://localhost:8081/v1/ingestion/upload' \
  -H 'X-API-Key: modcus-123456789' \
  -F 'file=@document.json' \
  -F 'ticker=TEST' \
  -F 'doc_type=annual_report'
# Response: {"job_id": "...", "status": "queued"}

# Check worker logs
docker compose logs celery-worker
# ERROR: Failed to read file /app/storage/uploads/...
```

### After Fix
```bash
# Upload a file
curl -X POST 'http://localhost:8081/v1/ingestion/upload' \
  -H 'X-API-Key: modcus-123456789' \
  -F 'file=@document.json' \
  -F 'ticker=TEST' \
  -F 'doc_type=annual_report'
# Response: {"job_id": "...", "status": "queued"}

# Check worker logs
docker compose logs celery-worker
# INFO: Starting document parsing
# INFO: Detecting format for file: /workspace/modcus_common/.storage/uploads/...
# SUCCESS: Job processed successfully
```

### Verification
1. **Check file exists**:
   ```bash
   docker compose exec celery-worker ls -la /workspace/modcus_common/.storage/uploads/
   ```

2. **Check both services see the same path**:
   ```bash
   docker compose exec modcus-api-ingest env | grep STORAGE
   docker compose exec celery-worker env | grep STORAGE
   # Both should output: STORAGE_PATH=/workspace/modcus_common/.storage
   ```

---

## Files Modified

| File                                        | Lines Changed | Description                                                |
| ------------------------------------------- | ------------- | ---------------------------------------------------------- |
| `modcus_api_ingest/api/routes_ingestion.py` | ~10           | Updated storage path logic to check `STORAGE_PATH` env var |
| `docker-compose.dev.yml`                    | ~2            | Added `STORAGE_PATH` to ingest and celery services         |

---

## Configuration

### Environment Variable

| Variable       | Description                     | Default                                |
| -------------- | ------------------------------- | -------------------------------------- |
| `STORAGE_PATH` | Base path for storage directory | None (uses `/app/storage` or fallback) |

### Examples

**Development (docker-compose.dev.yml)**:
```yaml
environment:
  STORAGE_PATH: /workspace/modcus_common/.storage
```

**Production (docker-compose.yml)**:
```yaml
# Not set - uses /app/storage
```

**Local Development (shell)**:
```bash
# Not set - uses fallback to project_root/modcus_common/.storage
python -m modcus_api_ingest.main
```

---

## Backward Compatibility

✅ **Fully backward compatible**:
- If `STORAGE_PATH` is not set, falls back to existing behavior
- Production Docker setups continue to work with `/app/storage`
- Local development continues to work with fallback path
- No breaking changes to API or database schema

---

## Migration Notes

No migration required. To deploy:

```bash
# Development
make dev-up

# Or manually
docker compose -f docker-compose.dev.yml up -d

# Production
docker compose up -d
```

---

## Best Practices

1. **Always use `STORAGE_PATH` in dev containers** to ensure consistency
2. **Mount the same path** in both API and worker services
3. **Use absolute paths** for `STORAGE_PATH` to avoid confusion
4. **Verify paths match** when troubleshooting ingestion failures

---

## Troubleshooting

### Issue: "File not found" errors in worker

**Check**:
1. Is `STORAGE_PATH` set consistently in both services?
   ```bash
   docker compose exec modcus-api-ingest env | grep STORAGE
   docker compose exec celery-worker env | grep STORAGE
   ```

2. Is the volume mounted correctly?
   ```bash
   docker compose exec celery-worker ls -la $STORAGE_PATH/uploads/
   ```

3. Does the API actually save to that path?
   ```bash
   docker compose logs modcus-api-ingest | grep "storage_path"
   ```

### Issue: Different paths in different environments

**Solution**: Always set `STORAGE_PATH` explicitly in your environment-specific compose files.

---

## Related Changes

- Storage path logic centralized in ingestion upload endpoint
- Docker compose configuration updated for dev environment
- Consistent volume mounting across all services

---

**End of Changelog**
