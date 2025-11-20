# Embedding Pipeline - Detailed Design

## Overview

The embedding pipeline transforms text chunks into high-dimensional vectors that capture semantic meaning. These vectors enable similarity-based retrieval.

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    EMBEDDING QUEUE (RabbitMQ)                  │
│  Job: { chunk_ids: [...], batch_size: 64, priority: 1 }       │
└────────────┬───────────────────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────────────────────┐
│                    EMBEDDING WORKER (Python + Celery)          │
│  1. Fetch chunks from PostgreSQL                               │
│  2. Batch chunks (32-64 per API call)                          │
│  3. Generate embeddings via provider (OpenAI/Cohere)           │
│  4. Retry with exponential backoff                             │
│  5. Store in Qdrant with metadata                              │
│  6. Link chunk_id → vector_id in PostgreSQL                    │
│  7. Track costs and usage                                      │
└────────────────────────────────────────────────────────────────┘
```

## Embedding Model Selection

### Why OpenAI text-embedding-3-large?

**Pros**:
- State-of-art quality (3072 dimensions)
- Competitive pricing ($0.13 per 1M tokens)
- Fast inference (<100ms for batch)
- Good multilingual support
- Stable API, rarely down

**Cons**:
- Cloud-only (no self-hosted)
- Vendor lock-in

**Alternatives**:

| Model | Dims | Quality | Cost | Self-Hosted |
|-------|------|---------|------|-------------|
| OpenAI text-embedding-3-large | 3072 | Excellent | $0.13/1M | No |
| OpenAI text-embedding-3-small | 1536 | Good | $0.02/1M | No |
| Cohere embed-v3 | 1024 | Excellent | $0.10/1M | No |
| Voyage AI voyage-2 | 1024 | Excellent | $0.12/1M | No |
| sentence-transformers (local) | 384-768 | Good | Free | Yes |

**Decision**: Use OpenAI text-embedding-3-large for production, with fallback to self-hosted sentence-transformers for cost optimization.

## Batch Processing

### Why Batching?

- **Cost**: Fewer API calls
- **Speed**: Parallel embedding generation
- **Throughput**: Process 1000 chunks in 30 seconds vs. 15 minutes

### Optimal Batch Size

**OpenAI limits**:
- Max tokens per request: 8191
- Max array size: 2048 inputs
- Rate limit: 1M tokens/min

**Strategy**:
- Batch 32-64 chunks per request (≈ 50K tokens)
- Parallel workers (4-8) to maximize throughput
- Dynamic batching based on chunk size

## Implementation

### Embedding Worker (Python + Celery)

```python
# apps/workers/embedding-worker/src/worker.py

import os
from celery import Celery
from typing import List
import time
import logging

# Celery setup
app = Celery('embedding_worker')
app.config_from_object('celeryconfig')

logger = logging.getLogger(__name__)

@app.task(
    bind=True,
    max_retries=3,
    default_retry_delay=60,
    autoretry_for=(Exception,)
)
def generate_embeddings(self, chunk_ids: List[str], batch_size: int = 64):
    """
    Generate embeddings for a batch of chunks.
    """
    from src.embedding_generator import EmbeddingGenerator
    from src.storage.postgres_client import PostgresClient
    from src.storage.qdrant_client import QdrantClient

    try:
        # Initialize clients
        pg_client = PostgresClient()
        qdrant_client = QdrantClient()
        embedding_gen = EmbeddingGenerator()

        # Fetch chunks
        chunks = pg_client.get_chunks_by_ids(chunk_ids)

        if not chunks:
            logger.warning(f"No chunks found for IDs: {chunk_ids}")
            return

        # Generate embeddings in batches
        results = []
        for i in range(0, len(chunks), batch_size):
            batch = chunks[i:i + batch_size]
            batch_texts = [chunk['content'] for chunk in batch]

            # Generate embeddings
            embeddings = embedding_gen.generate(batch_texts)

            # Store in Qdrant
            for chunk, embedding in zip(batch, embeddings):
                vector_id = qdrant_client.upsert(
                    collection_name='document_chunks',
                    vector=embedding,
                    payload={
                        'chunk_id': chunk['id'],
                        'document_id': chunk['document_id'],
                        'user_id': chunk['user_id'],
                        'content': chunk['content'],
                        'chunk_index': chunk['chunk_index'],
                        'parent_heading': chunk['parent_heading'],
                        'page_number': chunk['page_number'],
                        'quality_score': chunk['quality_score'],
                        'created_at': chunk['created_at'].isoformat(),
                    }
                )

                # Update chunk with vector_id
                pg_client.update_chunk_vector_id(chunk['id'], vector_id)

                results.append({
                    'chunk_id': chunk['id'],
                    'vector_id': vector_id,
                })

            # Rate limiting
            time.sleep(0.1)

        logger.info(f"Generated {len(results)} embeddings successfully")
        return results

    except Exception as e:
        logger.error(f"Embedding generation failed: {e}")
        raise self.retry(exc=e, countdown=2 ** self.request.retries)


# Celery configuration
# celeryconfig.py
broker_url = os.getenv('CELERY_BROKER_URL', 'amqp://localhost')
result_backend = os.getenv('CELERY_RESULT_BACKEND', 'redis://localhost:6379/0')

task_serializer = 'json'
accept_content = ['json']
result_serializer = 'json'
timezone = 'UTC'
enable_utc = True

# Worker configuration
worker_prefetch_multiplier = 1
worker_max_tasks_per_child = 1000
task_acks_late = True
task_reject_on_worker_lost = True
```

### Embedding Generator

```python
# apps/workers/embedding-worker/src/embedding_generator.py

from typing import List
import openai
import os
import time
from tenacity import retry, stop_after_attempt, wait_exponential
import logging

logger = logging.getLogger(__name__)

class EmbeddingGenerator:
    """
    Generates embeddings using OpenAI API with retry logic.
    """

    def __init__(
        self,
        model: str = "text-embedding-3-large",
        api_key: str = None,
    ):
        self.model = model
        self.api_key = api_key or os.getenv('OPENAI_API_KEY')
        openai.api_key = self.api_key

        # Cost tracking (per 1M tokens)
        self.cost_per_million = {
            "text-embedding-3-large": 0.13,
            "text-embedding-3-small": 0.02,
        }

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10),
        reraise=True
    )
    def generate(self, texts: List[str]) -> List[List[float]]:
        """
        Generate embeddings for a list of texts.
        """
        start_time = time.time()

        try:
            response = openai.embeddings.create(
                input=texts,
                model=self.model,
            )

            embeddings = [item.embedding for item in response.data]

            # Calculate cost
            tokens_used = response.usage.total_tokens
            cost = (tokens_used / 1_000_000) * self.cost_per_million[self.model]

            duration = time.time() - start_time

            logger.info(
                f"Generated {len(embeddings)} embeddings in {duration:.2f}s. "
                f"Tokens: {tokens_used}, Cost: ${cost:.4f}"
            )

            # Track metrics (send to monitoring system)
            self._track_metrics(len(texts), tokens_used, cost, duration)

            return embeddings

        except openai.RateLimitError as e:
            logger.warning(f"Rate limit hit: {e}. Retrying...")
            raise
        except openai.APIError as e:
            logger.error(f"OpenAI API error: {e}")
            raise
        except Exception as e:
            logger.error(f"Unexpected error: {e}")
            raise

    def _track_metrics(
        self,
        batch_size: int,
        tokens_used: int,
        cost: float,
        duration: float
    ):
        """
        Send metrics to monitoring system (Prometheus/Datadog).
        """
        # Implementation here (e.g., send to statsd)
        pass


# Alternative: Cohere Provider
class CohereEmbeddingGenerator:
    def __init__(self, api_key: str = None):
        import cohere
        self.api_key = api_key or os.getenv('COHERE_API_KEY')
        self.client = cohere.Client(self.api_key)

    @retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1))
    def generate(self, texts: List[str]) -> List[List[float]]:
        response = self.client.embed(
            texts=texts,
            model='embed-english-v3.0',
            input_type='search_document'
        )
        return response.embeddings


# Local self-hosted option
class LocalEmbeddingGenerator:
    """
    Self-hosted using sentence-transformers (free, slower).
    """

    def __init__(self, model_name: str = "all-mpnet-base-v2"):
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer(model_name)

    def generate(self, texts: List[str]) -> List[List[float]]:
        embeddings = self.model.encode(texts, convert_to_numpy=True)
        return embeddings.tolist()
```

### Qdrant Client

```python
# apps/workers/embedding-worker/src/storage/qdrant_client.py

from qdrant_client import QdrantClient as QdrantSDK
from qdrant_client.models import Distance, VectorParams, PointStruct
from typing import List, Dict, Any
import os
import uuid

class QdrantClient:
    """
    Client for interacting with Qdrant vector database.
    """

    def __init__(
        self,
        url: str = None,
        api_key: str = None,
    ):
        self.url = url or os.getenv('QDRANT_URL', 'http://localhost:6333')
        self.api_key = api_key or os.getenv('QDRANT_API_KEY')

        self.client = QdrantSDK(
            url=self.url,
            api_key=self.api_key,
        )

    def ensure_collection(
        self,
        collection_name: str,
        vector_size: int = 3072,
        distance: Distance = Distance.COSINE,
    ):
        """
        Create collection if it doesn't exist.
        """
        collections = self.client.get_collections().collections
        collection_names = [c.name for c in collections]

        if collection_name not in collection_names:
            self.client.create_collection(
                collection_name=collection_name,
                vectors_config=VectorParams(
                    size=vector_size,
                    distance=distance,
                ),
            )
            print(f"Created collection: {collection_name}")

    def upsert(
        self,
        collection_name: str,
        vector: List[float],
        payload: Dict[str, Any],
        point_id: str = None,
    ) -> str:
        """
        Insert or update a vector.
        """
        point_id = point_id or str(uuid.uuid4())

        self.client.upsert(
            collection_name=collection_name,
            points=[
                PointStruct(
                    id=point_id,
                    vector=vector,
                    payload=payload,
                )
            ],
        )

        return point_id

    def search(
        self,
        collection_name: str,
        query_vector: List[float],
        limit: int = 10,
        score_threshold: float = None,
        filter: Dict[str, Any] = None,
    ) -> List[Dict[str, Any]]:
        """
        Search for similar vectors.
        """
        results = self.client.search(
            collection_name=collection_name,
            query_vector=query_vector,
            limit=limit,
            score_threshold=score_threshold,
            query_filter=filter,
        )

        return [
            {
                'id': result.id,
                'score': result.score,
                'payload': result.payload,
            }
            for result in results
        ]

    def delete(self, collection_name: str, point_ids: List[str]):
        """
        Delete vectors by IDs.
        """
        self.client.delete(
            collection_name=collection_name,
            points_selector={"ids": point_ids},
        )
```

## Versioning Strategy

**Problem**: Embedding models evolve. How to handle model upgrades?

**Solution**: Version embeddings and support multiple versions simultaneously.

```python
# Store model version in chunk metadata
chunk.embedding_model = "text-embedding-3-large"
chunk.embedding_version = "v1"

# In Qdrant, use separate collections per version
collection_name = f"document_chunks_{embedding_version}"

# Gradual migration:
# 1. Deploy new version in parallel
# 2. Re-embed documents incrementally
# 3. Switch search to new version
# 4. Deprecate old version
```

## Cost Optimization

### 1. Caching

Cache embeddings for duplicate chunks:

```python
import hashlib
import redis

class EmbeddingCache:
    def __init__(self):
        self.redis_client = redis.Redis(host='localhost', port=6379)
        self.ttl = 86400 * 30  # 30 days

    def get(self, text: str) -> List[float] | None:
        cache_key = self._get_key(text)
        cached = self.redis_client.get(cache_key)
        if cached:
            return json.loads(cached)
        return None

    def set(self, text: str, embedding: List[float]):
        cache_key = self._get_key(text)
        self.redis_client.setex(
            cache_key,
            self.ttl,
            json.dumps(embedding)
        )

    def _get_key(self, text: str) -> str:
        text_hash = hashlib.md5(text.encode()).hexdigest()
        return f"emb:v1:{text_hash}"
```

### 2. Lazy Loading

Don't embed all chunks immediately. Embed on first search/query.

### 3. Batch Prioritization

- **High priority**: User-uploaded documents (immediate)
- **Low priority**: Bulk imports (background, slower)

## Performance Metrics

Track these metrics:

- **Throughput**: Chunks embedded per second
- **Latency**: Time per batch
- **Cost**: $ per 1M tokens
- **Error Rate**: Failed embeddings / total
- **Queue Depth**: Pending chunks

## Next: Vector Search Layer

Now that embeddings are generated, we need a powerful search layer to retrieve relevant chunks.
