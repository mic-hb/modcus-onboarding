# Architecture Validation Summary: Modcus v0.2 Implementation Guide

**Epic:** epic:7a9bea0a-9642-4cd3-b4d3-cbc00fd624f6  
**Related Specs:** 

- spec:7a9bea0a-9642-4cd3-b4d3-cbc00fd624f6/30980d13-feb1-4ff7-b94c-cffbb4f7c25f (Epic Brief)
- spec:7a9bea0a-9642-4cd3-b4d3-cbc00fd624f6/b9af845b-04dc-4f5a-a355-6265f5235c8f (Core Flows)
- spec:7a9bea0a-9642-4cd3-b4d3-cbc00fd624f6/6c854017-2ec5-4b3e-9f8a-b72aaad87f95 (Technical Plan)

**Validation Workflow:** workflow:271192ed-bf0b-4f43-9915-d77b9e7dbb04/architecture-validation  
**Status:** ✅ Validated  
**Date:** 2026-02-03

---

## Executive Summary

This document captures the outcomes of the architecture validation process for Modcus v0.2. All 8 critical architectural decisions from the Technical Plan have been stress-tested for **robustness**, **simplicity**, and **codebase fit**. Each decision has been validated against:

- Existing v0.1 patterns (file:AGENTS.md constraints)
- Real-world failure scenarios
- Integration complexity
- Performance implications
- Maintainability concerns

**Validation Outcome:** ✅ **All decisions approved with implementation guidance**

---

## 1. Validated Architectural Decisions

### Decision 1: Graph-Level Context Pattern for Agent Lifecycle

**✅ VALIDATED DECISION:** Agents instantiated once per request via `QueryGraphContext`. LangGraph nodes call agent methods with immutable state updates.

**Why This Matters:**

- Prevents memory leaks (agents properly scoped to request)
- Enables dependency injection (testable, mockable)
- Avoids race conditions (immutable state)
- Efficient (single instantiation per request)

**Implementation Pattern:**

```python
# modcus_api_query/services/orchestration/context.py
class QueryGraphContext:
    """Container for all agents and dependencies needed by the query graph."""
    
    def __init__(
        self,
        llm_service: LLMService,
        settings: QuerySettings,
        session: AsyncSession,
        index: VectorStoreIndex
    ):
        # Instantiate all agents once
        self.validation_agent = ValidationAgent(llm_service, settings)
        self.ticker_agent = TickerInferenceAgent(session)
        self.retrieval_agent = RetrievalAgent(index, settings, session)
        self.analysis_agent = AnalysisAgent(llm_service, get_all_tools(index))
        self.formatter_agent = ResponseFormatterAgent(settings)

# modcus_api_query/services/orchestration/query_graph.py
def create_query_graph(context: QueryGraphContext) -> StateGraph:
    """Create the query graph with agents from context."""
    
    graph = StateGraph(QueryState)
    
    # Define nodes that use context agents
    async def validation_node(state: QueryState) -> QueryState:
        result = await context.validation_agent.validate_and_classify(
            state["query"], state["mode"], state["level"]
        )
        # Immutable state update
        return {**state, "intent": result.intent, "complexity": result.complexity}
    
    async def ticker_inference_node(state: QueryState) -> QueryState:
        result = await context.ticker_agent.infer_tickers(
            state["query"], state.get("tickers")
        )
        return {**state, "inferred_tickers": result.tickers}
    
    # Add nodes
    graph.add_node("validate", validation_node)
    graph.add_node("infer_ticker", ticker_inference_node)
    # ... other nodes
    
    return graph.compile()

# modcus_api_query/api/routes_query.py
@router.post("/query")
async def query_endpoint(
    request: QueryRequest,
    session: AsyncSession = Depends(get_session),
    llm_service: LLMService = Depends(get_llm_service),
    settings: QuerySettings = Depends(get_settings),
    index: VectorStoreIndex = Depends(get_index)
):
    # Create context for this request
    context = QueryGraphContext(llm_service, settings, session, index)
    
    # Create and invoke graph
    graph = create_query_graph(context)
    initial_state = QueryState(query=request.query, mode=request.mode, ...)
    result = await graph.ainvoke(initial_state)
    
    return QueryResponse(answer=result["answer"], sources=result["sources"])
```

**Key Points:**

- ✅ One context per request (clean lifecycle)
- ✅ Agents reused across nodes (efficient)
- ✅ Immutable state updates (safe)
- ✅ Testable (mock context in tests)

---

### Decision 2: Hybrid Index Synchronization (Database-First + Reload)

**✅ VALIDATED DECISION:** Query service checks database for new documents before querying. Reloads LlamaIndex if stale.

**Why This Matters:**

- Prevents "document not found" errors after ingestion
- Avoids unnecessary index reloads (performance)
- Ensures data consistency between services

**Implementation Pattern:**

```python
# modcus_api_query/services/agents/retrieval_agent.py
class RetrievalAgent:
    def __init__(self, index: VectorStoreIndex, settings: RAGSettings, session: AsyncSession):
        self.index = index
        self.settings = settings
        self.session = session
        self.index_loaded_at = datetime.utcnow()
    
    async def retrieve_context(
        self,
        query: str,
        tickers: List[str],
        collection: str = "main_kb"
    ) -> RetrievalResult:
        # Step 1: Database-first check
        docs = await self.session.execute(
            select(Document)
            .where(Document.ticker.in_(tickers))
            .where(Document.status == "indexed")
        )
        docs = docs.scalars().all()
        
        if not docs:
            return RetrievalResult(chunks=[], sources=[], total_retrieved=0)
        
        # Step 2: Check if index is stale
        latest_doc_time = max(doc.indexed_at for doc in docs)
        if latest_doc_time > self.index_loaded_at:
            logger.info(f"Index stale, reloading (last load: {self.index_loaded_at}, latest doc: {latest_doc_time})")
            await self._reload_index(collection)
        
        # Step 3: Query index
        query_engine = self.index.as_query_engine(
            similarity_top_k=self.settings.retrieval_top_k,
            filters=MetadataFilters(
                filters=[MetadataFilter(key="ticker", value=ticker) for ticker in tickers]
            )
        )
        
        response = await query_engine.aquery(query)
        
        return RetrievalResult(
            chunks=response.source_nodes,
            sources=self._extract_sources(response.source_nodes),
            total_retrieved=len(response.source_nodes)
        )
    
    async def _reload_index(self, collection: str):
        """Reload index from vector store."""
        vector_store = VectorStoreFactory.create(self.settings, collection)
        self.index = VectorStoreIndex.from_vector_store(vector_store)
        self.index_loaded_at = datetime.utcnow()
        logger.info(f"Index reloaded at {self.index_loaded_at}")
```

**Ephemeral Artifact Management (Mode 3):**

```python
# modcus_api_query/services/chat/ephemeral_manager.py
class EphemeralArtifactManager:
    """Manages ephemeral artifacts for Mode 3 (Document Analysis)."""
    
    async def create_artifact(
        self,
        files: List[UploadFile],
        user_id: Optional[str],
        session_id: Optional[str]
    ) -> EphemeralArtifact:
        artifact_id = str(uuid.uuid4())
        collection_name = f"ephemeral_{artifact_id}"
        
        # Create artifact record
        artifact = EphemeralArtifact(
            id=artifact_id,
            user_id=user_id,
            session_id=session_id,
            vector_collection=collection_name,
            status="processing",
            ttl_hours=24,
            expires_at=datetime.utcnow() + timedelta(hours=24)
        )
        
        # Index documents into separate collection
        await self._index_documents(files, collection_name)
        
        artifact.status = "ready"
        return artifact
    
    async def cleanup_expired_artifacts(self):
        """Background job to clean up expired artifacts."""
        expired = await self.session.execute(
            select(EphemeralArtifact)
            .where(EphemeralArtifact.expires_at < datetime.utcnow())
            .where(EphemeralArtifact.status != "deleted")
        )
        
        for artifact in expired.scalars():
            # Delete vector collection
            await self._delete_collection(artifact.vector_collection)
            
            # Delete files
            for file_path in artifact.file_paths:
                os.remove(file_path)
            
            # Mark as deleted
            artifact.status = "deleted"
            artifact.deleted_at = datetime.utcnow()
```

**Key Points:**

- ✅ Database is source of truth
- ✅ Reload only when needed (performance)
- ✅ Query service manages ephemeral collections
- ✅ Automatic cleanup of expired artifacts

---

### Decision 3: Custom Agent Implementation with Stateless LLM Access

**✅ VALIDATED DECISION:** Custom agent loop with explicit tool orchestration. Agents call `LLMService` methods (not raw LLM instances). Each call gets fresh LLM.

**Why This Matters:**

- Full control over tool execution and logging
- Works with LLM rotation (stateless agents)
- Agent-level retry on rotation errors
- Cannot use LlamaIndex ReActAgent (LLM reference staleness)

**Implementation Pattern:**

```python
# modcus_api_query/services/agents/analysis_agent.py
class AnalysisAgent:
    """Custom agent with explicit tool orchestration and LLM rotation support."""
    
    def __init__(self, llm_service: LLMService, tools: List[BaseTool]):
        self.llm_service = llm_service  # Not a fixed LLM instance
        self.tools = {tool.metadata.name: tool for tool in tools}
        self.max_iterations = 5
        self.max_llm_retries = 3
    
    async def analyze(
        self,
        query: str,
        context: str,
        complexity: str,
        level: str,
        tickers: List[str]
    ) -> AnalysisResult:
        """Run analysis with tool calling loop."""
        
        # Build initial prompt based on complexity
        prompt = self._build_prompt(query, context, complexity, level, tickers)
        tool_call_history = []
        
        for iteration in range(self.max_iterations):
            # Get LLM response (stateless - fresh LLM each call)
            response = await self._call_llm_with_retry(prompt)
            
            # Parse for tool calls
            tool_calls = self._parse_tool_calls(response.text)
            
            if not tool_calls:
                # No more tools needed, return final answer
                return AnalysisResult(
                    answer=response.text,
                    tool_calls=tool_call_history,
                    reasoning=self._extract_reasoning(response.text)
                )
            
            # Execute tools
            tool_results = []
            for tool_call in tool_calls:
                result = await self._execute_tool(tool_call)
                tool_results.append(result)
                tool_call_history.append({
                    "tool": tool_call.name,
                    "args": tool_call.args,
                    "result": result.output
                })
            
            # Add tool results to prompt for next iteration
            prompt = self._add_tool_results(prompt, tool_results)
        
        # Max iterations reached
        return AnalysisResult(
            answer="Analysis incomplete (max iterations reached)",
            tool_calls=tool_call_history,
            reasoning="Exceeded maximum tool calling iterations"
        )
    
    async def _call_llm_with_retry(self, prompt: str) -> LLMResponse:
        """Call LLM with automatic retry on rotation."""
        for attempt in range(self.max_llm_retries):
            try:
                response = await self.llm_service.complete(
                    prompt=prompt,
                    context={"stage": "ANALYSIS", "attempt": attempt}
                )
                return response
            except RotationError as e:
                if attempt < self.max_llm_retries - 1:
                    logger.info(f"LLM rotated, retrying (attempt {attempt + 1}/{self.max_llm_retries})")
                    continue
                raise
        
        raise Exception("Max LLM retries exceeded")
    
    async def _execute_tool(self, tool_call: ToolCall) -> ToolResult:
        """Execute tool with logging and error handling."""
        tool = self.tools.get(tool_call.name)
        
        if not tool:
            return ToolResult(error=f"Tool {tool_call.name} not found")
        
        try:
            start_time = time.time()
            result = await tool.acall(**tool_call.args)
            latency_ms = int((time.time() - start_time) * 1000)
            
            # Log tool call
            await self._log_tool_call(tool_call, result, latency_ms, status="success")
            
            return ToolResult(output=result, error=None)
        
        except Exception as e:
            logger.error(f"Tool {tool_call.name} failed: {e}")
            await self._log_tool_call(tool_call, None, 0, status="failed", error=str(e))
            return ToolResult(output=None, error=str(e))
```

**LLM Service with No Embedding Rotation:**

```python
# modcus_common/services/llm/llm_service.py
class LLMService:
    """LLM service with rotation for completions, fixed model for embeddings."""
    
    def __init__(self, session_factory, settings: LLMSettings):
        self.rotator = LLMRotator(session_factory, settings)
        self.logger = LLMCallLogger(session_factory)
        self.settings = settings
        
        # Fixed embedding model (NO ROTATION)
        self.embedding_model = self._create_embedding_model()
    
    async def complete(self, prompt: str, context: Dict) -> LLMResponse:
        """Complete with rotation support."""
        llm = self.rotator.get_current_llm()  # Fresh LLM each call
        
        try:
            response = await llm.acomplete(prompt)
            await self.logger.log_call(context, response, status="success")
            return response
        
        except Exception as e:
            if self.rotator.should_rotate(e):
                self.rotator.rotate()
                raise RotationError("LLM rotated, please retry") from e
            else:
                await self.logger.log_call(context, None, status="failed", error=str(e))
                raise
    
    async def embed(self, texts: List[str]) -> List[List[float]]:
        """Generate embeddings with FIXED model (no rotation)."""
        # Embeddings must be consistent - no rotation
        return await self.embedding_model.aget_text_embedding_batch(texts)
    
    def _create_embedding_model(self):
        """Create fixed embedding model."""
        if self.settings.embedding_provider == "openai":
            return OpenAIEmbedding(
                model=self.settings.embedding_model,
                api_key=self.settings.openai_api_key
            )
        # ... other providers
```

**Key Points:**

- ✅ Stateless agents (fresh LLM per call)
- ✅ Agent-level retry on rotation
- ✅ Comprehensive tool call logging
- ✅ No embedding rotation (consistency)
- ❌ Cannot use ReActAgent (acceptable trade-off)

---

### Decision 4: Simplified Sliding Window for Conversation History

**✅ VALIDATED DECISION:** Defer full branching to v0.3. Use sliding window (keep last N turns + summary) for v0.2.

**Why This Matters:**

- Simpler implementation (no parent-child relationships)
- Avoids complex branching UX
- Still handles context window limits
- Can evolve to full branching in v0.3

**Implementation Pattern:**

```python
# modcus_api_query/services/chat/session_manager.py
class ChatSessionManager:
    """Manages conversation sessions with sliding window history."""
    
    def __init__(self, session: AsyncSession, settings: QuerySettings):
        self.session = session
        self.settings = settings
        self.context_window_limit = 8000  # tokens
        self.sliding_window_size = 5  # keep last 5 turns
    
    async def get_history_for_query(
        self,
        session_id: str
    ) -> List[ConversationTurn]:
        """Get conversation history with automatic summarization."""
        
        # Load all turns
        result = await self.session.execute(
            select(ConversationTurn)
            .where(ConversationTurn.session_id == session_id)
            .order_by(ConversationTurn.turn_number)
        )
        turns = result.scalars().all()
        
        if not turns:
            return []
        
        # Calculate total tokens
        total_tokens = sum(self._estimate_tokens(turn) for turn in turns)
        
        # If within limit, return full history
        if total_tokens < self.context_window_limit * 0.8:
            return turns
        
        # Sliding window: keep last N turns
        recent_turns = turns[-self.sliding_window_size:]
        
        # Summarize older turns
        old_turns = turns[:-self.sliding_window_size]
        if old_turns:
            summary_turn = await self._create_summary_turn(session_id, old_turns)
            return [summary_turn] + recent_turns
        
        return recent_turns
    
    async def _create_summary_turn(
        self,
        session_id: str,
        turns: List[ConversationTurn]
    ) -> ConversationTurn:
        """Create a summary of older turns."""
        
        # Build summary prompt
        conversation_text = "\n\n".join([
            f"User: {turn.query}\nAssistant: {turn.answer}"
            for turn in turns
        ])
        
        summary_prompt = f"""Summarize the following conversation concisely:

{conversation_text}

Provide a brief summary (2-3 sentences) capturing the key topics discussed."""
        
        # Generate summary via LLM
        llm_service = get_llm_service()  # Dependency injection
        response = await llm_service.complete(summary_prompt, context={"stage": "SUMMARIZATION"})
        
        # Create pseudo-turn for summary
        summary_turn = ConversationTurn(
            id=f"summary_{session_id}_{turns[0].turn_number}_{turns[-1].turn_number}",
            session_id=session_id,
            turn_number=0,  # Special turn number for summaries
            query="[CONVERSATION SUMMARY]",
            answer=response.text,
            mode="summary",
            level="",
            complexity_level="",
            sources=[],
            metadata_json={"summarized_turns": [t.turn_number for t in turns]},
            latency_ms=0,
            llm_call_ids=[]
        )
        
        return summary_turn
    
    def _estimate_tokens(self, turn: ConversationTurn) -> int:
        """Estimate token count for a turn."""
        # Rough estimate: 1 token ≈ 4 characters
        return (len(turn.query) + len(turn.answer)) // 4
```

**Database Schema (No Branching Fields):**

```python
# modcus_common/models/conversation.py
class ConversationSession(SQLModel, table=True):
    __tablename__ = "conversation_session"
    
    id: str = Field(primary_key=True)
    user_id: Optional[str] = Field(index=True)
    status: str = Field(index=True)  # active, expired, closed
    turn_count: int
    last_activity_at: datetime
    expires_at: datetime = Field(index=True)
    created_at: datetime
    
    # NO branching fields (deferred to v0.3):
    # parent_session_id, is_summary, summarized_from_turn
```

**Key Points:**

- ✅ Simple sliding window (last N turns)
- ✅ Automatic summarization when needed
- ✅ No complex branching (deferred to v0.3)
- ✅ Transparent to user (no UX complexity)

---

### Decision 5: Staged Retry with Intermediate Results

**✅ VALIDATED DECISION:** Resume from failure point. Save intermediate results per stage.

**Why This Matters:**

- No wasted work (don't re-parse already-parsed documents)
- Clear stage tracking (know exactly where failure occurred)
- User can retry without re-doing successful stages

**Implementation Pattern:**

```python
# modcus_common/models/job.py
class IngestionStage(str, Enum):
    PARSE = "parse"
    CHUNK = "chunk"
    EMBED = "embed"
    INDEX = "index"

class Job(SQLModel, table=True):
    # ... existing fields
    
    # Stage tracking
    current_stage: str = Field(default=IngestionStage.PARSE)
    completed_stages: List[str] = Field(default=[], sa_column=Column(JSON))
    stage_artifacts: Dict = Field(default={}, sa_column=Column(JSON))
    
    # Retry configuration
    retry_strategy: str = Field(default="transient_only")  # none, transient_only, all_errors
    max_retries: int = Field(default=3)
    attempts: int = Field(default=0)

# modcus_api_ingest/services/tasks/ingestion_tasks.py
@celery_app.task(bind=True)
def process_ingestion_job(self, job_id: str):
    """Process ingestion job with staged retry."""
    
    with SessionLocal() as session:
        job = session.get(Job, job_id)
        
        # Attach per-job logger
        logger_id = attach_job_logger(job_id, job.log_file_path)
        
        try:
            job.status = "running"
            job.attempts += 1
            session.commit()
            
            # Stage 1: Parse
            if IngestionStage.PARSE not in job.completed_stages:
                logger.info("Stage 1: Parsing document")
                parsed = parse_document(job.raw_file_path, job.parser)
                
                # Save intermediate result
                job.stage_artifacts["parsed"] = {
                    "text": parsed.text,
                    "metadata": parsed.metadata,
                    "tables_count": len(parsed.tables)
                }
                job.completed_stages.append(IngestionStage.PARSE)
                job.current_stage = IngestionStage.CHUNK
                job.progress = 25
                session.commit()
            else:
                logger.info("Stage 1: Skipping (already completed)")
                parsed = deserialize_parsed_document(job.stage_artifacts["parsed"])
            
            # Stage 2: Chunk
            if IngestionStage.CHUNK not in job.completed_stages:
                logger.info("Stage 2: Chunking document")
                chunks = chunk_document(parsed, job.doc_type, job.chunk_size)
                
                job.stage_artifacts["chunks"] = {
                    "count": len(chunks),
                    "chunk_ids": [chunk.id_ for chunk in chunks]
                }
                job.completed_stages.append(IngestionStage.CHUNK)
                job.current_stage = IngestionStage.EMBED
                job.progress = 50
                session.commit()
            else:
                logger.info("Stage 2: Skipping (already completed)")
                chunks = load_chunks_from_artifacts(job.stage_artifacts["chunks"])
            
            # Stage 3: Embed
            if IngestionStage.EMBED not in job.completed_stages:
                logger.info("Stage 3: Generating embeddings")
                embeddings = await generate_embeddings(chunks, job.embedding_model)
                
                job.stage_artifacts["embeddings"] = {
                    "model": job.embedding_model,
                    "count": len(embeddings)
                }
                job.completed_stages.append(IngestionStage.EMBED)
                job.current_stage = IngestionStage.INDEX
                job.progress = 75
                session.commit()
            else:
                logger.info("Stage 3: Skipping (already completed)")
                embeddings = load_embeddings_from_artifacts(job.stage_artifacts["embeddings"])
            
            # Stage 4: Index
            if IngestionStage.INDEX not in job.completed_stages:
                logger.info("Stage 4: Indexing into vector store")
                index_result = await index_document(chunks, embeddings, job_id)
                
                job.completed_stages.append(IngestionStage.INDEX)
                job.status = "completed"
                job.progress = 100
                session.commit()
            
            # Create Document record
            create_document_record(session, job, index_result)
            
        except Exception as e:
            # Classify error
            error_type = classify_error(e)
            job.error_code = error_type.code
            job.error_message = str(e)
            job.retryable = error_type.is_transient
            
            # Determine if should retry
            should_retry = (
                job.retry_strategy == "all_errors" or
                (job.retry_strategy == "transient_only" and error_type.is_transient)
            ) and job.attempts < job.max_retries
            
            if should_retry:
                job.status = "retrying"
                logger.info(f"Retrying job (attempt {job.attempts}/{job.max_retries})")
                # Celery will auto-retry
            else:
                job.status = "failed"
                logger.error(f"Job failed: {e}")
            
            session.commit()
            
            if not should_retry:
                raise  # Don't retry
        
        finally:
            detach_logger(logger_id)

def classify_error(e: Exception) -> ErrorType:
    """Classify error as transient or permanent."""
    
    # Transient errors (should retry)
    if isinstance(e, (RateLimitError, NetworkError, TimeoutError)):
        return ErrorType(code="TRANSIENT", is_transient=True)
    
    # LLM provider errors
    if "rate limit" in str(e).lower():
        return ErrorType(code="RATE_LIMIT", is_transient=True)
    
    if "timeout" in str(e).lower():
        return ErrorType(code="TIMEOUT", is_transient=True)
    
    # Permanent errors (don't retry)
    if isinstance(e, (ValueError, FileNotFoundError)):
        return ErrorType(code="PERMANENT", is_transient=False)
    
    if "password" in str(e).lower() or "encrypted" in str(e).lower():
        return ErrorType(code="PROTECTED_FILE", is_transient=False)
    
    # Default: treat as permanent
    return ErrorType(code="UNKNOWN", is_transient=False)
```

**Key Points:**

- ✅ Resume from failure point (efficient)
- ✅ Clear stage tracking
- ✅ Intermediate results preserved
- ✅ Configurable retry strategy
- ✅ Error classification (transient vs permanent)

---

### Decision 6: Context-Based Session Management

**✅ VALIDATED DECISION:** Use async context managers for session lifecycle. One session per request/task.

**Why This Matters:**

- Prevents connection leaks
- Automatic commit/rollback
- Clear transaction boundaries
- Works with async LangGraph and sync Celery

**Implementation Pattern:**

```python
# modcus_common/db/session.py
from contextlib import asynccontextmanager
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

# Async engine for Query service
async_engine = create_async_engine(
    settings.db_url.replace("postgresql://", "postgresql+asyncpg://"),
    pool_size=10,
    max_overflow=20,
    echo=False
)

AsyncSessionLocal = async_sessionmaker(
    async_engine,
    class_=AsyncSession,
    expire_on_commit=False
)

@asynccontextmanager
async def get_async_session():
    """Async session context manager."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        finally:
            await session.close()

# Sync engine for Ingestion service (Celery)
sync_engine = create_engine(
    settings.db_url,
    pool_size=5,
    max_overflow=10,
    echo=False
)

SessionLocal = sessionmaker(sync_engine, expire_on_commit=False)

@contextmanager
def get_sync_session():
    """Sync session context manager."""
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# FastAPI dependency for Query service
async def get_session() -> AsyncSession:
    """FastAPI dependency for async sessions."""
    async with get_async_session() as session:
        yield session
```

**Usage in LangGraph:**

```python
# modcus_api_query/api/routes_query.py
@router.post("/query")
async def query_endpoint(
    request: QueryRequest,
    session: AsyncSession = Depends(get_session)  # Injected by FastAPI
):
    # Session is managed by FastAPI dependency
    # Automatically committed/rolled back
    
    # Create context with session
    context = QueryGraphContext(
        llm_service=get_llm_service(),
        settings=get_settings(),
        session=session,  # Pass session to context
        index=get_index()
    )
    
    # Invoke graph
    graph = create_query_graph(context)
    result = await graph.ainvoke(initial_state)
    
    # Session auto-commits on success, rolls back on error
    return QueryResponse(...)
```

**Usage in Celery:**

```python
# modcus_api_ingest/services/tasks/ingestion_tasks.py
@celery_app.task
def process_ingestion_job(job_id: str):
    # Use sync session context manager
    with get_sync_session() as session:
        job = session.get(Job, job_id)
        
        # ... process job
        
        # Session auto-commits on success
```

**Key Points:**

- ✅ Async sessions for Query service
- ✅ Sync sessions for Celery tasks
- ✅ Automatic commit/rollback
- ✅ No connection leaks
- ✅ Works with FastAPI dependency injection

---

### Decision 7: Hybrid Configuration (YAML + Env Vars)

**✅ VALIDATED DECISION:** Use YAML files for structured config, environment variables for secrets. Env vars override YAML.

**Why This Matters:**

- Avoids 50-100 environment variables
- Structured config is readable (YAML)
- Secrets stay in env vars (secure)
- Deployment flexibility (override via env)

**Implementation Pattern:**

```yaml
# config.yaml (checked into git)
rag:
  vector_store: chromadb  # Default for dev
  chunk_sizes:
    annual_report: 1024
    financial_report: 1024
    news: 2048
    public_expose: 2048
  chunk_overlap: 200
  retrieval:
    top_k: 10
    enable_reranking: false

llm:
  provider: gemini
  model: gemini-1.5-pro
  embedding:
    provider: openai
    model: text-embedding-3-small
  rotation:
    enabled: true
    max_retries: 3

query:
  timeout_seconds: 360
  complexity_inference_mode: combined
  conversation:
    timeout_minutes: 120
    sliding_window_size: 5
  ephemeral_artifacts:
    ttl_hours: 24

ingestion:
  celery:
    concurrency: 2
  max_queued: 10
  retry:
    default_strategy: transient_only
    default_max_retries: 3

logging:
  level: INFO
  rotation: "50 MB"
  retention: "7 days"

metrics:
  enabled: true
  port: 9090
```

```bash
# .env (NOT checked into git, secrets only)
DB_PASSWORD=secret123
REDIS_PASSWORD=secret456

# LLM API keys
GEMINI_API_KEY=secret789
OPENAI_API_KEY=secret012
ANTHROPIC_API_KEY=secret345

# Vector store credentials
QDRANT_API_KEY=secret678
```

```python
# modcus_common/settings/base.py
from pydantic import BaseSettings
from pydantic_settings import SettingsConfigDict
import yaml
from pathlib import Path

class BaseSettings(BaseSettings):
    """Base settings with YAML + env var support."""
    
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore"
    )
    
    # Database (from env vars)
    db_url: str = Field(..., env="DB_URL")
    db_password: str = Field(..., env="DB_PASSWORD")
    
    # Redis (from env vars)
    redis_url: str = Field(default="redis://localhost:6379/0", env="REDIS_URL")
    
    # Logging (from YAML)
    log_level: str = "INFO"
    log_rotation: str = "50 MB"
    log_retention: str = "7 days"
    
    @classmethod
    def from_yaml(cls, yaml_path: str = "config.yaml"):
        """Load settings from YAML file, override with env vars."""
        
        # Load YAML
        config_file = Path(yaml_path)
        if config_file.exists():
            with open(config_file) as f:
                yaml_config = yaml.safe_load(f)
        else:
            yaml_config = {}
        
        # Flatten nested YAML for Pydantic
        flat_config = cls._flatten_yaml(yaml_config)
        
        # Create settings (env vars override YAML)
        return cls(**flat_config)
    
    @staticmethod
    def _flatten_yaml(d: dict, parent_key: str = "", sep: str = "_") -> dict:
        """Flatten nested YAML dict."""
        items = []
        for k, v in d.items():
            new_key = f"{parent_key}{sep}{k}" if parent_key else k
            if isinstance(v, dict):
                items.extend(BaseSettings._flatten_yaml(v, new_key, sep=sep).items())
            else:
                items.append((new_key, v))
        return dict(items)

# modcus_common/settings/llm.py
class LLMSettings(BaseSettings):
    """LLM-specific settings."""
    
    # From YAML
    llm_provider: str = "gemini"
    llm_model: str = "gemini-1.5-pro"
    embedding_provider: str = "openai"
    embedding_model: str = "text-embedding-3-small"
    llm_rotation_enabled: bool = True
    llm_rotation_max_retries: int = 3
    
    # From env vars (secrets)
    gemini_api_key: str = Field(..., env="GEMINI_API_KEY")
    openai_api_key: str = Field(..., env="OPENAI_API_KEY")
    anthropic_api_key: Optional[str] = Field(None, env="ANTHROPIC_API_KEY")

# modcus_common/settings/rag.py
class RAGSettings(BaseSettings):
    """RAG-specific settings."""
    
    # From YAML
    vector_store: str = "chromadb"
    chunk_sizes: Dict[str, int] = Field(default_factory=lambda: {
        "annual_report": 1024,
        "financial_report": 1024,
        "news": 2048
    })
    chunk_overlap: int = 200
    retrieval_top_k: int = 10
    retrieval_enable_reranking: bool = False
    
    # From env vars (if using external vector stores)
    qdrant_url: str = Field(default="http://localhost:6333", env="QDRANT_URL")
    qdrant_api_key: Optional[str] = Field(None, env="QDRANT_API_KEY")
    chromadb_persist_dir: str = Field(default="./chroma_db", env="CHROMADB_PERSIST_DIR")

# modcus_api_query/settings.py
class QuerySettings(BaseSettings, LLMSettings, RAGSettings):
    """Query service settings (inherits from base + LLM + RAG)."""
    
    # From YAML
    query_timeout_seconds: int = 360
    complexity_inference_mode: str = "combined"
    conversation_timeout_minutes: int = 120
    conversation_sliding_window_size: int = 5
    ephemeral_artifacts_ttl_hours: int = 24
    
    @classmethod
    def load(cls):
        """Load settings from YAML + env vars."""
        return cls.from_yaml("config.yaml")
```

**Key Points:**

- ✅ YAML for structured config (readable, version-controlled)
- ✅ Env vars for secrets (secure, not in git)
- ✅ Env vars override YAML (deployment flexibility)
- ✅ Hierarchical settings (base → specialized)
- ✅ Type-safe with Pydantic validation

---

### Decision 8: Index Reload Trigger & Ephemeral Collection Management

**✅ VALIDATED DECISION:** (Covered in Decision 2 above)

This decision is integrated into the Hybrid Index Synchronization pattern.

---

## 2. Database Schema Updates

Based on validated decisions, the following schema changes are required:

### Updated Job Model (Staged Retry)

```python
class Job(SQLModel, table=True):
    # ... existing fields
    
    # Stage tracking (NEW)
    current_stage: str = Field(default="parse")
    completed_stages: List[str] = Field(default=[], sa_column=Column(JSON))
    stage_artifacts: Dict = Field(default={}, sa_column=Column(JSON))
    
    # Retry configuration (NEW)
    retry_strategy: str = Field(default="transient_only")
    max_retries: int = Field(default=3)
```

### Simplified ConversationSession (No Branching)

```python
class ConversationSession(SQLModel, table=True):
    id: str = Field(primary_key=True)
    user_id: Optional[str] = Field(index=True)
    status: str = Field(index=True)
    turn_count: int
    last_activity_at: datetime
    expires_at: datetime = Field(index=True)
    created_at: datetime
    
    # NO branching fields (deferred to v0.3)
```

### New ToolCall Model (Tool Execution Tracking)

```python
class ToolCall(SQLModel, table=True):
    """Track tool executions for observability."""
    __tablename__ = "tool_call"
    
    id: str = Field(primary_key=True)
    conversation_id: Optional[str] = Field(index=True)
    job_id: Optional[str] = Field(index=True)
    
    tool_name: str = Field(index=True)
    tool_args: Dict = Field(sa_column=Column(JSON))
    tool_output: Optional[str]
    
    status: str = Field(index=True)  # success, failed
    error_message: Optional[str]
    latency_ms: int
    
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

---

## 3. Configuration Files Structure

```
modcus_core/
├── config.yaml                    # Main config (checked into git)
├── config.production.yaml         # Production overrides
├── config.development.yaml        # Development overrides
├── .env                           # Secrets (NOT in git)
├── .env.example                   # Template for .env
│
├── modcus_common/
│   ├── settings/
│   │   ├── base.py               # BaseSettings with YAML loading
│   │   ├── llm.py                # LLMSettings
│   │   ├── rag.py                # RAGSettings
│   │   └── __init__.py
│
├── modcus_api_query/
│   ├── settings.py               # QuerySettings (inherits base + LLM + RAG)
│   └── ...
│
└── modcus_api_ingest/
    ├── settings.py               # IngestSettings (inherits base + LLM + RAG)
    └── ...
```

---

## 4. Integration Checklist

Before proceeding to ticket breakdown, ensure these integration points are understood:

### ✅ LangGraph Integration

- [ ] QueryGraphContext pattern implemented
- [ ] Immutable state updates in all nodes
- [ ] Agents instantiated once per request
- [ ] Session passed via context

### ✅ LLM Rotation Integration

- [ ] LLMService with stateless agent support
- [ ] RotationError exception defined
- [ ] Agent-level retry logic
- [ ] No embedding rotation (fixed model)

### ✅ Index Synchronization

- [ ] Database-first check in RetrievalAgent
- [ ] Index reload logic when stale
- [ ] Ephemeral collection management
- [ ] Cleanup job for expired artifacts

### ✅ Conversation History

- [ ] Sliding window implementation
- [ ] Summarization via LLM
- [ ] No branching (deferred to v0.3)
- [ ] Token estimation logic

### ✅ Staged Retry

- [ ] Stage tracking in Job model
- [ ] Intermediate result serialization
- [ ] Error classification logic
- [ ] Resume from failure point

### ✅ Custom Agent

- [ ] Tool orchestration loop
- [ ] Tool call logging
- [ ] LLM retry on rotation
- [ ] Max iterations limit

### ✅ Session Management

- [ ] Async context managers
- [ ] Sync context managers for Celery
- [ ] FastAPI dependency injection
- [ ] Connection pool configuration

### ✅ Configuration

- [ ] YAML config loading
- [ ] Env var override
- [ ] Settings hierarchy
- [ ] Secrets in .env only

---

## 5. Next Steps

**Architecture validation is complete.** The Technical Plan is now validated and ready for implementation.

**Recommended Next Steps:**

1. **Ticket Breakdown** (workflow:271192ed-bf0b-4f43-9915-d77b9e7dbb04/ticket-breakdown)
  - Break down specs into coarse, actionable tickets
  - Organize by service (Ingestion, Query, Common)
  - Prioritize by dependency order
2. **File Reorganization** (before implementation)
  - Move v0.1 code to `legacy/v0.1/`
  - Move v0.0 legacy to `legacy/v0.0/`
  - Commit file moves first
3. **Implementation** (via Phases or Plan mode)
  - Start with Common module (database, settings, LLM service)
  - Then Ingestion service
  - Then Query service
  - Integration testing

---

**Document Status:** ✅ Validated & Complete  
**Next Action:** workflow:271192ed-bf0b-4f43-9915-d77b9e7dbb04/ticket-breakdown