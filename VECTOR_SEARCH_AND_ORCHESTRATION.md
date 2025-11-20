# Vector Search & LLM Orchestration - Detailed Design

## PART 1: VECTOR SEARCH LAYER

### Hybrid Search Architecture

```
User Query: "What are the benefits of RAG systems?"
                    │
                    ▼
        ┌───────────────────────┐
        │   Query Processing    │
        │  - Normalize query    │
        │  - Extract filters    │
        │  - Detect intent      │
        └───────────┬───────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│  Vector Search   │    │  Keyword Search  │
│   (Qdrant)       │    │ (Elasticsearch)  │
│  - Embed query   │    │  - BM25          │
│  - Cosine sim    │    │  - TF-IDF        │
│  - Top 50        │    │  - Top 30        │
└────────┬─────────┘    └─────────┬────────┘
         │                        │
         └───────────┬────────────┘
                     ▼
        ┌───────────────────────┐
        │  Fusion (RRF)         │
        │  - Merge results      │
        │  - Weighted scoring   │
        │  - Apply filters      │
        │  - Boost recency      │
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │  Re-ranking           │
        │  - Cross-encoder      │
        │  - Query-chunk score  │
        │  - Top 5-10 chunks    │
        └───────────┬───────────┘
                    ▼
             [Final Results]
```

### Implementation: Hybrid Search Service

```typescript
// apps/api/src/modules/search/hybrid-search.service.ts

import { Injectable, Logger } from '@nestjs/common';
import { VectorSearchService } from './vector-search.service';
import { KeywordSearchService } from './keyword-search.service';
import { RerankerService } from './reranker.service';
import { SearchQueryDto } from './dto/search-query.dto';

export interface SearchResult {
  chunkId: string;
  documentId: string;
  content: string;
  score: number;
  metadata: Record<string, any>;
}

@Injectable()
export class HybridSearchService {
  private readonly logger = new Logger(HybridSearchService.name);

  constructor(
    private readonly vectorSearch: VectorSearchService,
    private readonly keywordSearch: KeywordSearchService,
    private readonly reranker: RerankerService,
  ) {}

  async search(query: SearchQueryDto, userId: string): Promise<SearchResult[]> {
    const startTime = Date.now();

    try {
      // 1. Parallel search
      const [vectorResults, keywordResults] = await Promise.all([
        this.vectorSearch.search(query, userId, 50),
        this.keywordSearch.search(query, userId, 30),
      ]);

      this.logger.debug(
        `Vector: ${vectorResults.length}, Keyword: ${keywordResults.length}`,
      );

      // 2. Reciprocal Rank Fusion (RRF)
      const fusedResults = this.reciprocalRankFusion(
        vectorResults,
        keywordResults,
        0.7, // Vector weight
        0.3, // Keyword weight
      );

      // 3. Apply metadata filters
      const filteredResults = this.applyFilters(fusedResults, query.filters);

      // 4. Boost recent documents
      const boostedResults = this.applyRecencyBoost(filteredResults);

      // 5. Re-rank top candidates
      const topCandidates = boostedResults.slice(0, 20);
      const rerankedResults = await this.reranker.rerank(
        query.query,
        topCandidates,
      );

      // 6. Return top K
      const finalResults = rerankedResults.slice(0, query.topK || 5);

      const duration = Date.now() - startTime;
      this.logger.log(
        `Search completed in ${duration}ms. Found ${finalResults.length} results.`,
      );

      return finalResults;
    } catch (error) {
      this.logger.error(`Search failed: ${error.message}`);
      throw error;
    }
  }

  /**
   * Reciprocal Rank Fusion (RRF)
   * Combines results from multiple sources using reciprocal ranks.
   *
   * score(d) = Σ (1 / (k + rank(d)))
   * where k = 60 (constant)
   */
  private reciprocalRankFusion(
    vectorResults: SearchResult[],
    keywordResults: SearchResult[],
    vectorWeight: number = 0.7,
    keywordWeight: number = 0.3,
  ): SearchResult[] {
    const k = 60; // RRF constant
    const scoreMap = new Map<string, { result: SearchResult; score: number }>();

    // Score vector results
    vectorResults.forEach((result, index) => {
      const rank = index + 1;
      const score = vectorWeight * (1 / (k + rank));

      scoreMap.set(result.chunkId, {
        result,
        score,
      });
    });

    // Score keyword results
    keywordResults.forEach((result, index) => {
      const rank = index + 1;
      const keywordScore = keywordWeight * (1 / (k + rank));

      if (scoreMap.has(result.chunkId)) {
        // Combine scores
        const existing = scoreMap.get(result.chunkId)!;
        existing.score += keywordScore;
      } else {
        scoreMap.set(result.chunkId, {
          result,
          score: keywordScore,
        });
      }
    });

    // Sort by combined score
    const sorted = Array.from(scoreMap.values())
      .sort((a, b) => b.score - a.score)
      .map((item) => ({
        ...item.result,
        score: item.score,
      }));

    return sorted;
  }

  private applyFilters(
    results: SearchResult[],
    filters?: {
      documentIds?: string[];
      fileTypes?: string[];
      dateRange?: { from: Date; to: Date };
      tags?: string[];
    },
  ): SearchResult[] {
    if (!filters) return results;

    return results.filter((result) => {
      // Filter by document IDs
      if (
        filters.documentIds &&
        !filters.documentIds.includes(result.documentId)
      ) {
        return false;
      }

      // Filter by file types
      if (
        filters.fileTypes &&
        !filters.fileTypes.includes(result.metadata.fileType)
      ) {
        return false;
      }

      // Filter by date range
      if (filters.dateRange) {
        const createdAt = new Date(result.metadata.createdAt);
        if (
          createdAt < filters.dateRange.from ||
          createdAt > filters.dateRange.to
        ) {
          return false;
        }
      }

      // Filter by tags
      if (filters.tags && filters.tags.length > 0) {
        const resultTags = result.metadata.tags || [];
        const hasTag = filters.tags.some((tag) => resultTags.includes(tag));
        if (!hasTag) return false;
      }

      return true;
    });
  }

  private applyRecencyBoost(results: SearchResult[]): SearchResult[] {
    const now = Date.now();
    const maxAge = 365 * 24 * 60 * 60 * 1000; // 1 year in ms

    return results.map((result) => {
      const createdAt = new Date(result.metadata.createdAt).getTime();
      const age = now - createdAt;

      // Exponential decay: newer = higher boost
      const recencyBoost = Math.exp(-age / maxAge) * 0.1; // Max 10% boost

      return {
        ...result,
        score: result.score * (1 + recencyBoost),
      };
    });
  }
}
```

### Re-ranking with Cross-Encoder

```typescript
// apps/api/src/modules/search/reranker.service.ts

import { Injectable } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class RerankerService {
  private readonly rerankerUrl = process.env.RERANKER_URL || 'http://localhost:5001';

  /**
   * Re-rank results using a cross-encoder model.
   * Cross-encoders score query-document pairs more accurately than bi-encoders.
   */
  async rerank(query: string, results: SearchResult[]): Promise<SearchResult[]> {
    if (results.length === 0) return [];

    try {
      // Call reranker API (Python service with cross-encoder)
      const response = await axios.post(`${this.rerankerUrl}/rerank`, {
        query,
        documents: results.map((r) => r.content),
      });

      const scores = response.data.scores as number[];

      // Combine with existing scores (weighted average)
      const reranked = results.map((result, index) => ({
        ...result,
        score: 0.5 * result.score + 0.5 * scores[index],
      }));

      // Sort by new scores
      reranked.sort((a, b) => b.score - a.score);

      return reranked;
    } catch (error) {
      // Fallback: return original ranking
      console.error('Reranking failed:', error.message);
      return results;
    }
  }
}
```

---

## PART 2: LLM ORCHESTRATION LAYER

### Architecture

```
Retrieved Chunks + User Query + Memory
                │
                ▼
    ┌───────────────────────┐
    │  Context Assembly     │
    │  - Format chunks      │
    │  - Add citations      │
    │  - Check token limit  │
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │  Prompt Construction  │
    │  - Select template    │
    │  - Inject context     │
    │  - Add constraints    │
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │  LLM Generation       │
    │  - Stream response    │
    │  - Parse citations    │
    │  - Validate output    │
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │  Post-Processing      │
    │  - Extract sources    │
    │  - Format markdown    │
    │  - Store in memory    │
    └───────────────────────┘
```

### Prompt Templates

```typescript
// apps/api/src/modules/chat/templates/qa-prompt.template.ts

export const QA_PROMPT_TEMPLATE = `You are a helpful AI assistant that answers questions based ONLY on the provided context.

STRICT RULES:
1. Only use information from the context below
2. Cite sources using [1], [2], etc.
3. If the answer is not in the context, say "I don't have enough information to answer that."
4. Be concise and accurate
5. Do not speculate or add information not in the context

CONTEXT:
{{#each chunks}}
[{{add @index 1}}] {{this.content}}
Source: {{this.documentTitle}}, Page {{this.pageNumber}}

{{/each}}

{{#if userMemory}}
USER MEMORY (from previous conversations):
{{userMemory}}
{{/if}}

QUESTION: {{query}}

ANSWER (cite sources and be concise):`;

export const SUMMARIZE_PROMPT_TEMPLATE = `Summarize the following documents concisely. Focus on key points and insights.

DOCUMENTS:
{{#each chunks}}
### {{this.documentTitle}}
{{this.content}}

{{/each}}

Provide a structured summary with:
- Main topics
- Key insights
- Important details

SUMMARY:`;

export const CHAT_PROMPT_TEMPLATE = `You are a helpful AI assistant engaged in a conversation. Use the context and conversation history to provide accurate, helpful responses.

CONTEXT FROM DOCUMENTS:
{{#each chunks}}
[{{add @index 1}}] {{this.content}}
{{/each}}

CONVERSATION HISTORY:
{{#each history}}
{{#if (eq this.role "user")}}
User: {{this.content}}
{{else}}
Assistant: {{this.content}}
{{/if}}
{{/each}}

{{#if userMemory}}
USER PREFERENCES & MEMORY:
{{userMemory}}
{{/if}}

User: {{query}}
Assistant:`;
```

### LLM Orchestrator Implementation

```typescript
// apps/api/src/modules/chat/llm-orchestrator.service.ts

import { Injectable, Logger } from '@nestjs/common';
import { OpenAI } from 'openai';
import { Anthropic } from '@anthropic-ai/sdk';
import { PromptBuilderService } from './prompt-builder.service';
import { SearchResult } from '../search/hybrid-search.service';

export interface LLMResponse {
  content: string;
  citations: Array<{
    chunkId: string;
    documentTitle: string;
    pageNumber?: number;
  }>;
  tokensUsed: number;
  model: string;
}

@Injectable()
export class LLMOrchestratorService {
  private readonly logger = new Logger(LLMOrchestratorService.name);
  private readonly openai: OpenAI;
  private readonly anthropic: Anthropic;

  constructor(private readonly promptBuilder: PromptBuilderService) {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY,
    });

    this.anthropic = new Anthropic({
      apiKey: process.env.ANTHROPIC_API_KEY,
    });
  }

  async generateResponse(
    query: string,
    chunks: SearchResult[],
    conversationHistory: any[] = [],
    userMemory?: string,
    provider: 'openai' | 'anthropic' = 'openai',
  ): Promise<LLMResponse> {
    // 1. Build prompt
    const prompt = await this.promptBuilder.buildQAPrompt(
      query,
      chunks,
      conversationHistory,
      userMemory,
    );

    // 2. Check token limit
    const tokenCount = this.estimateTokens(prompt);
    if (tokenCount > 7000) {
      // Truncate chunks
      chunks = this.truncateChunks(chunks, 5000);
    }

    // 3. Generate response
    if (provider === 'openai') {
      return this.generateWithOpenAI(prompt, chunks);
    } else {
      return this.generateWithAnthropic(prompt, chunks);
    }
  }

  private async generateWithOpenAI(
    prompt: string,
    chunks: SearchResult[],
  ): Promise<LLMResponse> {
    const model = process.env.OPENAI_CHAT_MODEL || 'gpt-4-turbo-preview';

    const response = await this.openai.chat.completions.create({
      model,
      messages: [
        {
          role: 'system',
          content: 'You are a helpful assistant that answers based on provided context.',
        },
        {
          role: 'user',
          content: prompt,
        },
      ],
      temperature: 0.3, // Low temperature for factual responses
      max_tokens: 1000,
      stream: false,
    });

    const content = response.choices[0].message.content || '';
    const citations = this.extractCitations(content, chunks);

    return {
      content,
      citations,
      tokensUsed: response.usage?.total_tokens || 0,
      model,
    };
  }

  private async generateWithAnthropic(
    prompt: string,
    chunks: SearchResult[],
  ): Promise<LLMResponse> {
    const model = process.env.ANTHROPIC_MODEL || 'claude-3-5-sonnet-20250220';

    const response = await this.anthropic.messages.create({
      model,
      max_tokens: 1000,
      temperature: 0.3,
      messages: [
        {
          role: 'user',
          content: prompt,
        },
      ],
    });

    const content =
      response.content[0].type === 'text' ? response.content[0].text : '';

    const citations = this.extractCitations(content, chunks);

    return {
      content,
      citations,
      tokensUsed: response.usage.input_tokens + response.usage.output_tokens,
      model,
    };
  }

  /**
   * Stream response for real-time UI updates
   */
  async *streamResponse(
    query: string,
    chunks: SearchResult[],
    conversationHistory: any[] = [],
    userMemory?: string,
  ): AsyncGenerator<string> {
    const prompt = await this.promptBuilder.buildQAPrompt(
      query,
      chunks,
      conversationHistory,
      userMemory,
    );

    const stream = await this.openai.chat.completions.create({
      model: 'gpt-4-turbo-preview',
      messages: [
        { role: 'system', content: 'You are a helpful assistant.' },
        { role: 'user', content: prompt },
      ],
      temperature: 0.3,
      max_tokens: 1000,
      stream: true,
    });

    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta?.content;
      if (delta) {
        yield delta;
      }
    }
  }

  private extractCitations(
    content: string,
    chunks: SearchResult[],
  ): LLMResponse['citations'] {
    // Extract [1], [2], etc. from content
    const citationPattern = /\[(\d+)\]/g;
    const matches = content.matchAll(citationPattern);
    const citations: LLMResponse['citations'] = [];

    for (const match of matches) {
      const index = parseInt(match[1]) - 1;
      if (index >= 0 && index < chunks.length) {
        const chunk = chunks[index];
        citations.push({
          chunkId: chunk.chunkId,
          documentTitle: chunk.metadata.documentTitle,
          pageNumber: chunk.metadata.pageNumber,
        });
      }
    }

    return citations;
  }

  private estimateTokens(text: string): number {
    // Rough estimate: 1 token ≈ 4 characters
    return Math.ceil(text.length / 4);
  }

  private truncateChunks(
    chunks: SearchResult[],
    maxTokens: number,
  ): SearchResult[] {
    let currentTokens = 0;
    const truncated: SearchResult[] = [];

    for (const chunk of chunks) {
      const chunkTokens = this.estimateTokens(chunk.content);
      if (currentTokens + chunkTokens > maxTokens) {
        break;
      }
      truncated.push(chunk);
      currentTokens += chunkTokens;
    }

    return truncated;
  }
}
```

### Hallucination Prevention

**Techniques**:
1. **Grounded prompts**: "Answer ONLY based on context"
2. **Citation requirement**: Force model to cite sources
3. **Low temperature**: 0.1-0.3 for factual responses
4. **Post-validation**: Check if cited chunks actually support the claim

```typescript
async validateResponse(
  response: string,
  chunks: SearchResult[],
): Promise<boolean> {
  // Use another LLM call to verify claims are supported by chunks
  const validationPrompt = `
    Does the following answer contain ONLY information from the provided chunks?
    Answer with YES or NO and explain any hallucinations.

    CHUNKS:
    ${chunks.map((c) => c.content).join('\n\n')}

    ANSWER:
    ${response}
  `;

  // Call LLM for validation
  // If hallucinations detected, regenerate or flag to user
}
```

## Next Steps

With these core components in place, we now need to:
1. Implement the full NestJS backend API
2. Build the React frontend
3. Set up Docker/Kubernetes deployment
4. Configure monitoring and logging
