# Settings Endpoint Implementation

**Date**: February 10, 2026
**Version**: v0.2.2
**Status**: Completed
**Priority**: High

---

## Summary

Implemented a new `/v1/settings` REST endpoint in both Query service (port 8080) and Ingest service (port 8081) to expose currently loaded configuration. This enables runtime configuration visibility for debugging, deployment verification, and troubleshooting without requiring SSH access to containers.

---

## Changes

### 1. Settings Endpoint Implementation

**Files**: 
- `modcus_common/utils/settings_formatter.py` (new)
- `modcus_common/models/dtos.py` (updated)
- `modcus_api_query/api/routes_settings.py` (new)
- `modcus_api_ingest/api/routes_settings.py` (new)
- `modcus_api_query/api/deps.py` (updated)
- `modcus_api_query/api/__init__.py` (updated)
- `modcus_api_query/main.py` (updated)
- `modcus_api_ingest/api/__init__.py` (updated)
- `modcus_api_ingest/main.py` (updated)
- `modcus_common/pyproject.toml` (updated)

**New Endpoint**: `GET /v1/settings`

**Features**:
- Dual format support: JSON (default) and pretty ASCII tables
- Source labeling: Every field tagged with `[YAML]` or `[ENV]`
- Complete configuration coverage: Base, LLM, RAG, and service-specific settings
- API key authentication: Protected by X-API-Key header

**Query Parameters**:
- `format` (optional, string): `"json"` (default) or `"pretty"`

---

### 2. JSON Format Response

**Endpoint**: `GET /v1/settings`

Returns complete configuration as nested JSON object with source labels:

```json
{
  "service": "query",
  "database": {
    "db_url": "postgresql://modcus:modcus123@postgres:5432/modcus [ENV]",
    "db_password": "null [ENV]"
  },
  "llm_provider": {
    "llm_provider": "vertex [YAML]",
    "llm_model": "gemini-2.5-flash [YAML]",
    "llm_temperature": "0.0 [YAML]"
  },
  "vector_store": {
    "rag_vector_store": "chromadb [YAML]"
  },
  "query_timeout": {
    "query_timeout_seconds": "360 [YAML]",
    "query_complexity_inference_mode": "combined [YAML]"
  }
}
```

**Key characteristics:**
- All values are strings with source labels appended
- Nested structure organized by logical categories
- Service identifier field (`"service": "query"` or `"ingest"`)
- All configuration fields from BaseSettings, LLMSettingsMixin, RAGSettingsMixin, and service-specific settings

---

### 3. Pretty Format Response

**Endpoint**: `GET /v1/settings?format=pretty`

Returns formatted ASCII tables using the `rich` library:

```
╔══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║                                                                                                                      ║
║                                          ⚙️  MODCUS QUERY SERVICE SETTINGS                                           ║
║                                                                                                                      ║
╚══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝

                                                  📋 BASE SETTINGS                                                   
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━┓
┃ Setting                                         ┃ Value                                              ┃ Source     ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━┩
│ app.app_name                                    │ modcus                                             │ [YAML]     │
│ database.db_url                                 │ postgresql://modcus:modcus123@postgres:5432/modcus │ [ENV]      │
│ database.db_password                            │ null                                               │ [ENV]      │
│ llm_provider.llm_provider                       │ vertex                                             │ [YAML]     │
│ llm_provider.llm_model                          │ gemini-2.5-flash                                   │ [YAML]     │
└─────────────────────────────────────────────────┴────────────────────────────────────────────────────┴────────────┘

                                      🤖 LLM SETTINGS                                      
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━┓
┃ Setting                                  ┃ Value                           ┃ Source     ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━┩
│ llm_provider.llm_provider                │ vertex                          │ [YAML]     │
│ llm_api_keys.gemini_api_key              │ your_gemini_api_key_here        │ [ENV]      │
│ vertex_ai.gcp_project_id                 │ silken-tangent-299814           │ [ENV]      │
└──────────────────────────────────────────┴─────────────────────────────────┴────────────┘

                                                 🔍 RAG SETTINGS                                                  
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━┓
┃ Setting                                      ┃ Value                                              ┃ Source     ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━┩
│ vector_store.rag_vector_store                │ chromadb                                           │ [YAML]     │
│ chromadb.chromadb_host                       │ chromadb                                           │ [ENV]      │
└──────────────────────────────────────────────┴────────────────────────────────────────────────────┴────────────┘

                        🔧 QUERY SERVICE-SPECIFIC SETTINGS                          
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━┓
┃ Setting                                                 ┃ Value    ┃ Source     ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━┩
│ query_timeout.query_timeout_seconds                     │ 360      │ [YAML]     │
└─────────────────────────────────────────────────────────┴──────────┴────────────┘
```

**Key characteristics:**
- 4 separate tables with emojis (📋, 🤖, 🔍, 🔧/🔄)
- Professional box-drawing borders using `rich` library
- Columns: Setting, Value, Source
- Human-readable format for debugging

---

### 4. Source Labeling

Every configuration field is tagged with its source:

- **[YAML]**: Loaded from `config.yaml` or `config.{env}.yaml` files
  - Feature flags, numeric settings, timeouts
  - Pattern-based validation fields
  - List fields (celery_accept_content)

- **[ENV]**: Loaded from environment variables or `.env` files
  - Database URLs and passwords (db_url, db_password)
  - Redis connection strings (redis_url, redis_password)
  - All API keys (gemini_api_key, openai_api_key, etc.)
  - Cloud storage credentials (s3_secret_key, azure_storage_key)
  - Authentication tokens (api_key, qdrant_api_key)

**ENV_FIELDS** constant in `settings_formatter.py`:
```python
ENV_FIELDS = {
    "db_url", "db_password", "redis_url", "redis_password", "api_key",
    "s3_bucket", "s3_region", "s3_access_key", "s3_secret_key",
    "gcs_bucket", "gcs_project_id",
    "azure_storage_account", "azure_storage_key", "azure_container",
    "gemini_api_key", "openai_api_key", "anthropic_api_key",
    "gcp_project_id", "gcp_location", "google_application_credentials",
    "qdrant_url", "qdrant_api_key",
    "chromadb_persist_dir", "chromadb_host", "chromadb_port", "chromadb_use_http",
    "pgvector_connection_string",
}
```

---

### 5. Configuration Coverage

**Base Settings** (both services):
- App: app_name
- Database: db_url, db_password
- Redis: redis_url, redis_password
- Logging: log_level, log_dir, log_rotation, log_retention
- Storage: storage_backend, storage_local_base
- Metrics: metrics_enabled, metrics_port
- Auth: api_key
- Rate Limiting: requests_per_minute, burst_size
- Celery: result_backend, task_serializer, accept_content, result_serializer
- Cloud Storage: S3, GCS, Azure credentials

**LLM Settings** (both services):
- Provider: llm_provider, llm_model, temperature, max_tokens
- Embedding: embedding_provider, embedding_model, dimension
- Rotation: llm_rotation_enabled, llm_rotation_max_retries
- API Keys: gemini_api_key, openai_api_key, anthropic_api_key
- Vertex AI: gcp_project_id, gcp_location, google_application_credentials

**RAG Settings** (both services):
- Vector Store: rag_vector_store
- Chunk Sizes: annual_report, financial_report, news, public_expose
- Chunking: rag_chunk_overlap
- Retrieval: rag_retrieval_top_k, rag_retrieval_enable_reranking
- Connections: Qdrant, ChromaDB, PGVector

**Query Service-specific**:
- Query Timeout: query_timeout_seconds, query_complexity_inference_mode
- Conversation: timeout_minutes, sliding_window_size, context_window_tokens
- Ephemeral Artifacts: TTL hours

**Ingest Service-specific**:
- Celery Worker: concurrency
- Queue: max_queued
- Retry Strategy: strategy, max_retries
- Retry Timing: initial_delay, backoff_factor, max_delay

---

### 6. Implementation Details

#### Settings Formatter (`modcus_common/utils/settings_formatter.py`)

**Key Functions**:

```python
# Returns JSON-serializable dict with source labels
def format_settings_json(settings: Any) -> Dict[str, Any]:
    result = {
        **build_base_settings(settings),
        **build_llm_settings(settings),
        **build_rag_settings(settings),
    }
    # Add service-specific settings using duck typing
    if _is_ingest_settings(settings):
        result["service"] = "ingest"
        result["celery_worker"] = ...
    elif _is_query_settings(settings):
        result["service"] = "query"
        result["query_timeout"] = ...
    return result

# Returns formatted ASCII tables
def format_settings_pretty(settings: Any, service_name: str) -> str:
    data = format_settings_json(settings)
    data = dto_to_dict(data)  # Convert Pydantic models to dicts
    
    console = Console(width=120, record=True)
    # Create and print 4 tables with emojis
    return console.export_text()
```

**Duck Typing for Service Detection**:
```python
def _is_ingest_settings(settings: Any) -> bool:
    """Check if settings is an IngestSettings instance."""
    return hasattr(settings, "ingestion_celery_concurrency")

def _is_query_settings(settings: Any) -> bool:
    """Check if settings is a QuerySettings instance."""
    return hasattr(settings, "query_timeout_seconds")
```

#### Route Implementation

**Query Service** (`modcus_api_query/api/routes_settings.py`):
```python
@router.get("/settings")
async def get_settings_endpoint(
    format: Literal["json", "pretty"] = Query(default="json"),
    settings: QuerySettings = Depends(get_settings),
    api_key: str = Depends(verify_api_key),
):
    log = logger.bind(endpoint="settings", service="query", format=format)
    log.debug("Retrieving settings")
    
    if format == "pretty":
        pretty_output = format_settings_pretty(settings, "QUERY SERVICE")
        return PlainTextResponse(content=pretty_output)
    else:
        return format_settings_json(settings)
```

**Ingest Service** (`modcus_api_ingest/api/routes_settings.py`):
```python
@router.get("/settings")
async def get_settings_endpoint(
    format: Literal["json", "pretty"] = Query(default="json"),
    settings: IngestSettings = Depends(get_settings),
    api_key: str = Depends(verify_api_key),
):
    log = logger.bind(endpoint="settings", service="ingest", format=format)
    log.debug("Retrieving settings")
    
    if format == "pretty":
        pretty_output = format_settings_pretty(settings, "INGEST SERVICE")
        return PlainTextResponse(content=pretty_output)
    else:
        return format_settings_json(settings)
```

---

## DTO Types as Strings

**Challenge**: DTOs originally expected typed values (bool, int) but formatter appends source labels as strings (e.g., `"true [YAML]"`).

**Solution**: Changed all DTO field types to `str` in `modcus_common/models/dtos.py`:

**Before**:
```python
class MetricsSettingsDTO(BaseModel):
    metrics_enabled: bool
    metrics_port: int
```

**After**:
```python
class MetricsSettingsDTO(BaseModel):
    metrics_enabled: str
    metrics_port: str
```

**Result**: Values like `"false [YAML]"` and `"9090 [YAML]"` are now valid.

---

## Testing

### Test Results

**JSON Format Test**:
```bash
curl http://localhost:8080/v1/settings \
  -H "X-API-Key: modcus-123456789"
```
Result: ✅ Returns valid JSON with all settings and source labels

**Pretty Format Test**:
```bash
curl "http://localhost:8080/v1/settings?format=pretty" \
  -H "X-API-Key: modcus-123456789"
```
Result: ✅ Returns formatted ASCII tables with 4 sections

**Authentication Test**:
```bash
curl http://localhost:8080/v1/settings
```
Result: ✅ Returns 401 Unauthorized (missing API key)

**Ingest Service Test**:
```bash
curl http://localhost:8081/v1/settings \
  -H "X-API-Key: modcus-123456789"
```
Result: ✅ Returns ingest-specific settings (celery_worker, queue, retry_strategy)

---

## Documentation Updates

Updated the following documentation files:

1. **README.md**
   - Added `/v1/settings` to API endpoints tables
   - Added verification command to Docker Compose quick start
   - Added v0.2.2 release section with usage examples

2. **AGENTS.md**
   - Added settings verification to Manual Verification Checklist
   - Includes examples for both JSON and pretty formats

3. **docs/docs/MODCUS_MASTER_DOCUMENT.md**
   - Updated Appendix C: API Endpoint Summary
   - Added v1.2 to Change Log

4. **agent-os/standards/configuration/visibility.md** (new)
   - Comprehensive standard for configuration visibility
   - Requirements for endpoint specification
   - Response format specifications
   - Source labeling rules

5. **agent-os/standards/index.yml**
   - Added `visibility` entry to configuration section

---

## Use Cases

1. **Deployment Verification**: Confirm environment variables are loaded correctly
2. **Debugging**: Check configuration without SSH access
3. **Troubleshooting**: Identify misconfigurations causing issues
4. **Documentation**: Runtime view of actual vs. expected configuration
5. **Security Audit**: View all settings in one place (requires API key)

---

## Security Considerations

- **API Key Required**: All requests must include valid `X-API-Key` header
- **No Masking**: All values displayed as-is including passwords and API keys
- **Admin Access**: Endpoint intended for authorized administrators only
- **Network Security**: Consider network-level restrictions in production

---

## Future Enhancements

Potential improvements for future iterations:
- Filter by category (e.g., `?category=llm`)
- Diff mode to compare with defaults (e.g., `?diff=true`)
- Export to file format (YAML, JSON)
- Historical configuration tracking
- Configuration validation endpoint

---

## References

- **Spec**: `agent-os/specs/2026-02-10-0755-settings-endpoint/`
- **Implementation**: 
  - `modcus_common/utils/settings_formatter.py`
  - `modcus_common/models/dtos.py`
  - `modcus_api_query/api/routes_settings.py`
  - `modcus_api_ingest/api/routes_settings.py`

---

**End of Changelog**
