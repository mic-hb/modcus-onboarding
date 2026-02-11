# Modcus Configuration System Architecture Analysis

---

## Executive Summary

The Modcus platform uses a sophisticated, hierarchical configuration system that combines YAML-based structured configuration with environment variable overrides and Pydantic validation. This design enables environment-specific configurations while keeping secrets out of version control.

---

## 1. Inheritance Hierarchy

The configuration system follows a clear mixin-based inheritance pattern:

```bash
PydanticBaseSettings (from pydantic_settings)
         │
         ▼
    BaseSettings (modcus_common/settings/base.py)
    ├─ Core settings: db_url, redis_url, log_level, storage_backend, api_key
    ├─ YAML loading mechanism (from_yaml)
    └─ Configuration merging & flattening utilities
         │
         ├──► LLMSettingsMixin (modcus_common/settings/llm.py)
         │    ├─ llm_provider, llm_model, llm_temperature
         │    ├─ embedding_provider, embedding_model
         │    ├─ llm_rotation_enabled, llm_rotation_max_retries
         │    └─ API keys: gemini_api_key, openai_api_key, anthropic_api_key
         │
         └──► RAGSettingsMixin (modcus_common/settings/rag.py)
              ├─ rag_vector_store
              ├─ rag_chunk_sizes_* (per document type)
              ├─ rag_chunk_overlap, rag_retrieval_top_k
              └─ Vector store connections: qdrant_url, chromadb_persist_dir
         │
         ▼
    Service-Specific Settings
    ├─ QuerySettings (modcus_api_query/settings.py)
    │  └─ query_timeout_seconds, query_conversation_timeout_minutes, etc.
    │
    └─ IngestSettings (modcus_api_ingest/settings.py)
       └─ ingestion_celery_concurrency, ingestion_max_queued, etc.
```

**Key Design Decisions**

1. **Mixin Pattern**: `LLMSettingsMixin` and `RAGSettingsMixin` are not Pydantic models but plain Python classes. This allows multiple inheritance without diamond problem issues.
2. **Single Base**: All settings inherit from `BaseSettings` which extends Pydantic's `BaseSettings`, ensuring consistent behavior.
3. **Service Specialization**: Each service defines its own settings class combining the mixins it needs (both services use both mixins).

---

## 2. YAML Loading Mechanism

### The `from_yaml` Method (`BaseSettings`)

Located in `/home/michb/dev/02-projects/02-aiml/ngoper/data_extractor.worktrees/refactor-separate-api-platform-ingest/modcus_common/settings/base.py` (lines 76-117):

```py
@classmethod
def from_yaml(cls, config_path: str = "config.yaml", env: Optional[str] = None) -> "BaseSettings":
    # 1. Determine environment (from parameter or APP_ENV env var)
    if env is None:
        env = os.getenv("APP_ENV", "development")
    
    # 2. Load base configuration
    base_config = cls._load_yaml_file(config_path)
    
    # 3. Load environment-specific configuration
    env_config_path = f"{Path(config_path).stem}.{env}.yaml"
    env_config = cls._load_yaml_file(env_config_path)
    
    # 4. Merge configurations (environment takes precedence)
    merged_config = cls._merge_configs(base_config, env_config)
    
    # 5. Flatten nested YAML structure
    flat_config = cls._flatten_yaml(merged_config)
    
    # 6. Create settings instance
    return cls(**flat_config)
```

### Environment-Specific Overlay System

The configuration system supports three levels of configuration files:

| File                      | Purpose                           |
| ------------------------- | --------------------------------- |
| `config.yaml`             | Base configuration with defaults  |
| `config.development.yaml` | Development environment overrides |
| `config.production.yaml`  | Production environment overrides  |

Example: Production Overrides (`config.production.yaml`):

```yaml
rag:
  vector_store: qdrant  # Override: chromadb → qdrant
  chunk_sizes:
    annual_report: 2048  # Override: 1024 → 2048
ingestion:
  celery:
    concurrency: 4  # Override: 2 → 4
```

### Configuration Merging Algorithm

The `_merge_configs` method (lines 136-162) performs deep recursive merging:

```py
@staticmethod
def _merge_configs(base: Dict[str, Any], override: Dict[str, Any]) -> Dict[str, Any]:
    result = base.copy()
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            # Recursive merge for nested dictionaries
            result[key] = BaseSettings._merge_configs(result[key], value)
        else:
            # Override takes precedence for non-dicts
            result[key] = value
    return result
```

### YAML Flattening

The `_flatten_yaml` method (lines 164-194) converts nested YAML structures to flat keys:

```yaml
# Input YAML
rag:
  chunk_sizes:
    annual_report: 1024
  retrieval_top_k: 10
```

```py
# Output after flattening
{
  "rag_chunk_sizes_annual_report": 1024,
  "rag_retrieval_top_k": 10
}
```

This enables direct mapping to Pydantic Field names using underscore notation.

---

## 3. Environment Variable Override Mechanism

### Pydantic Settings Integration

`BaseSettings` uses Pydantic's `SettingsConfigDict` (line 37-41):

```py
model_config = SettingsConfigDict(
    env_file=".env",
    env_file_encoding="utf-8",
    extra="ignore",
)
```

### Override Priority (Highest to Lowest)

1. Environment variables (e.g., `DB_URL`, `GEMINI_API_KEY`)
2. Environment-specific YAML (`config.{env}.yaml`)
3. Base YAML (`config.yaml`)
4. Pydantic Field defaults

### Field-Level Environment Mapping

Some fields explicitly declare environment variable names:

```py
# BaseSettings
db_url: str = Field(default="sqlite:///./modcus.db", env="DB_URL")
redis_url: Optional[str] = Field(default="redis://localhost:6379/0", env="REDIS_URL")

# RAGSettingsMixin
qdrant_url: str = Field(default="http://localhost:6333", env="QDRANT_URL")
chromadb_persist_dir: str = Field(default="/app/rag_storage/chroma_db", env="CHROMADB_PERSIST_DIR")

# LLMSettingsMixin (note: no env mapping - loaded from YAML or .env via Pydantic)
gemini_api_key: Optional[str] = Field(default=None, description="...")
```

### Secrets Management Strategy

| Secret Type         | Storage Location   | Example                            |
| ------------------- | ------------------ | ---------------------------------- |
| Database passwords  | `.env` file        | `POSTGRES_PASSWORD`                |
| LLM API keys        | `.env` file        | `GEMINI_API_KEY`, `OPENAI_API_KEY` |
| Application API key | `.env` file        | `API_KEY`                          |
| GCP credentials     | `.env` file (path) | `GOOGLE_APPLICATION_CREDENTIALS`   |
| GCP project ID      | `.env` file        | `GCP_PROJECT_ID`                   |
| GCP location        | `.env` file        | `GCP_LOCATION`                     |
| Structured config   | `config.yaml`      | Chunk sizes, timeouts              |

---

## 4. Pydantic Validation Role

### Field Validation Features

The settings classes leverage Pydantic's validation extensively:

-  Type Validation:
   ```
   llm_temperature: float = Field(default=0.0)
   query_timeout_seconds: int = Field(default=360, ge=1, le=3600)
   ```
-  Constraints:
    ```py
    query_complexity_inference_mode: str = Field(
        default="combined",
        pattern="^(simple|combined|advanced)$"  # Regex validation
    )

    ingestion_celery_concurrency: int = Field(default=2, ge=1, le=16)  # Range validation
    ```
-   Documentation:
    ```py
    gemini_api_key: Optional[str] = Field(
        default=None,
        description="Google Gemini API key (optional, for Gemini provider)",
    )
    ```

### Validation During Loading

When `from_yaml()` calls `cls(**flat_config)`, Pydantic:
1. Validates all field types
2. Applies constraints (min/max values, patterns)
3. Converts types automatically (e.g., string "1024" → int 1024)
4. Raises `ValidationError` on invalid data

### Custom Properties

`QuerySettings` adds computed properties (lines 73-96):

```py
@property
def database_url(self) -> str:
    """Returns db_url with async driver prefixes."""
    url = self.db_url
    if url.startswith("sqlite://"):
        url = url.replace("sqlite://", "sqlite+aiosqlite://", 1)
    elif url.startswith("postgresql://"):
        url = url.replace("postgresql://", "postgresql+asyncpg://", 1)
    return url
```

---

## 5. Settings Instantiation and Usage

### Query Service (`modcus_api_query`)

Instantiation (`main.py`, line 29):

```py
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    settings = QuerySettings.load()  # Load at startup
    deps.set_settings(settings)       # Store in global state
```

Dependency Injection (`api/deps.py`, lines 79-90):

```py
def get_settings() -> "QuerySettings":
    if _settings is None:
        raise RuntimeError("Settings not initialized")
    return _settings
```

Usage in Routes (`api/routes_query.py`):

```py
@router.post("/v1/query/chat")
async def chat_query(
    settings: QuerySettings = Depends(get_settings),  # Injected
):
    timeout = settings.query_timeout_seconds
    # ...
```

### Ingest Service (`modcus_api_ingest`)

Instantiation (`main.py`, line 21):

```py
settings = IngestSettings.load()  # Module-level at import time
```

Dependency Injection (`api/deps.py`, lines 35-39):

```py
_cached_settings: IngestSettings = None
def get_settings() -> IngestSettings:
    global _cached_settings
    if _cached_settings is None:
        _cached_settings = IngestSettings.load()
    return _cached_settings
```

Celery Worker (`services/tasks/celery_app.py`):

```py
settings = IngestSettings.load()  # Loaded at module import
```

### Service-Specific Load Methods

Both services provide convenience `load()` class methods:

```py
# QuerySettings (modcus_api_query/settings.py, lines 98-124)
@classmethod
def load(cls, config_path: str = "config.yaml", env: Optional[str] = None) -> "QuerySettings":
    return cls.from_yaml(config_path, env)

# IngestSettings (modcus_api_ingest/settings.py, lines 58-84)
@classmethod
def load(cls, config_path: str = "config.yaml", env: Optional[str] = None) -> "IngestSettings":
    return cls.from_yaml(config_path, env)
```

---

## 6. Configuration Architecture Strengths

1. **Separation of Concerns**: Structured config in YAML, secrets in .env
2. **Environment Flexibility**: Easy switching between dev/prod via `APP_ENV`
3. **Type Safety**: Full Pydantic validation prevents runtime errors
4. **Hierarchical Organization**: Mixins allow reusable configuration groups
5. **Override Flexibility**: Environment variables can override any YAML value
6. **Documentation**: Field descriptions and YAML comments explain configuration
7. **Validation**: Constraints ensure valid configuration ranges

---

## 7. File Locations Summary

| File                             | Purpose                                |
| -------------------------------- | -------------------------------------- |
| `modcus_common/settings/base.py` | `BaseSettings` class with YAML loading |
| `modcus_common/settings/llm.py`  | `LLMSettingsMixin`                     |
| `modcus_common/settings/rag.py`  | `RAGSettingsMixin`                     |
| `modcus_api_query/settings.py`   | `QuerySettings`                        |
| `modcus_api_ingest/settings.py`  | `IngestSettings`                       |
| `config.yaml`                    | Base configuration                     |
| `config.development.yaml`        | Development overrides                  |
| `config.production.yaml`         | Production overrides                   |
| `.env.example`                   | Environment variable template          |

---

This configuration architecture provides a robust, maintainable, and secure foundation for the Modcus platform's multi-service deployment needs.