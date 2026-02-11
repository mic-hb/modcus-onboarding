# Configuration System Architecture Review (Post-Fix)

## 1. System Architecture Overview

The Modcus configuration system uses a **three-layer hierarchical architecture** with Pydantic Settings:

### 1.1 Configuration Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 3: Environment Variables (.env)                          │
│ - Secrets (API keys, passwords)                                │
│ - Runtime environment overrides                                │
│ - Highest precedence                                           │
└─────────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────────┐
│ Layer 2: Environment-Specific YAML                             │
│ - config.development.yaml                                      │
│ - config.production.yaml                                       │
│ - Overrides base configuration                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▲
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: Base YAML Configuration (config.yaml)                 │
│ - Default values                                               │
│ - Structured, version-controlled                               │
│ - Lowest precedence                                            │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Configuration Loading Flow

```
Service Startup
      │
      ▼
Settings.load() / from_yaml()
      │
      ├── 1. Determine APP_ENV (default: development)
      │
      ├── 2. Load base config.yaml
      │
      ├── 3. Load config.{APP_ENV}.yaml (if exists)
      │
      ├── 4. Merge configs (env-specific takes precedence)
      │
      ├── 5. Flatten nested YAML structure
      │   Example: rag.chunk_sizes.annual_report
      │        → rag_chunk_sizes_annual_report
      │
      ├── 6. Create Pydantic model instance
      │
      └── 7. Apply environment variable overrides
          (Pydantic reads from .env automatically)
```

## 2. Settings Class Hierarchy

### 2.1 Core Settings Classes

```
PydanticBaseSettings (pydantic-settings)
    │
    ├── BaseSettings (modcus_common/settings/base.py)
    │   ├── App configuration (app_name)
    │   ├── Database settings (db_url, db_password)
    │   ├── Redis settings (redis_url, redis_password)
    │   ├── Logging settings (log_level, log_rotation, etc.)
    │   ├── Storage settings (storage_backend, storage_local_base)
    │   ├── Metrics settings (metrics_enabled, metrics_port)
    │   ├── Security (api_key with validation)
    │   ├── Rate limiting (rate_limiting_requests_per_minute)
    │   ├── Celery config (celery_result_backend, etc.)
    │   └── Cloud storage (s3_*, gcs_*, azure_*)
    │
    ├── LLMSettingsMixin (modcus_common/settings/llm.py)
    │   ├── Provider settings (llm_provider, llm_model)
    │   ├── Generation settings (llm_temperature, llm_max_tokens)
    │   ├── Embedding settings (embedding_provider, embedding_model, embedding_dimension)
    │   ├── Rotation settings (llm_rotation_enabled, llm_rotation_max_retries)
    │   └── API keys (gemini_api_key, openai_api_key, anthropic_api_key)
    │       └── GCP settings (gcp_project_id, gcp_location, google_application_credentials)
    │
    └── RAGSettingsMixin (modcus_common/settings/rag.py)
        ├── Vector store (rag_vector_store)
        ├── Chunk sizes (rag_chunk_sizes_*)
        ├── Chunking (rag_chunk_overlap)
        ├── Retrieval (rag_retrieval_top_k, rag_retrieval_enable_reranking)
        └── Connection settings (qdrant_url, chromadb_persist_dir, etc.)

Service-Specific Settings:
    ├── QuerySettings (modcus_api_query/settings.py)
    │   └── query_timeout_seconds, query_complexity_inference_mode
    │       query_conversation_*, query_context_window_tokens
    │       query_ephemeral_artifacts_ttl_hours
    │
    └── IngestSettings (modcus_api_ingest/settings.py)
        └── ingestion_celery_*, ingestion_retry_*
```

### 2.2 Mixin Pattern Explanation

The settings system uses **multiple inheritance with mixins** to share configuration across services:

```python
# Base settings available to all services
class BaseSettings(PydanticBaseSettings):
    # Database, logging, storage, security

# Shared LLM configuration
class LLMSettingsMixin:
    # LLM provider, model, embedding settings

# Shared RAG configuration  
class RAGSettingsMixin:
    # Vector store, chunking settings

# Service-specific composition
class QuerySettings(BaseSettings, LLMSettingsMixin, RAGSettingsMixin):
    # Query-specific settings + inherited mixins

class IngestSettings(BaseSettings, LLMSettingsMixin, RAGSettingsMixin):
    # Ingest-specific settings + inherited mixins
```

**Benefits:**
- **DRY Principle**: Common settings defined once, used everywhere
- **Type Safety**: Pydantic validates all fields at load time
- **Flexibility**: Services can have service-specific settings
- **Consistency**: Same configuration structure across services

## 3. Configuration Loading Mechanism

### 3.1 YAML Flattening Process

The `_flatten_yaml()` method converts nested YAML to flat keys:

| YAML Structure                       | Flattened Key                        | Python Field                         |
| ------------------------------------ | ------------------------------------ | ------------------------------------ |
| `rag.vector_store`                   | `rag_vector_store`                   | `rag_vector_store`                   |
| `rag.chunk_sizes.annual_report`      | `rag_chunk_sizes_annual_report`      | `rag_chunk_sizes_annual_report`      |
| `query.conversation.timeout_minutes` | `query_conversation_timeout_minutes` | `query_conversation_timeout_minutes` |
| `llm.rotation.enabled`               | `llm_rotation_enabled`               | `llm_rotation_enabled`               |
| `log.level`                          | `log_level`                          | `log_level`                          |

### 3.2 Configuration Merge Strategy

```python
# Base config (config.yaml)
base = {
    "log": {"level": "INFO"},
    "rag": {"vector_store": "chromadb"}
}

# Environment config (config.development.yaml)
override = {
    "log": {"level": "DEBUG"},
    "rag": {"vector_store": "chromadb"}
}

# Merged result (override takes precedence)
merged = {
    "log": {"level": "DEBUG"},  # Overridden
    "rag": {"vector_store": "chromadb"}
}
```

### 3.3 Environment Variable Override

Pydantic Settings automatically reads environment variables:

| Source               | Priority    | Example                   |
| -------------------- | ----------- | ------------------------- |
| Environment Variable | 1 (Highest) | `DB_URL=postgresql://...` |
| Environment YAML     | 2           | `config.production.yaml`  |
| Base YAML            | 3 (Lowest)  | `config.yaml`             |

## 4. Service-Specific Configuration Patterns

### 4.1 Query Service Pattern

```python
# modcus_api_query/main.py
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Load settings during startup
    settings = QuerySettings.load()
    
    # Store in deps module for dependency injection
    deps.set_settings(settings)
    
    # Initialize services with settings
    engine = create_async_engine(settings.database_url)
    
    yield
    
    # Cleanup
    await engine.dispose()

# Routes use dependency injection
@app.get("/v1/health")
async def health(settings: QuerySettings = Depends(get_settings)):
    return {"status": "ok", "version": settings.app_name}
```

### 4.2 Ingest Service Pattern

```python
# modcus_api_ingest/main.py
# Module-level singleton
settings = IngestSettings.load()

# Initialize engine with settings
engine = get_engine(settings)

# Routes access global settings
@app.post("/v1/rag/jobs")
def create_job(job: JobCreate):
    # Uses global settings
    max_queued = settings.ingestion_max_queued
    ...

# Celery tasks load fresh settings
@celery_app.task
def process_job(job_id: str):
    settings = IngestSettings.load()  # Fresh load in worker context
    ...
```

### 4.3 Service Classes Pattern

```python
# Services receive settings via injection
class ChunkingService:
    def __init__(self, settings: IngestSettings):
        self.chunk_sizes = {
            "annual_report": settings.rag_chunk_sizes_annual_report,
            "financial_report": settings.rag_chunk_sizes_financial_report,
            ...
        }
```

## 5. Configuration Validation System

### 5.1 Field-Level Validation (Pydantic)

```python
# Pattern validation
llm_provider: str = Field(
    default="vertex",
    pattern="^(vertex|gemini|openai|anthropic)$"
)

# Range validation
llm_temperature: float = Field(
    default=0.0,
    ge=0.0,  # Greater than or equal
    le=2.0   # Less than or equal
)

# Length validation
api_key: str = Field(
    default="",
    min_length=16
)
```

### 5.2 Custom Validators

```python
@field_validator("api_key", mode="before")
@classmethod
def validate_api_key(cls, v: str) -> str:
    """Validate API key length when set."""
    if v and len(v) < 16:
        raise ValueError("API key must be at least 16 characters when set")
    return v

@model_validator(mode="after")
def validate_vertex_settings(self):
    """Validate that GCP project ID is set when using vertex provider."""
    if self.llm_provider == "vertex" and not self.gcp_project_id:
        raise ValueError("gcp_project_id is required when using 'vertex' provider")
    return self
```

### 5.3 Validation Timing

Validation occurs at **configuration load time**, not runtime:

```python
# Validation happens here ↓
settings = QuerySettings.load()

# If validation fails, exception is raised immediately
# Service never starts with invalid configuration
```

## 6. Configuration Sources by Field Type

### 6.1 YAML-Only Fields (Non-Secrets)

| Category      | Examples                                        |
| ------------- | ----------------------------------------------- |
| App Settings  | `app_name`                                      |
| Logging       | `log_level`, `log_rotation`, `log_retention`    |
| RAG Config    | `rag_chunk_sizes_*`, `rag_retrieval_top_k`      |
| Query Config  | `query_timeout_seconds`, `query_conversation_*` |
| Ingest Config | `ingestion_celery_*`, `ingestion_retry_*`       |
| Rate Limiting | `rate_limiting_*`                               |
| Celery        | `celery_*`                                      |

### 6.2 Environment-Only Fields (Secrets)

| Category      | Examples                                                |
| ------------- | ------------------------------------------------------- |
| Database      | `db_url`, `db_password`                                 |
| Redis         | `redis_url`, `redis_password`                           |
| LLM Keys      | `gemini_api_key`, `openai_api_key`, `anthropic_api_key` |
| GCP           | `gcp_project_id`, `google_application_credentials`      |
| Vector Store  | `qdrant_api_key`, `pgvector_connection_string`          |
| Cloud Storage | `s3_access_key`, `s3_secret_key`, `azure_storage_key`   |
| Security      | `api_key`                                               |

### 6.3 Hybrid Fields (YAML default + Env override)

Some fields can be set in YAML but overridden by environment:

| Field             | YAML Key          | Env Var   |
| ----------------- | ----------------- | --------- |
| `db_url`          | -                 | `DB_URL`  |
| `api_key`         | -                 | `API_KEY` |
| `storage_backend` | `storage.backend` | -         |
| `llm_provider`    | `llm.provider`    | -         |

**Note**: Pydantic Settings prioritizes environment variables over YAML values when both are present.

## 7. File Locations and Responsibilities

### 7.1 Configuration Files

| File                      | Purpose                           | Environment           |
| ------------------------- | --------------------------------- | --------------------- |
| `config.yaml`             | Base configuration with defaults  | All                   |
| `config.development.yaml` | Development overrides             | `APP_ENV=development` |
| `config.production.yaml`  | Production overrides              | `APP_ENV=production`  |
| `.env`                    | Secrets and environment variables | All (gitignored)      |
| `.env.example`            | Template for .env                 | Documentation         |

### 7.2 Settings Code Files

| File                             | Responsibility                               |
| -------------------------------- | -------------------------------------------- |
| `modcus_common/settings/base.py` | BaseSettings class, YAML loading, validation |
| `modcus_common/settings/llm.py`  | LLMSettingsMixin, LLM/Embedding config       |
| `modcus_common/settings/rag.py`  | RAGSettingsMixin, Vector store config        |
| `modcus_api_query/settings.py`   | QuerySettings, Query-specific config         |
| `modcus_api_ingest/settings.py`  | IngestSettings, Ingest-specific config       |

## 8. Key Improvements from Fixes

### 8.1 Critical Fixes

1. **SQLite → PostgreSQL**: Database URL now defaults to PostgreSQL with pattern validation
2. **Logging Key Fix**: Environment configs now use `log:` instead of `logging:`
3. **Provider Validation**: All provider fields have regex pattern validation
4. **Temperature Range**: Enforced 0.0-2.0 range for LLM temperature
5. **Vertex Validation**: Model validator ensures GCP project ID is set for Vertex

### 8.2 Security Improvements

1. **API Key Validation**: Minimum 16 characters required
2. **Pattern Validation**: Invalid provider values rejected at load time
3. **Storage Validation**: Cloud storage settings validated when backend selected

### 8.3 Completeness Improvements

1. **Retry Timing**: Full retry configuration (initial_delay, backoff_factor, max_delay)
2. **Rate Limiting**: API rate limiting settings added
3. **Celery Config**: Complete Celery-specific settings
4. **Storage Backends**: Support for S3, GCS, Azure with validation

## 9. Architecture Robustness Assessment

### Strengths

✅ **Hierarchical Loading**: Clear precedence (Env > Env-YAML > Base-YAML)  
✅ **Type Safety**: Pydantic validates all fields at load time  
✅ **Validation at Load**: Invalid configs fail fast, before service starts  
✅ **Mixin Pattern**: DRY principle, shared config across services  
✅ **Environment Separation**: Easy switching between dev/prod  
✅ **Secret Separation**: Secrets in .env, config in YAML  

### Areas for Improvement

⚠️ **Inconsistent Loading**: Different patterns across services (singleton vs DI)  
⚠️ **Unused Fields**: Some fields defined but not actively used (see 04-field-usage-analysis.md)  
⚠️ **Documentation Gap**: .env.example has misleading comments  

---

**Review Date**: 2026-02-07  
**Post-Fix Version**: v0.2.1
