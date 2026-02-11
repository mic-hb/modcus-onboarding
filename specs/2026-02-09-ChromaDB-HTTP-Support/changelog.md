# ChromaDB HTTP Client Support

**Date**: 2026-02-09
**Version**: v0.2.2
**Status**: Completed
**Priority**: Critical

## Summary

Added HTTP client support for ChromaDB vector store, enabling the Query and Ingest services to connect to a running ChromaDB server via HTTP API instead of using local file storage. This fixes the "Vector index unavailable" error when querying documents.

## Problem Statement

### The Issue
The Query service was failing to connect to the vector store with error:
```
Query Service: Could not load index from storage: 
[Errno 2] No such file or directory: '/app/modcus_api_query/vector_store/docstore.json'
```

When attempting to query documents:
```
POST /v1/query HTTP/1.1" 503 Service Unavailable
{"detail":"Vector index is currently unavailable. Please try again later."}
```

### Root Cause
1. **Mismatch in Client Types**: Documents were indexed into ChromaDB using an HTTP client (connecting to port 8000), but the Query service was trying to load from local persistent storage using `StorageContext.load_index_from_storage()`

2. **Missing Dependencies**: The Query service lacked the `llama-index-vector-stores-chroma` package required to connect to ChromaDB

3. **No HTTP Configuration**: The settings system only supported persistent client mode (local files), not HTTP server mode

4. **Hardcoded Logic**: The `_load_vector_index()` function in `main.py` was hardcoded to use `load_index_from_storage()` instead of connecting to ChromaDB

## Solution Overview

Implemented a flexible configuration system that supports both:
- **Persistent Client Mode**: Local file storage (original behavior)
- **HTTP Client Mode**: Connect to ChromaDB server via HTTP API (new)

## Changes Made

### 1. Settings Configuration (`modcus_common/settings/rag.py`)

Added three new configuration fields to `RAGSettingsMixin`:

```python
chromadb_host: str = Field(
    default="localhost",
    description="ChromaDB server host (from CHROMADB_HOST env var)",
)
chromadb_port: int = Field(
    default=8000,
    description="ChromaDB server port (from CHROMADB_PORT env var)",
)
chromadb_use_http: bool = Field(
    default=False,
    description="Use HTTP client instead of persistent client (from CHROMADB_USE_HTTP env var)",
)
```

**Impact**: Services can now choose between HTTP and persistent clients via environment variables.

### 2. ChromaDB Client Factory (`modcus_common/services/vector_store/chromadb_config.py`)

Updated `create_chromadb_client()` to support both modes:

```python
def create_chromadb_client(
    persist_dir: str, 
    host: str = "localhost", 
    port: int = 8000, 
    use_http: bool = False
):
    import chromadb

    if use_http:
        # Connect to ChromaDB server via HTTP
        return chromadb.HttpClient(host=host, port=port)
    else:
        # Use local file storage
        os.makedirs(persist_dir, exist_ok=True)
        return chromadb.PersistentClient(path=persist_dir)
```

**Impact**: Vector store factory can now create either type of client based on configuration.

### 3. VectorStoreFactory Update (`modcus_common/services/vector_store/vector_store_factory.py`)

Updated the ChromaDB branch to pass new configuration:

```python
elif vector_store_type == "chromadb":
    chromadb_persist_dir = getattr(
        settings, "chromadb_persist_dir", "/app/rag_storage/chroma_db"
    )
    chromadb_host = getattr(settings, "chromadb_host", "localhost")
    chromadb_port = getattr(settings, "chromadb_port", 8000)
    chromadb_use_http = getattr(settings, "chromadb_use_http", False)

    client = create_chromadb_client(
        persist_dir=chromadb_persist_dir,
        host=chromadb_host,
        port=chromadb_port,
        use_http=chromadb_use_http,
    )
    return ChromaVectorStore(
        chroma_collection=client.get_or_create_collection(collection)
    )
```

**Impact**: Factory now reads all ChromaDB settings and creates appropriate client.

### 4. Query Service Initialization (`modcus_api_query/main.py`)

Replaced the hardcoded `load_index_from_storage()` with proper ChromaDB connection:

**Before (Broken)**:
```python
async def _load_vector_index(settings: QuerySettings):
    from llama_index.core import StorageContext, load_index_from_storage
    storage_context = StorageContext.from_defaults(
        persist_dir=getattr(settings, 'vector_store_path', './vector_store')
    )
    index = load_index_from_storage(storage_context)
    return index
```

**After (Fixed)**:
```python
async def _load_vector_index(settings: QuerySettings):
    from modcus_common.services.vector_store.vector_store_factory import (
        VectorStoreFactory,
    )
    from llama_index.core import VectorStoreIndex

    # Create vector store connection to ChromaDB
    vector_store = VectorStoreFactory.create(
        settings=settings,
        collection="default",
    )

    # Create index from vector store
    index = VectorStoreIndex.from_vector_store(vector_store=vector_store)
    print("Query Service: Connected to ChromaDB")
    return index
```

**Impact**: Query service now properly connects to ChromaDB instead of looking for local files.

### 5. Dependencies (`modcus_api_query/pyproject.toml`)

Added required vector store packages:

```toml
dependencies = [
    # ... existing dependencies ...
    "llama-index-core>=0.14.0",
    "llama-index-vector-stores-chroma>=0.4.0",
    "llama-index-vector-stores-qdrant>=0.4.0",
    "llama-index-vector-stores-postgres>=0.4.0",
    # ...
]
```

**Impact**: Query service can now import and use vector store adapters.

### 6. Environment Variables

#### Updated `.env.example`:

```bash
# ChromaDB settings
# ChromaDB can operate in two modes:
# 1. HTTP Client Mode (recommended for Docker): Connects to a running ChromaDB server
# 2. Persistent Client Mode: Uses local file storage

# ChromaDB connection mode
CHROMADB_USE_HTTP=true

# ChromaDB HTTP connection settings (used when CHROMADB_USE_HTTP=true)
CHROMADB_HOST=host.docker.internal
CHROMADB_PORT=8000

# Legacy URL format (used by some components)
CHROMADB_URL=http://host.docker.internal:8000

# ChromaDB persistent storage settings (used when CHROMADB_USE_HTTP=false)
CHROMADB_PERSIST_DIR=./chroma_db
```

#### Docker Compose Configuration:

**docker-compose.query.yml**:
```yaml
environment:
  CHROMADB_URL: ${CHROMADB_URL:-http://host.docker.internal:8000}
  CHROMADB_HOST: ${CHROMADB_HOST:-host.docker.internal}
  CHROMADB_PORT: ${CHROMADB_PORT:-8000}
  CHROMADB_USE_HTTP: "true"
```

**docker-compose.ingest.yml**:
```yaml
environment:
  CHROMADB_URL: ${CHROMADB_URL:-http://host.docker.internal:8000}
  CHROMADB_HOST: ${CHROMADB_HOST:-host.docker.internal}
  CHROMADB_PORT: ${CHROMADB_PORT:-8000}
  CHROMADB_USE_HTTP: "true"
```

**Impact**: Consistent configuration across all services via environment variables.

## Configuration Guide

### For Docker Compose (Recommended)

1. **Copy `.env.example` to `.env`**:
   ```bash
   cp .env.example .env
   ```

2. **Ensure ChromaDB settings are**:
   ```bash
   CHROMADB_USE_HTTP=true
   CHROMADB_HOST=host.docker.internal
   CHROMADB_PORT=8000
   CHROMADB_URL=http://host.docker.internal:8000
   ```

3. **Start dependencies first**:
   ```bash
   docker compose -f docker-compose.deps.yml up -d
   docker compose -f docker-compose.vector.yml up -d
   ```

4. **Start services**:
   ```bash
   docker compose -f docker-compose.ingest.yml up -d
   docker compose -f docker-compose.query.yml up -d
   ```

### For Local Development (Native)

1. **Set environment variables**:
   ```bash
   export CHROMADB_USE_HTTP=true
   export CHROMADB_HOST=localhost
   export CHROMADB_PORT=8000
   ```

2. **Run services**:
   ```bash
   cd modcus_api_ingest
   uvicorn modcus_api_ingest.main:app --reload --port 8081
   
   cd modcus_api_query
   uvicorn modcus_api_query.main:app --reload --port 8080
   ```

### For Persistent Client Mode (File Storage)

If you prefer file-based storage instead of HTTP server:

```bash
CHROMADB_USE_HTTP=false
CHROMADB_PERSIST_DIR=/path/to/chroma_db
```

## Environment Variables Reference

| Variable               | Default                 | Description                |
| ---------------------- | ----------------------- | -------------------------- |
| `CHROMADB_USE_HTTP`    | `false`                 | Enable HTTP client mode    |
| `CHROMADB_HOST`        | `localhost`             | ChromaDB server hostname   |
| `CHROMADB_PORT`        | `8000`                  | ChromaDB server port       |
| `CHROMADB_URL`         | `http://localhost:8000` | Full ChromaDB URL (legacy) |
| `CHROMADB_PERSIST_DIR` | `./chroma_db`           | Local storage directory    |

## Testing

### Test 1: Verify ChromaDB Connection

```bash
# Check query service logs
docker compose -f docker-compose.query.yml logs modcus-api-query | grep -E "ChromaDB|Vector index"

# Expected output:
# Query Service: Connected to ChromaDB
# Query Service: Vector index loaded
```

### Test 2: Query Documents

```bash
# Send test query
curl -X POST http://localhost:8080/v1/query \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the financial highlights?",
    "mode": "company-analysis",
    "ticker": "GOTO_TEST",
    "level": "novice"
  }'
```

**Before Fix**: `503 Service Unavailable - Vector index unavailable`  
**After Fix**: Returns query response with citations

### Test 3: Verify Document Ingestion

```bash
# Ingest a document
curl -X POST http://localhost:8081/v1/rag/jobs \
  -H "X-API-Key: your-api-key" \
  -F "file=@document.pdf" \
  -F "ticker=TEST" \
  -F "doc_type=annual_report"

# Verify it appears in ChromaDB
curl http://localhost:8000/api/v1/collections
```

## Backward Compatibility

✅ **Fully backward compatible**

- Default behavior unchanged (`CHROMADB_USE_HTTP=false` uses persistent client)
- Existing configurations continue to work
- Migration path: Add `CHROMADB_USE_HTTP=true` to switch to HTTP mode

## Migration Guide

### For Existing Docker Compose Users

1. Update your `.env` file with new variables:
   ```bash
   echo "CHROMADB_USE_HTTP=true" >> .env
   echo "CHROMADB_HOST=host.docker.internal" >> .env
   echo "CHROMADB_PORT=8000" >> .env
   ```

2. Rebuild query service:
   ```bash
   docker compose -f docker-compose.query.yml up -d --build
   ```

### For New Users

Simply use the updated `.env.example` as your starting point - everything is pre-configured for Docker Compose.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Compose Network                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐         ┌──────────────────────────┐   │
│  │  Query Service  │◄────────│  ChromaDB (port 8000)    │   │
│  │  (port 8080)    │  HTTP   │  - Vector embeddings     │   │
│  └─────────────────┘         │  - Document chunks       │   │
│          │                   └──────────────────────────┘   │
│          │                            ▲                     │
│          │                            │                     │
│  ┌───────┴───────┐                    │                     │
│  │  Ingest       │────────────────────┘                     │
│  │  Service      │  HTTP (when CHROMADB_USE_HTTP=true)      │
│  │  (port 8081)  │                                          │
│  └───────────────┘                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Related Changes

- **Query Service**: Now properly initializes vector store from ChromaDB
- **Ingest Service**: Can use HTTP or persistent client (configurable)
- **Docker Compose**: Updated all compose files with proper env vars
- **Settings System**: Extended RAGSettingsMixin with ChromaDB connection options
- **Documentation**: Updated `.env.example` with comprehensive configuration guide

## Files Modified

1. `modcus_common/settings/rag.py` - Added ChromaDB HTTP settings
2. `modcus_common/services/vector_store/chromadb_config.py` - HTTP client support
3. `modcus_common/services/vector_store/vector_store_factory.py` - Pass HTTP config
4. `modcus_api_query/main.py` - Connect to ChromaDB instead of local storage
5. `modcus_api_query/pyproject.toml` - Added vector store dependencies
6. `.env.example` - Added new environment variables
7. `docker-compose.query.yml` - Added ChromaDB env vars
8. `docker-compose.ingest.yml` - Added ChromaDB env vars

## References

- ChromaDB Documentation: https://docs.trychroma.com/
- LlamaIndex Vector Stores: https://docs.llamaindex.ai/en/stable/module_guides/storing/vector_stores/
- ChromaDB HTTP Client: https://docs.trychroma.com/guides/rest
- Related Fix: Query service vector index initialization

## Checklist

- [x] Identify root cause (persistent vs HTTP client mismatch)
- [x] Add HTTP client configuration to settings
- [x] Update ChromaDB client factory
- [x] Fix Query service initialization
- [x] Add required dependencies
- [x] Update .env.example
- [x] Update Docker Compose configurations
- [x] Test query endpoint
- [x] Verify backward compatibility
- [x] Create comprehensive documentation
