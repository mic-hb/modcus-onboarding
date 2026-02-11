# Configuration Fields Usage Review Report

## Summary

I have reviewed all configuration fields across the Modcus project. Here are my findings:

---

## 1. FIELDS THAT ARE ACTIVELY USED

### Base Settings ([`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py))

| Field                | Used In                                                                                                                                                                                                                                                                                                             | Status                                            |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| `app_name`           | [`modcus_common/logging/app_logging.py` (line 57)](../../../modcus_common/logging/app_logging.py#L57)                                                                                                                                                                                                               | **✅ USED**                                        |
| `db_url`             | [`modcus_api_ingest/services/tasks/ingestion_tasks.py` (line 32)](../../../modcus_api_ingest/services/tasks/ingestion_tasks.py#L32), [`modcus_api_ingest/api/deps.py` (line 20)](../../../modcus_api_ingest/api/deps.py#L20), [`modcus_common/db/__init__.py` (line 28)](../../../modcus_common/db/__init__.py#L28) | **✅ USED**                                        |
| `db_password`        | -                                                                                                                                                                                                                                                                                                                   | **❌ NOT USED** - Field defined but never accessed |
| `redis_url`          | [`modcus_api_ingest/services/tasks/celery_app.py` (lines 8-9)](../../../modcus_api_ingest/services/tasks/celery_app.py#L8)                                                                                                                                                                                          | **✅ USED**                                        |
| `redis_password`     | -                                                                                                                                                                                                                                                                                                                   | **❌ NOT USED** - Field defined but never accessed |
| `log_level`          | [`modcus_common/logging/app_logging.py` (lines 59, 143)](../../../modcus_common/logging/app_logging.py#L59)                                                                                                                                                                                                         | **✅ USED**                                        |
| `log_dir`            | [`modcus_common/logging/app_logging.py` (lines 44, 57, 94)](../../../modcus_common/logging/app_logging.py#L44)                                                                                                                                                                                                      | **✅ USED**                                        |
| `log_rotation`       | [`modcus_common/logging/app_logging.py` (lines 62, 141)](../../../modcus_common/logging/app_logging.py#L62)                                                                                                                                                                                                         | **✅ USED**                                        |
| `log_retention`      | [`modcus_common/logging/app_logging.py` (lines 63, 142)](../../../modcus_common/logging/app_logging.py#L63)                                                                                                                                                                                                         | **✅ USED**                                        |
| `storage_backend`    | Legacy only - NOT USED in current code (only in `legacy/` health routes)                                                                                                                                                                                                                                            | **❌ NOT USED**                                    |
| `local_storage_base` | -                                                                                                                                                                                                                                                                                                                   | **❌ NOT USED** - Field defined but never accessed |
| `metrics_enabled`    | -                                                                                                                                                                                                                                                                                                                   | **❌ NOT USED** - Field defined but never accessed |
| `metrics_port`       | -                                                                                                                                                                                                                                                                                                                   | **❌ NOT USED** - Field defined but never accessed |
| `api_key`            | [`modcus_api_ingest/api/deps.py` (lines 71, 74)](../../../modcus_api_ingest/api/deps.py#L71)                                                                                                                                                                                                                        | **✅ USED**                                        |

---

### LLM Settings ([`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py))

| Field                            | Used In                                                                                                                                                                                                                                                                                              | Status                     |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| `llm_provider`                   | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py#L109), [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py), [`modcus_api_query/api/routes_health.py`](../../../modcus_api_query/api/routes_health.py) | **✅ USED**                 |
| `llm_model`                      | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py#L110), [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py)                                                                                            | **✅ USED**                 |
| `llm_temperature`                | [`modcus_common/services/llm/llm_rotator.py` (lines 109, 118, 141, 150)](../../../modcus_common/services/llm/llm_rotator.py#L109) via `getattr`                                                                                                                                                      | **✅ USED** (via `getattr`) |
| `llm_max_tokens`                 | [`modcus_common/services/llm/llm_rotator.py` (lines 110, 119, 142, 151)](../../../modcus_common/services/llm/llm_rotator.py#L110) via `getattr`                                                                                                                                                      | **✅ USED** (via `getattr`) |
| `embedding_provider`             | [`modcus_common/services/llm/llm_service.py` (multiple lines)](../../../modcus_common/services/llm/llm_service.py)                                                                                                                                                                                   | **✅ USED**                 |
| `embedding_model`                | [`modcus_common/services/llm/llm_service.py` (multiple lines)](../../../modcus_common/services/llm/llm_service.py)                                                                                                                                                                                   | **✅ USED**                 |
| `embedding_dimension`            | [`modcus_common/services/llm/llm_service.py` (line 70)](../../../modcus_common/services/llm/llm_service.py#L70)                                                                                                                                                                                      | **✅ USED**                 |
| `llm_rotation_enabled`           | -                                                                                                                                                                                                                                                                                                    | **❌ NOT USED**             |
| `llm_rotation_max_retries`       | -                                                                                                                                                                                                                                                                                                    | **❌ NOT USED**             |
| `gemini_api_key`                 | Legacy only (`legacy/` files)                                                                                                                                                                                                                                                                        | **❌ USELESS**              |
| `openai_api_key`                 | Legacy only (`legacy/v0.1/modcus_common/services/llm/openai_embedding.py`)                                                                                                                                                                                                                           | **❌ USELESS**              |
| `anthropic_api_key`              | -                                                                                                                                                                                                                                                                                                    | **❌ NOT USED**             |
| `gcp_project_id`                 | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py), [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py)                                                                                                 | **✅ USED**                 |
| `gcp_location`                   | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py), [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py)                                                                                                 | **✅ USED**                 |
| `google_application_credentials` | Legacy only (`legacy/` files)                                                                                                                                                                                                                                                                        | **❌ USELESS**              |

---

### RAG Settings ([`modcus_common/settings/rag.py`](../../../modcus_common/settings/rag.py))

| Field                              | Used In                                                                                                                                                                        | Status                                               |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- |
| `rag_vector_store`                 | [`modcus_common/services/vector_store/vector_store_factory.py` (line 25)](../../../modcus_common/services/vector_store/vector_store_factory.py#L25), [`tests`](../../../tests) | **✅ USED**                                           |
| `rag_chunk_sizes_annual_report`    | [`modcus_api_ingest/services/chunking/chunking_service.py` (line 49, 57)](../../../modcus_api_ingest/services/chunking/chunking_service.py#L49), [`tests`](../../../tests)     | **✅ USED**                                           |
| `rag_chunk_sizes_financial_report` | [`modcus_api_ingest/services/chunking/chunking_service.py` (line 50, 57)](../../../modcus_api_ingest/services/chunking/chunking_service.py#L50)                                | **✅ USED**                                           |
| `rag_chunk_sizes_news`             | [`modcus_api_ingest/services/chunking/chunking_service.py` (line 51, 57)](../../../modcus_api_ingest/services/chunking/chunking_service.py#L51), [`tests`](../../../tests)     | **✅ USED**                                           |
| `rag_chunk_sizes_public_expose`    | [`modcus_api_ingest/services/chunking/chunking_service.py` (line 52, 57)](../../../modcus_api_ingest/services/chunking/chunking_service.py#L52)                                | **✅ USED**                                           |
| `rag_chunk_overlap`                | [`modcus_api_ingest/services/chunking/chunking_service.py` (line 24)](../../../modcus_api_ingest/services/chunking/chunking_service.py#L24)                                    | **✅ USED**                                           |
| `rag_retrieval_top_k`              | -                                                                                                                                                                              | **❌ NOT USED** - Code uses `retrieval_top_k` instead |
| `rag_retrieval_enable_reranking`   | [`modcus_api_query/services/agents/retrieval_agent.py` (line 103)](../../../modcus_api_query/services/agents/retrieval_agent.py#:L103) via `getattr`                           | **✅ USED** (via `getattr` with wrong name)           |
| `qdrant_url`                       | [`modcus_common/services/vector_store/vector_store_factory.py` (line 28)](../../../modcus_common/services/vector_store/vector_store_factory.py#L28)                            | **✅ USED**                                           |
| `qdrant_api_key`                   | [`modcus_common/services/vector_store/vector_store_factory.py` (line 29)](../../../modcus_common/services/vector_store/vector_store_factory.py#L29)                            | **✅ USED**                                           |
| `chromadb_persist_dir`             | [`modcus_common/services/vector_store/vector_store_factory.py` (line 40-41)](../../../modcus_common/services/vector_store/vector_store_factory.py#L40)                         | **✅ USED**                                           |
| `pgvector_connection_string`       | [`modcus_common/services/vector_store/vector_store_factory.py` (line 50, 57)](../../../modcus_common/services/vector_store/vector_store_factory.py#L50)                        | **✅ USED**                                           |

---

### Query Settings ([`modcus_api_query/settings.py`](../../../modcus_api_query/settings.py))

| Field                                    | Used In                                                                                                                                                                  | Status         |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------- |
| `query_timeout_seconds`                  | Only in docstring example                                                                                                                                                | **❌ NOT USED** |
| `query_complexity_inference_mode`        | [`modcus_api_query/services/agents/validation_agent.py` (lines 131, 150)](../../../modcus_api_query/services/agents/validation_agent.py#L131), [`tests`](../../../tests) | **✅ USED**     |
| `query_conversation_timeout_minutes`     | [`modcus_api_query/services/chat/session_manager.py` (line 75)](../../../modcus_api_query/services/chat/session_manager.py#L75)                                          | **✅ USED**     |
| `query_conversation_sliding_window_size` | [`modcus_api_query/services/chat/session_manager.py` (line 291)](../../../modcus_api_query/services/chat/session_manager.py#L291)                                        | **✅ USED**     |
| `query_context_window_tokens`            | [`modcus_api_query/services/chat/session_manager.py` (line 281)](../../../modcus_api_query/services/chat/session_manager.py#L281) via `getattr`                          | **✅ USED**     |
| `query_ephemeral_artifacts_ttl_hours`    | [`modcus_api_query/services/chat/ephemeral_manager.py` (line 84)](../../../modcus_api_query/services/chat/ephemeral_manager.py#L84)                                      | **✅ USED**     |

---

### Ingest Settings ([`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py))

| Field                                 | Used In                                                                                                                   | Status        |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ------------- |
| `ingestion_celery_concurrency`        | [`modcus_api_ingest/services/tasks/celery_app.py` (line 18)](../../../modcus_api_ingest/services/tasks/celery_app.py#L18) | **✅ USED**    |
| `ingestion_max_queued`                | Legacy only (`legacy/` files)                                                                                             | **❌ USELESS** |
| `ingestion_retry_default_strategy`    | [`modcus_api_ingest/api/routes_ingestion.py` (line 66, 146, 201)](../../../modcus_api_ingest/api/routes_ingestion.py#L66) | **✅ USED**    |
| `ingestion_retry_default_max_retries` | [`modcus_api_ingest/api/routes_ingestion.py` (line 67, 147, 203)](../../../modcus_api_ingest/api/routes_ingestion.py#L67) | **✅ USED**    |

---

## 2. FIELDS THAT ARE DEFINED BUT NEVER USED (USELESS)

| Field                            | Location                                                                    | Recommendation                                               |
| -------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `db_password`                    | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | **🗑️ REMOVE** or implement password extraction from `DB_URL`  |
| `redis_password`                 | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | **🗑️ REMOVE** or implement Redis AUTH support                 |
| `storage_backend`                | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | **🗑️ REMOVE** - Only used in `legacy` health routes           |
| `local_storage_base`             | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | **🗑️ REMOVE** or implement storage service                    |
| `metrics_enabled`                | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | **🗑️ REMOVE** or implement metrics endpoint                   |
| `metrics_port`                   | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | **🗑️ REMOVE** or implement metrics endpoint                   |
| `llm_rotation_enabled`           | [`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py)   | **🗑️ REMOVE** - LLM rotator doesn't check this flag           |
| `llm_rotation_max_retries`       | [`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py)   | **🗑️ REMOVE** or implement in rotator logic                   |
| `gemini_api_key`                 | [`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py)   | **🗑️ REMOVE** - Only used in `legacy` code                    |
| `openai_api_key`                 | [`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py)   | **🗑️ REMOVE** - Only used in `legacy` code                    |
| `anthropic_api_key`              | [`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py)   | **🗑️ REMOVE** - Only used in `legacy` code                    |
| `google_application_credentials` | [`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py)   | **🗑️ REMOVE** - Only used in `legacy` code                    |
| `rag_retrieval_top_k`            | [`modcus_common/settings/rag.py`](../../../modcus_common/settings/rag.py)   | **🔃 CONSOLIDATE** - Code uses `retrieval_top_k` (wrong name) |
| `query_timeout_seconds`          | [`modcus_api_query/settings.py`](../../../modcus_api_query/settings.py)     | **🗑️ REMOVE** or implement timeout logic                      |
| `ingestion_max_queued`           | [`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py)   | **🗑️ REMOVE** - Only used in `legacy` routes                  |

---

## 3. REDUNDANT OR MISNAMED FIELDS

| Issue                | Details                                                                                                                                                                                                                | Recommendation                                                                                                                                                 |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Naming mismatch      | `rag_retrieval_top_k` is defined but code uses `retrieval_top_k` (without `rag_` prefix) in [`modcus_api_query/services/agents/retrieval_agent.py`](../../../modcus_api_query/services/agents/retrieval_agent.py#L210) | Either rename the setting to `retrieval_top_k` or update code to use `rag_retrieval_top_k`                                                                     |
| Naming inconsistency | `rag_retrieval_enable_reranking` is accessed via `getattr(settings, 'retrieval_enable_reranking', False)` - missing `rag_` prefix                                                                                      | Update code in [`modcus_api_query/services/agents/retrieval_agent.py`](../../../modcus_api_query/services/agents/retrieval_agent.py) to use correct field name |

---

## 4. UNUSED ENVIRONMENT VARIABLES IN [`.env.example`](../../../.env.example)

These variables are defined in [`.env.example`](../../../.env.example) but not used in the settings classes:

| Variable                       | Issue                                                 |
| ------------------------------ | ----------------------------------------------------- |
| `POSTGRES_HOST`                | ℹ️ Only used to construct `DB_URL`, not read directly  |
| `POSTGRES_PORT`                | ℹ️ Only used to construct `DB_URL`, not read directly  |
| `POSTGRES_DB`                  | ℹ️ Only used to construct `DB_URL`, not read directly  |
| `POSTGRES_USER`                | ℹ️ Only used to construct `DB_URL`, not read directly  |
| `POSTGRES_PASSWORD`            | ℹ️ Only used to construct `DB_URL`, not read directly  |
| `REDIS_HOST_PORT`              | 🐋 Docker-only, not used by application                |
| `VECTOR_STORE`                 | ⚠️ Duplicate of `rag_vector_store` in YAML             |
| `QDRANT_PORT`                  | 🐋 Docker-only, not used by application                |
| `QDRANT_GRPC_PORT`             | 🐋 Docker-only, not used by application                |
| `CHROMADB_URL`                 | ⚠️ Not used - only `CHROMADB_PERSIST_DIR` is used      |
| `CHROMADB_PORT`                | 🐋 Docker-only, not used by application                |
| `QUERY_API_HOST_PORT`          | 🐋 Docker-only, not used by application                |
| `INGEST_API_HOST_PORT`         | 🐋 Docker-only, not used by application                |
| `UID`                          | 🐋 Docker-only, not used by application                |
| `GID`                          | 🐋 Docker-only, not used by application                |
| `CELERY_CONCURRENCY`           | ⚠️ Duplicate of `ingestion_celery_concurrency` in YAML |
| `QUERY_TIMEOUT_S`              | ⚠️ Duplicate of `query_timeout_seconds`                |
| `COMPLEXITY_INFERENCE_MODE`    | ⚠️ Duplicate of `query_complexity_inference_mode`      |
| `CONVERSATION_TIMEOUT_MINUTES` | ⚠️ Duplicate of `query_conversation_timeout_minutes`   |
| `EPHEMERAL_ARTIFACT_TTL_HOURS` | ⚠️ Duplicate of `query_ephemeral_artifacts_ttl_hours`  |

---

## 5. RECOMMENDATIONS

### High Priority (Fix Naming Issues)

1. **Fix `rag_retrieval_top_k` / `retrieval_top_k` mismatch:**
   - The setting is defined as `rag_retrieval_top_k` in `rag.py`
   - But code in `retrieval_agent.py` looks for `retrieval_top_k`
   - Fix: Update `retrieval_agent.py` line 210 to use `rag_retrieval_top_k`

2. **Fix `rag_retrieval_enable_reranking` access:**
   - Code uses `getattr(settings, 'retrieval_enable_reranking', False)`
   - Should be `getattr(settings, 'rag_retrieval_enable_reranking', False)`

### Medium Priority (Remove Unused Fields)

1. **Remove truly unused fields from settings classes:**
   - `db_password` (unless implementing DB URL parsing)
   - `redis_password` (unless implementing Redis AUTH)
   - `storage_backend` (only used in legacy)
   - `local_storage_base` (never used)
   - `metrics_enabled` / `metrics_port` (unless implementing Prometheus)
   - `llm_rotation_enabled` / `llm_rotation_max_retries` (unless implementing)
   - `gemini_api_key`, `openai_api_key`, `anthropic_api_key` (API keys now in DB)
   - `google_application_credentials` (now using ADC)
   - `query_timeout_seconds` (unless implementing request timeouts)
   - `ingestion_max_queued` (unless implementing queue limits)

2. **Clean up `.env.example`:**
   - Remove duplicates that are now in YAML config
   - Clarify which are Docker-only vs application settings
   - Group by purpose (Docker, Application, Optional)

### Low Priority (Documentation)

1. **Document settings hierarchy:**
   - YAML config files are the primary configuration method
   - Environment variables only for secrets and Docker overrides
   - Add comments in `.env.example` to clarify Docker-only variables

---

## Files Affected

Settings Definitions:
- [modcus_common/settings/base.py](../../../modcus_common/settings/base.py) - 6 unused fields
- [modcus_common/settings/llm.py](../../../modcus_common/settings/llm.py) - 7 unused fields
- [modcus_common/settings/rag.py](../../../modcus_common/settings/rag.py) - 1 naming mismatch
- [modcus_api_query/settings.py](../../../modcus_api_query/settings.py) - 1 unused field
- [modcus_api_ingest/settings.py](../../../modcus_api_ingest/settings.py) - 1 unused field

Code Files Needing Updates:
- [modcus_api_query/services/agents/retrieval_agent.py](../../../modcus_api_query/services/agents/retrieval_agent.py#L103) - Lines 103, 210 (wrong setting names)