# Technical Plan: Modcus v0.2 Architecture

**Epic:** spec:7a9bea0a-9642-4cd3-b4d3-cbc00fd624f6/30980d13-feb1-4ff7-b94c-cffbb4f7c25f  
**Related Specs:** spec:7a9bea0a-9642-4cd3-b4d3-cbc00fd624f6/b9af845b-04dc-4f5a-a355-6265f5235c8f  
**Version:** 0.2.0  
**Status:** Draft  
**Created:** 2026-02-03

---

## 1. Architectural Approach

### 1.1 Core Architectural Decisions

**Decision 1: LangGraph as Query Orchestrator (Query Service Only)**

LangGraph state machine lives exclusively in `modcus_api_query/services/orchestration/`. This decision:

- Keeps orchestration logic localized to Query service (Ingestion doesn't need it)
- Aligns with monorepo principle: no parallel frameworks across services
- Simplifies dependency management (LangGraph only in Query service's `pyproject.toml`)
- Enables independent testing of orchestration logic

**Trade-offs:**

- ✅ Clear ownership and boundaries
- ✅ Simpler deployment (Ingestion service lighter)
- ⚠️ Cannot reuse LangGraph for future Ingestion orchestration (acceptable - Celery handles that)

---

**Decision 2: Per-Service LlamaIndex Instances**

Each service creates its own `VectorStoreIndex` instance pointing to the same vector store backend:

- Ingestion service: Creates index, inserts nodes
- Query service: Loads index from vector store on startup
- No shared index object, no reload mechanism needed

**Trade-offs:**

- ✅ Services truly independent (can restart separately)
- ✅ No complex reload logic (LlamaIndex handles vector store connection)
- ✅ Simpler mental model (each service owns its index instance)
- ⚠️ Slight overhead on Query service startup (loads index from vector store)

**Future (v0.3):** Introduce Index-as-a-Service for centralized index management and caching.

---

**Decision 3: Service Classes for Agents**

Each of the 5 agents is implemented as a service class with clear interface:

- `ValidationAgent` - Intent classification, schema validation
- `TickerInferenceAgent` - Ticker extraction and resolution
- `RetrievalAgent` - LlamaIndex query orchestration
- `AnalysisAgent` - LLM synthesis with tool calling
- `ResponseFormatterAgent` - Output structuring

LangGraph nodes are thin wrappers that call agent methods. This separates orchestration (LangGraph) from business logic (service classes).

**Trade-offs:**

- ✅ Testable in isolation (unit test each agent)
- ✅ Reusable (agents can be called outside LangGraph if needed)
- ✅ Clear separation of concerns
- ⚠️ More boilerplate than pure node functions (acceptable for maintainability)

---

**Decision 4: Hybrid Tool System**

Tools leverage both LlamaIndex and LangGraph capabilities:

- **RAG Query Tools**: LlamaIndex `QueryEngineTool` (wraps query engines)
- **Function Tools**: LlamaIndex `FunctionTool` (financial calculators, data manipulation, text analysis)
- **Analysis Agent**: Wraps LlamaIndex `ReActAgent` as a LangGraph node
- **LangGraph**: Orchestrates high-level flow, delegates tool calling to LlamaIndex

**Trade-offs:**

- ✅ Leverage LlamaIndex's mature tool calling infrastructure
- ✅ LangGraph focuses on orchestration, not tool execution
- ✅ Proven pattern from reference project
- ⚠️ Two frameworks in play (acceptable - each does what it's best at)

---

**Decision 5: Selective Database Schema Simplification**

Fresh schema removes LightRAG-specific models, preserves proven patterns, adds v0.2 features:

**Remove:**

- `RagArtifact` - LightRAG-specific (working_dir, graph_storage, etc.)
- `IngestionJob` - Redundant with Job model's stage tracking

**Keep & Simplify:**

- `Job` - Remove LightRAG fields, add LlamaIndex fields
- `Document` - Keep structure, remove LightRAG references
- `Company` - Keep as-is (minimal, works well)
- `LLMCall` - Keep comprehensive logging (critical for cost tracking)
- `ApiKey` - Keep with rotation support
- `Batch` - Keep for batch ingestion
- `StoredFile` - Keep for file tracking
- `DeletionLog` - Keep for audit trail

**Add:**

- `ConversationSession` - Chat session management
- `ConversationTurn` - Individual Q&A pairs
- `EphemeralArtifact` - Mode 3 user uploads

**Trade-offs:**

- ✅ Removes obsolete complexity
- ✅ Adds needed features
- ✅ Preserves proven patterns (LLMCall, ApiKey)
- ✅ Fresh start allows better field naming

---

**Decision 6: LLM Service Layer for Rotation**

`LLMService` in `modcus_common/services/llm/` encapsulates:

- LLM provider management (creates LlamaIndex LLM instances)
- API key rotation on failures
- Comprehensive call logging
- Metrics tracking

**Interface:**

```python
class LLMService:
    async def complete(prompt: str, **kwargs) -> str
    async def chat(messages: List[Dict], **kwargs) -> str
    async def embed(texts: List[str]) -> List[List[float]]
    
    # Internal: rotation, logging, metrics
```

**Trade-offs:**

- ✅ Clean abstraction for rotation logic
- ✅ Centralized logging and metrics
- ✅ Easy to test and mock
- ✅ Reusable across both services
- ⚠️ Adds abstraction layer (acceptable - encapsulates complexity)

---

**Decision 7: Thin Vector Store Wrapper**

Minimal wrapper around LlamaIndex vector stores:

- `VectorStoreFactory` creates configured LlamaIndex vector store instances
- No custom abstraction layer
- Configuration-driven selection (Qdrant, ChromaDB, PGVector)

**Trade-offs:**

- ✅ Leverage LlamaIndex's native vector store support
- ✅ Minimal code to maintain
- ✅ Easy to add new backends (LlamaIndex does the work)
- ⚠️ Tied to LlamaIndex interface (acceptable - committed to LlamaIndex for v0.2)

---

**Decision 8: Database-First Conversation Storage**

Conversation sessions stored in PostgreSQL with LRU caching:

- `ConversationSession` table for session metadata
- `ConversationTurn` table for Q&A history
- In-memory LRU cache for active sessions (last 100)
- Automatic summarization when approaching context window

**Trade-offs:**

- ✅ Persistent across restarts
- ✅ Fast access for active sessions
- ✅ Simple mental model (database is source of truth)
- ⚠️ DB overhead per turn (acceptable - not high frequency)

---

### 1.2 System Architecture Overview

```mermaid
graph TB
    subgraph "Query Service :8080"
        QRoute[FastAPI Routes]
        LGOrch[LangGraph Orchestrator]
        
        subgraph "Agents (Service Classes)"
            VA[Validation Agent]
            TIA[Ticker Inference Agent]
            RA[Retrieval Agent]
            AA[Analysis Agent]
            RFA[Response Formatter Agent]
        end
        
        subgraph "LlamaIndex Components"
            QIndex[VectorStoreIndex]
            QEngine[Query Engine]
            ReactAgent[ReActAgent + Tools]
        end
        
        ChatMgr[Chat Session Manager]
    end
    
    subgraph "Ingestion Service :8081"
        IRoute[FastAPI Routes]
        Celery[Celery Workers]
        
        subgraph "Pipeline Stages"
            Parser[Document Parser]
            Chunker[Node Parser]
            Embedder[Embedding Generator]
        end
        
        IIndex[VectorStoreIndex]
    end
    
    subgraph "Common Module"
        LLMSvc[LLM Service]
        VectorFactory[Vector Store Factory]
        Models[Database Models]
        Settings[Settings Manager]
        Logging[Logging Utils]
        Metrics[Prometheus Metrics]
    end
    
    subgraph "Storage Backends"
        DB[(PostgreSQL)]
        VectorDB[(Vector Store)]
        Redis[(Redis)]
        FileStore[Local FS]
    end
    
    QRoute --> LGOrch
    LGOrch --> VA --> TIA --> RA --> AA --> RFA
    RA --> QIndex
    AA --> ReactAgent
    ReactAgent --> QEngine
    QIndex --> VectorFactory
    QRoute --> ChatMgr
    
    IRoute --> Celery
    Celery --> Parser --> Chunker --> Embedder --> IIndex
    IIndex --> VectorFactory
    
    VectorFactory --> VectorDB
    LLMSvc --> Models
    ChatMgr --> Models
    Models --> DB
    Celery --> Redis
    Parser --> FileStore
    
    AA --> LLMSvc
    Embedder --> LLMSvc
    LGOrch --> Logging
    Celery --> Logging
    LLMSvc --> Metrics
```

---

### 1.3 Request Flow Tracing

**Company Analysis Query (Mode 1):**

```
1. User → POST /v1/query {"query": "Analyze BBCA", "mode": "company-analysis", "level": "novice"}
2. FastAPI Route → Validate API key (deps.py)
3. Route → LangGraph.invoke(state)
4. LangGraph State Machine:
   a. Validation Agent → Classify intent (finance), infer complexity (reasoning)
   b. Ticker Inference Agent → Extract ticker (BBCA)
   c. Retrieval Agent → Query LlamaIndex (filter by BBCA docs)
   d. Analysis Agent → LlamaIndex ReActAgent with tools → LLMService
   e. Response Formatter → Structure output for novice level
5. LangGraph → Return final state
6. Route → Return JSON response
7. Background: Log LLM calls, update metrics
```

**Document Ingestion:**

```
1. User → POST /v1/ingestion/upload (file + metadata)
2. FastAPI Route → Validate file, check duplicates
3. Route → Create Job record, queue Celery task
4. Celery Worker:
   a. Parse → LlamaIndex PDFReader or JSON parser
   b. Chunk → LlamaIndex SentenceSplitter (configurable per doc type)
   c. Embed → LLMService.embed() → LlamaIndex embeddings
   d. Index → VectorStoreIndex.insert_nodes()
5. Update Job status → completed
6. Background: Per-job logs, LLM call logging, metrics
```

---

### 1.4 Key Constraints & Principles

**From **file:AGENTS.md**:**

1. **Service Boundaries**: Ingestion and Query are independent, communicate via HTTP (except shared `modcus_common`)
2. **No Parallel Frameworks**: LangGraph only in Query service, Celery only in Ingestion service
3. **Shared Code in Common**: Database models, settings, LLM service, logging utilities
4. **Configuration-Driven**: All behavior configurable via environment variables
5. **Observability First**: Log everything, measure everything, trace everything

**Additional Constraints:**

1. **Type Safety**: Comprehensive type hints on all public interfaces
2. **Graceful Degradation**: Failures should not crash services
3. **Idempotency**: Duplicate operations handled safely
4. **Testability**: Design for unit and integration testing
5. **Modularity**: Clear separation of concerns, single responsibility

---

### 1.5 Technology Stack Mapping


| Component               | Technology       | Location                                   | Purpose                            |
| ----------------------- | ---------------- | ------------------------------------------ | ---------------------------------- |
| **Query Orchestration** | LangGraph        | `modcus_api_query/services/orchestration/` | State machine, agent coordination  |
| **RAG Engine**          | LlamaIndex       | Both services                              | Vector indexing, query engines     |
| **Agent Logic**         | Python Classes   | `modcus_api_query/services/agents/`        | Business logic for 5 agents        |
| **Tool System**         | LlamaIndex Tools | `modcus_api_query/services/tools/`         | 28+ financial/data/text tools      |
| **LLM Management**      | Custom Service   | `modcus_common/services/llm/`              | Rotation, logging, metrics         |
| **Vector Store**        | Pluggable        | Via LlamaIndex                             | Qdrant, ChromaDB, PGVector         |
| **Background Jobs**     | Celery           | `modcus_api_ingest/` only                  | Async ingestion                    |
| **Database**            | PostgreSQL       | Via SQLModel                               | All persistent data                |
| **Caching**             | Python LRU       | In-memory                                  | Active conversation sessions       |
| **Logging**             | Loguru           | `modcus_common/logging/`                   | Structured logs, per-job isolation |
| **Metrics**             | Prometheus       | `modcus_common/services/metrics/`          | Operational observability          |


---

## 2. Data Model

### 2.1 Core Entities (Fresh Schema)

**Job** - Ingestion job tracking

```python
class Job(SQLModel, table=True):
    id: str  # UUID
    status: str  # queued, running, completed, failed
    progress: int  # 0-100
    
    # Source
    source_kind: str  # upload, url
    filename: Optional[str]
    source_url: Optional[str]
    
    # Batch tracking
    batch_id: Optional[str]
    
    # Retry logic
    attempts: int
    error_code: Optional[str]
    error_message: Optional[str]
    retryable: bool
    
    # Document metadata
    ticker: str
    doc_type: str  # annual_report, financial_report, news, etc.
    doc_tier: str  # A or B
    
    # Processing config
    parser: str  # docling, llamaparse, pdf
    chunk_size: int
    chunk_overlap: int
    embedding_model: str
    
    # Storage paths
    raw_file_path: Optional[str]
    parsed_file_path: Optional[str]
    log_file_path: Optional[str]
    
    # Timing
    duration_ms: Optional[int]
    created_at: datetime
    updated_at: datetime
    started_at: Optional[datetime]
    finished_at: Optional[datetime]
```

**Rationale:**

- Removed LightRAG-specific fields (ingest_llm_mode, rag_settings_json)
- Added LlamaIndex-specific fields (chunk_size, chunk_overlap)
- Added doc_tier for multi-tier classification
- Simplified while preserving essential tracking

---

**Document** - Document metadata and status

```python
class Document(SQLModel, table=True):
    id: int  # Auto-increment
    doc_id: str  # {ticker}__{doc_type}__{date}__{hash_prefix}
    
    # Company info
    ticker: str
    doc_type: str
    doc_tier: str  # A or B
    sector: Optional[str]
    
    # Provenance
    source_url: Optional[str]
    file_hash: str  # SHA-256
    file_path: str  # Storage location
    
    # Dates
    effective_date: str  # YYYY-MM-DD
    scraped_at: Optional[datetime]
    
    # Indexing metadata
    indexed_at: Optional[datetime]
    embedding_model: str
    chunk_count: int
    vector_collection: str  # Collection name in vector store
    
    # Status
    status: str  # PENDING, INDEXED, FAILED, DELETED
    last_error: Optional[str]
    
    # Timestamps
    created_at: datetime
    updated_at: datetime
```

**Rationale:**

- Added doc_tier for classification
- Added chunk_count for tracking
- Added vector_collection for multi-collection support
- Removed index_schema_version (not needed with fresh start)

---

**Company** - Company master data

```python
class Company(SQLModel, table=True):
    id: int
    ticker: str  # Unique
    name: str
    sector: Optional[str]
    market_cap: Optional[float]  # For stock recommendation ranking
    
    # Metadata
    aliases: List[str]  # JSON field for name variations
    
    created_at: datetime
    updated_at: datetime
```

**Rationale:**

- Added market_cap for stock recommendation candidate selection
- Added aliases for ticker inference (e.g., "PT Telkom" → TLKM)
- Minimal, focused on essential data

---

**LLMCall** - Comprehensive LLM interaction logging

```python
class LLMCall(SQLModel, table=True):
    id: str  # UUID
    
    # Context
    job_id: Optional[str]  # FK to Job (for ingestion)
    conversation_id: Optional[str]  # FK to ConversationSession (for queries)
    doc_id: Optional[str]  # Related document
    
    # Stage
    stage: str  # INGESTION, QUERY, VALIDATION, ANALYSIS, etc.
    operation: str  # embed, complete, chat, classify, etc.
    
    # Provider
    provider: str  # openai, anthropic, gemini, etc.
    model: str
    api_key_id: str  # FK to ApiKey
    
    # Performance
    latency_ms: int
    input_tokens: int
    output_tokens: int
    total_tokens: int
    cost_usd: Optional[float]
    
    # Content (truncated previews)
    system_prompt_preview: Optional[str]  # First 500 chars
    user_prompt_preview: Optional[str]
    response_preview: Optional[str]
    
    # Full content (file references)
    prompt_file_id: Optional[str]  # FK to StoredFile
    response_file_id: Optional[str]
    
    # Status
    status: str  # success, failed, timeout
    error_message: Optional[str]
    
    created_at: datetime
```

**Rationale:**

- Added conversation_id for query tracking
- Added operation field for granular tracking
- Added cost_usd for financial tracking
- Preserved comprehensive logging from v0.1

---

**ApiKey** - LLM provider API key management

```python
class ApiKey(SQLModel, table=True):
    id: UUID
    name: str  # Human-friendly identifier
    provider: str  # openai, anthropic, gemini, etc.
    api_key: str  # Encrypted
    
    # Status
    is_active: bool  # Currently selected for rotation
    is_enabled: bool  # Available for use
    
    # Rate limiting
    daily_request_limit: int
    daily_requests: int
    daily_input_tokens: int
    daily_output_tokens: int
    daily_total_tokens: int
    
    # Lifetime metrics
    total_requests: int
    total_input_tokens: int
    total_output_tokens: int
    
    # Timestamps
    last_used_at: Optional[datetime]
    daily_metrics_reset_at: Optional[datetime]
    created_at: datetime
    updated_at: datetime
```

**Rationale:**

- Preserved from v0.1 (proven design)
- Supports rotation and rate limiting
- Comprehensive metrics for cost tracking

---

**ConversationSession** - Chat session management (NEW)

```python
class ConversationSession(SQLModel, table=True):
    id: str  # conversation_id (UUID)
    
    # User context
    user_id: Optional[str]  # For future multi-tenancy
    
    # Session state
    status: str  # active, expired, closed
    turn_count: int
    
    # History management
    has_summary: bool  # True if history was summarized
    summary_branch_id: Optional[str]  # FK to another session (original)
    
    # Timeout
    last_activity_at: datetime
    expires_at: datetime
    timeout_minutes: int  # Configurable per session
    
    # Timestamps
    created_at: datetime
    updated_at: datetime
```

**Rationale:**

- Supports 2-hour timeout (configurable)
- Tracks summarization with branching
- Enables session expiration cleanup

---

**ConversationTurn** - Individual Q&A pairs (NEW)

```python
class ConversationTurn(SQLModel, table=True):
    id: str  # UUID
    session_id: str  # FK to ConversationSession
    turn_number: int
    
    # Request
    query: str
    mode: str
    level: str
    tickers: Optional[List[str]]  # JSON field
    
    # Response
    answer: str
    complexity_level: str  # simple, reasoning, deep-dive (inferred)
    sources: List[Dict]  # JSON field
    
    # Metadata
    metadata_json: Dict  # JSON field
    
    # Performance
    latency_ms: int
    llm_call_ids: List[str]  # JSON field, FKs to LLMCall
    
    created_at: datetime
```

**Rationale:**

- Captures full conversation context
- Links to LLM calls for cost tracking
- Supports history retrieval and summarization

---

**EphemeralArtifact** - User-uploaded documents (Mode 3) (NEW)

```python
class EphemeralArtifact(SQLModel, table=True):
    id: str  # artifact_id (UUID)
    
    # Ownership
    user_id: Optional[str]  # For future multi-tenancy
    session_id: Optional[str]  # Related conversation
    
    # Storage
    vector_collection: str  # ephemeral_{artifact_id}
    file_paths: List[str]  # JSON field, uploaded files
    
    # Lifecycle
    status: str  # processing, ready, expired, deleted
    ttl_hours: int  # Configurable (default 24)
    expires_at: datetime
    
    # Metadata
    document_count: int
    chunk_count: int
    
    # Timestamps
    created_at: datetime
    last_accessed_at: datetime
    deleted_at: Optional[datetime]
```

**Rationale:**

- Tracks ephemeral artifact lifecycle
- Supports 24-hour TTL with manual deletion
- Links to separate vector collection for isolation

---

**Batch** - Batch ingestion tracking (PRESERVED)

```python
class Batch(SQLModel, table=True):
    id: str
    status: str  # pending, running, completed, partial, failed
    
    # Configuration
    ticker: str
    doc_type: str
    doc_tier: str
    
    # Progress
    total_files: int
    completed_files: int
    failed_files: int
    skipped_files: int
    
    # Retry configuration
    retry_strategy: str  # none, transient_only, all_errors
    max_retries: int
    retry_delay_seconds: int
    
    # Timestamps
    created_at: datetime
    updated_at: datetime
    started_at: Optional[datetime]
    finished_at: Optional[datetime]
```

**Rationale:**

- Preserved from v0.1 with retry configuration added
- Supports configurable retry strategies
- Essential for batch operations

---

**StoredFile** - File storage tracking (PRESERVED & ENHANCED)

```python
class StoredFile(SQLModel, table=True):
    id: str = Field(default_factory=lambda: str(uuid4()), primary_key=True)
    kind: str  # raw, parsed, log, prompt, response
    file_path: str
    size_bytes: int
    checksum: str  # SHA-256
    
    # Relationships
    job_id: Optional[str]
    llm_call_id: Optional[str]
    artifact_id: Optional[str]  # For ephemeral artifacts
    
    created_at: datetime
```

**Rationale:**

- Preserved from v0.1 (proven design)
- **ENHANCED**: Added UUID auto-generation for `id` field to prevent null constraint violations
- Added artifact_id for ephemeral artifact tracking
- Supports full deletion (track what to delete)

---

**DeletionLog** - Audit trail (PRESERVED)

```python
class DeletionLog(SQLModel, table=True):
    id: str
    scope: str  # document, artifact, conversation
    target_id: str  # doc_id, artifact_id, conversation_id
    
    # Deletion details
    deleted_items: Dict  # JSON: {files: [...], chunks: N, records: [...]}
    reason: Optional[str]
    
    # Audit
    deleted_by: Optional[str]  # User or system
    deleted_at: datetime
```

**Rationale:**

- Preserved from v0.1
- Enhanced with deleted_items for full cleanup tracking
- Supports compliance and debugging

---

### 2.2 Entity Relationships

```mermaid
classDiagram
    class Job {
        +id: str
        +status: str
        +ticker: str
        +batch_id: str
    }
    
    class Document {
        +id: int
        +doc_id: str
        +ticker: str
        +vector_collection: str
    }
    
    class Company {
        +id: int
        +ticker: str
        +market_cap: float
    }
    
    class LLMCall {
        +id: str
        +job_id: str
        +conversation_id: str
        +api_key_id: str
    }
    
    class ApiKey {
        +id: UUID
        +provider: str
        +is_active: bool
    }
    
    class ConversationSession {
        +id: str
        +status: str
        +summary_branch_id: str
    }
    
    class ConversationTurn {
        +id: str
        +session_id: str
        +turn_number: int
    }
    
    class EphemeralArtifact {
        +id: str
        +vector_collection: str
        +expires_at: datetime
    }
    
    class Batch {
        +id: str
        +ticker: str
        +retry_strategy: str
    }
    
    class StoredFile {
        +id: str
        +job_id: str
        +artifact_id: str
    }
    
    class DeletionLog {
        +id: str
        +target_id: str
        +scope: str
    }
    
    Job "N" --> "1" Batch
    Job "1" --> "N" StoredFile
    Job "1" --> "N" LLMCall
    
    Document "N" --> "1" Company
    
    LLMCall "N" --> "1" ApiKey
    LLMCall "N" --> "1" ConversationSession
    LLMCall "1" --> "N" StoredFile
    
    ConversationSession "1" --> "N" ConversationTurn
    ConversationSession "1" --> "1" ConversationSession : summary_branch
    
    EphemeralArtifact "1" --> "N" StoredFile
```

---

### 2.3 Removed Models (from v0.1)

**RagArtifact** - REMOVED

- **Reason**: LightRAG-specific (working_dir, graph_storage, kv_storage, etc.)
- **Replacement**: Document.vector_collection tracks collection name
- **Migration**: Not needed (fresh schema)

**IngestionJob** - REMOVED

- **Reason**: Redundant with Job model
- **Replacement**: Job.status and Job.progress track pipeline stages
- **Migration**: Not needed (fresh schema)

---

### 2.4 Configuration Schema

**Settings Hierarchy:**

```python
# modcus_common/settings/base.py
class BaseSettings(BaseSettings):
    # Database
    db_url: str
    
    # Storage
    storage_backend: str  # localfs, s3, gcs, azure
    local_storage_base: str
    
    # Logging
    log_dir: str
    log_level: str
    log_retention: str
    
    # Redis
    redis_url: str

# modcus_common/settings/llm_settings.py
class LLMSettings(BaseSettings):
    # LLM Configuration
    llm_provider: str  # openai, anthropic, gemini
    llm_model: str
    llm_temperature: float
    llm_max_tokens: int
    
    # Embedding
    embedding_provider: str
    embedding_model: str
    embedding_dimension: int
    
    # Rotation
    enable_rotation: bool
    rotation_on_rate_limit: bool
    rotation_on_error: bool

# modcus_common/settings/rag_settings.py
class RAGSettings(BaseSettings):
    # Vector Store
    vector_store: str  # qdrant, chromadb, pgvector
    qdrant_url: Optional[str]
    chromadb_persist_dir: Optional[str]
    pgvector_connection_string: Optional[str]
    
    # Chunking
    default_chunk_size: int = 1024
    default_chunk_overlap: int = 200
    chunk_size_by_doc_type: Dict[str, int]  # Configurable per type
    
    # Retrieval
    retrieval_top_k: int = 10
    enable_reranking: bool = False

# modcus_api_query/settings.py
class QuerySettings(BaseSettings, LLMSettings, RAGSettings):
    # Query-specific
    query_timeout_s: int = 360
    
    # Complexity inference
    complexity_inference_mode: str = "combined"  # separate, combined
    
    # Conversation
    conversation_timeout_minutes: int = 120
    conversation_history_limit: Optional[int] = None  # Unlimited
    
    # Ephemeral artifacts
    ephemeral_artifact_ttl_hours: int = 24

# modcus_api_ingest/settings.py
class IngestSettings(BaseSettings, LLMSettings, RAGSettings):
    # Ingestion-specific
    ingestion_max_queued: int = 10
    ingestion_max_per_ticker: int = 3
    
    # Celery
    celery_broker_url: str
    celery_concurrency: int = 2
    
    # Batch retry
    default_retry_strategy: str = "transient_only"
    default_max_retries: int = 3
```

**Rationale:**

- Hierarchical settings (base → specialized)
- Each service extends base with service-specific settings
- All configurable parameters documented
- Type-safe with Pydantic validation

---

## 3. Component Architecture

### 3.1 Query Service Components

**3.1.1 LangGraph Orchestration Layer**

Location: `modcus_api_query/services/orchestration/`

**Components:**

- `query_graph.py` - Main state machine definition
- `state.py` - State schema (Pydantic model)
- `nodes.py` - Node function wrappers (call agents)
- `edges.py` - Conditional routing logic

**State Schema:**

```python
class QueryState(TypedDict):
    # Input
    query: str
    mode: str
    level: str
    tickers: Optional[List[str]]
    artifact_id: Optional[str]
    conversation_id: Optional[str]
    
    # Validation
    intent: str  # finance, off-topic
    complexity: str  # simple, reasoning, deep-dive
    
    # Ticker inference
    inferred_tickers: List[str]
    ticker_confidence: Dict[str, float]
    
    # Retrieval
    retrieved_chunks: List[Dict]
    sources: List[Dict]
    
    # Analysis
    analysis_result: str
    tool_calls: List[Dict]
    
    # Output
    answer: str
    metadata: Dict
    
    # Error handling
    error: Optional[str]
    partial: bool
```

**Graph Definition:**

```python
# Simplified pseudocode
graph = StateGraph(QueryState)

graph.add_node("validate", validation_node)
graph.add_node("infer_ticker", ticker_inference_node)
graph.add_node("retrieve", retrieval_node)
graph.add_node("analyze_simple", simple_analysis_node)
graph.add_node("analyze_reasoning", reasoning_analysis_node)
graph.add_node("analyze_deepdive", deepdive_analysis_node)
graph.add_node("format_response", formatter_node)

graph.add_edge(START, "validate")
graph.add_conditional_edges("validate", route_by_intent)
graph.add_edge("infer_ticker", "retrieve")
graph.add_conditional_edges("retrieve", route_by_complexity)
graph.add_edge("analyze_simple", "format_response")
graph.add_edge("format_response", END)
```

---

**3.1.2 Agent Service Classes**

Location: `modcus_api_query/services/agents/`

**ValidationAgent** (`validation_agent.py`)

- **Responsibility**: API auth, schema validation, intent classification, complexity inference
- **Dependencies**: LLMService (for classification), Settings
- **Interface**:
  ```python
  async def validate_and_classify(query: str, mode: str) -> ValidationResult
  ```

**TickerInferenceAgent** (`ticker_inference_agent.py`)

- **Responsibility**: Extract tickers, resolve company names, handle ambiguity
- **Dependencies**: Database session (Company table), regex patterns
- **Interface**:
  ```python
  async def infer_tickers(query: str, explicit_tickers: Optional[List[str]]) -> TickerResult
  ```

**RetrievalAgent** (`retrieval_agent.py`)

- **Responsibility**: Query LlamaIndex, filter by tickers, rerank, select chunks
- **Dependencies**: VectorStoreIndex, Settings (top_k, reranking)
- **Interface**:
  ```python
  async def retrieve_context(query: str, tickers: List[str], top_k: int) -> RetrievalResult
  ```

**AnalysisAgent** (`analysis_agent.py`)

- **Responsibility**: LLM synthesis with tool calling, complexity-specific prompts
- **Dependencies**: LlamaIndex ReActAgent, LLMService, Tools
- **Interface**:
  ```python
  async def analyze(query: str, context: str, complexity: str, level: str) -> AnalysisResult
  ```

**ResponseFormatterAgent** (`response_formatter.py`)

- **Responsibility**: Structure output, format for presentation level, add citations
- **Dependencies**: Settings (level formatting rules)
- **Interface**:
  ```python
  def format_response(analysis: str, sources: List, level: str) -> FormattedResponse
  ```

---

**3.1.3 Tool System**

Location: `modcus_api_query/services/tools/`

**Tool Categories:**

1. **Financial Tools** (`financial_tools.py`)
  - P/E ratio calculator, DCF model, growth rate calculator
  - CAR, NPL, ROA, ROE, NIM, LDR, BOPO calculators
  - Returns: LlamaIndex `FunctionTool` instances
2. **Data Tools** (`data_tools.py`)
  - Number formatting (Juta/Milyar/Triliun)
  - Ranking, filtering, aggregation
  - Date calculations
  - Returns: LlamaIndex `FunctionTool` instances
3. **Text Tools** (`text_tools.py`)
  - Summarization, sentiment analysis
  - Key figure extraction
  - Comparison generation
  - Returns: LlamaIndex `FunctionTool` instances
4. **RAG Tools** (`rag_tools.py`)
  - General Q&A query engine
  - Financial analysis query engine
  - Returns: LlamaIndex `QueryEngineTool` instances

**Tool Registration:**

```python
# modcus_api_query/services/tools/__init__.py
def get_all_tools(index: VectorStoreIndex) -> List[BaseTool]:
    financial = get_financial_tools()
    data = get_data_tools()
    text = get_text_tools()
    rag = get_rag_tools(index)
    return financial + data + text + rag
```

---

**3.1.4 Chat Session Manager**

Location: `modcus_api_query/services/chat/`

**Responsibilities:**

- Create/retrieve conversation sessions
- Manage conversation history
- Automatic summarization when approaching context window
- Session expiration and cleanup

**Interface:**

```python
class ChatSessionManager:
    async def create_session(user_id: Optional[str]) -> ConversationSession
    async def get_session(session_id: str) -> Optional[ConversationSession]
    async def add_turn(session_id: str, turn: ConversationTurn) -> None
    async def get_history(session_id: str, limit: Optional[int]) -> List[ConversationTurn]
    async def summarize_if_needed(session_id: str) -> bool
    async def expire_session(session_id: str) -> None
```

---

### 3.2 Ingestion Service Components

**3.2.1 Document Parser**

Location: `modcus_api_ingest/services/parsing/`

**Responsibilities:**

- Auto-detect format (Docling JSON, LlamaParse JSON, raw PDF)
- Parse documents using appropriate parser
- Extract text, tables, figures

**Parsers:**

- `docling_parser.py` - Handle Docling JSON format (supports both modern and legacy formats)
- `llamaparse_parser.py` - Handle LlamaParse JSON format
- `pdf_parser.py` - Use LlamaIndex `PDFReader` or `SimpleDirectoryReader`

**Interface:**

```python
class DocumentParser:
    async def parse(file_path: str) -> ParsedDocument
    
class ParsedDocument:
    text: str
    metadata: Dict
    tables: List[Table]
    figures: List[Figure]
```

**DoclingDocument Format Support:**

The Docling parser supports two JSON formats:

1. **Modern Format** (Docling v1.8.0+):
   ```json
   {
     "schema_name": "DoclingDocument",
     "version": "1.8.0",
     "name": "document_name",
     "body": {
       "children": [
         {"$ref": "#/texts/0"},
         {"$ref": "#/texts/1"}
       ]
     },
     "texts": [
       {"text": "Content section 1"},
       {"text": "Content section 2"}
     ],
     "tables": [...],
     "pictures": [...]
   }
   ```
   - Text content is stored in a `texts` array
   - `body.children` contains references to text items via `$ref` pointers
   - Tables are in a top-level `tables` array
   - Figures/images are in a top-level `pictures` array

2. **Legacy Format** (older Docling versions):
   ```json
   {
     "schema_name": "DoclingDocument",
     "pages": [
       {"text": "Page 1 content", "tables": [...]},
       {"text": "Page 2 content", "tables": [...]}
     ]
   }
   ```
   - Text content is stored per-page in `pages` array
   - Tables and figures are nested within each page

The parser auto-detects the format and extracts text accordingly.

---

**3.2.2 Chunking Pipeline**

Location: `modcus_api_ingest/services/chunking/`

**Responsibilities:**

- Chunk documents using LlamaIndex node parsers
- Apply configurable chunk size per document type
- Enrich chunks with metadata

**Interface:**

```python
class ChunkingService:
    def chunk_document(
        parsed_doc: ParsedDocument,
        doc_type: str,
        chunk_size: Optional[int] = None
    ) -> List[TextNode]
```

**Configuration:**

```python
# From settings
chunk_size_by_doc_type = {
    "annual_report": 1024,
    "financial_report": 1024,
    "news": 2048,  # Larger chunks for news (less precision needed)
}
```

---

**3.2.3 Embedding & Indexing**

Location: `modcus_api_ingest/services/indexing/`

**Responsibilities:**

- Generate embeddings via LLMService
- Insert nodes into LlamaIndex VectorStoreIndex
- Track progress and update Job status

**Interface:**

```python
class IndexingService:
    async def index_document(
        chunks: List[TextNode],
        job_id: str,
        collection: str = "main_kb"
    ) -> IndexingResult
```

---

**3.2.4 Celery Task Orchestration**

Location: `modcus_api_ingest/services/tasks/`

**Main Task:**

```python
@celery_app.task(bind=True)
def process_ingestion_job(self, job_id: str):
    # 1. Load job from database
    # 2. Attach per-job logger
    # 3. Parse document
    # 4. Chunk document
    # 5. Generate embeddings
    # 6. Index into vector store
    # 7. Update job status
    # 8. Detach logger
    # 9. Log metrics
```

**Retry Logic:**

```python
@celery_app.task(
    bind=True,
    autoretry_for=(RateLimitError, NetworkError),
    retry_backoff=True,
    retry_kwargs={'max_retries': 3}
)
```

---

### 3.3 Common Module Components

**3.3.1 LLM Service (Rotation & Logging)**

Location: `modcus_common/services/llm/`

**Components:**

- `llm_service.py` - Main service with rotation
- `llm_factory.py` - Creates LlamaIndex LLM instances
- `llm_rotator.py` - Key rotation logic
- `llm_logger.py` - Call logging and metrics

**LLMService Interface:**

```python
class LLMService:
    def __init__(self, provider: str, session_factory):
        self.provider = provider
        self.rotator = LLMRotator(provider, session_factory)
        self.logger = LLMLogger(session_factory)
    
    async def complete(
        self,
        prompt: str,
        system_prompt: Optional[str] = None,
        context: Optional[Dict] = None
    ) -> LLMResponse:
        # 1. Get current LLM from rotator
        # 2. Make call
        # 3. On error: rotate and retry
        # 4. Log call
        # 5. Return response
    
    async def chat(messages: List[Dict]) -> LLMResponse
    async def embed(texts: List[str]) -> List[List[float]]
```

**VertexAI Authentication:**

For VertexAI provider, credentials are loaded from a service account JSON file:

```python
# In llm_service.py _get_embedding_model()
if self.settings.embedding_provider.lower() == "vertex":
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
        credentials=credentials,  # Passed to VertexTextEmbedding
    )
```

**Required Environment Variables for VertexAI:**
- `GCP_PROJECT_ID`: GCP project ID
- `GCP_LOCATION`: GCP location (default: us-central1)
- `GOOGLE_APPLICATION_CREDENTIALS`: Path to service account JSON file

**Docker Compose Configuration:**
```yaml
environment:
  GOOGLE_APPLICATION_CREDENTIALS: /app/service-account.json
volumes:
  - ./service-account.json:/app/service-account.json:ro
```

**LLMRotator:**

```python
class LLMRotator:
    def get_current_llm(self) -> BaseLLM:
        # Get active API key from database
        # Create LlamaIndex LLM instance
        # Return configured LLM
    
    def rotate_on_error(self, error: Exception) -> bool:
        # Classify error (rate limit, auth, network, etc.)
        # If rotatable: mark current key inactive, activate next
        # Return True if rotated, False if no more keys
```

---

**3.3.2 Vector Store Factory**

Location: `modcus_common/services/vector_store/`

**Thin wrapper around LlamaIndex vector stores:**

```python
class VectorStoreFactory:
    @staticmethod
    def create(settings: RAGSettings) -> BaseVectorStore:
        if settings.vector_store == "qdrant":
            return QdrantVectorStore(
                client=QdrantClient(url=settings.qdrant_url),
                collection_name="main_kb"
            )
        elif settings.vector_store == "chromadb":
            return ChromaVectorStore(
                chroma_collection=chroma_client.get_or_create_collection("main_kb")
            )
        elif settings.vector_store == "pgvector":
            return PGVectorStore(
                connection_string=settings.pgvector_connection_string,
                table_name="embeddings"
            )
```

**Configuration Files:**

- `qdrant_config.py` - Qdrant client setup
- `chromadb_config.py` - ChromaDB client setup
- `pgvector_config.py` - PGVector connection setup

---

**3.3.3 Logging Utilities (Preserved from v0.1)**

Location: `modcus_common/logging/`

**Components:**

- `init_logging.py` - Global logger setup
- `job_logger.py` - Per-job log isolation (attach/detach)
- `structured_logger.py` - Structured log formatting

**Preserved Patterns:**

- Global logs: `modcus_common/.logs/{service_name}.log`
- Per-job logs: `modcus_common/.logs/jobs/{job_id}/pipeline.log`
- Loguru with context binding
- Rotation and retention policies

---

**3.3.4 Metrics & Monitoring**

Location: `modcus_common/services/metrics/`

**Prometheus Metrics:**

- `query_latency_seconds` - Histogram by mode, level
- `ingestion_duration_seconds` - Histogram by doc_type
- `llm_calls_total` - Counter by provider, model, status
- `llm_tokens_total` - Counter by provider, type (input/output)
- `llm_cost_usd_total` - Counter by provider
- `active_jobs` - Gauge by status
- `vector_store_chunks` - Gauge by collection

**Interface:**

```python
class MetricsService:
    @staticmethod
    def record_query(mode: str, level: str, latency_ms: int)
    
    @staticmethod
    def record_llm_call(provider: str, tokens: int, cost: float)
    
    @staticmethod
    def record_ingestion(doc_type: str, duration_ms: int, chunks: int)
```

---

### 3.4 Integration Patterns

**3.4.1 Service-to-Service Communication**

**Pattern:** HTTP APIs only (except `modcus_common`)

- Query service does NOT import from Ingestion service
- Ingestion service does NOT import from Query service
- Both import from `modcus_common` (models, settings, utilities)

**Example:**

```python
# ❌ WRONG
from modcus_api_ingest.services.parsing import DocumentParser

# ✅ CORRECT
response = httpx.post("http://ingest-service:8081/v1/ingestion/upload", ...)
```

---

**3.4.2 LlamaIndex Integration**

**Ingestion Service:**

```python
# Create index
vector_store = VectorStoreFactory.create(settings)
storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex(nodes, storage_context=storage_context)
```

**Query Service:**

```python
# Load index
vector_store = VectorStoreFactory.create(settings)
index = VectorStoreIndex.from_vector_store(vector_store)

# Query
query_engine = index.as_query_engine(similarity_top_k=10)
response = await query_engine.aquery(query)
```

---

**3.4.3 LangGraph Integration**

**Query Route → LangGraph:**

```python
@router.post("/query")
async def query_endpoint(request: QueryRequest):
    # Create initial state
    state = QueryState(
        query=request.query,
        mode=request.mode,
        level=request.level,
        tickers=request.tickers
    )
    
    # Invoke graph
    graph = get_query_graph()
    result = await graph.ainvoke(state)
    
    # Return formatted response
    return QueryResponse(
        answer=result["answer"],
        sources=result["sources"],
        metadata=result["metadata"]
    )
```

**LangGraph Node → Agent:**

```python
async def validation_node(state: QueryState) -> QueryState:
    agent = ValidationAgent(llm_service, settings)
    result = await agent.validate_and_classify(state["query"], state["mode"])
    
    state["intent"] = result.intent
    state["complexity"] = result.complexity
    return state
```

---

**3.4.4 Tool Integration**

**Analysis Agent wraps LlamaIndex ReActAgent:**

```python
class AnalysisAgent:
    def __init__(self, index: VectorStoreIndex, llm_service: LLMService):
        # Get all tools
        tools = get_all_tools(index)
        
        # Create ReActAgent
        self.agent = ReActAgent.from_tools(
            tools=tools,
            llm=llm_service.get_llm(),
            verbose=True
        )
    
    async def analyze(self, query: str, context: str, complexity: str) -> str:
        # Build prompt based on complexity
        prompt = self._build_prompt(query, context, complexity)
        
        # Run agent
        response = await self.agent.achat(prompt)
        
        return str(response)
```

---

### 3.5 Data Flow Patterns

**3.5.1 Ingestion Data Flow**

```mermaid
sequenceDiagram
    participant User
    participant API as Ingestion API
    participant Celery
    participant Parser
    participant Chunker
    participant LLMSvc as LLM Service
    participant Index as LlamaIndex
    participant VectorDB as Vector Store
    participant DB as PostgreSQL
    
    User->>API: POST /v1/ingestion/upload
    API->>DB: Create Job (status=queued)
    API->>Celery: Queue task
    API-->>User: {job_id, status}
    
    Celery->>DB: Update Job (status=running)
    Celery->>Parser: Parse document
    Parser-->>Celery: ParsedDocument
    
    Celery->>Chunker: Chunk document
    Chunker-->>Celery: List[TextNode]
    
    Celery->>LLMSvc: Generate embeddings
    LLMSvc->>DB: Log LLMCall
    LLMSvc-->>Celery: Embeddings
    
    Celery->>Index: Insert nodes
    Index->>VectorDB: Store vectors
    VectorDB-->>Index: Success
    
    Celery->>DB: Update Job (status=completed)
    Celery->>DB: Create Document record
```

---

**3.5.2 Query Data Flow (Mode 1)**

```mermaid
sequenceDiagram
    participant User
    participant API as Query API
    participant LG as LangGraph
    participant VA as Validation Agent
    participant TIA as Ticker Agent
    participant RA as Retrieval Agent
    participant AA as Analysis Agent
    participant RFA as Formatter Agent
    participant LLMSvc as LLM Service
    participant Index as LlamaIndex
    participant VectorDB as Vector Store
    participant DB as PostgreSQL
    
    User->>API: POST /v1/query
    API->>LG: Invoke graph
    
    LG->>VA: Validate & classify
    VA->>LLMSvc: Classify intent
    LLMSvc->>DB: Log LLMCall
    VA-->>LG: {intent, complexity}
    
    LG->>TIA: Infer tickers
    TIA->>DB: Query Company table
    TIA-->>LG: {tickers}
    
    LG->>RA: Retrieve context
    RA->>Index: Query
    Index->>VectorDB: Vector search
    VectorDB-->>Index: Chunks
    RA-->>LG: {chunks, sources}
    
    LG->>AA: Analyze
    AA->>LLMSvc: LLM synthesis + tools
    LLMSvc->>DB: Log LLMCall
    AA-->>LG: {analysis}
    
    LG->>RFA: Format response
    RFA-->>LG: {answer, sources, metadata}
    
    LG-->>API: Final state
    API->>DB: Save ConversationTurn
    API-->>User: QueryResponse
```

---

### 3.6 Failure Modes & Recovery

**LLM Provider Failure:**

1. LLMService detects error (rate limit, timeout, auth)
2. LLMRotator classifies error as rotatable
3. Mark current ApiKey as inactive
4. Activate next ApiKey
5. Retry call with new key
6. Log rotation event
7. If all keys exhausted: Return error to user

**Vector Store Unavailable:**

1. LlamaIndex raises connection error
2. Retry with exponential backoff (3 attempts)
3. If persistent: Return 503 Service Unavailable
4. Log error, alert monitoring

**Celery Worker Crash:**

1. Job remains in "running" status
2. Celery auto-restarts worker
3. Job timeout mechanism (30 min) marks as failed
4. User can retry job manually
5. Logs preserved for debugging

**Context Window Exceeded:**

1. ChatSessionManager detects approaching limit
2. Trigger summarization:
  - Create summary of turns 1-N
  - Create new session (summary branch)
  - Link original session to summary
3. Continue conversation with summary + recent turns

---

### 3.7 Configuration Strategy

**Environment Variables (Hierarchical):**

```bash
# Base (modcus_common)
DB_URL=postgresql://...
REDIS_URL=redis://...
LOG_LEVEL=INFO

# LLM (modcus_common)
LLM_PROVIDER=gemini
LLM_MODEL=gemini-1.5-pro
EMBEDDING_PROVIDER=openai
EMBEDDING_MODEL=text-embedding-3-small
ENABLE_LLM_ROTATION=true

# RAG (modcus_common)
VECTOR_STORE=qdrant
QDRANT_URL=http://localhost:6333
DEFAULT_CHUNK_SIZE=1024
DEFAULT_CHUNK_OVERLAP=200
RETRIEVAL_TOP_K=10

# Query Service
QUERY_TIMEOUT_S=360
COMPLEXITY_INFERENCE_MODE=combined
CONVERSATION_TIMEOUT_MINUTES=120
EPHEMERAL_ARTIFACT_TTL_HOURS=24

# Ingestion Service
CELERY_CONCURRENCY=2
INGESTION_MAX_QUEUED=10
DEFAULT_RETRY_STRATEGY=transient_only
DEFAULT_MAX_RETRIES=3
```

**Settings Loading:**

```python
# Each service loads hierarchically
settings = QuerySettings()  # Inherits BaseSettings + LLMSettings + RAGSettings
```

---

## 2. Data Model

### 2.1 Database Schema Design

**Design Philosophy:**

- Selective simplification: Remove obsolete (RagArtifact, IngestionJob), preserve proven (LLMCall, ApiKey), add new (ConversationSession, EphemeralArtifact)
- Fresh start: No migration from v0.1, clean field naming
- Type-safe: SQLModel with comprehensive type hints
- Auditable: Track all state changes, deletions, LLM calls

---

### 2.2 Core Tables

**Job** - Ingestion job tracking

```python
class Job(SQLModel, table=True):
    __tablename__ = "job"
    
    # Identity
    id: str = Field(primary_key=True)  # UUID
    batch_id: Optional[str] = Field(default=None, index=True)
    
    # Status
    status: str = Field(index=True)  # queued, running, completed, failed
    progress: int = Field(default=0, ge=0, le=100)
    
    # Source
    source_kind: str  # upload, url
    filename: Optional[str]
    source_url: Optional[str]
    
    # Document metadata
    ticker: str = Field(index=True)
    doc_type: str = Field(index=True)
    doc_tier: str = Field(index=True)  # A or B
    sector: Optional[str]
    
    # Processing configuration
    parser: str  # docling, llamaparse, pdf
    chunk_size: int
    chunk_overlap: int
    embedding_model: str
    
    # Storage paths
    raw_file_path: Optional[str]
    parsed_file_path: Optional[str]
    log_file_path: Optional[str]
    
    # Retry tracking
    attempts: int = Field(default=0)
    error_code: Optional[str]
    error_message: Optional[str]
    retryable: bool = Field(default=False)
    
    # Performance
    duration_ms: Optional[int]
    
    # Timestamps
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
    started_at: Optional[datetime]
    finished_at: Optional[datetime]
```

---

**Document** - Document metadata

```python
class Document(SQLModel, table=True):
    __tablename__ = "document"
    
    # Identity
    id: int = Field(primary_key=True)  # Auto-increment
    doc_id: str = Field(unique=True, index=True)  # {ticker}__{type}__{date}__{hash}
    
    # Company
    ticker: str = Field(index=True)
    doc_type: str = Field(index=True)
    doc_tier: str = Field(index=True)  # A or B
    sector: Optional[str]
    
    # Provenance
    source_url: Optional[str]
    file_hash: str = Field(index=True)  # SHA-256
    file_path: str
    
    # Dates
    effective_date: str = Field(index=True)  # YYYY-MM-DD
    scraped_at: Optional[datetime]
    
    # Indexing
    indexed_at: Optional[datetime]
    embedding_model: str
    chunk_count: int
    vector_collection: str  # Collection name (main_kb or ephemeral_*)
    
    # Status
    status: str = Field(index=True)  # PENDING, INDEXED, FAILED, DELETED
    last_error: Optional[str]
    
    # Timestamps
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

---

**Company** - Company master data

```python
class Company(SQLModel, table=True):
    __tablename__ = "company"
    
    # Identity
    id: int = Field(primary_key=True)
    ticker: str = Field(unique=True, index=True)
    
    # Basic info
    name: str
    sector: Optional[str] = Field(index=True)
    market_cap: Optional[float]  # For ranking in stock recommendations
    
    # Ticker inference support
    aliases: List[str] = Field(default=[], sa_column=Column(JSON))  # Name variations
    
    # Timestamps
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

---

**ConversationSession** - Chat session management (NEW)

```python
class ConversationSession(SQLModel, table=True):
    __tablename__ = "conversation_session"
    
    # Identity
    id: str = Field(primary_key=True)  # conversation_id (UUID)
    
    # User context
    user_id: Optional[str] = Field(index=True)  # For future multi-tenancy
    
    # Session state
    status: str = Field(index=True)  # active, expired, closed
    turn_count: int = Field(default=0)
    
    # History management
    has_summary: bool = Field(default=False)
    summary_branch_id: Optional[str]  # FK to original session
    summary_text: Optional[str]  # Compressed history
    
    # Timeout configuration
    timeout_minutes: int = Field(default=120)
    last_activity_at: datetime = Field(default_factory=datetime.utcnow)
    expires_at: datetime  # Computed: last_activity + timeout
    
    # Timestamps
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

---

**ConversationTurn** - Q&A pairs (NEW)

```python
class ConversationTurn(SQLModel, table=True):
    __tablename__ = "conversation_turn"
    
    # Identity
    id: str = Field(primary_key=True)  # UUID
    session_id: str = Field(foreign_key="conversation_session.id", index=True)
    turn_number: int
    
    # Request
    query: str
    mode: str
    level: str
    tickers: Optional[List[str]] = Field(default=None, sa_column=Column(JSON))
    
    # Response
    answer: str
    complexity_level: str  # simple, reasoning, deep-dive (inferred)
    sources: List[Dict] = Field(default=[], sa_column=Column(JSON))
    
    # Metadata
    metadata_json: Dict = Field(default={}, sa_column=Column(JSON))
    
    # Performance
    latency_ms: int
    llm_call_ids: List[str] = Field(default=[], sa_column=Column(JSON))
    
    # Timestamps
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

---

**EphemeralArtifact** - User uploads (Mode 3) (NEW)

```python
class EphemeralArtifact(SQLModel, table=True):
    __tablename__ = "ephemeral_artifact"
    
    # Identity
    id: str = Field(primary_key=True)  # artifact_id (UUID)
    
    # Ownership
    user_id: Optional[str] = Field(index=True)
    session_id: Optional[str]  # Related conversation
    
    # Storage
    vector_collection: str  # ephemeral_{artifact_id}
    file_paths: List[str] = Field(sa_column=Column(JSON))
    
    # Lifecycle
    status: str = Field(index=True)  # processing, ready, expired, deleted
    ttl_hours: int = Field(default=24)
    expires_at: datetime = Field(index=True)
    
    # Metadata
    document_count: int
    chunk_count: int
    
    # Timestamps
    created_at: datetime = Field(default_factory=datetime.utcnow)
    last_accessed_at: datetime = Field(default_factory=datetime.utcnow)
    deleted_at: Optional[datetime]
```

---

**LLMCall** - LLM interaction logging (PRESERVED & ENHANCED)

```python
class LLMCall(SQLModel, table=True):
    __tablename__ = "llm_call"
    
    # Identity
    id: str = Field(primary_key=True)  # UUID
    
    # Context
    job_id: Optional[str] = Field(index=True)  # Ingestion context
    conversation_id: Optional[str] = Field(index=True)  # Query context
    doc_id: Optional[str] = Field(index=True)  # Related document
    
    # Stage
    stage: str = Field(index=True)  # INGESTION, QUERY, VALIDATION, ANALYSIS
    operation: str  # embed, complete, chat, classify
    
    # Provider
    provider: str = Field(index=True)
    model: str = Field(index=True)
    api_key_id: str = Field(foreign_key="api_key.id", index=True)
    
    # Performance
    latency_ms: int
    input_tokens: int
    output_tokens: int
    total_tokens: int
    cost_usd: Optional[float]
    
    # Content previews (first 500 chars)
    system_prompt_preview: Optional[str]
    user_prompt_preview: Optional[str]
    response_preview: Optional[str]
    
    # Full content (file references)
    prompt_file_id: Optional[str] = Field(foreign_key="stored_file.id")
    response_file_id: Optional[str] = Field(foreign_key="stored_file.id")
    
    # Status
    status: str = Field(index=True)  # success, failed, timeout
    error_message: Optional[str]
    
    # Timestamps
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

---

**ApiKey** - API key management (PRESERVED)

```python
class ApiKey(SQLModel, table=True):
    __tablename__ = "api_key"
    
    # Identity
    id: UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    name: str = Field(index=True)
    provider: str = Field(index=True)
    
    # Key
    api_key: str = Field(unique=True)  # Encrypted
    
    # Status
    is_active: bool = Field(default=False)  # Currently selected
    is_enabled: bool = Field(default=True)  # Available for rotation
    
    # Rate limiting
    daily_request_limit: int = Field(default=10000)
    daily_requests: int = Field(default=0)
    daily_input_tokens: int = Field(default=0)
    daily_output_tokens: int = Field(default=0)
    daily_total_tokens: int = Field(default=0)
    
    # Lifetime metrics
    total_requests: int = Field(default=0)
    total_input_tokens: int = Field(default=0)
    total_output_tokens: int = Field(default=0)
    total_total_tokens: int = Field(default=0)
    
    # Timestamps
    last_used_at: Optional[datetime]
    daily_metrics_reset_at: Optional[datetime]
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

---

**Batch, StoredFile, DeletionLog** - PRESERVED (see Epic Brief Section 2.2 for details)

---

### 2.3 Indexes & Constraints

**Primary Keys:**

- All use UUIDs except auto-increment IDs (Document.id, Company.id)

**Unique Constraints:**

- `Document.doc_id` - Prevent duplicate documents
- `Company.ticker` - One company per ticker
- `ApiKey.api_key` - Prevent duplicate keys

**Foreign Keys:**

- `Job.batch_id` → `Batch.id`
- `ConversationTurn.session_id` → `ConversationSession.id`
- `LLMCall.api_key_id` → `ApiKey.id`
- `StoredFile.job_id` → `Job.id`

**Indexes:**

- `Job.status`, `Job.ticker`, `Job.batch_id`
- `Document.ticker`, `Document.doc_type`, `Document.status`, `Document.effective_date`
- `Company.ticker`, `Company.sector`
- `ConversationSession.status`, `ConversationSession.expires_at`
- `LLMCall.stage`, `LLMCall.provider`, `LLMCall.conversation_id`

---

## 3. Component Architecture

### 3.1 Query Service Architecture

**Module Structure:**

```
modcus_api_query/
├── api/
│   ├── deps.py                    # Dependency injection
│   ├── routes_query.py            # Main query endpoint
│   ├── routes_chat.py             # Chat endpoints
│   ├── routes_health.py           # Health checks
│   └── routes_admin.py            # Admin endpoints
├── services/
│   ├── orchestration/
│   │   ├── query_graph.py         # LangGraph state machine
│   │   ├── state.py               # State schema
│   │   ├── nodes.py               # Node wrappers
│   │   └── edges.py               # Conditional routing
│   ├── agents/
│   │   ├── validation_agent.py
│   │   ├── ticker_inference_agent.py
│   │   ├── retrieval_agent.py
│   │   ├── analysis_agent.py
│   │   └── response_formatter.py
│   ├── tools/
│   │   ├── financial_tools.py
│   │   ├── data_tools.py
│   │   ├── text_tools.py
│   │   └── rag_tools.py
│   ├── chat/
│   │   ├── session_manager.py
│   │   └── history_summarizer.py
│   └── index/
│       └── index_manager.py       # LlamaIndex instance
├── models/                        # Re-exports from modcus_common
├── settings.py                    # Service settings
└── main.py                        # FastAPI app
```

---

**3.1.1 LangGraph State Machine**

**File:** `modcus_api_query/services/orchestration/query_graph.py`

**Responsibilities:**

- Define state machine for query pipeline
- Coordinate 5 agents
- Handle conditional routing (complexity levels, modes)
- Manage error states

**Key Functions:**

```python
def create_query_graph() -> StateGraph:
    # Define graph structure
    # Add nodes for each agent
    # Add conditional edges
    # Return compiled graph

def route_by_intent(state: QueryState) -> str:
    # Route based on intent classification
    # off-topic → reject, finance → continue

def route_by_complexity(state: QueryState) -> str:
    # Route to appropriate analysis node
    # simple → simple_analysis, reasoning → reasoning_analysis, etc.
```

---

**3.1.2 Agent Service Classes**

**ValidationAgent** - Intent classification and complexity inference

**Responsibilities:**

- Validate request schema
- Classify query intent (finance vs. off-topic)
- Infer complexity level (simple, reasoning, deep-dive)
- Configurable: separate LLM call or combined

**Dependencies:**

- LLMService (for classification)
- Settings (complexity_inference_mode)

**Interface:**

```python
class ValidationAgent:
    async def validate_and_classify(
        query: str,
        mode: str,
        level: str
    ) -> ValidationResult
    
class ValidationResult:
    is_valid: bool
    intent: str  # finance, off-topic
    complexity: str  # simple, reasoning, deep-dive
    error_message: Optional[str]
```

---

**TickerInferenceAgent** - Ticker extraction and resolution

**Responsibilities:**

- Extract ticker symbols via regex
- Match company names from database
- Resolve ambiguities
- Handle multi-ticker queries

**Dependencies:**

- Database session (Company table)
- Regex patterns

**Interface:**

```python
class TickerInferenceAgent:
    async def infer_tickers(
        query: str,
        explicit_tickers: Optional[List[str]]
    ) -> TickerResult
    
class TickerResult:
    tickers: List[str]
    confidence: Dict[str, float]
    is_ambiguous: bool
    candidates: List[str]
```

---

**RetrievalAgent** - Context retrieval from LlamaIndex

**Responsibilities:**

- Query LlamaIndex VectorStoreIndex
- Filter by ticker scoping
- Optional reranking
- Chunk selection (top_k)

**Dependencies:**

- VectorStoreIndex instance
- Settings (retrieval_top_k, enable_reranking)

**Interface:**

```python
class RetrievalAgent:
    async def retrieve_context(
        query: str,
        tickers: List[str],
        collection: str = "main_kb",
        top_k: Optional[int] = None
    ) -> RetrievalResult
    
class RetrievalResult:
    chunks: List[TextNode]
    sources: List[Dict]  # Citations
    total_retrieved: int
```

---

**AnalysisAgent** - LLM synthesis with tools

**Responsibilities:**

- Wrap LlamaIndex ReActAgent
- Provide complexity-specific prompts
- Execute tool calls
- Generate structured analysis

**Dependencies:**

- LlamaIndex ReActAgent
- LLMService
- Tools (28+ financial/data/text tools)

**Interface:**

```python
class AnalysisAgent:
    async def analyze(
        query: str,
        context: str,
        complexity: str,
        level: str,
        tickers: List[str]
    ) -> AnalysisResult
    
class AnalysisResult:
    answer: str
    tool_calls: List[Dict]
    reasoning: Optional[str]
```

---

**ResponseFormatterAgent** - Output structuring

**Responsibilities:**

- Format answer for presentation level
- Add citations and metadata
- Structure JSON response

**Dependencies:**

- Settings (level formatting rules)

**Interface:**

```python
class ResponseFormatterAgent:
    def format_response(
        analysis: str,
        sources: List[Dict],
        level: str,
        metadata: Dict
    ) -> FormattedResponse
    
class FormattedResponse:
    answer: str
    sources: List[Dict]
    metadata: Dict
```

---

### 3.2 Ingestion Service Architecture

**Module Structure:**

```
modcus_api_ingest/
├── api/
│   ├── deps.py
│   ├── routes_ingestion.py        # Upload endpoints
│   ├── routes_batch.py            # Batch endpoints
│   ├── routes_jobs.py             # Job management
│   └── routes_health.py
├── services/
│   ├── parsing/
│   │   ├── docling_parser.py
│   │   ├── llamaparse_parser.py
│   │   └── pdf_parser.py
│   ├── chunking/
│   │   └── chunking_service.py
│   ├── indexing/
│   │   └── indexing_service.py
│   ├── tasks/
│   │   └── ingestion_tasks.py     # Celery tasks
│   └── index/
│       └── index_manager.py       # LlamaIndex instance
├── settings.py
└── main.py
```

---

**3.2.1 Document Parsers**

**Auto-Detection Logic:**

```python
class DocumentParser:
    @staticmethod
    def detect_format(file_path: str) -> str:
        # Check extension
        if file_path.endswith('.pdf'):
            return 'pdf'
        
        # Check JSON schema
        with open(file_path) as f:
            data = json.load(f)
            if data.get('schema_name') == 'DoclingDocument':
                return 'docling'
            elif 'job_id' in data and 'pages' in data:
                return 'llamaparse'
        
        raise ValueError("Unknown format")
    
    async def parse(file_path: str) -> ParsedDocument:
        format = self.detect_format(file_path)
        
        if format == 'docling':
            return DoclingParser().parse(file_path)
        elif format == 'llamaparse':
            return LlamaParseParser().parse(file_path)
        elif format == 'pdf':
            return PDFParser().parse(file_path)
```

---

**3.2.2 Chunking Service**

**Configurable Chunking:**

```python
class ChunkingService:
    def __init__(self, settings: RAGSettings):
        self.settings = settings
    
    def chunk_document(
        parsed_doc: ParsedDocument,
        doc_type: str
    ) -> List[TextNode]:
        # Get chunk size for doc type
        chunk_size = self.settings.chunk_size_by_doc_type.get(
            doc_type,
            self.settings.default_chunk_size
        )
        
        # Create node parser
        parser = SentenceSplitter(
            chunk_size=chunk_size,
            chunk_overlap=self.settings.default_chunk_overlap
        )
        
        # Create document
        doc = Document(text=parsed_doc.text, metadata=parsed_doc.metadata)
        
        # Generate nodes
        nodes = parser.get_nodes_from_documents([doc])
        
        return nodes
```

---

**3.2.3 Indexing Service**

**LlamaIndex Integration:**

```python
class IndexingService:
    def __init__(self, vector_store: BaseVectorStore, llm_service: LLMService):
        self.vector_store = vector_store
        self.llm_service = llm_service
        
        # Create index
        storage_context = StorageContext.from_defaults(vector_store=vector_store)
        self.index = VectorStoreIndex([], storage_context=storage_context)
    
    async def index_document(
        chunks: List[TextNode],
        job_id: str,
        collection: str = "main_kb"
    ) -> IndexingResult:
        # Generate embeddings via LLMService
        embeddings = await self.llm_service.embed([node.text for node in chunks])
        
        # Attach embeddings to nodes
        for node, embedding in zip(chunks, embeddings):
            node.embedding = embedding
        
        # Insert into index
        self.index.insert_nodes(chunks)
        
        return IndexingResult(
            chunk_count=len(chunks),
            collection=collection
        )
```

---

**3.2.4 Celery Task**

**Main Ingestion Task:**

```python
@celery_app.task(
    bind=True,
    autoretry_for=(RateLimitError, NetworkError),
    retry_backoff=True,
    retry_kwargs={'max_retries': 3}
)
def process_ingestion_job(self, job_id: str):
    # 1. Load job
    job = get_job(job_id)
    
    # 2. Attach per-job logger
    logger_id = attach_job_logger(job_id)
    
    try:
        # 3. Update status
        update_job(job_id, status="running", progress=10)
        
        # 4. Parse
        parser = DocumentParser()
        parsed = await parser.parse(job.raw_file_path)
        update_job(job_id, progress=30)
        
        # 5. Chunk
        chunker = ChunkingService(settings)
        chunks = chunker.chunk_document(parsed, job.doc_type)
        update_job(job_id, progress=50)
        
        # 6. Index
        indexer = IndexingService(vector_store, llm_service)
        result = await indexer.index_document(chunks, job_id)
        update_job(job_id, progress=90)
        
        # 7. Create Document record
        create_document(job, result)
        
        # 8. Complete
        update_job(job_id, status="completed", progress=100)
        
    except Exception as e:
        # Classify error
        retryable = classify_error(e)
        update_job(job_id, status="failed", error=str(e), retryable=retryable)
        
        if not retryable:
            raise  # Don't retry
    
    finally:
        detach_logger(logger_id)
```

---

### 3.3 Common Module Architecture

**3.3.1 LLM Service (Rotation & Logging)**

**File:** `modcus_common/services/llm/llm_service.py`

**Responsibilities:**

- Create LlamaIndex LLM instances
- Rotate API keys on failures
- Log all LLM calls
- Track metrics

**Interface:**

```python
class LLMService:
    def __init__(self, session_factory, settings: LLMSettings):
        self.rotator = LLMRotator(session_factory, settings)
        self.logger = LLMCallLogger(session_factory)
        self.settings = settings
    
    async def complete(
        prompt: str,
        system_prompt: Optional[str] = None,
        context: Optional[Dict] = None
    ) -> LLMResponse:
        llm = self.rotator.get_current_llm()
        
        try:
            response = await llm.acomplete(prompt)
            self.logger.log_call(context, response, status="success")
            return response
        
        except Exception as e:
            if self.rotator.should_rotate(e):
                self.rotator.rotate()
                return await self.complete(prompt, system_prompt, context)
            else:
                self.logger.log_call(context, None, status="failed", error=str(e))
                raise
    
    async def chat(messages: List[Dict]) -> LLMResponse
    async def embed(texts: List[str]) -> List[List[float]]
    
    def get_llm(self) -> BaseLLM:
        # For direct use in LlamaIndex components
        return self.rotator.get_current_llm()
```

---

**3.3.2 Vector Store Factory**

**File:** `modcus_common/services/vector_store/vector_store_factory.py`

**Thin wrapper around LlamaIndex vector stores:**

```python
class VectorStoreFactory:
    @staticmethod
    def create(settings: RAGSettings, collection: str = "main_kb") -> BaseVectorStore:
        if settings.vector_store == "qdrant":
            from llama_index.vector_stores.qdrant import QdrantVectorStore
            from qdrant_client import QdrantClient
            
            client = QdrantClient(url=settings.qdrant_url)
            return QdrantVectorStore(client=client, collection_name=collection)
        
        elif settings.vector_store == "chromadb":
            from llama_index.vector_stores.chroma import ChromaVectorStore
            import chromadb
            
            client = chromadb.PersistentClient(path=settings.chromadb_persist_dir)
            collection = client.get_or_create_collection(collection)
            return ChromaVectorStore(chroma_collection=collection)
        
        elif settings.vector_store == "pgvector":
            from llama_index.vector_stores.postgres import PGVectorStore
            
            return PGVectorStore.from_params(
                connection_string=settings.pgvector_connection_string,
                table_name=collection
            )
```

---

**3.3.3 Logging Utilities (Preserved)**

**Per-Job Log Isolation:**

```python
# modcus_common/logging/job_logger.py
def attach_job_logger(job_id: str, log_file_path: str) -> str:
    # Create file handler for job
    handler_id = f"job_{job_id}"
    logger.add(
        log_file_path,
        filter=lambda record: record["extra"].get("job_id") == job_id,
        format="{time} | {level} | {message}",
        rotation="50 MB",
        retention="7 days"
    )
    return handler_id

def detach_logger(handler_id: str):
    logger.remove(handler_id)
```

---

### 3.4 Integration Boundaries

**Query Service → Common Module:**

- Import: Models, Settings, LLMService, VectorStoreFactory, Logging
- No HTTP calls (same process)

**Ingestion Service → Common Module:**

- Import: Models, Settings, LLMService, VectorStoreFactory, Logging
- No HTTP calls (same process)

**Query Service → Ingestion Service:**

- HTTP API only (for future features)
- No direct imports

**Both Services → Storage Backends:**

- Via LlamaIndex abstractions (VectorStore)
- Via SQLModel (Database)
- Via file system (Logs, Files)

---

### 3.5 Deployment Architecture

**3.5.1 Docker Compose Configurations**

**Option 1: All-in-One** (`docker-compose.yml`)

```yaml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: modcus
      POSTGRES_USER: modcus
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:7-alpine
  
  qdrant:
    image: qdrant/qdrant:latest
    volumes:
      - qdrant_data:/qdrant/storage
  
  ingest:
    build:
      context: .
      dockerfile: modcus_api_ingest/Dockerfile
    ports:
      - "8081:8081"
    depends_on:
      - postgres
      - redis
      - qdrant
    environment:
      - DB_URL=postgresql://modcus:${DB_PASSWORD}@postgres:5432/modcus
      - REDIS_URL=redis://redis:6379/0
      - VECTOR_STORE=qdrant
      - QDRANT_URL=http://qdrant:6333
  
  celery:
    build:
      context: .
      dockerfile: modcus_api_ingest/Dockerfile
    command: celery -A modcus_api_ingest.services.tasks worker --loglevel=INFO
    depends_on:
      - postgres
      - redis
      - qdrant
  
  query:
    build:
      context: .
      dockerfile: modcus_api_query/Dockerfile
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - qdrant
```

---

**Option 2: Per-Service** (separate files)

Each service has its own compose file with dependencies declared.

---

**Option 3: Dependencies Only** (`docker-compose.deps.yml`)

```yaml
services:
  postgres:
    image: postgres:15
    ports:
      - "5432:5432"
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
```

Services run natively: `uvicorn modcus_api_query.main:app --reload`

---

**Document Status:** ✅ Ready for Review  
**Next Action:** workflow:271192ed-bf0b-4f43-9915-d77b9e7dbb04/architecture-validation