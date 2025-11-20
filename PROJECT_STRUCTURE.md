# RAG System Project Structure

```
dev-RAG/
├── README.md
├── ARCHITECTURE.md
├── docker-compose.yml              # Local development environment
├── .env.example
├── .gitignore
├── package.json                    # Root workspace config
├── pnpm-workspace.yaml
├── turbo.json                      # Turborepo config
│
├── apps/
│   ├── api/                        # NestJS Backend API
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── app.module.ts
│   │   │   ├── config/
│   │   │   │   ├── database.config.ts
│   │   │   │   ├── qdrant.config.ts
│   │   │   │   ├── redis.config.ts
│   │   │   │   └── llm.config.ts
│   │   │   ├── modules/
│   │   │   │   ├── auth/
│   │   │   │   │   ├── auth.module.ts
│   │   │   │   │   ├── auth.controller.ts
│   │   │   │   │   ├── auth.service.ts
│   │   │   │   │   ├── guards/
│   │   │   │   │   └── strategies/
│   │   │   │   ├── ingest/
│   │   │   │   │   ├── ingest.module.ts
│   │   │   │   │   ├── ingest.controller.ts
│   │   │   │   │   ├── ingest.service.ts
│   │   │   │   │   ├── ingest.processor.ts
│   │   │   │   │   ├── dto/
│   │   │   │   │   │   ├── upload-document.dto.ts
│   │   │   │   │   │   └── ingest-url.dto.ts
│   │   │   │   │   └── parsers/
│   │   │   │   │       ├── pdf.parser.ts
│   │   │   │   │       ├── pptx.parser.ts
│   │   │   │   │       ├── text.parser.ts
│   │   │   │   │       └── url.parser.ts
│   │   │   │   ├── chunks/
│   │   │   │   │   ├── chunks.module.ts
│   │   │   │   │   ├── chunks.service.ts
│   │   │   │   │   ├── chunking-strategies/
│   │   │   │   │   │   ├── recursive.strategy.ts
│   │   │   │   │   │   ├── semantic.strategy.ts
│   │   │   │   │   │   ├── html-aware.strategy.ts
│   │   │   │   │   │   └── code-aware.strategy.ts
│   │   │   │   │   └── quality-scorer.ts
│   │   │   │   ├── embeddings/
│   │   │   │   │   ├── embeddings.module.ts
│   │   │   │   │   ├── embeddings.service.ts
│   │   │   │   │   ├── embeddings.processor.ts
│   │   │   │   │   └── providers/
│   │   │   │   │       ├── openai.provider.ts
│   │   │   │   │       └── cohere.provider.ts
│   │   │   │   ├── search/
│   │   │   │   │   ├── search.module.ts
│   │   │   │   │   ├── search.controller.ts
│   │   │   │   │   ├── search.service.ts
│   │   │   │   │   ├── vector-search.service.ts
│   │   │   │   │   ├── keyword-search.service.ts
│   │   │   │   │   ├── hybrid-search.service.ts
│   │   │   │   │   ├── reranker.service.ts
│   │   │   │   │   └── dto/
│   │   │   │   │       └── search-query.dto.ts
│   │   │   │   ├── chat/
│   │   │   │   │   ├── chat.module.ts
│   │   │   │   │   ├── chat.controller.ts
│   │   │   │   │   ├── chat.gateway.ts  # WebSocket
│   │   │   │   │   ├── chat.service.ts
│   │   │   │   │   ├── llm-orchestrator.service.ts
│   │   │   │   │   ├── prompt-builder.service.ts
│   │   │   │   │   ├── templates/
│   │   │   │   │   │   ├── qa-prompt.template.ts
│   │   │   │   │   │   ├── summarize-prompt.template.ts
│   │   │   │   │   │   └── chat-prompt.template.ts
│   │   │   │   │   └── dto/
│   │   │   │   │       └── chat-message.dto.ts
│   │   │   │   ├── memory/
│   │   │   │   │   ├── memory.module.ts
│   │   │   │   │   ├── memory.service.ts
│   │   │   │   │   ├── session-memory.service.ts
│   │   │   │   │   └── long-term-memory.service.ts
│   │   │   │   ├── documents/
│   │   │   │   │   ├── documents.module.ts
│   │   │   │   │   ├── documents.controller.ts
│   │   │   │   │   ├── documents.service.ts
│   │   │   │   │   └── dto/
│   │   │   │   └── admin/
│   │   │   │       ├── admin.module.ts
│   │   │   │       ├── admin.controller.ts
│   │   │   │       └── admin.service.ts
│   │   │   ├── common/
│   │   │   │   ├── decorators/
│   │   │   │   ├── filters/
│   │   │   │   ├── interceptors/
│   │   │   │   ├── pipes/
│   │   │   │   └── utils/
│   │   │   ├── database/
│   │   │   │   ├── entities/
│   │   │   │   │   ├── user.entity.ts
│   │   │   │   │   ├── document.entity.ts
│   │   │   │   │   ├── chunk.entity.ts
│   │   │   │   │   ├── embedding.entity.ts
│   │   │   │   │   ├── conversation.entity.ts
│   │   │   │   │   ├── message.entity.ts
│   │   │   │   │   └── memory-vector.entity.ts
│   │   │   │   ├── migrations/
│   │   │   │   └── seeds/
│   │   │   └── shared/
│   │   │       ├── interfaces/
│   │   │       ├── types/
│   │   │       └── constants/
│   │   ├── test/
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── nest-cli.json
│   │   └── Dockerfile
│   │
│   ├── web/                        # React Frontend
│   │   ├── public/
│   │   ├── src/
│   │   │   ├── main.tsx
│   │   │   ├── App.tsx
│   │   │   ├── pages/
│   │   │   │   ├── ChatPage.tsx
│   │   │   │   ├── UploadPage.tsx
│   │   │   │   ├── SearchPage.tsx
│   │   │   │   ├── DocumentsPage.tsx
│   │   │   │   ├── HistoryPage.tsx
│   │   │   │   ├── SettingsPage.tsx
│   │   │   │   └── AdminDashboard.tsx
│   │   │   ├── components/
│   │   │   │   ├── chat/
│   │   │   │   │   ├── ChatInterface.tsx
│   │   │   │   │   ├── MessageList.tsx
│   │   │   │   │   ├── MessageItem.tsx
│   │   │   │   │   ├── ChatInput.tsx
│   │   │   │   │   └── SourceCitation.tsx
│   │   │   │   ├── upload/
│   │   │   │   │   ├── FileUpload.tsx
│   │   │   │   │   ├── DragDropZone.tsx
│   │   │   │   │   ├── UploadProgress.tsx
│   │   │   │   │   └── ProcessingStatus.tsx
│   │   │   │   ├── search/
│   │   │   │   │   ├── SearchBar.tsx
│   │   │   │   │   ├── SearchResults.tsx
│   │   │   │   │   ├── SearchFilters.tsx
│   │   │   │   │   └── ResultCard.tsx
│   │   │   │   ├── documents/
│   │   │   │   │   ├── DocumentList.tsx
│   │   │   │   │   ├── DocumentCard.tsx
│   │   │   │   │   └── DocumentViewer.tsx
│   │   │   │   ├── common/
│   │   │   │   │   ├── Layout.tsx
│   │   │   │   │   ├── Sidebar.tsx
│   │   │   │   │   ├── Header.tsx
│   │   │   │   │   ├── Button.tsx
│   │   │   │   │   ├── Input.tsx
│   │   │   │   │   ├── Modal.tsx
│   │   │   │   │   ├── Spinner.tsx
│   │   │   │   │   └── Toast.tsx
│   │   │   │   └── admin/
│   │   │   │       ├── MetricsDashboard.tsx
│   │   │   │       ├── UserManagement.tsx
│   │   │   │       └── SystemLogs.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useChat.ts
│   │   │   │   ├── useSearch.ts
│   │   │   │   ├── useUpload.ts
│   │   │   │   ├── useDocuments.ts
│   │   │   │   └── useWebSocket.ts
│   │   │   ├── store/
│   │   │   │   ├── index.ts
│   │   │   │   ├── authStore.ts
│   │   │   │   ├── chatStore.ts
│   │   │   │   ├── documentsStore.ts
│   │   │   │   └── uiStore.ts
│   │   │   ├── services/
│   │   │   │   ├── api.ts
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── chat.service.ts
│   │   │   │   ├── documents.service.ts
│   │   │   │   ├── search.service.ts
│   │   │   │   └── websocket.service.ts
│   │   │   ├── types/
│   │   │   │   ├── api.types.ts
│   │   │   │   ├── chat.types.ts
│   │   │   │   ├── document.types.ts
│   │   │   │   └── user.types.ts
│   │   │   ├── utils/
│   │   │   │   ├── format.ts
│   │   │   │   ├── validation.ts
│   │   │   │   └── storage.ts
│   │   │   └── styles/
│   │   │       └── globals.css
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── vite.config.ts
│   │   ├── tailwind.config.js
│   │   ├── postcss.config.js
│   │   └── Dockerfile
│   │
│   └── workers/                    # Python Worker Services
│       ├── embedding-worker/
│       │   ├── src/
│       │   │   ├── __init__.py
│       │   │   ├── main.py
│       │   │   ├── worker.py
│       │   │   ├── embedding_generator.py
│       │   │   ├── batch_processor.py
│       │   │   ├── providers/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── openai_provider.py
│       │   │   │   ├── cohere_provider.py
│       │   │   │   └── huggingface_provider.py
│       │   │   ├── storage/
│       │   │   │   ├── __init__.py
│       │   │   │   ├── qdrant_client.py
│       │   │   │   └── postgres_client.py
│       │   │   └── utils/
│       │   │       ├── __init__.py
│       │   │       ├── retry.py
│       │   │       └── metrics.py
│       │   ├── tests/
│       │   ├── requirements.txt
│       │   ├── Dockerfile
│       │   └── README.md
│       │
│       └── ingestion-worker/
│           ├── src/
│           │   ├── __init__.py
│           │   ├── main.py
│           │   ├── worker.py
│           │   ├── parsers/
│           │   │   ├── __init__.py
│           │   │   ├── pdf_parser.py
│           │   │   ├── pptx_parser.py
│           │   │   ├── text_parser.py
│           │   │   ├── html_parser.py
│           │   │   └── ocr_engine.py
│           │   ├── chunkers/
│           │   │   ├── __init__.py
│           │   │   ├── base_chunker.py
│           │   │   ├── recursive_chunker.py
│           │   │   ├── semantic_chunker.py
│           │   │   ├── html_aware_chunker.py
│           │   │   ├── code_aware_chunker.py
│           │   │   └── quality_scorer.py
│           │   ├── preprocessors/
│           │   │   ├── __init__.py
│           │   │   ├── text_cleaner.py
│           │   │   ├── metadata_extractor.py
│           │   │   └── deduplicator.py
│           │   └── utils/
│           │       ├── __init__.py
│           │       └── s3_client.py
│           ├── tests/
│           ├── requirements.txt
│           ├── Dockerfile
│           └── README.md
│
├── packages/                       # Shared Libraries
│   ├── shared-types/               # TypeScript types
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── document.types.ts
│   │   │   ├── chunk.types.ts
│   │   │   ├── embedding.types.ts
│   │   │   ├── search.types.ts
│   │   │   └── api.types.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── shared-utils/               # Shared utilities
│       ├── src/
│       │   ├── index.ts
│       │   ├── validation.ts
│       │   ├── formatting.ts
│       │   └── constants.ts
│       ├── package.json
│       └── tsconfig.json
│
├── infra/                          # Infrastructure as Code
│   ├── docker/
│   │   ├── api.Dockerfile
│   │   ├── web.Dockerfile
│   │   ├── embedding-worker.Dockerfile
│   │   └── ingestion-worker.Dockerfile
│   │
│   ├── kubernetes/
│   │   ├── namespace.yaml
│   │   ├── configmaps/
│   │   ├── secrets/
│   │   ├── deployments/
│   │   │   ├── api-deployment.yaml
│   │   │   ├── web-deployment.yaml
│   │   │   ├── embedding-worker-deployment.yaml
│   │   │   ├── ingestion-worker-deployment.yaml
│   │   │   ├── qdrant-deployment.yaml
│   │   │   ├── postgres-deployment.yaml
│   │   │   ├── redis-deployment.yaml
│   │   │   └── elasticsearch-deployment.yaml
│   │   ├── services/
│   │   │   ├── api-service.yaml
│   │   │   ├── web-service.yaml
│   │   │   ├── qdrant-service.yaml
│   │   │   ├── postgres-service.yaml
│   │   │   └── redis-service.yaml
│   │   ├── ingress/
│   │   │   └── ingress.yaml
│   │   ├── persistent-volumes/
│   │   │   ├── qdrant-pv.yaml
│   │   │   ├── postgres-pv.yaml
│   │   │   └── minio-pv.yaml
│   │   └── jobs/
│   │       └── db-migration-job.yaml
│   │
│   ├── terraform/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── modules/
│   │   │   ├── vpc/
│   │   │   ├── eks/
│   │   │   ├── rds/
│   │   │   └── s3/
│   │   └── environments/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── prod/
│   │
│   └── monitoring/
│       ├── prometheus/
│       │   ├── prometheus.yml
│       │   └── alerts.yml
│       ├── grafana/
│       │   └── dashboards/
│       │       ├── rag-system-overview.json
│       │       ├── ingestion-metrics.json
│       │       └── search-performance.json
│       └── elasticsearch/
│           └── index-templates/
│
├── scripts/
│   ├── setup.sh                    # Initial setup
│   ├── seed-db.sh                  # Database seeding
│   ├── generate-types.sh           # Type generation
│   ├── docker-build.sh             # Build all images
│   └── deploy.sh                   # Deploy to k8s
│
├── docs/
│   ├── API.md                      # API documentation
│   ├── DEPLOYMENT.md               # Deployment guide
│   ├── DEVELOPMENT.md              # Development guide
│   └── TROUBLESHOOTING.md          # Common issues
│
└── .github/
    └── workflows/
        ├── ci.yml                  # CI pipeline
        ├── cd.yml                  # CD pipeline
        ├── test.yml                # Run tests
        └── docker-build.yml        # Build images
```

## Key Design Decisions

### Monorepo Structure
- **Tool**: Turborepo with pnpm workspaces
- **Why**: Shared code, unified builds, atomic changes across services
- **Alternative**: Separate repos per service (harder to maintain consistency)

### Backend in NestJS (TypeScript)
- **Why**: Enterprise DI, decorators, built-in testing, TypeORM integration
- **Alternative**: Express (too minimal), Fastify (less ecosystem)

### Workers in Python
- **Why**: Rich ML/NLP libraries (spaCy, sentence-transformers), better PDF/PPTX parsers
- **Alternative**: Node.js (weaker ML ecosystem)

### Frontend in React
- **Why**: Largest ecosystem, mature streaming support, easy SSR with Vite
- **Alternative**: Vue (smaller community), Svelte (less enterprise adoption)

## File Naming Conventions

- **TypeScript**: `kebab-case.ts` (e.g., `llm-orchestrator.service.ts`)
- **React Components**: `PascalCase.tsx` (e.g., `ChatInterface.tsx`)
- **Python**: `snake_case.py` (e.g., `semantic_chunker.py`)
- **Config**: `lowercase.config.ts` (e.g., `qdrant.config.ts`)
- **DTOs**: `*.dto.ts` (e.g., `search-query.dto.ts`)
- **Entities**: `*.entity.ts` (e.g., `document.entity.ts`)
