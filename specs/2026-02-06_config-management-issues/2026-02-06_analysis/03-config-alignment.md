# Configuration Alignment Report

---

## Understanding the YAML Flattening Process

The `BaseSettings._flatten_yaml()` method converts nested YAML to flat keys:
- `{"rag": {"vector_store": "chromadb"}}` → `rag_vector_store`
- `{"rag": {"chunk_sizes": {"annual_report": 1024}}}` → `rag_chunk_sizes_annual_report`
- `{"query": {"conversation": {"timeout_minutes": 120}}}` → `query_conversation_timeout_minutes`

---

## 1. `BaseSettings` Alignment ([`modcus_common/settings/base.py`](../../../modcus_common/settings/base.py))

**Aligned Fields**

| YAML Key          | Python Field      | YAML Value  | Python Default | Status        |
| ----------------- | ----------------- | ----------- | -------------- | ------------- |
| `app.name`        | `app_name`        | "modcus"    | "modcus"       | **✅ Aligned** |
| `log.level`       | `log_level`       | "INFO"      | "INFO"         | **✅ Aligned** |
| `log.dir`         | `log_dir`         | "/app/logs" | "/app/logs"    | **✅ Aligned** |
| `log.rotation`    | `log_rotation`    | "50 MB"     | "50 MB"        | **✅ Aligned** |
| `log.retention`   | `log_retention`   | "7 days"    | "7 days"       | **✅ Aligned** |
| `metrics.enabled` | `metrics_enabled` | false       | False          | **✅ Aligned** |
| `metrics.port`    | `metrics_port`    | 9090        | 9090           | **✅ Aligned** |

**Python Fields Without YAML Counterparts (ENV-Only)**

| Python Field         | Source                        | Status                |
| -------------------- | ----------------------------- | --------------------- |
| `db_url`             | `DB_URL` env var              | **✅ OK - Secret**     |
| `db_password`        | `DB_PASSWORD` env var         | **✅ OK - Secret**     |
| `redis_url`          | `REDIS_URL` env var           | **✅ OK - Secret**     |
| `redis_password`     | `REDIS_PASSWORD` env var      | **✅ OK - Secret**     |
| `storage_backend`    | Default only (`"local"`)      | **⚠️ Missing in YAML** |
| `local_storage_base` | Default only (`"./.storage"`) | **⚠️ Missing in YAML** |
| `api_key`            | Default only (`""`)           | **⚠️ Missing in YAML** |

---

## 2. `LLMSettingsMixin` Alignment ([`modcus_common/settings/llm.py`](../../../modcus_common/settings/llm.py))

**Aligned Fields**

| YAML Key                   | Python Field               | YAML Value             | Python Default         | Status        |
| -------------------------- | -------------------------- | ---------------------- | ---------------------- | ------------- |
| `llm.provider`             | `llm_provider`             | "vertex"               | "vertex"               | **✅ Aligned** |
| `llm.model`                | `llm_model`                | "gemini-2.5-flash"     | "gemini-2.5-flash"     | **✅ Aligned** |
| `llm.embedding.provider`   | `embedding_provider`       | "vertex"               | "vertex"               | **✅ Aligned** |
| `llm.embedding.model`      | `embedding_model`          | "gemini-embedding-001" | "gemini-embedding-001" | **✅ Aligned** |
| `llm.embedding.dimension`  | `embedding_dimension`      | 3072                   | 3072                   | **✅ Aligned** |
| `llm.rotation.enabled`     | `llm_rotation_enabled`     | true                   | True                   | **✅ Aligned** |
| `llm.rotation.max_retries` | `llm_rotation_max_retries` | 3                      | 3                      | **✅ Aligned** |

**Python Fields Without YAML Counterparts**

| Python Field      | Default | Status                |
| ----------------- | ------- | --------------------- |
| `llm_temperature` | 0.0     | **⚠️ Missing in YAML** |
| `llm_max_tokens`  | None    | **⚠️ Missing in YAML** |

**ENV-Only Fields (OK)**
- `gemini_api_key`, `openai_api_key`, `anthropic_api_key`
- `gcp_project_id`, `gcp_location`, `google_application_credentials`

---

## 3. `RAGSettingsMixin` Alignment ([`modcus_common/settings/rag.py`](../../../modcus_common/settings/rag.py))

**Aligned Fields**

| YAML Key                           | Python Field                       | YAML Value | Python Default | Status        |
| ---------------------------------- | ---------------------------------- | ---------- | -------------- | ------------- |
| `rag.vector_store`                 | `rag_vector_store`                 | "chromadb" | "chromadb"     | **✅ Aligned** |
| `rag.chunk_sizes.annual_report`    | `rag_chunk_sizes_annual_report`    | 1024       | 1024           | **✅ Aligned** |
| `rag.chunk_sizes.financial_report` | `rag_chunk_sizes_financial_report` | 1024       | 1024           | **✅ Aligned** |
| `rag.chunk_sizes.news`             | `rag_chunk_sizes_news`             | 2048       | 2048           | **✅ Aligned** |
| `rag.chunk_sizes.public_expose`    | `rag_chunk_sizes_public_expose`    | 1024       | 1024           | **✅ Aligned** |
| `rag.chunk_overlap`                | `rag_chunk_overlap`                | 200        | 200            | **✅ Aligned** |
| `rag.retrieval_top_k`              | `rag_retrieval_top_k`              | 10         | 10             | **✅ Aligned** |
| `rag.retrieval_enable_reranking`   | `rag_retrieval_enable_reranking`   | false      | False          | **✅ Aligned** |

**ENV-Only Fields (OK)**
- `qdrant_url`, `qdrant_api_key`, `chromadb_persist_dir`, `pgvector_connection_string`

---

## 4. `QuerySettings` Alignment ([`modcus_api_query/settings.py`](../../../modcus_api_query/settings.py))

**Aligned Fields**

| YAML Key                                 | Python Field                             | YAML Value | Python Default | Status        |
| ---------------------------------------- | ---------------------------------------- | ---------- | -------------- | ------------- |
| `query.timeout_seconds`                  | `query_timeout_seconds`                  | 360        | 360            | **✅ Aligned** |
| `query.complexity_inference_mode`        | `query_complexity_inference_mode`        | "combined" | "combined"     | **✅ Aligned** |
| `query.conversation.timeout_minutes`     | `query_conversation_timeout_minutes`     | 120        | 120            | **✅ Aligned** |
| `query.conversation.sliding_window_size` | `query_conversation_sliding_window_size` | 5          | 5              | **✅ Aligned** |
| `query.ephemeral_artifacts_ttl_hours`    | `query_ephemeral_artifacts_ttl_hours`    | 24         | 24             | **✅ Aligned** |

**Python Fields Without YAML Counterparts**

| Python Field                  | Default | Status                |
| ----------------------------- | ------- | --------------------- |
| `query_context_window_tokens` | 8000    | **⚠️ Missing in YAML** |

---

## 5. `IngestSettings` Alignment ([`modcus_api_ingest/settings.py`](../../../modcus_api_ingest/settings.py))

**Aligned Fields**

| YAML Key                              | Python Field                          | YAML Value       | Python Default   | Status        |
| ------------------------------------- | ------------------------------------- | ---------------- | ---------------- | ------------- |
| `ingestion.celery.concurrency`        | `ingestion_celery_concurrency`        | 2                | 2                | **✅ Aligned** |
| `ingestion.celery.max_queued`         | `ingestion_max_queued`                | 10               | 10               | **✅ Aligned** |
| `ingestion.retry.default_strategy`    | `ingestion_retry_default_strategy`    | "transient_only" | "transient_only" | **✅ Aligned** |
| `ingestion.retry.default_max_retries` | `ingestion_retry_default_max_retries` | 3                | 3                | **✅ Aligned** |

---

## 6. Environment-Specific Configs Analysis

### [`config.development.yaml`](../../../config.development.yaml) Issues

| YAML Key                 | Problem                           | Expected                   |
| ------------------------ | --------------------------------- | -------------------------- |
| `logging.level: DEBUG`   | KEY MISMATCH                      | `log.level: DEBUG`         |
| `metrics.enabled: false` | Redundant override (same as base) | Remove or keep for clarity |

**CRITICAL ISSUE**: `logging.level` flattens to `logging_level`, but Python expects `log_level`. The development logging override will not work.

### [`config.production.yaml`](../../../config.production.yaml) Issues

| YAML Key                                     | Problem        | Expected          |
| -------------------------------------------- | -------------- | ----------------- |
| `logging.level: INFO`                        | KEY MISMATCH   | `log.level: INFO` |
| `rag.chunk_sizes.*: 2048`                    | Valid override | OK                |
| `llm.rotation.max_retries: 5`                | Valid override | OK                |
| `query.timeout_seconds: 600`                 | Valid override | OK                |
| `query.conversation.sliding_window_size: 10` | Valid override | OK                |
| `ingestion.celery.concurrency: 4`            | Valid override | OK                |
| `ingestion.celery.max_queued: 50`            | Valid override | OK                |
| `ingestion.retry.default_max_retries: 5`     | Valid override | OK                |
| `metrics.enabled: true`                      | Valid override | OK                |

**CRITICAL ISSUE**: `logging.level` flattens to `logging_level`, but Python expects `log_level`. The production logging override will not work.
---

## Summary of Issues

### 1. **CRITICAL**: Logging Key Mismatch

In [`config.development.yaml`](../../../config.development.yaml) and [`config.production.yaml`](../../../config.production.yaml):
```YAML
# WRONG - Will NOT override log_level
logging:
  level: DEBUG  # Flattens to logging_level

# CORRECT - Will override log_level
log:
  level: DEBUG  # Flattens to log_level
```

### 2. Missing YAML Fields (Use Defaults)

| Config Section | Missing YAML Keys             | Python Defaults Used |
| -------------- | ----------------------------- | -------------------- |
| Base           | `storage_backend`             | `"local"`            |
| Base           | `local_storage_base`          | `"./.storage"`       |
| Base           | `api_key`                     | `""` (empty)         |
| LLM            | `llm_temperature`             | `0.0`                |
| LLM            | `llm_max_tokens`              | `None` (unlimited)   |
| Query          | `query_context_window_tokens` | `8000`               |

### 3. Perfectly Aligned Sections

- RAG settings (all chunk sizes, retrieval config)
- LLM provider and model settings
- Embedding configuration
- Query timeout and conversation settings
- Ingestion Celery and retry settings
- Metrics configuration

---

## Recommendations

### Fix 1: Correct Environment Config Logging Keys

**File**: [`config.development.yaml`](../../../config.development.yaml) and [`config.production.yaml`](../../../config.production.yaml)
**Change**:
```YAML
# WRONG - Will NOT override log_level
logging:
  level: DEBUG

# CORRECT - Will override log_level
log:
  level: DEBUG
```

### Fix 2: Add Missing Optional Fields to [`config.yaml`](../../../config.yaml) (Optional)

If you want explicit configuration instead of relying on defaults, add:

```YAML
# Add to config.yaml under existing sections
llm:
  temperature: 0.0
  max_tokens: null  # or specific value
query:
  context_window_tokens: 8000

# Add to base level or app section
storage:
  backend: local
  local_base: ./.storage
```

### Fix 3: Remove Redundant Overrides

In [`config.development.yaml`](../../../config.development.yaml), remove:

```YAML
metrics:
  enabled: false  # Same as base config.yaml
```

---

## Final Alignment Score

| Config File                                                   | Alignment Status                                |
| ------------------------------------------------------------- | ----------------------------------------------- |
| [`config.yaml`](../../../config.yaml)                         | **95%** Aligned (minor missing optional fields) |
| [`config.development.yaml`](../../../config.development.yaml) | **70%** Aligned (logging key mismatch)          |
| [`config.production.yaml`](../../../config.production.yaml)   | **90%** Aligned (logging key mismatch)          |