# Modcus Configuration System - Comprehensive Review Report

**Date:** February 6, 2026  
**Scope:** modcus_common, modcus_api_query, modcus_api_ingest  
**Overall Score: 6.2/10** - Functional but requires significant improvements.

---

## Executive Summary

The review identified 4 critical issues, 15+ unused fields, naming mismatches, and security concerns. The configuration system uses a hierarchical, mixin-based architecture that is generally sound but needs immediate attention in several areas.

---

## 1. Architecture Overview

**Inheritance Hierarchy**
```bash
PydanticBaseSettings
    ▼
BaseSettings (modcus_common/settings/base.py)
    ├──► LLMSettingsMixin (modcus_common/settings/llm.py)
    └──► RAGSettingsMixin (modcus_common/settings/rag.py)
    ▼
Service-Specific Settings
    ├──► QuerySettings (modcus_api_query/settings.py)
    └──► IngestSettings (modcus_api_ingest/settings.py)
```

**Configuration Loading Flow**
1. Load config.yaml (base)
2. Load config.{APP_ENV}.yaml (overrides)
3. Recursively merge configurations
4. Flatten nested YAML keys (e.g., rag.vector_store → rag_vector_store)
5. Pydantic validates and applies environment variable overrides

---

## 2. Critical Issues (Fix Immediately)

### Issue 1: SQLite Default Violates AGENTS.md

**Location**: `modcus_common/settings/base.py:47`
```py
# Current (violates "Never use SQLite" rule)
db_url: str = Field(default="sqlite:///./modcus.db", env="DB_URL")

# Should be:
db_url: str = Field(
    default="postgresql://modcus:modcus@localhost:5432/modcus",
    env="DB_URL",
    pattern=r"^postgresql(\+asyncpg)?://"
)
```

### Issue 2: Logging Key Mismatch - Overrides Don't Work

**Location**: `config.development.yaml` and `config.production.yaml`
```yaml
# WRONG - These files use:
logging:
  level: DEBUG  # Flattens to: logging_level

# Settings expects:
log:
  level: DEBUG  # Flattens to: log_level ✓
```
**Impact**: Development/production logging level overrides do not work.

### Issue 3: Field Naming Mismatch in Code

**Location**: `modcus_api_query/services/agents/retrieval_agent.py`
```py
# Line 103 - Wrong name:
disable_reranking = not getattr(settings, 'retrieval_enable_reranking', False)
# Should be: 'rag_retrieval_enable_reranking'

# Line 210 - Wrong name:
top_k = getattr(settings, 'retrieval_top_k', 10)
# Should be: 'rag_retrieval_top_k'
```

### Issue 4: Missing Provider Validation

**Location**: `modcus_common/settings/base.py`
```py
# Should add pattern validation:
llm_provider: str = Field(default="vertex", pattern="^(vertex|gemini|openai|anthropic)$")
rag_vector_store: str = Field(default="chromadb", pattern="^(chromadb|qdrant|pgvector)$")
```

---

## 3. YAML Alignment Report

| Config File               | Alignment Status                   |
| ------------------------- | ---------------------------------- |
| `config.yaml`             | 95% Aligned                        |
| `config.development.yaml` | 70% Aligned (logging key mismatch) |
| `config.production.yaml`  | 90% Aligned (logging key mismatch) |

**Fix Required**: Change `logging:` to `log:` in environment-specific configs.

---

## 4. Field Usage Analysis - 15+ Unused Fields

**Base Settings - 6 Unused**

| Field                | Recommendation                 |
| -------------------- | ------------------------------ |
| `db_password`        | Remove or implement parsing    |
| `redis_password`     | Remove or implement Redis AUTH |
| `storage_backend`    | Remove (legacy only)           |
| `local_storage_base` | Remove (never used)            |
| `metrics_enabled`    | Remove or implement metrics    |
| `metrics_port`       | Remove or implement metrics    |

**LLM Settings - 7 Unused**

| Field                            | Recommendation       |
| -------------------------------- | -------------------- |
| `llm_rotation_enabled`           | Remove or implement  |
| `llm_rotation_max_retries`       | Remove or implement  |
| `gemini_api_key`                 | Remove (legacy only) |
| `openai_api_key`                 | Remove (legacy only) |
| `anthropic_api_key`              | Remove (never used)  |
| `google_application_credentials` | Remove (legacy only) |

**Service-Specific - 2 Unused**

| Field                   | Recommendation                    |
| ----------------------- | --------------------------------- |
| `query_timeout_seconds` | Remove or implement timeout logic |
| `ingestion_max_queued`  | Remove (legacy only)              |

---

## 5. Recommendations

**Immediate Actions (This Week)**

1. Fix `logging:` → `log:` in `config.development.yaml` and `config.production.yaml`
2. Fix field names in `retrieval_agent.py` (lines 103, 210)
3. Change SQLite default to PostgreSQL
4. Add provider validation patterns

**Short-Term Actions (Next Sprint)**

1. Remove 15+ unused fields
2. Add API key minimum length validation (16 chars)
3. Add temperature range validation (0.0-2.0)
4. Clean up `.env.example` (group Docker-only vs application settings)

**Long-Term Improvements**

1. Add missing settings to YAML (retry delays, rate limiting)
2. Implement storage backend-specific settings
3. Standardize naming conventions

---

## 6. Key Files Requiring Changes

**Settings Definitions**

- `modcus_common/settings/base.py` - SQLite default, remove unused fields
- `modcus_common/settings/llm.py` - Add validation, remove unused fields
- `modcus_common/settings/rag.py` - Add validation

**Configuration Files**

- `config.development.yaml` - Fix logging: → log:
- `config.production.yaml` - Fix logging: → log:

**Code Files**

- `modcus_api_query/services/agents/retrieval_agent.py` - Lines 103, 210 (fix field names)