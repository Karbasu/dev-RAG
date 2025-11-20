# Quick Reference Guide

Fast reference for common commands and API calls.

## Docker Commands

```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs (all services)
docker-compose logs -f

# View logs (specific service)
docker-compose logs -f api

# Restart a service
docker-compose restart api

# Check service status
docker-compose ps

# Rebuild images
docker-compose build

# Scale workers
docker-compose up -d --scale embedding-worker=4
```

## pnpm Commands

```bash
# Install dependencies
pnpm install

# Run API in dev mode
pnpm api:dev

# Run frontend in dev mode
pnpm web:dev

# Build all
pnpm build

# Run tests
pnpm test

# Run linter
pnpm lint

# Database migrations
pnpm db:migrate
pnpm migration:generate
pnpm migration:revert
```

## API Authentication

```bash
# Set your token as environment variable
export TOKEN="your-jwt-token-here"

# Or include in every request:
-H "Authorization: Bearer your-jwt-token"
```

### Register
```bash
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "Password123!",
    "fullName": "John Doe"
  }'
```

### Login
```bash
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "Password123!"
  }'
```

## Document Ingestion

### Upload File
```bash
curl -X POST http://localhost:3000/api/v1/ingest/file \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@document.pdf"
```

### Upload with Options
```bash
curl -X POST http://localhost:3000/api/v1/ingest/file \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@document.pdf" \
  -F "chunkingStrategy=semantic" \
  -F "chunkSize=1000"
```

### Ingest URL
```bash
curl -X POST http://localhost:3000/api/v1/ingest/url \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/article",
    "title": "Article Title"
  }'
```

### Check Status
```bash
curl http://localhost:3000/api/v1/ingest/status/$JOB_ID \
  -H "Authorization: Bearer $TOKEN"
```

## Document Management

### List Documents
```bash
curl http://localhost:3000/api/v1/documents \
  -H "Authorization: Bearer $TOKEN"
```

### Get Document
```bash
curl http://localhost:3000/api/v1/documents/$DOC_ID \
  -H "Authorization: Bearer $TOKEN"
```

### Get Chunks
```bash
curl http://localhost:3000/api/v1/documents/$DOC_ID/chunks \
  -H "Authorization: Bearer $TOKEN"
```

### Delete Document
```bash
curl -X DELETE http://localhost:3000/api/v1/documents/$DOC_ID \
  -H "Authorization: Bearer $TOKEN"
```

## Search

### Basic Search
```bash
curl -X POST http://localhost:3000/api/v1/search \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is RAG?",
    "topK": 5
  }'
```

### Search with Filters
```bash
curl -X POST http://localhost:3000/api/v1/search \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "machine learning",
    "topK": 10,
    "filters": {
      "fileTypes": ["pdf"],
      "dateRange": {
        "from": "2024-01-01",
        "to": "2024-12-31"
      },
      "tags": ["ai", "ml"]
    }
  }'
```

## Chat

### Send Message
```bash
curl -X POST http://localhost:3000/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Explain RAG systems",
    "conversationId": null
  }'
```

### Continue Conversation
```bash
curl -X POST http://localhost:3000/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What are the benefits?",
    "conversationId": "previous-conv-id"
  }'
```

### Use Specific Provider
```bash
curl -X POST http://localhost:3000/api/v1/chat \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Explain this",
    "provider": "anthropic"
  }'
```

### List Conversations
```bash
curl http://localhost:3000/api/v1/chat/conversations \
  -H "Authorization: Bearer $TOKEN"
```

### Get Conversation
```bash
curl http://localhost:3000/api/v1/chat/conversations/$CONV_ID \
  -H "Authorization: Bearer $TOKEN"
```

## Database Operations

### PostgreSQL
```bash
# Connect to database
docker exec -it rag-postgres psql -U rag_user -d rag_db

# Common queries
SELECT COUNT(*) FROM documents;
SELECT COUNT(*) FROM chunks;
SELECT * FROM documents WHERE status = 'indexed';
SELECT * FROM chunks WHERE document_id = 'uuid';
```

### Qdrant
```bash
# List collections
curl http://localhost:6333/collections

# Get collection info
curl http://localhost:6333/collections/document_chunks

# Count vectors
curl http://localhost:6333/collections/document_chunks/points/count

# Search (for testing)
curl -X POST http://localhost:6333/collections/document_chunks/points/search \
  -H "Content-Type: application/json" \
  -d '{
    "vector": [0.1, 0.2, ...],
    "limit": 5
  }'
```

### Redis
```bash
# Connect
docker exec -it rag-redis redis-cli

# Common commands
KEYS *
GET key
TTL key
FLUSHALL  # Clear all (careful!)
```

### Elasticsearch
```bash
# Cluster health
curl http://localhost:9200/_cluster/health

# List indices
curl http://localhost:9200/_cat/indices

# Search documents
curl http://localhost:9200/documents/_search?q=RAG
```

## Monitoring

### Prometheus Metrics
```bash
# Open Prometheus
open http://localhost:9090

# Example queries:
rag_ingestion_documents_total
rate(rag_search_duration_seconds[5m])
rag_llm_tokens_total
rag_queue_depth
```

### Grafana
```bash
# Open Grafana
open http://localhost:3001
# Login: admin / admin
```

### Check Service Health
```bash
# API
curl http://localhost:3000/health

# Qdrant
curl http://localhost:6333/health

# Elasticsearch
curl http://localhost:9200/_cluster/health

# RabbitMQ
curl -u admin:admin http://localhost:15672/api/health/checks/alarms
```

## Backup & Restore

### PostgreSQL
```bash
# Backup
docker exec rag-postgres pg_dump -U rag_user rag_db > backup.sql

# Restore
docker exec -i rag-postgres psql -U rag_user rag_db < backup.sql
```

### Qdrant
```bash
# Create snapshot
curl -X POST http://localhost:6333/collections/document_chunks/snapshots

# List snapshots
curl http://localhost:6333/collections/document_chunks/snapshots

# Download snapshot
curl -O http://localhost:6333/snapshots/document_chunks/$SNAPSHOT_NAME
```

## Troubleshooting Commands

### View Logs
```bash
# All services
docker-compose logs -f

# API only
docker-compose logs -f api

# Workers
docker-compose logs -f ingestion-worker embedding-worker

# Last 100 lines
docker-compose logs --tail=100 api
```

### Check Resource Usage
```bash
# Docker stats
docker stats

# Disk usage
docker system df

# Container processes
docker-compose top
```

### Restart Services
```bash
# Restart all
docker-compose restart

# Restart specific service
docker-compose restart api

# Restart workers
docker-compose restart ingestion-worker embedding-worker
```

### Clean Up
```bash
# Remove stopped containers
docker-compose down

# Remove with volumes (careful!)
docker-compose down -v

# Clean Docker system
docker system prune -a

# Clean specific volumes
docker volume rm rag_postgres_data
```

## Environment Variables

### Required
```bash
OPENAI_API_KEY=sk-proj-...
DATABASE_PASSWORD=secure_password
JWT_SECRET=your_jwt_secret_min_32_chars
```

### Optional (Common)
```bash
# Anthropic (for Claude)
ANTHROPIC_API_KEY=sk-ant-...

# Chunking
MAX_CHUNK_SIZE=1000
MIN_CHUNK_SIZE=200
CHUNK_OVERLAP=100

# Features
ENABLE_OCR=false
ENABLE_CACHING=true
ENABLE_RERANKING=true

# Models
OPENAI_EMBEDDING_MODEL=text-embedding-3-large
OPENAI_CHAT_MODEL=gpt-4-turbo-preview
ANTHROPIC_MODEL=claude-3-5-sonnet-20250220
```

## Useful Scripts

### Test Full Pipeline
```bash
#!/bin/bash
# Upload document
RESPONSE=$(curl -s -X POST http://localhost:3000/api/v1/ingest/file \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@test.pdf")

DOC_ID=$(echo $RESPONSE | jq -r '.documentId')
JOB_ID=$(echo $RESPONSE | jq -r '.jobId')

# Wait for completion
while true; do
  STATUS=$(curl -s http://localhost:3000/api/v1/ingest/status/$JOB_ID \
    -H "Authorization: Bearer $TOKEN" | jq -r '.status')
  echo "Status: $STATUS"
  [ "$STATUS" = "completed" ] && break
  sleep 2
done

# Search
curl -X POST http://localhost:3000/api/v1/search \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "test query", "topK": 5}'
```

### Monitor Queue Depth
```bash
#!/bin/bash
while true; do
  curl -s -u admin:admin http://localhost:15672/api/queues | \
    jq -r '.[] | "\(.name): \(.messages)"'
  sleep 5
done
```

### Check All Services
```bash
#!/bin/bash
echo "Checking services..."
docker-compose ps
echo ""
echo "API Health:"
curl -s http://localhost:3000/health | jq
echo ""
echo "Qdrant Health:"
curl -s http://localhost:6333/health
echo ""
echo "Elasticsearch Health:"
curl -s http://localhost:9200/_cluster/health | jq
```

## Common Issues & Quick Fixes

### Port Already in Use
```bash
# Find what's using the port
lsof -i :5432  # PostgreSQL
lsof -i :6379  # Redis

# Stop the service
sudo systemctl stop postgresql
```

### Out of Memory
```bash
# Check Docker memory
docker stats

# Increase in Docker Desktop: Settings > Resources > Memory

# Or reduce worker concurrency
docker-compose up -d --scale embedding-worker=1
```

### Worker Not Processing
```bash
# Check RabbitMQ
open http://localhost:15672

# Check worker logs
docker-compose logs -f embedding-worker

# Restart workers
docker-compose restart ingestion-worker embedding-worker
```

### Qdrant Connection Issues
```bash
# Check if running
docker-compose ps qdrant

# Check health
curl http://localhost:6333/health

# Restart
docker-compose restart qdrant
```

## Performance Tips

```bash
# Enable Redis caching
ENABLE_CACHING=true

# Reduce embedding batch size (if hitting rate limits)
EMBEDDING_BATCH_SIZE=32

# Use smaller embedding model
OPENAI_EMBEDDING_MODEL=text-embedding-3-small

# Scale workers for faster ingestion
docker-compose up -d --scale ingestion-worker=4

# Monitor performance
open http://localhost:3001  # Grafana
```

## Access URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Frontend | http://localhost:5173 | - |
| API | http://localhost:3000 | JWT token |
| API Docs | http://localhost:3000/api/docs | - |
| PostgreSQL | localhost:5432 | rag_user / rag_password |
| Redis | localhost:6379 | - |
| Qdrant | http://localhost:6333 | - |
| Elasticsearch | http://localhost:9200 | - |
| MinIO Console | http://localhost:9001 | minioadmin / minioadmin |
| RabbitMQ Console | http://localhost:15672 | admin / admin |
| Prometheus | http://localhost:9090 | - |
| Grafana | http://localhost:3001 | admin / admin |

---

**For detailed explanations, see [SETUP_AND_USAGE.md](SETUP_AND_USAGE.md)**
