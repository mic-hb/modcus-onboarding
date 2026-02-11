# Implementation Plan: Configuration System Fixes

## Overview

This plan addresses all critical issues identified in the Modcus configuration system review.
**Priority: High** | **Estimated Time: 1-2 days**

## Task List

### Phase 1: Critical Fixes (Must Complete First)

- [ ] **Task 1**: Fix logging key mismatch in environment configs
  - Fix `config.development.yaml`
  - Fix `config.production.yaml`
  - Verify flattening produces correct `log_level` field

- [ ] **Task 2**: Fix field naming mismatch in retrieval_agent.py
  - Line 103: `'retrieval_enable_reranking'` → `'rag_retrieval_enable_reranking'`
  - Line 210: `'retrieval_top_k'` → `'rag_retrieval_top_k'`
  - Verify both fields are accessed correctly

- [ ] **Task 3**: Change SQLite default to PostgreSQL in BaseSettings
  - Update `db_url` default value
  - Add pattern validation for PostgreSQL URLs
  - Ensure `db_url` requires PostgreSQL format

- [ ] **Task 4**: Add provider validation patterns
  - Add `pattern="^(vertex|gemini|openai|anthropic)$"` to `llm_provider`
  - Add `pattern="^(chromadb|qdrant|pgvector)$"` to `rag_vector_store`
  - Add `pattern="^(DEBUG|INFO|WARNING|ERROR|CRITICAL)$"` to `log_level`
  - Add `pattern="^(none|transient_only|all)$"` to retry strategy fields

### Phase 2: Remove Unused Fields

- [ ] **Task 5**: Remove unused fields from BaseSettings
  - Remove `db_password` (or implement parsing logic)
  - Remove `redis_password` (or implement Redis AUTH)
  - Remove `storage_backend` (legacy only)
  - Remove `local_storage_base` (never used)
  - Remove `metrics_enabled` (or implement metrics endpoint)
  - Remove `metrics_port` (or implement metrics endpoint)

- [ ] **Task 6**: Remove unused fields from LLMSettingsMixin
  - Remove `llm_rotation_enabled` (or implement in rotator)
  - Remove `llm_rotation_max_retries` (or implement in rotator)
  - Remove `gemini_api_key` (legacy only)
  - Remove `openai_api_key` (legacy only)
  - Remove `anthropic_api_key` (never used)
  - Remove `google_application_credentials` (legacy only)

- [ ] **Task 7**: Remove unused fields from service-specific settings
  - Remove `query_timeout_seconds` from QuerySettings (or implement timeout logic)
  - Remove `ingestion_max_queued` from IngestSettings (legacy only)

### Phase 3: Add Validation

- [ ] **Task 8**: Add API key minimum length validation
  - Add `min_length=16` to `api_key` field
  - Add `@field_validator` to ensure non-empty API keys have minimum length
  - Update error messages

- [ ] **Task 9**: Add temperature range validation
  - Add `ge=0.0, le=2.0` to `llm_temperature` field
  - Update docstrings

- [ ] **Task 10**: Add embedding dimension validation
  - Document expected dimensions per provider
  - Add validation or warnings for mismatched dimensions

### Phase 4: Cleanup & Documentation

- [ ] **Task 11**: Clean up `.env.example`
  - Group variables by purpose (Docker-only, Application, Secrets)
  - Remove duplicates that exist in YAML
  - Add clear comments explaining each group
  - Identify which variables are actually read by settings classes

- [ ] **Task 12**: Update `.env.example` to reflect actual usage
  - Keep only environment variables that override settings
  - Remove YAML duplicates
  - Clarify Docker-only variables

### Phase 5: Testing & Verification

- [ ] **Task 13**: Test configuration loading
  - Verify `config.development.yaml` overrides work
  - Verify `config.production.yaml` overrides work
  - Test environment variable overrides
  - Test validation errors

- [ ] **Task 14**: Verify all active fields are used
  - Run application and check no errors
  - Verify retrieval_agent.py uses correct field names
  - Check logs show correct log level

## Implementation Order

**Recommended sequence:**
1. Tasks 1-4 (Critical fixes - fixes broken functionality)
2. Tasks 5-7 (Remove unused - reduces technical debt)
3. Tasks 8-10 (Add validation - improves robustness)
4. Tasks 11-12 (Cleanup - improves maintainability)
5. Tasks 13-14 (Testing - ensures everything works)

## Risk Assessment

| Task | Risk Level | Mitigation |
|------|-----------|------------|
| Fix logging keys | Low | Simple YAML change |
| Fix field names | Low | Only affects retrieval_agent.py |
| Change SQLite default | Medium | May break local dev; document migration |
| Remove unused fields | Medium | Verify not used in tests first |
| Add validation | Low | Only rejects invalid values |
| Clean up .env.example | Low | Documentation change only |

## Success Criteria

- [ ] All 4 critical issues are resolved
- [ ] No configuration errors on startup
- [ ] All 15+ unused fields removed or implemented
- [ ] All validation patterns work correctly
- [ ] `.env.example` is clean and accurate
- [ ] Tests pass
- [ ] Documentation updated

## Files to Modify

### Settings Definitions
- `modcus_common/settings/base.py` (Tasks 3, 4, 5, 8)
- `modcus_common/settings/llm.py` (Tasks 4, 6, 9, 10)
- `modcus_common/settings/rag.py` (Task 4)
- `modcus_api_query/settings.py` (Task 7)
- `modcus_api_ingest/settings.py` (Task 7)

### Configuration Files
- `config.development.yaml` (Task 1)
- `config.production.yaml` (Task 1)
- `.env.example` (Tasks 11, 12)

### Code Files
- `modcus_api_query/services/agents/retrieval_agent.py` (Task 2)

## Notes

- Check for tests that might reference removed fields
- Update AGENTS.md if any standards change
- Consider adding migration notes for SQLite → PostgreSQL change
- Verify Docker Compose still works after all changes
