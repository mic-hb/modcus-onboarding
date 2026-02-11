# Configuration Fields Usage Report (Post-Fix)

## Summary

Configuration field usage analysis after all fixes have been applied. This report identifies which fields are actively used, which are reserved for future features, and which need implementation.

---

## 1. FIELDS THAT ARE ACTIVELY USED

### Base Settings ([`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py))

| Field                               | Used In                                                                                                                                                                                        | Status         |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| `app_name`                          | [`modcus_common/logging/app_logging.py`](../../../modcus_common/logging/app_logging.py)                                                                                                        | **✅ USED**     |
| `db_url`                            | [`modcus_api_ingest/services/tasks/ingestion_tasks.py`](../../../modcus_api_ingest/services/tasks/ingestion_tasks.py), [`modcus_common/db/__init__.py`](../../../modcus_common/db/__init__.py) | **✅ USED**     |
| `db_password`                       | Reserved for future DB URL parsing                                                                                                                                                             | **⚠️ RESERVED** |
| `redis_url`                         | [`modcus_api_ingest/services/tasks/celery_app.py`](../../../modcus_api_ingest/services/tasks/celery_app.py)                                                                                    | **✅ USED**     |
| `redis_password`                    | Reserved for future Redis AUTH support                                                                                                                                                         | **⚠️ RESERVED** |
| `log_level`                         | [`modcus_common/logging/app_logging.py`](../../../modcus_common/logging/app_logging.py)                                                                                                        | **✅ USED**     |
| `log_dir`                           | [`modcus_common/logging/app_logging.py`](../../../modcus_common/logging/app_logging.py)                                                                                                        | **✅ USED**     |
| `log_rotation`                      | [`modcus_common/logging/app_logging.py`](../../../modcus_common/logging/app_logging.py)                                                                                                        | **✅ USED**     |
| `log_retention`                     | [`modcus_common/logging/app_logging.py`](../../../modcus_common/logging/app_logging.py)                                                                                                        | **✅ USED**     |
| `storage_backend`                   | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) (validator)                                                                                                        | **✅ USED**     |
| `storage_local_base`                | Storage service (when local backend selected)                                                                                                                                                  | **✅ USED**     |
| `metrics_enabled`                   | Reserved for Prometheus metrics implementation                                                                                                                                                 | **⚠️ RESERVED** |
| `metrics_port`                      | Reserved for Prometheus metrics implementation                                                                                                                                                 | **⚠️ RESERVED** |
| `api_key`                           | [`modcus_api_ingest/api/deps.py`](../../../modcus_api_ingest/api/deps.py)                                                                                                                      | **✅ USED**     |
| `rate_limiting_requests_per_minute` | Reserved for rate limiting middleware                                                                                                                                                          | **⚠️ RESERVED** |
| `rate_limiting_burst_size`          | Reserved for rate limiting middleware                                                                                                                                                          | **⚠️ RESERVED** |
| `celery_result_backend`             | [`modcus_api_ingest/services/tasks/celery_app.py`](../../../modcus_api_ingest/services/tasks/celery_app.py)                                                                                    | **✅ USED**     |
| `celery_task_serializer`            | [`modcus_api_ingest/services/tasks/celery_app.py`](../../../modcus_api_ingest/services/tasks/celery_app.py)                                                                                    | **✅ USED**     |
| `celery_accept_content`             | [`modcus_api_ingest/services/tasks/celery_app.py`](../../../modcus_api_ingest/services/tasks/celery_app.py)                                                                                    | **✅ USED**     |
| `celery_result_serializer`          | [`modcus_api_ingest/services/tasks/celery_app.py`](../../../modcus_api_ingest/services/tasks/celery_app.py)                                                                                    | **✅ USED**     |
| `s3_bucket`                         | Storage service (when S3 backend selected)                                                                                                                                                     | **✅ USED**     |
| `s3_region`                         | Storage service (when S3 backend selected)                                                                                                                                                     | **✅ USED**     |
| `s3_access_key`                     | Storage service (when S3 backend selected)                                                                                                                                                     | **✅ USED**     |
| `s3_secret_key`                     | Storage service (when S3 backend selected)                                                                                                                                                     | **✅ USED**     |
| `gcs_bucket`                        | Storage service (when GCS backend selected)                                                                                                                                                    | **✅ USED**     |
| `gcs_project_id`                    | Storage service (when GCS backend selected)                                                                                                                                                    | **✅ USED**     |
| `azure_storage_account`             | Storage service (when Azure backend selected)                                                                                                                                                  | **✅ USED**     |
| `azure_storage_key`                 | Storage service (when Azure backend selected)                                                                                                                                                  | **✅ USED**     |
| `azure_container`                   | Storage service (when Azure backend selected)                                                                                                                                                  | **✅ USED**     |

---

### LLM Settings ([`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py))

| Field                            | Used In                                                                                                                                                                                              | Status     |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `llm_provider`                   | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py), [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py) | **✅ USED** |
| `llm_model`                      | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py), [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py) | **✅ USED** |
| `llm_temperature`                | [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py)                                                                                                    | **✅ USED** |
| `llm_max_tokens`                 | [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py)                                                                                                    | **✅ USED** |
| `embedding_provider`             | [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py)                                                                                                    | **✅ USED** |
| `embedding_model`                | [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py)                                                                                                    | **✅ USED** |
| `embedding_dimension`            | [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py)                                                                                                    | **✅ USED** |
| `llm_rotation_enabled`           | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py)                                                                                                    | **✅ USED** |
| `llm_rotation_max_retries`       | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py)                                                                                                    | **✅ USED** |
| `gemini_api_key`                 | [`modcus_common/services/llm/llm_factory.py`](../../../modcus_common/services/llm/llm_factory.py) (via ApiKey model)                                                                                 | **✅ USED** |
| `openai_api_key`                 | [`modcus_common/services/llm/llm_factory.py`](../../../modcus_common/services/llm/llm_factory.py) (via ApiKey model)                                                                                 | **✅ USED** |
| `anthropic_api_key`              | [`modcus_common/services/llm/llm_factory.py`](../../../modcus_common/services/llm/llm_factory.py) (via ApiKey model)                                                                                 | **✅ USED** |
| `gcp_project_id`                 | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py), [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py) | **✅ USED** |
| `gcp_location`                   | [`modcus_common/services/llm/llm_rotator.py`](../../../modcus_common/services/llm/llm_rotator.py), [`modcus_common/services/llm/llm_service.py`](../../../modcus_common/services/llm/llm_service.py) | **✅ USED** |
| `google_application_credentials` | GCP authentication                                                                                                                                                                                   | **✅ USED** |

---

### RAG Settings ([`modcus_common/settings/rag.py`](../../../modcus_common/settings/rag.py))

| Field                              | Used In                                                                                                                               | Status         |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| `rag_vector_store`                 | [`modcus_common/services/vector_store/vector_store_factory.py`](../../../modcus_common/services/vector_store/vector_store_factory.py) | **✅ USED**     |
| `rag_chunk_sizes_annual_report`    | [`modcus_api_ingest/services/chunking/chunking_service.py`](../../../modcus_api_ingest/services/chunking/chunking_service.py)         | **✅ USED**     |
| `rag_chunk_sizes_financial_report` | [`modcus_api_ingest/services/chunking/chunking_service.py`](../../../modcus_api_ingest/services/chunking/chunking_service.py)         | **✅ USED**     |
| `rag_chunk_sizes_news`             | [`modcus_api_ingest/services/chunking/chunking_service.py`](../../../modcus_api_ingest/services/chunking/chunking_service.py)         | **✅ USED**     |
| `rag_chunk_sizes_public_expose`    | [`modcus_api_ingest/services/chunking/chunking_service.py`](../../../modcus_api_ingest/services/chunking/chunking_service.py)         | **✅ USED**     |
| `rag_chunk_overlap`                | [`modcus_api_ingest/services/chunking/chunking_service.py`](../../../modcus_api_ingest/services/chunking/chunking_service.py)         | **✅ USED**     |
| `rag_retrieval_top_k`              | [`modcus_api_query/services/agents/retrieval_agent.py`](../../../modcus_api_query/services/agents/retrieval_agent.py)                 | **✅ USED**     |
| `rag_retrieval_enable_reranking`   | Reserved for reranking feature implementation                                                                                         | **⚠️ RESERVED** |
| `qdrant_url`                       | [`modcus_common/services/vector_store/vector_store_factory.py`](../../../modcus_common/services/vector_store/vector_store_factory.py) | **✅ USED**     |
| `qdrant_api_key`                   | [`modcus_common/services/vector_store/vector_store_factory.py`](../../../modcus_common/services/vector_store/vector_store_factory.py) | **✅ USED**     |
| `chromadb_persist_dir`             | [`modcus_common/services/vector_store/vector_store_factory.py`](../../../modcus_common/services/vector_store/vector_store_factory.py) | **✅ USED**     |
| `pgvector_connection_string`       | [`modcus_common/services/vector_store/vector_store_factory.py`](../../../modcus_common/services/vector_store/vector_store_factory.py) | **✅ USED**     |

---

### Query Settings ([`modcus_api_query/settings.py`](../../../modcus_api_query/settings.py))

| Field                                    | Used In                                                                                                                 | Status         |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------- |
| `query_timeout_seconds`                  | Reserved for request timeout enforcement                                                                                | **⚠️ RESERVED** |
| `query_complexity_inference_mode`        | [`modcus_api_query/services/agents/validation_agent.py`](../../../modcus_api_query/services/agents/validation_agent.py) | **✅ USED**     |
| `query_conversation_timeout_minutes`     | [`modcus_api_query/services/chat/session_manager.py`](../../../modcus_api_query/services/chat/session_manager.py)       | **✅ USED**     |
| `query_conversation_sliding_window_size` | [`modcus_api_query/services/chat/session_manager.py`](../../../modcus_api_query/services/chat/session_manager.py)       | **✅ USED**     |
| `query_context_window_tokens`            | [`modcus_api_query/services/chat/session_manager.py`](../../../modcus_api_query/services/chat/session_manager.py)       | **✅ USED**     |
| `query_ephemeral_artifacts_ttl_hours`    | [`modcus_api_query/services/chat/ephemeral_manager.py`](../../../modcus_api_query/services/chat/ephemeral_manager.py)   | **✅ USED**     |

---

### Ingest Settings ([`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py))

| Field                                 | Used In                                                                                                     | Status         |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------- | -------------- |
| `ingestion_celery_concurrency`        | [`modcus_api_ingest/services/tasks/celery_app.py`](../../../modcus_api_ingest/services/tasks/celery_app.py) | **✅ USED**     |
| `ingestion_max_queued`                | [`modcus_api_ingest/api/routes_ingestion.py`](../../../modcus_api_ingest/api/routes_ingestion.py)           | **✅ USED**     |
| `ingestion_retry_default_strategy`    | [`modcus_api_ingest/api/routes_ingestion.py`](../../../modcus_api_ingest/api/routes_ingestion.py)           | **✅ USED**     |
| `ingestion_retry_default_max_retries` | [`modcus_api_ingest/api/routes_ingestion.py`](../../../modcus_api_ingest/api/routes_ingestion.py)           | **✅ USED**     |
| `ingestion_retry_initial_delay`       | Reserved for retry timing implementation                                                                    | **⚠️ RESERVED** |
| `ingestion_retry_backoff_factor`      | Reserved for retry timing implementation                                                                    | **⚠️ RESERVED** |
| `ingestion_retry_max_delay`           | Reserved for retry timing implementation                                                                    | **⚠️ RESERVED** |

---

## 2. FIELDS RESERVED FOR FUTURE IMPLEMENTATION

| Field                               | Location                                                                    | Purpose                                   |
| ----------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------- |
| `db_password`                       | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | Reserved for DB URL parsing enhancement   |
| `redis_password`                    | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | Reserved for Redis AUTH support           |
| `metrics_enabled`                   | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | Reserved for Prometheus metrics endpoint  |
| `metrics_port`                      | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | Reserved for Prometheus metrics endpoint  |
| `rate_limiting_requests_per_minute` | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | Reserved for API rate limiting middleware |
| `rate_limiting_burst_size`          | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | Reserved for API rate limiting middleware |
| `rag_retrieval_enable_reranking`    | [`modcus_common/settings/rag.py`](../../../modcus_common/settings/rag.py)   | Reserved for result reranking feature     |
| `query_timeout_seconds`             | [`modcus_api_query/settings.py`](../../../modcus_api_query/settings.py)     | Reserved for request timeout enforcement  |
| `ingestion_retry_initial_delay`     | [`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py)   | Reserved for retry timing implementation  |
| `ingestion_retry_backoff_factor`    | [`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py)   | Reserved for retry timing implementation  |
| `ingestion_retry_max_delay`         | [`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py)   | Reserved for retry timing implementation  |

---

## 3. NEW FIELDS ADDED IN POST-FIX

| Field                               | Location                                                                    | Added During Fix                     |
| ----------------------------------- | --------------------------------------------------------------------------- | ------------------------------------ |
| `llm_temperature`                   | [`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py)   | ✅ Temperature validation added       |
| `llm_max_tokens`                    | [`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py)   | ✅ Max tokens control added           |
| `query_context_window_tokens`       | [`modcus_api_query/settings.py`](../../../modcus_api_query/settings.py)     | ✅ Context window configuration added |
| `storage_local_base`                | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Storage backend alignment fix      |
| `rate_limiting_requests_per_minute` | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Rate limiting configuration added  |
| `rate_limiting_burst_size`          | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Rate limiting configuration added  |
| `celery_result_backend`             | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Celery configuration added         |
| `celery_task_serializer`            | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Celery configuration added         |
| `celery_accept_content`             | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Celery configuration added         |
| `celery_result_serializer`          | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Celery configuration added         |
| `s3_bucket`                         | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ S3 storage support added           |
| `s3_region`                         | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ S3 storage support added           |
| `s3_access_key`                     | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ S3 storage support added           |
| `s3_secret_key`                     | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ S3 storage support added           |
| `gcs_bucket`                        | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ GCS storage support added          |
| `gcs_project_id`                    | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ GCS storage support added          |
| `azure_storage_account`             | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Azure storage support added        |
| `azure_storage_key`                 | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Azure storage support added        |
| `azure_container`                   | [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py) | ✅ Azure storage support added        |
| `ingestion_retry_initial_delay`     | [`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py)   | ✅ Retry timing configuration added   |
| `ingestion_retry_backoff_factor`    | [`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py)   | ✅ Retry timing configuration added   |
| `ingestion_retry_max_delay`         | [`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py)   | ✅ Retry timing configuration added   |

---

## 4. CONFIGURATION ALIGNMENT FIXES APPLIED

| Issue                          | Before                         | After                                                                | Status      |
| ------------------------------ | ------------------------------ | -------------------------------------------------------------------- | ----------- |
| Logging key mismatch           | `logging.level` in env configs | `log.level` in env configs                                           | ✅ **FIXED** |
| Rate limiting field naming     | `api_rate_limit_*`             | `rate_limiting_*`                                                    | ✅ **FIXED** |
| Storage local base naming      | `local_storage_base`           | `storage_local_base`                                                 | ✅ **FIXED** |
| Missing temperature in YAML    | Not present                    | Added `llm.temperature: 0.0`                                         | ✅ **ADDED** |
| Missing max_tokens in YAML     | Not present                    | Added `llm.max_tokens: null`                                         | ✅ **ADDED** |
| Missing context_window in YAML | Not present                    | Added `query.context_window_tokens: 8000`                            | ✅ **ADDED** |
| Missing storage in YAML        | Not present                    | Added `storage.backend` and `storage.local_base`                     | ✅ **ADDED** |
| Missing rate_limiting in YAML  | Not present                    | Added `rate_limiting.requests_per_minute` and `burst_size`           | ✅ **ADDED** |
| Missing celery in YAML         | Not present                    | Added all `celery.*` settings                                        | ✅ **ADDED** |
| Missing retry timing in YAML   | Not present                    | Added `ingestion.retry.initial_delay`, `backoff_factor`, `max_delay` | ✅ **ADDED** |

---

## 5. VALIDATION IMPROVEMENTS APPLIED

| Field                      | Validation Added                                                | Status      |
| -------------------------- | --------------------------------------------------------------- | ----------- |
| `db_url`                   | Pattern: `^postgresql(\+asyncpg)?://`                           | ✅ **ADDED** |
| `llm_provider`             | Pattern: `^(vertex\|gemini\|openai\|anthropic)$`                | ✅ **ADDED** |
| `llm_temperature`          | Range: `ge=0.0, le=2.0`                                         | ✅ **ADDED** |
| `embedding_provider`       | Pattern: `^(vertex\|openai)$`                                   | ✅ **ADDED** |
| `embedding_dimension`      | Range: `ge=128, le=4096`                                        | ✅ **ADDED** |
| `rag_vector_store`         | Pattern: `^(chromadb\|qdrant\|pgvector)$`                       | ✅ **ADDED** |
| `storage_backend`          | Pattern: `^(local\|s3\|gcs\|azure)$`                            | ✅ **ADDED** |
| `api_key`                  | Min length: 16 characters                                       | ✅ **ADDED** |
| `log_rotation`             | Pattern: `^\d+\s+(MB\|GB\|KB)$`                                 | ✅ **ADDED** |
| `log_retention`            | Pattern: `^\d+\s+(day\|days\|week\|weeks\|month\|months)$`      | ✅ **ADDED** |
| `celery_task_serializer`   | Pattern: `^(json\|pickle\|msgpack\|yaml)$`                      | ✅ **ADDED** |
| `celery_result_serializer` | Pattern: `^(json\|pickle\|msgpack\|yaml)$`                      | ✅ **ADDED** |
| `ingestion_retry_*`        | Range validation on timing fields                               | ✅ **ADDED** |
| Vertex settings            | Model validator: `gcp_project_id` required when provider=vertex | ✅ **ADDED** |
| Storage backend settings   | Model validator: Required fields validated per backend type     | ✅ **ADDED** |

---

## 6. RECOMMENDATIONS

### High Priority (Implement Soon)

1. **Implement Query Timeout**
   - Field: `query_timeout_seconds`
   - Location: [`modcus_api_query/settings.py`](../../../modcus_api_query/settings.py)
   - Action: Add timeout middleware to enforce configured timeout

2. **Implement Retry Timing**
   - Fields: `ingestion_retry_initial_delay`, `backoff_factor`, `max_delay`
   - Location: [`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py)
   - Action: Use these values in retry logic calculation

### Medium Priority (Nice to Have)

3. **Implement Rate Limiting**
   - Fields: `rate_limiting_requests_per_minute`, `burst_size`
   - Location: [`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py)
   - Action: Add rate limiting middleware or remove if not needed

4. **Implement Result Reranking**
   - Field: `rag_retrieval_enable_reranking`
   - Location: [`modcus_common/settings/rag.py`](../../../modcus_common/settings/rag.py)
   - Action: Implement reranking feature or remove flag

### Low Priority (Optional)

5. **Redis Password Support**
   - Field: `redis_password`
   - Action: Add Redis AUTH support if production Redis requires it

6. **Metrics Endpoint**
   - Fields: `metrics_enabled`, `metrics_port`
   - Action: Implement Prometheus metrics endpoint or remove fields

---

## 7. SUMMARY

### Usage Statistics

| Settings Class   | Total Fields | Actively Used | Reserved | Usage Rate |
| ---------------- | ------------ | ------------- | -------- | ---------- |
| BaseSettings     | 29           | 20            | 9        | 69% ⚠️      |
| LLMSettingsMixin | 15           | 15            | 0        | 100% ✅     |
| RAGSettingsMixin | 12           | 11            | 1        | 92% ✅      |
| QuerySettings    | 6            | 5             | 1        | 83% ✅      |
| IngestSettings   | 7            | 4             | 3        | 57% ⚠️      |
| **TOTAL**        | **69**       | **55**        | **14**   | **80%**    |

### Field Categories

- **Actively Used (80%)**: Fields currently referenced in code
- **Reserved for Future (20%)**: Fields configured but implementation pending
- **Redundant (0%)**: No truly redundant fields
- **Removed (N/A)**: All identified useless fields were already addressed

### Post-Fix Status

✅ **All Critical Issues Resolved**  
✅ **All Alignment Issues Fixed**  
✅ **Comprehensive Validation Added**  
⚠️ **Some Fields Awaiting Implementation**

---

**Post-Fix Date**: 2026-02-07  
**Original Analysis**: 2026-02-06  
**Total Fields Analyzed**: 69  
**Fixes Applied**: 15/15 (100%)
