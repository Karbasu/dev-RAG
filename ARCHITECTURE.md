# RAG System Architecture

## System Overview

This is a production-grade Retrieval Augmented Generation (RAG) system designed for multi-format document ingestion, intelligent chunking, semantic search, and context-aware LLM responses.

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND LAYER                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   Chat UI    │  │   Upload UI  │  │  Search UI   │  │   Admin UI   │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                 │                 │                 │            │
│         └─────────────────┴─────────────────┴─────────────────┘            │
│                                   │                                         │
│                                   ▼                                         │
│                        ┌──────────────────────┐                            │
│                        │   API Gateway (NGINX) │                            │
│                        └──────────┬───────────┘                            │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                        BACKEND API LAYER (NestJS)                           │
│                        ┌──────────┴───────────┐                            │
│                        │   Main API Service    │                            │
│  ┌─────────────────────┼───────────────────────┼─────────────────────────┐ │
│  │                     │                       │                         │ │
│  ▼                     ▼                       ▼                         ▼ │
│ ┌────────┐      ┌────────────┐        ┌─────────────┐        ┌──────────┐ │
│ │ Ingest │      │   Search   │        │    Chat     │        │  Memory  │ │
│ │Controller     │ Controller │        │ Controller  │        │Controller│ │
│ └───┬────┘      └─────┬──────┘        └──────┬──────┘        └────┬─────┘ │
│     │                 │                      │                    │       │
└─────┼─────────────────┼──────────────────────┼────────────────────┼───────┘
      │                 │                      │                    │
      │                 │                      │                    │
┌─────┼─────────────────┼──────────────────────┼────────────────────┼───────┐
│     │        PROCESSING & ORCHESTRATION LAYER                     │       │
│     ▼                 │                      ▼                    │       │
│ ┌─────────────────┐   │              ┌──────────────────┐         │       │
│ │ Ingestion Queue │   │              │ LLM Orchestrator │         │       │
│ │   (Bull/Redis)  │   │              │    Service       │         │       │
│ └────────┬────────┘   │              └────────┬─────────┘         │       │
│          │            │                       │                   │       │
│          ▼            │                       │                   │       │
│ ┌──────────────────┐  │              ┌────────▼─────────┐         │       │
│ │  Ingestion       │  │              │  Prompt Builder  │         │       │
│ │  Worker Service  │  │              │   + Templates    │         │       │
│ └────────┬─────────┘  │              └──────────────────┘         │       │
│          │            │                                            │       │
│          ▼            │                                            │       │
│ ┌──────────────────┐  │                                            │       │
│ │  Format Parser   │  │                                            │       │
│ │  - PDF Parser    │  │                                            │       │
│ │  - PPTX Parser   │  │                                            │       │
│ │  - URL Scraper   │  │                                            │       │
│ │  - OCR Engine    │  │                                            │       │
│ └────────┬─────────┘  │                                            │       │
│          │            │                                            │       │
│          ▼            │                                            │       │
│ ┌──────────────────┐  │                                            │       │
│ │ Chunking Service │  │                                            │       │
│ │  - Recursive     │  │                                            │       │
│ │  - Semantic      │  │                                            │       │
│ │  - HTML-aware    │  │                                            │       │
│ │  - Code-aware    │  │                                            │       │
│ └────────┬─────────┘  │                                            │       │
│          │            │                                            │       │
│          ▼            │                                            │       │
│ ┌──────────────────┐  │                                            │       │
│ │ Embedding Worker │  │                                            │       │
│ │  - Batch Queue   │  │                                            │       │
│ │  - Retry Logic   │  │                                            │       │
│ │  - Model Version │  │                                            │       │
│ └────────┬─────────┘  │                                            │       │
│          │            │                                            │       │
└──────────┼────────────┼────────────────────────────────────────────┼───────┘
           │            │                                            │
           ▼            ▼                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STORAGE LAYER                                     │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│  │  Vector Database │  │   PostgreSQL     │  │   Redis Cache    │         │
│  │   (Qdrant)       │  │   - Documents    │  │   - Sessions     │         │
│  │  - Embeddings    │  │   - Chunks       │  │   - Search Cache │         │
│  │  - Collections   │  │   - Users        │  │   - Rate Limits  │         │
│  │  - Metadata      │  │   - Logs         │  │                  │         │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘         │
│                                                                             │
│  ┌──────────────────┐  ┌──────────────────┐                               │
│  │   S3/MinIO       │  │   Elasticsearch  │                               │
│  │   - Raw Files    │  │   - Full-text    │                               │
│  │   - Processed    │  │   - Logs         │                               │
│  └──────────────────┘  └──────────────────┘                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                     EXTERNAL SERVICES                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│  │   OpenAI API     │  │   Anthropic      │  │   Monitoring     │         │
│  │   - Embeddings   │  │   - Claude       │  │   - Prometheus   │         │
│  │   - GPT-4        │  │                  │  │   - Grafana      │         │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow: Raw File → Chunk → Vector → Search → Answer

### Flow 1: Document Ingestion Pipeline

```
┌──────────────┐
│  User Upload │
│  (PDF/PPTX)  │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 1. INGESTION SERVICE                                     │
│    - Validate file type & size                           │
│    - Generate document UUID                              │
│    - Store raw file in S3/MinIO                          │
│    - Create document record in PostgreSQL                │
│    - Enqueue job to processing queue                     │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 2. FORMAT PARSING (Worker picks job from queue)         │
│    - PDF: Extract text with layout info (pdf-parse)     │
│           Extract tables (tabula-py fallback)           │
│           Detect headings via font size/bold            │
│    - PPTX: Extract slide text, notes, shapes            │
│            Preserve slide order & structure             │
│    - OCR: Tesseract for images (optional)               │
│    Output: Structured text with metadata                │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 3. CHUNKING & PREPROCESSING                             │
│    - Choose chunking strategy based on content type     │
│      * Recursive: For general text (500-1000 tokens)    │
│      * Semantic: Group by topic similarity              │
│      * HTML-aware: Preserve structure                   │
│      * Code-aware: Respect function boundaries          │
│    - Add metadata to each chunk:                        │
│      * document_id, chunk_index, parent_heading         │
│      * char_count, token_count, created_at              │
│    - Deduplicate similar chunks (cosine > 0.95)         │
│    - Quality scoring (length, coherence, info density)  │
│    - Store chunks in PostgreSQL                         │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 4. EMBEDDING GENERATION                                  │
│    - Batch chunks (32-64 per batch)                     │
│    - Generate embeddings via OpenAI/Cohere API          │
│      * Model: text-embedding-3-large (3072 dims)        │
│    - Retry with exponential backoff (3 attempts)        │
│    - Store embeddings in Qdrant vector DB               │
│      * Collection per tenant/project                    │
│      * Metadata: chunk_id, doc_id, timestamp            │
│    - Link: chunk_id ↔ vector_id (stored in PostgreSQL)  │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 5. INDEXING COMPLETE                                     │
│    - Update document status: "indexed"                   │
│    - Trigger webhook/notification (optional)             │
│    - Log to Elasticsearch for analytics                  │
└──────────────────────────────────────────────────────────┘
```

### Flow 2: Query → Retrieval → Answer Generation

```
┌──────────────┐
│  User Query  │
│ "What is...?"│
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 1. QUERY PROCESSING                                      │
│    - Check cache (Redis) for identical query            │
│    - Extract metadata filters from query                │
│      * Date range, document type, tags                  │
│    - Detect query intent (factual/summarize/compare)    │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 2. HYBRID SEARCH                                         │
│    A. Vector Search (Semantic)                           │
│       - Embed query using same model                     │
│       - Search Qdrant with cosine similarity             │
│       - Get top-k=50 candidates                          │
│                                                          │
│    B. Keyword Search (Lexical)                           │
│       - BM25 search on Elasticsearch                     │
│       - Get top-k=30 candidates                          │
│                                                          │
│    C. Fusion (RRF - Reciprocal Rank Fusion)             │
│       - Merge results with weighted scores:             │
│         score = 0.7 * vec_score + 0.3 * bm25_score      │
│       - Apply metadata filters                          │
│       - Boost recent documents (+10%)                   │
│       - Boost headings/titles (+15%)                    │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 3. RE-RANKING                                            │
│    - Take top-20 chunks from fusion                     │
│    - Use cross-encoder model for fine-grained scoring   │
│      * Model: ms-marco-MiniLM-L-12-v2                   │
│    - Re-score query-chunk relevance                     │
│    - Select top-5 chunks for context                    │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 4. CONTEXT ASSEMBLY                                      │
│    - Retrieve full chunk text from PostgreSQL           │
│    - Fetch surrounding chunks (±1) for continuity       │
│    - Load user memory vectors (past conversations)      │
│    - Arrange chunks by relevance score                  │
│    - Check total token count < context limit            │
│      * GPT-4: 8K context                                │
│      * Claude: 100K context                             │
│    - Truncate/summarize if needed                       │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 5. PROMPT CONSTRUCTION                                   │
│    Template:                                             │
│    """                                                   │
│    You are a helpful assistant. Answer based ONLY on    │
│    the provided context. Cite sources using [1], [2].   │
│                                                          │
│    Context:                                              │
│    [1] {chunk_1_text} (Source: {doc_title}, p.{page})   │
│    [2] {chunk_2_text} (Source: {doc_title}, p.{page})   │
│    ...                                                   │
│                                                          │
│    User Memory:                                          │
│    - You previously discussed {topic} with user         │
│                                                          │
│    Question: {user_query}                               │
│                                                          │
│    Answer (cite sources, be concise, avoid speculation):│
│    """                                                   │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 6. LLM GENERATION                                        │
│    - Send prompt to LLM (OpenAI GPT-4 / Claude)         │
│    - Stream response to user                            │
│    - Parse citations from response                      │
└──────┬───────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ 7. POST-PROCESSING                                       │
│    - Store conversation in session memory               │
│    - Generate memory embedding for long-term storage    │
│    - Log query, chunks used, response (analytics)       │
│    - Cache result in Redis (TTL: 1 hour)                │
└──────────────────────────────────────────────────────────┘
```

## Component Breakdown

### 1. Ingestion Service
- **Tech Stack**: NestJS + Bull Queue + Redis
- **Responsibilities**:
  - File validation & virus scanning (ClamAV)
  - Multi-format parsing orchestration
  - Progress tracking & webhooks
  - Error handling & retry logic

### 2. Chunking & Preprocessing Service
- **Tech Stack**: Python (spaCy, LangChain)
- **Responsibilities**:
  - Adaptive chunking strategies
  - Metadata enrichment
  - Quality scoring
  - Deduplication

### 3. Embedding Worker Service
- **Tech Stack**: Python + Celery + RabbitMQ
- **Responsibilities**:
  - Batch processing
  - API rate limiting
  - Cost tracking
  - Model version management

### 4. Vector Database (Qdrant)
- **Why Qdrant**:
  - High performance (HNSW + disk-backed)
  - Rich filtering capabilities
  - Horizontal scaling
  - REST + gRPC APIs
- **Alternatives**: Milvus (more complex), LanceDB (embedded), Weaviate (GraphQL)

### 5. Retrieval API
- **Tech Stack**: NestJS + Qdrant Client + Elasticsearch
- **Responsibilities**:
  - Hybrid search orchestration
  - Re-ranking
  - Caching
  - Filter management

### 6. LLM Orchestration Layer
- **Tech Stack**: NestJS + LangChain/LlamaIndex
- **Responsibilities**:
  - Prompt engineering
  - Context optimization
  - Multi-turn conversation
  - Citation extraction
  - Hallucination detection

### 7. Chat Session Memory
- **Tech Stack**: PostgreSQL + Redis
- **Responsibilities**:
  - Session state management
  - Conversation history
  - User preferences
  - Memory decay (older messages summarized)

### 8. Frontend
- **Tech Stack**: React + Tailwind + Zustand + React Query
- **Responsibilities**:
  - Chat interface with streaming
  - Document upload with progress
  - Search UI
  - Admin dashboard

## Scaling Considerations

### Horizontal Scaling
- **Stateless Services**: NestJS API (load balanced)
- **Worker Pools**: Ingestion workers, embedding workers (auto-scale on queue depth)
- **Database Sharding**: Qdrant collections per tenant

### Performance Optimization
- **Caching**: Redis for queries, embeddings, search results
- **CDN**: Static assets, processed documents
- **Database Indexing**: B-tree on doc_id, chunk_id; GiST on vectors
- **Connection Pooling**: PostgreSQL (pg-pool), Qdrant (persistent connections)

### Cost Optimization
- **Embedding Batching**: Reduce API calls (32-64 chunks/batch)
- **Lazy Loading**: Generate embeddings on-demand for large docs
- **Model Selection**: Use smaller models for re-ranking
- **Caching**: Avoid redundant LLM calls

## Security & Compliance

- **Authentication**: JWT + OAuth2
- **Authorization**: RBAC (user/admin roles)
- **Data Encryption**: At-rest (AES-256), in-transit (TLS 1.3)
- **PII Detection**: spaCy NER + regex patterns
- **Audit Logging**: All queries, ingestions, access

## Monitoring & Observability

- **Metrics**: Prometheus (ingestion rate, latency, errors)
- **Logs**: Elasticsearch + Kibana
- **Tracing**: OpenTelemetry + Jaeger
- **Alerts**: Grafana (queue depth, API errors, DB load)

## Deployment

- **Containerization**: Docker multi-stage builds
- **Orchestration**: Kubernetes (deployments, services, ingress)
- **CI/CD**: GitHub Actions → Docker Hub → K8s rolling update
- **Infrastructure**: Terraform (AWS/GCP)

## Technology Stack Summary

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Frontend | React + Tailwind + Vite | Fast dev, component reuse, modern UX |
| Backend API | NestJS + TypeScript | Enterprise-grade, DI, testability |
| Vector DB | Qdrant | Performance, filters, easy ops |
| Relational DB | PostgreSQL | ACID, JSON support, pgvector option |
| Cache | Redis | Speed, pub/sub, session store |
| Queue | Bull (Redis) | Reliable, UI dashboard, retries |
| Search | Elasticsearch | Full-text, analytics, logs |
| Storage | MinIO (S3-compatible) | Self-hosted, cost-effective |
| Embeddings | OpenAI text-embedding-3-large | State-of-art quality (3072 dims) |
| LLM | GPT-4 / Claude-3.5-Sonnet | Reasoning, long context |
| Orchestration | Kubernetes | Auto-scaling, self-healing |
| Monitoring | Prometheus + Grafana | Industry standard, extensible |

## Next Steps

1. Set up monorepo structure
2. Implement core services
3. Deploy to staging
4. Load testing & optimization
5. Production deployment
