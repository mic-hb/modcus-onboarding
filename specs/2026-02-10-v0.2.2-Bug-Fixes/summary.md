# Modcus v0.2.2 Release Summary

**Release Date**: February 10, 2026
**Status**: ✅ Complete
**Previous Version**: v0.2.1

---

## Overview

This release contains critical bug fixes and improvements to the Modcus platform, focusing on authentication, async operations, storage consistency, and observability. The release addresses issues discovered during v0.2.1 deployment and testing.

---

## Summary of Changes

### 1. ChromaDB HTTP Client Support
**Issue**: Query service couldn't connect to ChromaDB in Docker - "Vector index unavailable" error
**Files**: `modcus_common/settings/rag.py`, `modcus_common/services/vector_store/chromadb_config.py`, `modcus_common/services/vector_store/vector_store_factory.py`, `modcus_api_query/main.py`, `modcus_api_query/pyproject.toml`, `.env.example`, Docker Compose files
**Fix**: Added HTTP client support for ChromaDB to enable Query and Ingest services to connect via HTTP API instead of local file storage

**Key Changes**:
- Added `chromadb_host`, `chromadb_port`, `chromadb_use_http` settings
- Updated `create_chromadb_client()` to support `HttpClient` mode
- Rewrote Query service `_load_vector_index()` to use `VectorStoreFactory`
- Added `llama-index-vector-stores-chroma` dependency
- Updated Docker Compose configurations

**Commit**: `b49f0f4f206e4b032c3f14176bc4e3f4e7378a8f`

---

### 2. ConversationTurn NameError Fix
**Issue**: `NameError: name 'ConversationTurn' is not defined` in Query API orchestration
**File**: `modcus_api_query/services/orchestration/state.py`
**Fix**: Changed TYPE_CHECKING-only import to regular import for `ConversationTurn`

**Before**:
```python
if TYPE_CHECKING:
    from modcus_common.models.conversation import ConversationTurn
```

**After**:
```python
from modcus_common.models.conversation import ConversationTurn
```

**Root Cause**: TypedDict annotations can be evaluated at runtime in some Python contexts, requiring the type to be available at runtime.

**Commit**: `05bbd86906e8fad60fe490f4b23c7f16612df0e7`

---

### 3. Query API Logging System
**Issue**: Missing comprehensive logging in Query API compared to Ingest API
**Files**: 11 files including `modcus_api_query/main.py`, `modcus_api_query/api/routes_query.py`, all agents, orchestration nodes, and background services
**Fix**: Implemented comprehensive structured logging throughout the Query API

**Coverage**:
- Service initialization and shutdown logging
- Per-request query tracking with `query_id`
- All 5 agents (Validation, TickerInference, Retrieval, Analysis, ResponseFormatter)
- All 8 orchestration nodes
- Health and artifact endpoints
- Background cleanup scheduler

**Total**: ~1,040 lines of logging code across 11 files

**Documentation**: See `agent-os/specs/2026-02-09-0320-query-api-logging-system/` for complete implementation spec

---

### 4. Vertex AI LLM Credentials Support
**Issue**: Vertex AI LLM class couldn't authenticate using service account credentials
**Files**: `modcus_common/services/llm/llm_factory.py`, `modcus_common/services/llm/llm_rotator.py`
**Fix**: Added `credentials` parameter support for Vertex AI LLM authentication

**Key Changes**:
- Added `credentials` parameter to `create_llm_instance()` function
- Updated `_get_current_llm_async()` to load and pass credentials
- Uses `service_account.Credentials.from_service_account_file()` for credential loading
- Aligns LLM authentication with existing Vertex Embedding implementation

**Configuration**:
```yaml
gcp:
  project_id: your-gcp-project-id
  location: us-central1
  application_credentials: /path/to/service-account.json
```

---

### 5. Async Error Fix - Loading Active Keys
**Issue**: `RuntimeError: asyncio.run() cannot be called from a running event loop`
**File**: `modcus_common/services/llm/llm_rotator.py`
**Fix**: Changed `_get_current_llm_async()` to call async method directly instead of sync wrapper

**Before** (line 95):
```python
if not self.active_keys:
    self._load_active_keys()  # BUG: Calls asyncio.run() internally!
```

**After**:
```python
if not self.active_keys:
    await self._load_active_keys_async()  # FIXED: Direct async call
```

**Root Cause**: Async methods should `await` other async methods, not call sync wrappers that use `asyncio.run()`.

---

### 6. Ingestion Storage Path Mismatch Fix
**Issue**: Uploaded files couldn't be found by Celery worker - path mismatch between Ingest API and worker
**Files**: `modcus_api_ingest/api/routes_ingestion.py`, `docker-compose.dev.yml`
**Fix**: Introduced `STORAGE_PATH` environment variable for consistent storage location

**Before**: Hardcoded path checks with inconsistent fallback behavior
**After**: Environment-driven path selection

```python
if os.environ.get("STORAGE_PATH"):
    storage_dir = Path(os.environ["STORAGE_PATH"]) / "uploads"
elif Path("/app/storage").exists():
    storage_dir = Path("/app/storage/uploads")
else:
    storage_dir = Path(__file__).resolve().parents[2] / "modcus_common" / ".storage" / "uploads"
```

**Docker Compose Configuration**:
```yaml
environment:
  STORAGE_PATH: /workspace/modcus_common/.storage  # Both services use same path
```

---

### 7. LLMCall Database UUID Type Fix
**Issue**: `column "api_key_id" is of type uuid but expression is of type character varying` database error
**File**: `modcus_common/models/llm_call.py`
**Fix**: Changed `api_key_id` field type from `str` to `UUID` to match foreign key reference

**Before**:
```python
class LLMCall(SQLModel, table=True):
    ...
    api_key_id: Optional[str] = Field(default=None, foreign_key="apikey.id", index=True)
```

**After**:
```python
from uuid import UUID

class LLMCall(SQLModel, table=True):
    ...
    api_key_id: Optional[UUID] = Field(default=None, foreign_key="apikey.id", index=True)
```

**Root Cause**: The `ApiKey` model defines `id` as `UUID` type, but `LLMCall` was referencing it as `str`, causing PostgreSQL to reject inserts.

---

### 8. Settings Endpoint Implementation

**Issue**: No way to verify loaded configuration without SSH access to containers
**Files**: 10 files across all services
**Fix**: Implemented `/v1/settings` endpoint in both Query and Ingest services with dual format support

**Features**:
- **JSON format** (default): Complete configuration as nested object with source labels
- **Pretty format**: ASCII tables using `rich` library with 4 sections (📋 Base, 🤖 LLM, 🔍 RAG, 🔧/🔄 Service-specific)
- **Source labeling**: Every field tagged with `[YAML]` or `[ENV]`
- **Complete coverage**: All settings from BaseSettings, LLMSettingsMixin, RAGSettingsMixin, and service-specific settings

**New Files**:
- `modcus_common/utils/settings_formatter.py` - Formatting logic
- `modcus_api_query/api/routes_settings.py` - Query service endpoint
- `modcus_api_ingest/api/routes_settings.py` - Ingest service endpoint

**Modified Files**:
- `modcus_common/models/dtos.py` - Added 40+ settings DTOs
- `modcus_common/pyproject.toml` - Added `rich` dependency
- `modcus_api_query/api/deps.py` - Added `verify_api_key()` dependency
- `modcus_api_query/api/__init__.py` - Export settings_router
- `modcus_api_query/main.py` - Register settings_router
- `modcus_api_ingest/api/__init__.py` - Export settings_router
- `modcus_api_ingest/main.py` - Register settings_router

**Usage**:
```bash
# Query service - JSON format
curl http://localhost:8080/v1/settings -H "X-API-Key: $API_KEY"

# Query service - Pretty ASCII tables
curl "http://localhost:8080/v1/settings?format=pretty" -H "X-API-Key: $API_KEY"

# Ingest service
curl http://localhost:8081/v1/settings -H "X-API-Key: $API_KEY"
```

**Documentation**: See `docs/docs/specs/2026-02-10-Settings-Endpoint/changelog.md` for complete details

---

## Files Modified Summary

| Component          | Files Changed     | Lines Changed    |
| ------------------ | ----------------- | ---------------- |
| **Common Library** | 9 files           | ~852 lines       |
| **Query Service**  | 14 files          | ~1,115 lines     |
| **Ingest Service** | 5 files           | ~85 lines        |
| **Configuration**  | 4 files           | ~40 lines        |
| **Documentation**  | 8 changelog files | ~4,000 lines     |
| **Total**          | **40 files**      | **~6,092 lines** |

### Detailed File List

**Common Library (`modcus_common/`)**:
- `services/llm/llm_factory.py` - Vertex credentials support
- `services/llm/llm_rotator.py` - Async error fix, credentials loading
- `models/llm_call.py` - UUID type fix for api_key_id
- `settings/rag.py` - ChromaDB HTTP settings
- `services/vector_store/chromadb_config.py` - HTTP client support
- `services/vector_store/vector_store_factory.py` - ChromaDB configuration
- `utils/settings_formatter.py` - Settings endpoint formatter (NEW)
- `models/dtos.py` - Settings DTOs (enhanced)
- `pyproject.toml` - Added rich dependency

**Query Service (`modcus_api_query/`)**:
- `main.py` - ChromaDB connection, logging, settings router
- `api/routes_query.py` - Comprehensive query logging
- `api/routes_health.py` - Health endpoint logging
- `api/routes_artifacts.py` - Artifact endpoint logging
- `api/routes_settings.py` - Settings endpoint (NEW)
- `api/deps.py` - Added verify_api_key dependency
- `api/__init__.py` - Export settings_router
- `services/agents/validation_agent.py` - Agent logging
- `services/agents/ticker_inference_agent.py` - Agent logging
- `services/agents/retrieval_agent.py` - Agent logging
- `services/agents/analysis_agent.py` - Agent logging
- `services/agents/response_formatter.py` - Agent logging
- `services/orchestration/nodes.py` - Node logging
- `services/orchestration/state.py` - ConversationTurn fix
- `services/chat/cleanup_scheduler.py` - Background service logging
- `pyproject.toml` - ChromaDB dependencies

**Ingest Service (`modcus_api_ingest/`)**:
- `api/routes_ingestion.py` - Storage path fix
- `api/routes_settings.py` - Settings endpoint (NEW)
- `api/__init__.py` - Export settings_router
- `main.py` - Register settings_router

**Configuration Files**:
- `.env.example` - New environment variables
- `docker-compose.query.yml` - ChromaDB and service account configuration
- `docker-compose.ingest.yml` - ChromaDB configuration
- `docker-compose.dev.yml` - Storage path configuration

**Documentation**:
- `docs/docs/specs/2026-02-09-ChromaDB-HTTP-Support/changelog.md`
- `docs/docs/specs/2026-02-09-ConversationTurn-Fix/changelog.md`
- `docs/docs/specs/2026-02-09-Query-Logging-System/changelog.md`
- `docs/docs/specs/2026-02-10_1145_Vertex-LLM-Credentials-Support/changelog.md`
- `docs/docs/specs/2026-02-10_1200_Async-Error-Fix/changelog.md`
- `docs/docs/specs/2026-02-10_1215_Ingestion-Storage-Path-Fix/changelog.md`
- `docs/docs/specs/2026-02-10_1300_LLMLCall-UUID-Type-Fix/changelog.md`
- `docs/docs/specs/2026-02-10-Settings-Endpoint/changelog.md`

---

## Environment Variables

### New Environment Variables

```bash
# ChromaDB HTTP Client Settings
CHROMADB_USE_HTTP=true              # Use HTTP client (required for Docker)
CHROMADB_HOST=host.docker.internal  # ChromaDB server host
CHROMADB_PORT=8000                  # ChromaDB server port

# Storage Path (for dev containers)
STORAGE_PATH=/workspace/modcus_common/.storage  # Shared storage base path

# Vertex AI Service Account (if using Vertex AI)
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
GCP_PROJECT_ID=your-gcp-project-id
GCP_LOCATION=us-central1
```

### Updated Environment Variables
- `CHROMADB_URL` - Now used as legacy format alongside new settings

---

## Testing Checklist

### Prerequisites
```bash
# Start dependencies
docker compose -f docker-compose.deps.yml up -d

# Verify ChromaDB is running
curl http://localhost:8000/api/v2/heartbeat
```

### Test 1: Health Endpoints
```bash
# Query service
curl http://localhost:8080/v1/health
# Expected: {"status": "healthy", "index_status": "available"}

# Ingest service
curl http://localhost:8081/v1/health
# Expected: {"status": "healthy"}
```

### Test 2: Query Service ChromaDB Connection
```bash
# Check Query service logs
docker compose -f docker-compose.query.yml logs modcus-api-query | grep -E "ChromaDB|Vector index"
# Expected: "Query Service: Connected to ChromaDB"
```

### Test 3: Query Processing (Async Fix)
```bash
# Test query (this would fail with async error before fix)
curl -X POST http://localhost:8080/v1/query \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the financial highlights?",
    "mode": "company-analysis",
    "ticker": "GOTO_TEST",
    "level": "novice"
  }'
# Expected: 200 OK with response (not 500 error)
```

### Test 4: Document Ingestion (Storage Path Fix)
```bash
# Upload a file
curl -X POST 'http://localhost:8081/v1/ingestion/upload' \
  -H 'X-API-Key: $API_KEY' \
  -F 'file=@document.json' \
  -F 'ticker=TEST' \
  -F 'doc_type=annual_report'

# Check worker logs
docker compose logs celery-worker
# Expected: File found at correct path, processing starts
```

### Test 5: Verify Logging
```bash
# Check Query service logs for structured logging
docker compose logs modcus-api-query | grep query_id
# Expected: Structured logs with query_id correlation
```

### Test 6: LLMCall UUID Type Fix
```bash
# Test query with API key logging (this would fail with UUID error before fix)
curl -X POST http://localhost:8080/v1/query \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the financial highlights?",
    "mode": "company-analysis",
    "ticker": "TEST"
  }'
# Expected: 200 OK or processing error (not database UUID error)

# Check that LLM calls are logged without database errors
docker compose logs modcus-api-query | grep -i "llm_call\|uuid"
# Expected: No "column api_key_id is of type uuid" errors
```

### Test 7: Settings Endpoint
```bash
# Query service - JSON format
curl http://localhost:8080/v1/settings \
  -H "X-API-Key: $API_KEY"
# Expected: Valid JSON with all settings and source labels

# Query service - Pretty format
curl "http://localhost:8080/v1/settings?format=pretty" \
  -H "X-API-Key: $API_KEY"
# Expected: Formatted ASCII tables with 4 sections

# Ingest service
curl http://localhost:8081/v1/settings \
  -H "X-API-Key: $API_KEY"
# Expected: Valid JSON with ingest-specific settings (celery, queue, retry)
```

---

## Deployment Notes

### Docker Deployment
1. Ensure `.env` file has all required variables (see Environment Variables section)
2. Copy service account JSON if using Vertex AI
3. Start dependencies: `docker compose -f docker-compose.deps.yml up -d`
4. Run migrations if needed: `cd modcus_migration && alembic upgrade head`
5. Start ingest: `docker compose -f docker-compose.ingest.yml up -d`
6. Start query: `docker compose -f docker-compose.query.yml up -d`

### Development Deployment
```bash
# Using Makefile
make dev-up

# Or manually
docker compose -f docker-compose.dev.yml up -d
```

### Breaking Changes
- None (all changes are backward compatible)
- ChromaDB HTTP client is recommended for Docker deployments
- Existing persistent client mode still works with `CHROMADB_USE_HTTP=false`

---

## Migration Guide

### From v0.2.1

No migration required! The fixes are backward compatible:

1. Pull latest code
2. Copy new `.env.example` fields to your `.env`:
   ```bash
   # ChromaDB settings
   CHROMADB_USE_HTTP=true
   CHROMADB_HOST=host.docker.internal
   CHROMADB_PORT=8000

   # Storage path (for dev containers)
   STORAGE_PATH=/workspace/modcus_common/.storage

   # Vertex AI (if applicable)
   GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
   ```
3. Rebuild and restart services
4. Verify all endpoints work correctly

### Database Changes
- No schema changes required
- Existing data is fully compatible

---

## Verification Commands

```bash
# Quick health check
curl http://localhost:8080/v1/health && curl http://localhost:8081/v1/health

# Verify ChromaDB connection in Query service logs
docker logs modcus-api-query 2>&1 | grep "Connected to ChromaDB"

# Test ingestion and query flow
curl -X POST http://localhost:8081/v1/ingestion/upload \
  -H "X-API-Key: $API_KEY" \
  -F "file=@test.pdf" -F "ticker=TEST"

curl -X POST http://localhost:8080/v1/query \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "test", "mode": "company-analysis", "ticker": "TEST"}'

# Verify loaded configuration
curl http://localhost:8080/v1/settings -H "X-API-Key: $API_KEY"
curl "http://localhost:8080/v1/settings?format=pretty" -H "X-API-Key: $API_KEY"
```

---

## Release Sign-off

- [x] ChromaDB HTTP support implemented and tested
- [x] ConversationTurn NameError fixed
- [x] Query API logging system implemented (~1,040 lines)
- [x] Vertex AI credentials support added
- [x] Async error in LLM rotator fixed
- [x] Ingestion storage path mismatch fixed
- [x] LLMCall UUID type mismatch fixed
- [x] Settings endpoint implemented (~700 lines)
- [x] All code reviewed and tested
- [x] Documentation updated (8 changelog files)
- [x] Environment variables configured
- [x] Docker Compose files updated
- [x] End-to-end testing completed
- [x] Release summary created

**Status**: ✅ **All Fixes Complete and Tested**

---

## References

### Changelogs
- `docs/docs/specs/2026-02-09-ChromaDB-HTTP-Support/changelog.md`
- `docs/docs/specs/2026-02-09-ConversationTurn-Fix/changelog.md`
- `docs/docs/specs/2026-02-09-Query-Logging-System/changelog.md`
- `docs/docs/specs/2026-02-10_1145_Vertex-LLM-Credentials-Support/changelog.md`
- `docs/docs/specs/2026-02-10_1200_Async-Error-Fix/changelog.md`
- `docs/docs/specs/2026-02-10_1215_Ingestion-Storage-Path-Fix/changelog.md`
- `docs/docs/specs/2026-02-10_1300_LLMLCall-UUID-Type-Fix/changelog.md`
- `docs/docs/specs/2026-02-10-Settings-Endpoint/changelog.md`

### Implementation Specs
- `agent-os/specs/2026-02-09-0320-query-api-logging-system/` - Complete Query API logging spec

### Commits
- `b49f0f4f206e4b032c3f14176bc4e3f4e7378a8f` - ChromaDB HTTP Support
- `05bbd86906e8fad60fe490f4b23c7f16612df0e7` - ConversationTurn Fix

---

**End of Release Summary v0.2.2**
