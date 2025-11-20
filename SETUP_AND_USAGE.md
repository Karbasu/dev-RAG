# Setup and Usage Guide

Complete step-by-step guide to set up, configure, and use the production-grade RAG system.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Configuration](#configuration)
4. [Running Locally](#running-locally)
5. [Using the System](#using-the-system)
6. [Testing](#testing)
7. [Production Deployment](#production-deployment)
8. [Troubleshooting](#troubleshooting)
9. [Advanced Usage](#advanced-usage)

---

## Prerequisites

### Required Software

1. **Docker & Docker Compose**
   - Docker Desktop 4.0+ (Windows/Mac)
   - Docker Engine 20.10+ (Linux)
   - Docker Compose 2.0+

   ```bash
   # Verify installation
   docker --version
   docker-compose --version
   ```

2. **Node.js & pnpm**
   - Node.js 20.x or higher
   - pnpm 8.x or higher

   ```bash
   # Install Node.js (via nvm recommended)
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
   nvm install 20
   nvm use 20

   # Install pnpm
   npm install -g pnpm

   # Verify
   node --version  # Should be v20.x.x
   pnpm --version  # Should be 8.x.x
   ```

3. **Python 3.11+** (for workers)
   ```bash
   # Check version
   python3 --version  # Should be 3.11+

   # Install pip
   python3 -m ensurepip --upgrade
   ```

4. **Git**
   ```bash
   git --version
   ```

### System Requirements

**Minimum for Development:**
- 8 GB RAM
- 4 CPU cores
- 20 GB free disk space

**Recommended for Production:**
- 16 GB RAM
- 8 CPU cores
- 100 GB SSD

### API Keys Required

1. **OpenAI API Key** (Required)
   - Sign up at https://platform.openai.com
   - Create API key at https://platform.openai.com/api-keys
   - Cost: ~$0.13 per 1M tokens for embeddings

2. **Anthropic API Key** (Optional, for Claude)
   - Sign up at https://console.anthropic.com
   - Create API key in settings
   - Cost: ~$3 per 1M tokens

---

## Initial Setup

### Step 1: Clone the Repository

```bash
# Clone the repo
git clone https://github.com/yourusername/dev-RAG.git
cd dev-RAG

# Checkout the branch (if not on main)
git checkout claude/build-rag-system-01Wwuaa5S7HYywAwbF38HTac
```

### Step 2: Configure Environment Variables

```bash
# Copy example environment file
cp .env.example .env

# Open .env in your editor
nano .env  # or vim, code, etc.
```

**Edit `.env` and add your API keys:**

```bash
# REQUIRED: Add your OpenAI API key
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxx

# OPTIONAL: Add Anthropic API key for Claude
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Database password (change for production)
DATABASE_PASSWORD=your_secure_password

# JWT secret (change for production)
JWT_SECRET=your_jwt_secret_key_min_32_chars

# Leave other values as defaults for local development
```

### Step 3: Start Infrastructure Services

**Using Docker Compose (Recommended):**

```bash
# Start all infrastructure services
docker-compose up -d postgres redis qdrant elasticsearch minio rabbitmq

# Verify all services are running
docker-compose ps

# Expected output:
# NAME                  STATUS    PORTS
# rag-postgres          Up        0.0.0.0:5432->5432/tcp
# rag-redis             Up        0.0.0.0:6379->6379/tcp
# rag-qdrant            Up        0.0.0.0:6333->6333/tcp
# rag-elasticsearch     Up        0.0.0.0:9200->9200/tcp
# rag-minio             Up        0.0.0.0:9000-9001->9000-9001/tcp
# rag-rabbitmq          Up        0.0.0.0:5672,15672->5672,15672/tcp
```

**Wait for services to be healthy:**

```bash
# Check health
docker-compose ps

# Wait until all services show "healthy" status (1-2 minutes)
# You can also check logs:
docker-compose logs -f postgres redis qdrant
```

### Step 4: Install Dependencies

```bash
# Install all workspace dependencies
pnpm install

# This will install dependencies for:
# - Root workspace
# - apps/api (NestJS backend)
# - apps/web (React frontend)
# - packages/shared-types
# - packages/shared-utils
```

### Step 5: Set Up Database

```bash
# Run database migrations
pnpm db:migrate

# Expected output:
# Migration InitialSchema1234567890 has been executed successfully

# (Optional) Seed with sample data
pnpm db:seed
```

### Step 6: Initialize Qdrant Collections

```bash
# Create vector collections
curl -X PUT "http://localhost:6333/collections/document_chunks" \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": {
      "size": 3072,
      "distance": "Cosine"
    }
  }'

curl -X PUT "http://localhost:6333/collections/memory_vectors" \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": {
      "size": 3072,
      "distance": "Cosine"
    }
  }'

# Verify collections created
curl http://localhost:6333/collections
```

### Step 7: Initialize MinIO Buckets

```bash
# Install MinIO client
# Mac:
brew install minio/stable/mc

# Linux:
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configure MinIO
mc alias set local http://localhost:9000 minioadmin minioadmin

# Create bucket
mc mb local/rag-documents

# Verify
mc ls local
```

---

## Running Locally

### Option 1: Run Everything with Docker (Easiest)

```bash
# Start all services (infrastructure + application)
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down
```

**Access URLs:**
- Frontend: http://localhost:5173
- API: http://localhost:3000
- API Docs: http://localhost:3000/api/docs
- MinIO Console: http://localhost:9001 (minioadmin/minioadmin)
- RabbitMQ Console: http://localhost:15672 (admin/admin)
- Grafana: http://localhost:3001 (admin/admin)

### Option 2: Run Application in Development Mode (Recommended for Development)

This gives you hot-reload and better debugging.

**Terminal 1: Start Infrastructure**
```bash
docker-compose up -d postgres redis qdrant elasticsearch minio rabbitmq prometheus grafana
```

**Terminal 2: Start API (NestJS)**
```bash
cd apps/api
pnpm install
pnpm dev

# Expected output:
# [Nest] 12345  - Nest application successfully started
# [Nest] 12345  - Mapped {/api/v1/auth/register, POST} route
# [Nest] 12345  - Listening on http://localhost:3000
```

**Terminal 3: Start Frontend (React)**
```bash
cd apps/web
pnpm install
pnpm dev

# Expected output:
# VITE v5.0.0  ready in 500 ms
# ➜  Local:   http://localhost:5173/
# ➜  Network: use --host to expose
```

**Terminal 4: Start Workers**
```bash
# Option A: Use Docker
docker-compose up -d ingestion-worker embedding-worker

# Option B: Run locally
cd apps/workers/ingestion-worker
pip install -r requirements.txt
celery -A src.worker worker --loglevel=info --concurrency=4

# Terminal 5 (for embedding worker):
cd apps/workers/embedding-worker
pip install -r requirements.txt
celery -A src.worker worker --loglevel=info --concurrency=2
```

---

## Using the System

### 1. Create an Account

**Via API:**
```bash
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePassword123!",
    "fullName": "John Doe"
  }'

# Response:
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "fullName": "John Doe"
  },
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

**Via Frontend:**
1. Open http://localhost:5173
2. Click "Sign Up"
3. Fill in email, password, full name
4. Click "Create Account"

### 2. Login

**Via API:**
```bash
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "SecurePassword123!"
  }'

# Save the accessToken for subsequent requests
export TOKEN="eyJhbGciOiJIUzI1NiIs..."
```

**Via Frontend:**
1. Click "Login"
2. Enter credentials
3. You'll be redirected to dashboard

### 3. Upload Documents

**Via API (Upload File):**
```bash
# Upload a PDF
curl -X POST http://localhost:3000/api/v1/ingest/file \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/document.pdf"

# Response:
{
  "documentId": "uuid",
  "title": "document.pdf",
  "status": "pending",
  "jobId": "job-uuid"
}

# Upload a PPTX
curl -X POST http://localhost:3000/api/v1/ingest/file \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/presentation.pptx"
```

**Via API (Ingest URL):**
```bash
curl -X POST http://localhost:3000/api/v1/ingest/url \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/article",
    "title": "Example Article"
  }'
```

**Via Frontend:**
1. Navigate to "Upload" page
2. Click "Choose File" or drag-and-drop
3. Select PDF, PPTX, or TXT file
4. Click "Upload"
5. Watch progress bar (parsing → chunking → embedding)

### 4. Monitor Ingestion Status

**Via API:**
```bash
# Check ingestion job status
curl -X GET http://localhost:3000/api/v1/ingest/status/$JOB_ID \
  -H "Authorization: Bearer $TOKEN"

# Response:
{
  "jobId": "uuid",
  "status": "processing",
  "stage": "embedding",
  "progress": 65,
  "chunksCreated": 45,
  "embeddingsGenerated": 30,
  "estimatedTimeRemaining": 30
}

# Statuses: queued, processing, completed, failed
```

**Via Frontend:**
- Real-time progress updates on upload page
- Notifications when indexing completes

### 5. Search Documents

**Via API:**
```bash
# Semantic search
curl -X POST http://localhost:3000/api/v1/search \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the benefits of RAG systems?",
    "topK": 5,
    "filters": {
      "fileTypes": ["pdf"],
      "dateRange": {
        "from": "2024-01-01",
        "to": "2024-12-31"
      }
    }
  }'

# Response:
{
  "results": [
    {
      "chunkId": "uuid",
      "documentId": "uuid",
      "documentTitle": "RAG Systems Overview.pdf",
      "content": "RAG systems combine retrieval with generation...",
      "score": 0.92,
      "pageNumber": 3,
      "metadata": {
        "parentHeading": "Introduction to RAG",
        "qualityScore": 0.85
      }
    },
    ...
  ],
  "totalResults": 5,
  "searchDuration": 245
}
```

**Via Frontend:**
1. Navigate to "Search" page
2. Enter query in search bar
3. (Optional) Apply filters (date, document type, tags)
4. View results with highlighted snippets
5. Click result to view full document

### 6. Chat with RAG

**Via API:**
```bash
# Start a conversation
curl -X POST http://localhost:3000/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Explain RAG systems based on my documents",
    "conversationId": null
  }'

# Response:
{
  "conversationId": "uuid",
  "message": {
    "id": "uuid",
    "role": "assistant",
    "content": "Based on your documents, RAG (Retrieval Augmented Generation) systems combine document retrieval with LLM generation. [1] They work by...",
    "citations": [
      {
        "chunkId": "uuid",
        "documentTitle": "RAG Overview.pdf",
        "pageNumber": 3
      }
    ]
  }
}

# Continue conversation
curl -X POST http://localhost:3000/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What are the main components?",
    "conversationId": "previous-uuid"
  }'
```

**Via WebSocket (Streaming):**
```javascript
// JavaScript example
const ws = new WebSocket('ws://localhost:3000/chat/stream');

ws.on('open', () => {
  ws.send(JSON.stringify({
    token: 'Bearer eyJhbGci...',
    message: 'Explain RAG systems',
    conversationId: null
  }));
});

ws.on('message', (data) => {
  const chunk = JSON.parse(data);
  if (chunk.type === 'content') {
    process.stdout.write(chunk.content);
  } else if (chunk.type === 'complete') {
    console.log('\nCitations:', chunk.citations);
  }
});
```

**Via Frontend:**
1. Navigate to "Chat" page
2. Type message: "What is RAG?"
3. Watch streaming response appear in real-time
4. Click citations [1], [2] to view sources
5. Continue conversation naturally

### 7. Manage Documents

**List Documents:**
```bash
curl -X GET http://localhost:3000/api/v1/documents \
  -H "Authorization: Bearer $TOKEN"

# Response:
{
  "documents": [
    {
      "id": "uuid",
      "title": "RAG Systems.pdf",
      "fileType": "pdf",
      "fileSize": 2048000,
      "status": "indexed",
      "pageCount": 15,
      "wordCount": 3500,
      "createdAt": "2024-01-15T10:30:00Z",
      "indexedAt": "2024-01-15T10:32:00Z"
    },
    ...
  ]
}
```

**Get Document Details:**
```bash
curl -X GET http://localhost:3000/api/v1/documents/$DOCUMENT_ID \
  -H "Authorization: Bearer $TOKEN"
```

**Get Document Chunks:**
```bash
curl -X GET http://localhost:3000/api/v1/documents/$DOCUMENT_ID/chunks \
  -H "Authorization: Bearer $TOKEN"

# Response:
{
  "chunks": [
    {
      "id": "uuid",
      "content": "Introduction to RAG systems...",
      "chunkIndex": 0,
      "tokenCount": 650,
      "qualityScore": 0.87,
      "pageNumber": 1,
      "parentHeading": "Introduction"
    },
    ...
  ]
}
```

**Delete Document:**
```bash
curl -X DELETE http://localhost:3000/api/v1/documents/$DOCUMENT_ID \
  -H "Authorization: Bearer $TOKEN"
```

### 8. View Conversation History

**List Conversations:**
```bash
curl -X GET http://localhost:3000/api/v1/chat/conversations \
  -H "Authorization: Bearer $TOKEN"
```

**Get Conversation Messages:**
```bash
curl -X GET http://localhost:3000/api/v1/chat/conversations/$CONVERSATION_ID \
  -H "Authorization: Bearer $TOKEN"

# Response:
{
  "conversation": {
    "id": "uuid",
    "title": "RAG Systems Discussion",
    "createdAt": "2024-01-15T10:00:00Z",
    "messages": [
      {
        "id": "uuid",
        "role": "user",
        "content": "What is RAG?",
        "createdAt": "2024-01-15T10:00:00Z"
      },
      {
        "id": "uuid",
        "role": "assistant",
        "content": "RAG stands for Retrieval Augmented Generation...",
        "citations": [...],
        "createdAt": "2024-01-15T10:00:05Z"
      }
    ]
  }
}
```

---

## Testing

### Test the Full Pipeline

**1. Upload a test document:**
```bash
# Create a test PDF
echo "RAG systems combine retrieval with generation. They are powerful tools for question answering." > test.txt

# Upload
curl -X POST http://localhost:3000/api/v1/ingest/file \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test.txt"
```

**2. Wait for indexing (check status):**
```bash
# Poll status endpoint
while true; do
  STATUS=$(curl -s http://localhost:3000/api/v1/ingest/status/$JOB_ID -H "Authorization: Bearer $TOKEN" | jq -r '.status')
  echo "Status: $STATUS"
  [ "$STATUS" = "completed" ] && break
  sleep 2
done
```

**3. Search the document:**
```bash
curl -X POST http://localhost:3000/api/v1/search \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "RAG systems", "topK": 3}'
```

**4. Chat with RAG:**
```bash
curl -X POST http://localhost:3000/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message": "What are RAG systems?", "conversationId": null}'
```

### Run Automated Tests

```bash
# Unit tests
pnpm test

# E2E tests
pnpm test:e2e

# Coverage
pnpm test:cov

# Load tests (requires k6)
k6 run scripts/load-test.js
```

### Verify Services

**Check PostgreSQL:**
```bash
docker exec -it rag-postgres psql -U rag_user -d rag_db -c "SELECT COUNT(*) FROM documents;"
```

**Check Qdrant:**
```bash
curl http://localhost:6333/collections/document_chunks
```

**Check Redis:**
```bash
docker exec -it rag-redis redis-cli PING
```

**Check Elasticsearch:**
```bash
curl http://localhost:9200/_cluster/health
```

---

## Production Deployment

### Option 1: Docker Compose (Small Scale)

```bash
# Create production .env
cp .env.example .env.production

# Edit with production values
nano .env.production

# Start with production config
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Enable SSL with Let's Encrypt
# (Add nginx-proxy and letsencrypt-companion containers)
```

### Option 2: Kubernetes (Recommended)

**Prerequisites:**
- Kubernetes cluster (EKS, GKE, AKS)
- kubectl configured
- Helm 3.x installed

**Deploy:**

```bash
# Create namespace
kubectl create namespace rag-system

# Create secrets
kubectl create secret generic rag-secrets \
  --from-literal=openai-api-key=$OPENAI_API_KEY \
  --from-literal=database-password=$DATABASE_PASSWORD \
  --from-literal=jwt-secret=$JWT_SECRET \
  -n rag-system

# Deploy PostgreSQL (using Helm)
helm install postgres bitnami/postgresql \
  --set auth.password=$DATABASE_PASSWORD \
  --namespace rag-system

# Deploy Qdrant
kubectl apply -f infra/kubernetes/deployments/qdrant-deployment.yaml

# Deploy Redis
kubectl apply -f infra/kubernetes/deployments/redis-deployment.yaml

# Deploy application
kubectl apply -f infra/kubernetes/deployments/api-deployment.yaml
kubectl apply -f infra/kubernetes/deployments/web-deployment.yaml
kubectl apply -f infra/kubernetes/deployments/worker-deployment.yaml

# Deploy services
kubectl apply -f infra/kubernetes/services/

# Deploy ingress
kubectl apply -f infra/kubernetes/ingress/ingress.yaml

# Check status
kubectl get pods -n rag-system
```

### Option 3: Terraform (Infrastructure as Code)

```bash
cd infra/terraform/environments/prod

# Initialize
terraform init

# Plan
terraform plan -var-file=prod.tfvars

# Apply
terraform apply -var-file=prod.tfvars

# This will create:
# - VPC, subnets, security groups
# - EKS cluster
# - RDS PostgreSQL
# - ElastiCache Redis
# - S3 buckets
# - Load balancers
```

---

## Troubleshooting

### Issue: Services won't start

**Symptom:** `docker-compose up` fails

**Solutions:**
```bash
# Check if ports are already in use
lsof -i :5432  # PostgreSQL
lsof -i :6379  # Redis
lsof -i :6333  # Qdrant

# Stop conflicting services
sudo systemctl stop postgresql
sudo systemctl stop redis

# Or change ports in docker-compose.yml

# Check Docker resources
docker system df
docker system prune  # Clean up if needed
```

### Issue: Worker not processing jobs

**Symptom:** Documents stuck in "processing" status

**Solutions:**
```bash
# Check worker logs
docker-compose logs -f ingestion-worker embedding-worker

# Check RabbitMQ queue
curl -u admin:admin http://localhost:15672/api/queues

# Restart workers
docker-compose restart ingestion-worker embedding-worker

# Check for API key issues
docker-compose exec embedding-worker env | grep OPENAI_API_KEY
```

### Issue: OpenAI rate limit errors

**Symptom:** "Rate limit exceeded" in logs

**Solutions:**
```bash
# Reduce worker concurrency
# Edit docker-compose.yml:
# command: celery -A src.worker worker --concurrency=1

# Or add retry delay in code
# (Already implemented with exponential backoff)

# Check usage at https://platform.openai.com/usage
```

### Issue: Qdrant connection refused

**Symptom:** "Connection refused" to Qdrant

**Solutions:**
```bash
# Check if Qdrant is running
docker-compose ps qdrant

# Check logs
docker-compose logs qdrant

# Restart Qdrant
docker-compose restart qdrant

# Wait for health check
docker-compose ps  # Should show "healthy"

# Verify manually
curl http://localhost:6333/collections
```

### Issue: Search returns no results

**Symptom:** Search query returns empty array

**Solutions:**
```bash
# Check if documents are indexed
curl http://localhost:3000/api/v1/documents \
  -H "Authorization: Bearer $TOKEN"

# Check if embeddings exist
curl http://localhost:6333/collections/document_chunks/points/count

# Check search logs
docker-compose logs api | grep search

# Try a broader query
curl -X POST http://localhost:3000/api/v1/search \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"query": "test", "topK": 10}'
```

### Issue: Out of memory

**Symptom:** Containers crashing with OOM

**Solutions:**
```bash
# Increase Docker memory limit
# Docker Desktop: Settings > Resources > Memory (set to 8GB+)

# Reduce worker concurrency
# Edit docker-compose.yml worker concurrency settings

# Add memory limits to containers
# In docker-compose.yml:
# deploy:
#   resources:
#     limits:
#       memory: 2G
```

### Issue: Slow search performance

**Symptom:** Searches take >2 seconds

**Solutions:**
```bash
# Enable Redis caching
# Ensure ENABLE_CACHING=true in .env

# Increase Qdrant HNSW parameters
# (Adjust in collection creation)

# Check database indexes
docker exec -it rag-postgres psql -U rag_user -d rag_db \
  -c "SELECT schemaname, tablename, indexname FROM pg_indexes WHERE tablename = 'chunks';"

# Monitor with Grafana
# Open http://localhost:3001 and check "Search Performance" dashboard
```

---

## Advanced Usage

### Custom Chunking Strategy

```bash
# Set chunking strategy per document
curl -X POST http://localhost:3000/api/v1/ingest/file \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@document.pdf" \
  -F "chunkingStrategy=semantic" \
  -F "chunkSize=1000"

# Options: recursive, semantic, html_aware, code_aware
```

### Adjust Embedding Model

```bash
# Edit .env
OPENAI_EMBEDDING_MODEL=text-embedding-3-small  # Cheaper, less accurate

# Restart services
docker-compose restart api embedding-worker
```

### Use Claude Instead of GPT-4

```bash
# Edit .env
DEFAULT_LLM_PROVIDER=anthropic

# Or specify per request
curl -X POST http://localhost:3000/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "message": "Explain this",
    "provider": "anthropic"
  }'
```

### Enable OCR for Scanned PDFs

```bash
# Edit .env
ENABLE_OCR=true

# Restart ingestion worker
docker-compose restart ingestion-worker

# Now scanned PDFs will be processed with Tesseract
```

### Monitoring & Metrics

**View Prometheus metrics:**
```bash
# Open Prometheus
open http://localhost:9090

# Example queries:
rag_ingestion_documents_total
rag_search_duration_seconds
rag_llm_tokens_total
```

**View Grafana dashboards:**
```bash
# Open Grafana
open http://localhost:3001

# Login: admin / admin
# Navigate to "Dashboards" > "RAG System Overview"
```

### Database Backups

```bash
# Backup PostgreSQL
docker exec rag-postgres pg_dump -U rag_user rag_db > backup.sql

# Backup Qdrant
curl -X POST http://localhost:6333/collections/document_chunks/snapshots

# Download snapshot
curl -O http://localhost:6333/collections/document_chunks/snapshots/$SNAPSHOT_ID
```

### Scaling Workers

```bash
# Scale ingestion workers
docker-compose up -d --scale ingestion-worker=4

# Scale embedding workers
docker-compose up -d --scale embedding-worker=4
```

---

## Next Steps

1. **Read the Documentation**: Review all `*.md` files for deep understanding
2. **Experiment**: Upload various document types, try different queries
3. **Customize**: Adjust chunking strategies, embedding models, LLM prompts
4. **Monitor**: Set up alerts in Grafana for production
5. **Optimize**: Tune performance based on your use case
6. **Scale**: Deploy to Kubernetes when ready for production

## Getting Help

- **Documentation**: See all `*.md` files in the repo
- **Issues**: https://github.com/yourusername/dev-RAG/issues
- **Logs**: Always check `docker-compose logs` for errors

---

**You're now ready to use the RAG system!** 🚀

Start by uploading a document and asking questions about it. The system will retrieve relevant chunks and generate accurate, cited answers.
