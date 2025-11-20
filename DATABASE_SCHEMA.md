# Database Schema Design

## Technology Choice: PostgreSQL + Qdrant

### Why PostgreSQL for Relational Data?
- ACID compliance for transactional integrity
- Rich JSON/JSONB support for flexible metadata
- Mature ecosystem with TypeORM integration
- pgvector extension available (backup vector storage)
- Excellent performance with proper indexing

### Why Qdrant for Vector Storage?
- Purpose-built for high-dimensional vectors
- HNSW algorithm for fast ANN search
- Rich filtering on metadata (date, tags, etc.)
- Horizontal scaling capabilities
- Better performance than pgvector at scale
- Disk-backed storage (lower memory usage)

## PostgreSQL Schema

### 1. Users Table
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  full_name VARCHAR(255),
  role VARCHAR(50) DEFAULT 'user', -- 'user', 'admin'
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  last_login_at TIMESTAMP,
  metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
```

### 2. Documents Table
```sql
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(500) NOT NULL,
  file_name VARCHAR(500) NOT NULL,
  file_type VARCHAR(50) NOT NULL, -- 'pdf', 'pptx', 'txt', 'url'
  file_size BIGINT, -- bytes
  file_url TEXT, -- S3/MinIO URL
  content_hash VARCHAR(64), -- SHA-256 hash for deduplication
  status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'processing', 'indexed', 'failed'
  error_message TEXT,
  page_count INTEGER,
  word_count INTEGER,
  language VARCHAR(10) DEFAULT 'en',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  indexed_at TIMESTAMP,
  metadata JSONB DEFAULT '{}'::jsonb, -- { tags, author, source_url, custom_fields }

  CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'indexed', 'failed'))
);

CREATE INDEX idx_documents_user_id ON documents(user_id);
CREATE INDEX idx_documents_status ON documents(status);
CREATE INDEX idx_documents_file_type ON documents(file_type);
CREATE INDEX idx_documents_created_at ON documents(created_at DESC);
CREATE INDEX idx_documents_content_hash ON documents(content_hash);
CREATE INDEX idx_documents_metadata ON documents USING GIN (metadata);
```

### 3. Chunks Table
```sql
CREATE TABLE chunks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  chunk_index INTEGER NOT NULL, -- Position in document
  content TEXT NOT NULL,
  content_hash VARCHAR(64), -- For deduplication
  chunk_type VARCHAR(50) DEFAULT 'text', -- 'text', 'code', 'table', 'heading'
  token_count INTEGER NOT NULL,
  char_count INTEGER NOT NULL,

  -- Chunking strategy metadata
  strategy VARCHAR(50) NOT NULL, -- 'recursive', 'semantic', 'html_aware', 'code_aware'
  parent_heading TEXT,
  section_title TEXT,
  page_number INTEGER,

  -- Quality metrics
  quality_score FLOAT DEFAULT 0.0, -- 0.0 to 1.0
  info_density FLOAT, -- Calculated metric

  -- Embeddings link
  vector_id VARCHAR(255), -- Qdrant point ID
  embedding_model VARCHAR(100) DEFAULT 'text-embedding-3-large',
  embedding_version VARCHAR(20) DEFAULT 'v1',

  created_at TIMESTAMP DEFAULT NOW(),
  metadata JSONB DEFAULT '{}'::jsonb, -- { keywords, entities, sentiment }

  UNIQUE(document_id, chunk_index)
);

CREATE INDEX idx_chunks_document_id ON chunks(document_id);
CREATE INDEX idx_chunks_vector_id ON chunks(vector_id);
CREATE INDEX idx_chunks_embedding_model ON chunks(embedding_model);
CREATE INDEX idx_chunks_quality_score ON chunks(quality_score DESC);
CREATE INDEX idx_chunks_content_hash ON chunks(content_hash);
CREATE INDEX idx_chunks_metadata ON chunks USING GIN (metadata);

-- Full-text search index
CREATE INDEX idx_chunks_content_fts ON chunks USING GIN (to_tsvector('english', content));
```

### 4. Conversations Table
```sql
CREATE TABLE conversations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title VARCHAR(500),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_conversations_user_id ON conversations(user_id);
CREATE INDEX idx_conversations_updated_at ON conversations(updated_at DESC);
```

### 5. Messages Table
```sql
CREATE TABLE messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  role VARCHAR(50) NOT NULL, -- 'user', 'assistant', 'system'
  content TEXT NOT NULL,
  tokens_used INTEGER,

  -- Source tracking for RAG
  source_chunk_ids UUID[], -- Array of chunk IDs used
  citations JSONB, -- [{ chunk_id, doc_title, page, relevance_score }]

  -- LLM metadata
  llm_model VARCHAR(100),
  llm_temperature FLOAT,
  llm_max_tokens INTEGER,

  created_at TIMESTAMP DEFAULT NOW(),
  metadata JSONB DEFAULT '{}'::jsonb,

  CONSTRAINT valid_role CHECK (role IN ('user', 'assistant', 'system'))
);

CREATE INDEX idx_messages_conversation_id ON messages(conversation_id);
CREATE INDEX idx_messages_created_at ON messages(created_at DESC);
CREATE INDEX idx_messages_role ON messages(role);
```

### 6. Memory Vectors Table (Long-term Memory)
```sql
CREATE TABLE memory_vectors (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  conversation_id UUID REFERENCES conversations(id) ON DELETE SET NULL,

  memory_type VARCHAR(50) NOT NULL, -- 'session', 'permanent', 'preference'
  content TEXT NOT NULL,
  summary TEXT, -- Condensed version

  vector_id VARCHAR(255) NOT NULL, -- Qdrant point ID
  embedding_model VARCHAR(100) DEFAULT 'text-embedding-3-large',

  -- Memory management
  importance_score FLOAT DEFAULT 0.5, -- 0.0 to 1.0
  access_count INTEGER DEFAULT 0,
  last_accessed_at TIMESTAMP DEFAULT NOW(),
  decay_factor FLOAT DEFAULT 1.0, -- Decreases over time

  -- Time-based decay
  created_at TIMESTAMP DEFAULT NOW(),
  expires_at TIMESTAMP, -- For session memory

  metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_memory_vectors_user_id ON memory_vectors(user_id);
CREATE INDEX idx_memory_vectors_conversation_id ON memory_vectors(conversation_id);
CREATE INDEX idx_memory_vectors_memory_type ON memory_vectors(memory_type);
CREATE INDEX idx_memory_vectors_importance_score ON memory_vectors(importance_score DESC);
CREATE INDEX idx_memory_vectors_last_accessed ON memory_vectors(last_accessed_at DESC);
```

### 7. Search Logs Table (Analytics)
```sql
CREATE TABLE search_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE SET NULL,
  query TEXT NOT NULL,
  query_embedding_id VARCHAR(255), -- Qdrant point ID

  -- Search parameters
  search_type VARCHAR(50) NOT NULL, -- 'semantic', 'keyword', 'hybrid'
  filters JSONB,
  top_k INTEGER DEFAULT 10,

  -- Results
  results_count INTEGER,
  top_chunk_ids UUID[],
  top_scores FLOAT[],

  -- Performance metrics
  search_duration_ms INTEGER,
  embedding_duration_ms INTEGER,
  rerank_duration_ms INTEGER,

  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_search_logs_user_id ON search_logs(user_id);
CREATE INDEX idx_search_logs_created_at ON search_logs(created_at DESC);
CREATE INDEX idx_search_logs_search_type ON search_logs(search_type);
```

### 8. Ingestion Jobs Table
```sql
CREATE TABLE ingestion_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  status VARCHAR(50) DEFAULT 'queued', -- 'queued', 'processing', 'completed', 'failed'

  -- Job stages
  stage VARCHAR(50), -- 'parsing', 'chunking', 'embedding', 'indexing'
  progress_percentage INTEGER DEFAULT 0,

  -- Timing
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  duration_seconds INTEGER,

  -- Results
  chunks_created INTEGER DEFAULT 0,
  embeddings_generated INTEGER DEFAULT 0,
  error_message TEXT,

  -- Cost tracking
  api_calls INTEGER DEFAULT 0,
  tokens_used INTEGER DEFAULT 0,
  estimated_cost DECIMAL(10, 4) DEFAULT 0.0,

  metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_ingestion_jobs_document_id ON ingestion_jobs(document_id);
CREATE INDEX idx_ingestion_jobs_status ON ingestion_jobs(status);
CREATE INDEX idx_ingestion_jobs_created_at ON ingestion_jobs(created_at DESC);
```

## Qdrant Collections Schema

### Collection 1: document_chunks
```javascript
{
  "collection_name": "document_chunks",
  "vector_config": {
    "size": 3072,  // text-embedding-3-large dimensions
    "distance": "Cosine"  // Cosine similarity
  },
  "hnsw_config": {
    "m": 16,  // Number of edges per node
    "ef_construct": 100,  // Size of dynamic candidate list
    "ef_search": 50  // Size of candidate list for search
  },
  "optimizers_config": {
    "indexing_threshold": 20000  // Start indexing after 20K vectors
  },
  "payload_schema": {
    "chunk_id": "keyword",  // UUID from PostgreSQL
    "document_id": "keyword",  // UUID
    "user_id": "keyword",  // UUID
    "content": "text",  // Full chunk text (for reranking)
    "chunk_index": "integer",
    "chunk_type": "keyword",  // 'text', 'code', 'heading'
    "document_title": "text",
    "file_type": "keyword",
    "page_number": "integer",
    "section_title": "text",
    "parent_heading": "text",
    "quality_score": "float",
    "token_count": "integer",
    "tags": ["keyword"],  // Array of tags
    "created_at": "datetime",
    "metadata": "object"  // Flexible metadata
  }
}
```

### Collection 2: memory_vectors
```javascript
{
  "collection_name": "memory_vectors",
  "vector_config": {
    "size": 3072,
    "distance": "Cosine"
  },
  "hnsw_config": {
    "m": 16,
    "ef_construct": 100,
    "ef_search": 50
  },
  "payload_schema": {
    "memory_id": "keyword",  // UUID from PostgreSQL
    "user_id": "keyword",
    "conversation_id": "keyword",
    "memory_type": "keyword",  // 'session', 'permanent', 'preference'
    "content": "text",
    "summary": "text",
    "importance_score": "float",
    "access_count": "integer",
    "decay_factor": "float",
    "created_at": "datetime",
    "last_accessed_at": "datetime",
    "metadata": "object"
  }
}
```

### Collection 3: query_cache
```javascript
{
  "collection_name": "query_cache",
  "vector_config": {
    "size": 3072,
    "distance": "Cosine"
  },
  "hnsw_config": {
    "m": 16,
    "ef_construct": 50,
    "ef_search": 30
  },
  "payload_schema": {
    "query": "text",
    "query_hash": "keyword",  // MD5 of normalized query
    "user_id": "keyword",
    "result_chunk_ids": ["keyword"],  // Top results
    "filters": "object",
    "ttl": "datetime",  // Expiration time
    "hit_count": "integer",
    "created_at": "datetime"
  }
}
```

## Data Relationships

```
users (1) ────────── (N) documents
  │                      │
  │                      └── (1) ────────── (N) chunks
  │                                           │
  │                                           └── vector_id → Qdrant:document_chunks
  │
  ├── (1) ────────── (N) conversations
  │                      │
  │                      └── (1) ────────── (N) messages
  │                                           │
  │                                           └── source_chunk_ids[] → chunks
  │
  └── (1) ────────── (N) memory_vectors
                          │
                          └── vector_id → Qdrant:memory_vectors
```

## Indexing Strategy

### High-Cardinality Columns (B-tree)
- UUIDs (id, user_id, document_id, etc.)
- Timestamps (created_at, updated_at)
- Status fields (for filtering)

### Full-Text Search (GIN)
- chunks.content (to_tsvector)
- documents.metadata (JSONB)
- chunks.metadata (JSONB)

### Composite Indexes (Multi-column)
```sql
-- Frequently joined columns
CREATE INDEX idx_chunks_doc_quality ON chunks(document_id, quality_score DESC);
CREATE INDEX idx_messages_conv_time ON messages(conversation_id, created_at DESC);

-- Filter + sort combinations
CREATE INDEX idx_documents_user_status_time ON documents(user_id, status, created_at DESC);
```

## Data Retention Policies

### Session Memory
- TTL: 7 days
- Auto-cleanup job runs daily
```sql
DELETE FROM memory_vectors
WHERE memory_type = 'session'
  AND expires_at < NOW();
```

### Search Logs
- Retention: 90 days
- Aggregated to analytics tables after 30 days

### Inactive Documents
- Documents with status='failed' for >30 days → archive to cold storage

## Backup Strategy

### PostgreSQL
- Daily full backups
- WAL archiving (point-in-time recovery)
- Backup retention: 30 days

### Qdrant
- Snapshot backups daily
- Stored in S3/MinIO
- Retention: 14 days

## Scaling Considerations

### Read Replicas
- PostgreSQL: Read replicas for analytics queries
- Qdrant: Sharding by user_id or tenant_id

### Partitioning
```sql
-- Partition search_logs by month
CREATE TABLE search_logs (
  -- columns...
) PARTITION BY RANGE (created_at);

CREATE TABLE search_logs_2024_01 PARTITION OF search_logs
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

### Archival
- Move old chunks to separate archive table
- Keep metadata in main table, content in cold storage

## Example Queries

### 1. Get User's Recent Documents with Chunk Count
```sql
SELECT
  d.id,
  d.title,
  d.file_type,
  d.status,
  d.created_at,
  COUNT(c.id) as chunk_count
FROM documents d
LEFT JOIN chunks c ON d.id = c.document_id
WHERE d.user_id = $1
  AND d.status = 'indexed'
GROUP BY d.id
ORDER BY d.created_at DESC
LIMIT 20;
```

### 2. Get Conversation with Messages and Sources
```sql
SELECT
  m.id,
  m.role,
  m.content,
  m.created_at,
  m.citations,
  array_agg(
    json_build_object(
      'chunk_id', c.id,
      'content', c.content,
      'document_title', d.title,
      'page_number', c.page_number
    )
  ) as sources
FROM messages m
LEFT JOIN unnest(m.source_chunk_ids) WITH ORDINALITY AS s(chunk_id, ord) ON true
LEFT JOIN chunks c ON c.id = s.chunk_id
LEFT JOIN documents d ON d.id = c.document_id
WHERE m.conversation_id = $1
GROUP BY m.id
ORDER BY m.created_at ASC;
```

### 3. Get High-Quality Chunks for a Document
```sql
SELECT
  c.id,
  c.content,
  c.chunk_index,
  c.quality_score,
  c.parent_heading,
  c.page_number
FROM chunks c
WHERE c.document_id = $1
  AND c.quality_score > 0.7
ORDER BY c.chunk_index ASC;
```

### 4. User Memory Decay Update (Run periodically)
```sql
UPDATE memory_vectors
SET
  decay_factor = decay_factor * 0.95,  -- 5% decay
  importance_score = importance_score * decay_factor
WHERE
  user_id = $1
  AND memory_type != 'permanent'
  AND last_accessed_at < NOW() - INTERVAL '7 days';
```
