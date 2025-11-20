# Production-Grade RAG System

A complete, enterprise-ready Retrieval Augmented Generation (RAG) system with multi-format document ingestion, semantic search, and intelligent LLM orchestration.

## Features

### Document Processing
- **Multi-Format Support**: PDF, PPTX, Text, Markdown, URLs
- **Advanced Parsing**: Layout-aware PDF extraction, PPTX slide/notes extraction
- **OCR Support**: Tesseract integration for scanned documents (optional)
- **Smart Chunking**: Recursive, semantic, HTML-aware, and code-aware strategies
- **Quality Scoring**: Automatic chunk quality assessment and filtering

### Search & Retrieval
- **Hybrid Search**: Combines vector (semantic) and keyword (BM25) search
- **Intelligent Re-ranking**: Cross-encoder model for precision
- **Rich Filtering**: By date, document type, tags, quality score
- **Metadata Boosting**: Recent documents, headings, and important sections

### LLM Integration
- **Multi-Provider**: OpenAI (GPT-4) and Anthropic (Claude-3.5)
- **Streaming Responses**: Real-time answer generation
- **Citation Tracking**: Automatic source attribution
- **Hallucination Prevention**: Grounded prompts and validation
- **Context Optimization**: Intelligent chunk selection and token management

### Memory System
- **Session Memory**: Conversation-aware responses
- **Long-term Memory**: User preferences and learned facts
- **Memory Decay**: Time-based importance scoring
- **Vector-based Retrieval**: Semantic memory search

### Enterprise Features
- **Multi-tenancy**: User isolation and access control
- **Scalability**: Kubernetes-ready with auto-scaling
- **Monitoring**: Prometheus metrics + Grafana dashboards
- **Cost Tracking**: Per-query cost analysis and optimization
- **Security**: JWT auth, encryption at rest/in transit, PII detection

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend (React)                         │
│  - Chat Interface  - Document Upload  - Search  - Admin         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────┐
│                      API Gateway (NestJS)                        │
│  - Auth  - Ingest  - Search  - Chat  - Memory  - Admin         │
└─────┬──────────────────────────────────────────────────────┬────┘
      │                                                       │
┌─────┴──────────────┐                         ┌─────────────┴────┐
│  Processing Layer  │                         │  Storage Layer   │
│  - Ingestion Queue │                         │  - PostgreSQL    │
│  - Chunking        │                         │  - Qdrant (Vec)  │
│  - Embedding       │                         │  - Redis (Cache) │
│  - Re-ranking      │                         │  - MinIO (S3)    │
└────────────────────┘                         └──────────────────┘
```

## Quick Start

> **📖 For detailed setup instructions, troubleshooting, and usage examples, see [SETUP_AND_USAGE.md](SETUP_AND_USAGE.md)**

### Prerequisites

- Docker & Docker Compose
- Node.js 20+
- Python 3.11+
- pnpm 8+
- OpenAI API key (for embeddings & LLM)

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/yourusername/rag-system.git
   cd rag-system
   ```

2. **Set up environment variables**
   ```bash
   cp .env.example .env
   # Edit .env and add your API keys (OPENAI_API_KEY required)
   ```

3. **Start all services with Docker**
   ```bash
   docker-compose up -d
   ```

4. **Access the application**
   - Frontend: http://localhost:5173
   - API: http://localhost:3000
   - API Docs: http://localhost:3000/api/docs

**That's it!** The system is now running. See [SETUP_AND_USAGE.md](SETUP_AND_USAGE.md) for:
- Step-by-step setup guide
- How to upload documents
- How to search and chat
- Troubleshooting tips
- Production deployment

### Alternative: Development Mode

For development with hot-reload:

1. **Start infrastructure only**
   ```bash
   docker-compose up -d postgres redis qdrant elasticsearch minio rabbitmq
   ```

2. **Install dependencies**
   ```bash
   pnpm install
   ```

3. **Run database migrations**
   ```bash
   pnpm db:migrate
   ```

6. **Start services**
   ```bash
   # Terminal 1: API
   pnpm api:dev

   # Terminal 2: Frontend
   pnpm web:dev

   # Terminal 3: Workers (optional, or use Docker)
   docker-compose up ingestion-worker embedding-worker
   ```

7. **Access the application**
   - Frontend: http://localhost:5173
   - API: http://localhost:3000
   - API Docs: http://localhost:3000/api/docs
   - Grafana: http://localhost:3001 (admin/admin)
   - MinIO Console: http://localhost:9001 (minioadmin/minioadmin)

### Using Docker Compose (Recommended)

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down

# Rebuild images
docker-compose build
```

## Project Structure

```
dev-RAG/
├── apps/
│   ├── api/                    # NestJS backend
│   ├── web/                    # React frontend
│   └── workers/                # Python workers
│       ├── ingestion-worker/
│       └── embedding-worker/
├── packages/
│   ├── shared-types/           # Shared TypeScript types
│   └── shared-utils/           # Shared utilities
├── infra/
│   ├── docker/                 # Dockerfiles
│   ├── kubernetes/             # K8s manifests
│   ├── terraform/              # IaC
│   └── monitoring/             # Prometheus, Grafana
├── docs/                       # Documentation
├── scripts/                    # Utility scripts
└── docker-compose.yml
```

## API Endpoints

### Authentication
- `POST /auth/register` - Register new user
- `POST /auth/login` - Login
- `POST /auth/refresh` - Refresh token

### Ingestion
- `POST /ingest/file` - Upload document
- `POST /ingest/url` - Ingest from URL
- `GET /ingest/status/:jobId` - Check ingestion status

### Search
- `POST /search` - Hybrid search
- `GET /search/:id` - Get search result details

### Chat
- `POST /chat` - Send message (with RAG)
- `GET /chat/conversations` - List conversations
- `GET /chat/conversations/:id` - Get conversation history
- `WS /chat/stream` - WebSocket for streaming

### Documents
- `GET /documents` - List user documents
- `GET /documents/:id` - Get document details
- `GET /documents/:id/chunks` - Get document chunks
- `DELETE /documents/:id` - Delete document

### Memory
- `GET /memory` - Get user memories
- `POST /memory` - Store memory
- `DELETE /memory/:id` - Delete memory

## Configuration

### Environment Variables

See `.env.example` for all configuration options.

**Required:**
- `OPENAI_API_KEY` - OpenAI API key
- `DATABASE_PASSWORD` - PostgreSQL password
- `JWT_SECRET` - JWT signing secret

**Optional:**
- `ANTHROPIC_API_KEY` - For Claude models
- `ENABLE_OCR` - Enable OCR for scanned PDFs
- `ENABLE_RERANKING` - Enable cross-encoder re-ranking
- `MAX_CHUNK_SIZE` - Maximum chunk size in tokens

### Chunking Configuration

Edit chunk sizes in `.env`:
```bash
MAX_CHUNK_SIZE=1000
MIN_CHUNK_SIZE=200
CHUNK_OVERLAP=100
```

### Embedding Model

Change embedding model:
```bash
OPENAI_EMBEDDING_MODEL=text-embedding-3-large  # or text-embedding-3-small
```

### LLM Model

Configure LLM:
```bash
OPENAI_CHAT_MODEL=gpt-4-turbo-preview
ANTHROPIC_MODEL=claude-3-5-sonnet-20250220
```

## Deployment

### Kubernetes

1. **Create namespace**
   ```bash
   kubectl create namespace rag-system
   ```

2. **Deploy infrastructure**
   ```bash
   kubectl apply -f infra/kubernetes/deployments/
   ```

3. **Deploy application**
   ```bash
   kubectl apply -f infra/kubernetes/services/
   ```

4. **Configure ingress**
   ```bash
   kubectl apply -f infra/kubernetes/ingress/
   ```

### Terraform (AWS)

```bash
cd infra/terraform/environments/prod
terraform init
terraform plan
terraform apply
```

## Monitoring

### Prometheus Metrics

- `rag_ingestion_documents_total` - Total documents ingested
- `rag_search_duration_seconds` - Search latency
- `rag_llm_tokens_total` - LLM tokens used
- `rag_embedding_cost_dollars` - Embedding costs
- `rag_queue_depth` - Worker queue depth

### Grafana Dashboards

Access Grafana at http://localhost:3001

**Available Dashboards:**
- RAG System Overview
- Ingestion Metrics
- Search Performance
- LLM Usage & Costs
- Infrastructure Health

## Development

### Run Tests

```bash
# Unit tests
pnpm test

# E2E tests
pnpm test:e2e

# Coverage
pnpm test:cov
```

### Linting

```bash
pnpm lint
pnpm lint:fix
```

### Database Migrations

```bash
# Generate migration
pnpm migration:generate

# Run migrations
pnpm migration:run

# Revert migration
pnpm migration:revert
```

## Performance

### Benchmarks

**Ingestion:**
- PDF (100 pages): ~30 seconds
- PPTX (50 slides): ~15 seconds
- Embedding: 1000 chunks in 60 seconds

**Search:**
- Hybrid search: <500ms (p95)
- Vector search: <200ms (p95)
- Re-ranking: +100ms

**Chat:**
- Response latency: 1-3 seconds
- Streaming: First token in <500ms

### Optimization Tips

1. **Increase worker concurrency** for faster ingestion
2. **Enable caching** for repeat queries
3. **Use smaller embedding model** for cost savings
4. **Adjust chunk size** based on your use case
5. **Enable re-ranking only for critical queries**

## Troubleshooting

### Common Issues

**1. Qdrant connection refused**
```bash
# Check if Qdrant is running
docker-compose ps qdrant

# Restart Qdrant
docker-compose restart qdrant
```

**2. Worker not processing jobs**
```bash
# Check RabbitMQ
docker-compose logs rabbitmq

# Restart workers
docker-compose restart ingestion-worker embedding-worker
```

**3. Out of memory**
```bash
# Increase Docker memory limit
# Or reduce worker concurrency
```

**4. OpenAI rate limit**
```bash
# Add backoff/retry logic (already implemented)
# Or use multiple API keys
```

## Cost Estimation

**Per 1000 documents (avg 10 pages each):**
- Embeddings: ~$2.00
- LLM queries: ~$5.00 (1000 queries)
- Storage: ~$0.50/month
- Compute: ~$1.00/month

**Total: ~$8.50 per 1000 documents**

## Security

- JWT authentication with refresh tokens
- Row-level security (user_id filtering)
- Encryption at rest (AES-256)
- Encryption in transit (TLS 1.3)
- PII detection and masking
- Rate limiting (100 req/min per user)
- Audit logging

## Documentation

### Getting Started
- **[Setup and Usage Guide](SETUP_AND_USAGE.md)** ⭐ - Complete step-by-step guide to set up and use the system

### Technical Documentation
- [Architecture](ARCHITECTURE.md) - System architecture and data flow diagrams
- [Database Schema](DATABASE_SCHEMA.md) - Complete database design with PostgreSQL and Qdrant
- [Project Structure](PROJECT_STRUCTURE.md) - Monorepo layout and folder organization

### Implementation Details
- [Ingestion Engine](INGESTION_ENGINE.md) - Multi-format document parsing (PDF, PPTX, Text, URLs)
- [Chunking System](CHUNKING_SYSTEM.md) - Advanced chunking strategies with implementations
- [Embedding Pipeline](EMBEDDING_PIPELINE.md) - Embedding generation with batching and versioning
- [Vector Search](VECTOR_SEARCH_AND_ORCHESTRATION.md) - Hybrid search and LLM orchestration
- [Memory Layer & Engineering Decisions](MEMORY_AND_ENGINEERING_DECISIONS.md) - Memory system and all technical decisions

## License

MIT License - see LICENSE file for details

## Support

- Documentation: [/docs](/docs)
- Issues: [GitHub Issues](https://github.com/yourusername/rag-system/issues)

## Acknowledgments

- OpenAI for embeddings and LLM
- Qdrant for vector search
- NestJS and React communities
- All open-source contributors

---

Built with precision for production RAG systems
