# Vertex AI LLM Credentials Support

**Date**: February 10, 2026
**Version**: v0.2.2
**Status**: Completed

---

## Summary

Added support for Google Cloud service account credentials authentication in the Vertex AI LLM class, aligning it with the existing Vertex AI Embedding implementation. This enables proper authentication using service account JSON files instead of relying solely on environment-based authentication.

---

## Changes

### 1. LLM Factory - Vertex Credentials Support

**File**: `modcus_common/services/llm/llm_factory.py`

Added `credentials` parameter to `create_llm_instance()` function for Vertex AI provider:

**Before**:
```python
def create_llm_instance(
    provider: str,
    model: str,
    api_key: Optional[str] = None,
    temperature: float = 0.0,
    max_tokens: int | None = None,
    project_id: Optional[str] = None,
    location: Optional[str] = None,
    **kwargs,
) -> LLM:
```

**After**:
```python
def create_llm_instance(
    provider: str,
    model: str,
    api_key: Optional[str] = None,
    temperature: float = 0.0,
    max_tokens: int | None = None,
    project_id: Optional[str] = None,
    location: Optional[str] = None,
    credentials: Optional[Any] = None,
    **kwargs,
) -> LLM:
```

**Vertex AI Instantiation** (lines 69-76):
```python
return Vertex(
    model=model,
    credentials=credentials,  # NEW: Pass credentials object
    location=location,
    temperature=temperature,
    **kwargs,
)
```

**Key Points**:
- `credentials` is an optional parameter that accepts a Google OAuth2 credentials object
- When provided, it's passed directly to the LlamaIndex `Vertex` class
- This mirrors the existing pattern used by `create_embedding_model()` for Vertex embeddings

---

### 2. LLM Rotator - Credentials Loading

**File**: `modcus_common/services/llm/llm_rotator.py`

Updated `_get_current_llm_async()` to load and pass credentials for Vertex AI:

**Implementation** (lines 104-121):
```python
# Check if using VertexAI
if self.settings.llm_provider.lower() == "vertex":
    # VertexAI embeddings use project_id, location, and credentials
    credentials = None
    if self.settings.google_application_credentials:
        credentials = service_account.Credentials.from_service_account_file(
            self.settings.google_application_credentials
        )

    return create_llm_instance(
        provider=self.settings.llm_provider,
        model=self.settings.llm_model,
        project_id=self.settings.gcp_project_id,
        location=self.settings.gcp_location,
        temperature=getattr(self.settings, "llm_temperature", 0.0),
        max_tokens=getattr(self.settings, "llm_max_tokens", None),
        credentials=credentials,  # NEW: Pass loaded credentials
    )
```

**Key Points**:
- Loads credentials from the service account JSON file path specified in settings
- Uses `google.oauth2.service_account.Credentials.from_service_account_file()` to create the credentials object
- Only loads credentials when `google_application_credentials` path is configured

---

### 3. LLM Service - Embedding Model Credentials

**File**: `modcus_common/services/llm/llm_service.py`

The embedding model already supported credentials (no changes needed), serving as the reference implementation:

**Existing Implementation** (lines 58-87):
```python
def _get_embedding_model(self) -> Any:
    if self._embedding_model is None:
        if self.settings.embedding_provider.lower() == "vertex":
            # VertexAI embeddings use project_id, location, and credentials
            credentials = None
            if self.settings.google_application_credentials:
                credentials = service_account.Credentials.from_service_account_file(
                    self.settings.google_application_credentials
                )
            self._embedding_model = create_embedding_model(
                provider=self.settings.embedding_provider,
                model=self.settings.embedding_model,
                project_id=self.settings.gcp_project_id,
                location=self.settings.gcp_location,
                credentials=credentials,
            )
```

---

## Configuration

### Required Settings

Add to your configuration file or environment:

```yaml
# config.yaml
llm:
  provider: vertex
  model: gemini-2.5-flash

gcp:
  project_id: your-gcp-project-id
  location: us-central1
  # Path to service account JSON file
  application_credentials: /path/to/service-account.json
```

Or via environment variables:
```bash
export LLM_PROVIDER=vertex
export LLM_MODEL=gemini-2.5-flash
export GCP_PROJECT_ID=your-gcp-project-id
export GCP_LOCATION=us-central1
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
```

---

## Authentication Flow

1. **Service Account File**: Store your Google Cloud service account JSON key file securely
2. **Configuration**: Set `google_application_credentials` to the file path
3. **Runtime Loading**: The LLM rotator loads credentials using `service_account.Credentials.from_service_account_file()`
4. **Authentication**: Credentials object is passed to Vertex AI LLM constructor
5. **API Calls**: All Vertex AI API calls are authenticated using the service account

---

## Backward Compatibility

✅ **Fully backward compatible**:
- Credentials parameter is optional
- If not provided, Vertex AI falls back to default authentication methods
- No breaking changes to existing API
- Existing configurations continue to work without modification

---

## Testing

### Verification

1. **Upload Service Account File**:
   ```bash
   # In dev container, mount the service account file
   docker cp service-account.json modcus-dev:/workspace/service-account.json
   ```

2. **Verify Credentials Loading**:
   The service will log successful initialization if credentials are valid.

3. **Test Query**:
   ```bash
   curl -X POST http://localhost:8080/v1/query \
     -H "X-API-Key: your-api-key" \
     -H "Content-Type: application/json" \
     -d '{
       "query": "What is the revenue of Apple?",
       "mode": "company-analysis",
       "level": "novice"
     }'
   ```

---

## Security Considerations

- **Service Account File**: Keep the JSON key file secure and never commit it to version control
- **File Permissions**: Set appropriate file permissions (e.g., `chmod 600 service-account.json`)
- **Container Mounts**: In Docker, mount the credentials file as read-only
- **Environment Variables**: Consider using environment variables for the credentials path rather than hardcoding

---

## Files Modified

| File                                        | Lines Changed | Description                                              |
| ------------------------------------------- | ------------- | -------------------------------------------------------- |
| `modcus_common/services/llm/llm_factory.py` | ~15           | Added `credentials` parameter to `create_llm_instance()` |
| `modcus_common/services/llm/llm_rotator.py` | ~20           | Load and pass credentials for Vertex AI                  |

---

## Related Changes

- Vertex AI Embedding already supported credentials (unchanged)
- Service account authentication aligns LLM with embedding authentication patterns
- Enables consistent authentication across all Vertex AI operations

---

**End of Changelog**
