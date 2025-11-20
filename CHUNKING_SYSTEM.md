# Advanced Chunking System - Detailed Design

## Overview

The chunking system is critical for RAG quality. Poor chunking leads to:
- Context fragmentation (breaking mid-sentence)
- Information loss (splitting related concepts)
- Poor retrieval (chunks too small or too large)

Our system uses **adaptive chunking strategies** based on content type.

## Chunking Strategies

### 1. Recursive Text Splitter (General Text)

**Use Case**: General documents, articles, books

**Strategy**:
1. Try to split on paragraph boundaries (`\n\n`)
2. If chunks too large, split on sentences (`. `)
3. If still too large, split on clauses (`, `)
4. Last resort: split on word boundaries

**Parameters**:
- `chunk_size`: 800 tokens (≈ 3200 characters)
- `chunk_overlap`: 100 tokens (≈ 400 characters)
- `min_chunk_size`: 200 tokens
- `max_chunk_size`: 1000 tokens

**Implementation**:
```python
# apps/workers/ingestion-worker/src/chunkers/recursive_chunker.py

from typing import List, Dict, Any
import re
import tiktoken
from dataclasses import dataclass

@dataclass
class Chunk:
    content: str
    index: int
    token_count: int
    char_count: int
    metadata: Dict[str, Any]

class RecursiveChunker:
    """
    Recursively splits text into chunks, prioritizing natural boundaries.
    """

    def __init__(
        self,
        chunk_size: int = 800,
        chunk_overlap: int = 100,
        min_chunk_size: int = 200,
        max_chunk_size: int = 1000,
        encoding_name: str = "cl100k_base"  # GPT-4 tokenizer
    ):
        self.chunk_size = chunk_size
        self.chunk_overlap = chunk_overlap
        self.min_chunk_size = min_chunk_size
        self.max_chunk_size = max_chunk_size
        self.encoding = tiktoken.get_encoding(encoding_name)

        # Splitting hierarchy (in order of preference)
        self.separators = [
            "\n\n\n",  # Multiple blank lines
            "\n\n",    # Paragraph breaks
            "\n",      # Line breaks
            ". ",      # Sentence endings
            "! ",
            "? ",
            "; ",      # Clauses
            ", ",
            " ",       # Words
        ]

    def chunk(self, text: str, document_id: str = None) -> List[Chunk]:
        """
        Chunk text using recursive splitting.
        """
        chunks = []
        splits = self._recursive_split(text, self.separators)

        current_chunk = []
        current_tokens = 0
        chunk_index = 0

        for split in splits:
            split_tokens = len(self.encoding.encode(split))

            # If adding this split exceeds chunk size
            if current_tokens + split_tokens > self.chunk_size and current_chunk:
                # Save current chunk
                chunk_text = "".join(current_chunk).strip()
                if len(self.encoding.encode(chunk_text)) >= self.min_chunk_size:
                    chunks.append(
                        Chunk(
                            content=chunk_text,
                            index=chunk_index,
                            token_count=current_tokens,
                            char_count=len(chunk_text),
                            metadata={
                                "document_id": document_id,
                                "chunking_strategy": "recursive",
                            },
                        )
                    )
                    chunk_index += 1

                # Start new chunk with overlap
                overlap_text = self._get_overlap(current_chunk)
                current_chunk = [overlap_text] if overlap_text else []
                current_tokens = len(self.encoding.encode(overlap_text)) if overlap_text else 0

            current_chunk.append(split)
            current_tokens += split_tokens

        # Add final chunk
        if current_chunk:
            chunk_text = "".join(current_chunk).strip()
            if len(self.encoding.encode(chunk_text)) >= self.min_chunk_size:
                chunks.append(
                    Chunk(
                        content=chunk_text,
                        index=chunk_index,
                        token_count=current_tokens,
                        char_count=len(chunk_text),
                        metadata={
                            "document_id": document_id,
                            "chunking_strategy": "recursive",
                        },
                    )
                )

        return chunks

    def _recursive_split(self, text: str, separators: List[str]) -> List[str]:
        """
        Recursively split text using hierarchy of separators.
        """
        if not separators:
            # Base case: split on characters
            return list(text)

        separator = separators[0]
        splits = text.split(separator)

        result = []
        for split in splits:
            split_tokens = len(self.encoding.encode(split))

            if split_tokens > self.max_chunk_size:
                # Recursively split with next separator
                result.extend(self._recursive_split(split, separators[1:]))
            else:
                result.append(split + separator if separator != " " else split + " ")

        return result

    def _get_overlap(self, chunks: List[str]) -> str:
        """
        Get overlap text from previous chunk.
        """
        overlap_text = ""
        overlap_tokens = 0

        # Iterate backwards to get last N tokens
        for chunk in reversed(chunks):
            chunk_tokens = len(self.encoding.encode(chunk))
            if overlap_tokens + chunk_tokens > self.chunk_overlap:
                break
            overlap_text = chunk + overlap_text
            overlap_tokens += chunk_tokens

        return overlap_text


# Example usage
if __name__ == "__main__":
    chunker = RecursiveChunker(chunk_size=500, chunk_overlap=50)

    text = """
    Artificial intelligence (AI) is intelligence demonstrated by machines,
    in contrast to the natural intelligence displayed by humans and animals.

    Leading AI textbooks define the field as the study of "intelligent agents":
    any device that perceives its environment and takes actions that maximize
    its chance of successfully achieving its goals.

    The term "artificial intelligence" is often used to describe machines
    (or computers) that mimic "cognitive" functions that humans associate
    with the human mind, such as "learning" and "problem solving".
    """

    chunks = chunker.chunk(text)
    for chunk in chunks:
        print(f"Chunk {chunk.index}: {chunk.token_count} tokens")
        print(chunk.content[:100] + "...")
        print()
```

### 2. Semantic Chunker (Topic-Based)

**Use Case**: Long documents where content shifts topics

**Strategy**:
1. Embed sentences using a smaller model (e.g., all-MiniLM-L6-v2)
2. Calculate cosine similarity between consecutive sentences
3. Split when similarity drops below threshold (topic change)
4. Merge small chunks into larger ones

**Parameters**:
- `similarity_threshold`: 0.7 (lower = more splits)
- `min_chunk_size`: 200 tokens
- `max_chunk_size`: 1000 tokens

**Implementation**:
```python
# apps/workers/ingestion-worker/src/chunkers/semantic_chunker.py

from typing import List
import numpy as np
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity
import nltk
import tiktoken

nltk.download('punkt', quiet=True)

class SemanticChunker:
    """
    Chunks text based on semantic similarity between sentences.
    Splits when topic changes (low similarity).
    """

    def __init__(
        self,
        similarity_threshold: float = 0.7,
        min_chunk_size: int = 200,
        max_chunk_size: int = 1000,
        model_name: str = "all-MiniLM-L6-v2"
    ):
        self.similarity_threshold = similarity_threshold
        self.min_chunk_size = min_chunk_size
        self.max_chunk_size = max_chunk_size
        self.model = SentenceTransformer(model_name)
        self.encoding = tiktoken.get_encoding("cl100k_base")

    def chunk(self, text: str, document_id: str = None) -> List[Chunk]:
        """
        Chunk text based on semantic similarity.
        """
        # Split into sentences
        sentences = nltk.sent_tokenize(text)

        if len(sentences) < 2:
            # Not enough sentences to split
            return [
                Chunk(
                    content=text,
                    index=0,
                    token_count=len(self.encoding.encode(text)),
                    char_count=len(text),
                    metadata={"chunking_strategy": "semantic"},
                )
            ]

        # Embed sentences
        embeddings = self.model.encode(sentences)

        # Calculate similarities between consecutive sentences
        similarities = []
        for i in range(len(embeddings) - 1):
            sim = cosine_similarity(
                embeddings[i].reshape(1, -1),
                embeddings[i + 1].reshape(1, -1)
            )[0][0]
            similarities.append(sim)

        # Find split points (low similarity)
        split_indices = [0]
        for i, sim in enumerate(similarities):
            if sim < self.similarity_threshold:
                split_indices.append(i + 1)
        split_indices.append(len(sentences))

        # Create chunks
        chunks = []
        chunk_index = 0

        for i in range(len(split_indices) - 1):
            start = split_indices[i]
            end = split_indices[i + 1]
            chunk_sentences = sentences[start:end]

            # Merge small chunks
            chunk_text = " ".join(chunk_sentences)
            token_count = len(self.encoding.encode(chunk_text))

            # Skip if too small
            if token_count < self.min_chunk_size and i < len(split_indices) - 2:
                # Merge with next chunk
                continue

            # Split if too large
            if token_count > self.max_chunk_size:
                # Fall back to recursive splitter
                from .recursive_chunker import RecursiveChunker
                recursive_chunker = RecursiveChunker(
                    chunk_size=(self.min_chunk_size + self.max_chunk_size) // 2
                )
                sub_chunks = recursive_chunker.chunk(chunk_text, document_id)

                for sub_chunk in sub_chunks:
                    sub_chunk.index = chunk_index
                    sub_chunk.metadata["chunking_strategy"] = "semantic_recursive"
                    chunks.append(sub_chunk)
                    chunk_index += 1
            else:
                chunks.append(
                    Chunk(
                        content=chunk_text,
                        index=chunk_index,
                        token_count=token_count,
                        char_count=len(chunk_text),
                        metadata={
                            "document_id": document_id,
                            "chunking_strategy": "semantic",
                            "topic_break": True,
                        },
                    )
                )
                chunk_index += 1

        return chunks
```

### 3. HTML-Aware Chunker

**Use Case**: Web pages, documentation

**Strategy**:
1. Parse HTML structure (preserve hierarchy)
2. Split by semantic HTML tags (`<section>`, `<article>`, `<div>`)
3. Keep headings with their content
4. Preserve code blocks intact

**Implementation**:
```python
# apps/workers/ingestion-worker/src/chunkers/html_aware_chunker.py

from typing import List, Optional
from bs4 import BeautifulSoup, Tag
import tiktoken
from dataclasses import dataclass

class HTMLAwareChunker:
    """
    Chunks HTML content while preserving structure and hierarchy.
    """

    def __init__(
        self,
        chunk_size: int = 800,
        min_chunk_size: int = 200,
    ):
        self.chunk_size = chunk_size
        self.min_chunk_size = min_chunk_size
        self.encoding = tiktoken.get_encoding("cl100k_base")

        # Tags that indicate semantic boundaries
        self.boundary_tags = {'section', 'article', 'div', 'main', 'aside'}
        self.heading_tags = {'h1', 'h2', 'h3', 'h4', 'h5', 'h6'}
        self.preserve_tags = {'pre', 'code', 'table'}  # Don't split these

    def chunk(self, html: str, document_id: str = None) -> List[Chunk]:
        """
        Chunk HTML content.
        """
        soup = BeautifulSoup(html, 'html.parser')

        # Remove script, style, nav, footer
        for tag in soup(['script', 'style', 'nav', 'footer', 'header']):
            tag.decompose()

        chunks = []
        self._process_element(soup, chunks, document_id)

        # Renumber chunks
        for i, chunk in enumerate(chunks):
            chunk.index = i

        return chunks

    def _process_element(
        self,
        element: Tag,
        chunks: List[Chunk],
        document_id: str,
        parent_heading: str = None
    ):
        """
        Recursively process HTML elements.
        """
        if isinstance(element, str):
            return

        current_heading = parent_heading

        # Update heading context
        if element.name in self.heading_tags:
            current_heading = element.get_text().strip()

        # Check if this element should be preserved intact
        if element.name in self.preserve_tags:
            content = element.get_text()
            token_count = len(self.encoding.encode(content))

            chunks.append(
                Chunk(
                    content=content,
                    index=len(chunks),
                    token_count=token_count,
                    char_count=len(content),
                    metadata={
                        "document_id": document_id,
                        "chunking_strategy": "html_aware",
                        "element_type": element.name,
                        "parent_heading": current_heading,
                    },
                )
            )
            return

        # Check if this is a boundary element
        if element.name in self.boundary_tags:
            content = element.get_text().strip()
            token_count = len(self.encoding.encode(content))

            if token_count <= self.chunk_size:
                # Keep element intact
                if token_count >= self.min_chunk_size:
                    chunks.append(
                        Chunk(
                            content=content,
                            index=len(chunks),
                            token_count=token_count,
                            char_count=len(content),
                            metadata={
                                "document_id": document_id,
                                "chunking_strategy": "html_aware",
                                "element_type": element.name,
                                "parent_heading": current_heading,
                            },
                        )
                    )
                return
            # Else: too large, process children

        # Process children recursively
        for child in element.children:
            if isinstance(child, Tag):
                self._process_element(child, chunks, document_id, current_heading)
```

### 4. Code-Aware Chunker

**Use Case**: Source code files

**Strategy**:
1. Parse code with AST (Abstract Syntax Tree)
2. Split by functions/classes (keep definitions intact)
3. Include docstrings with code
4. Preserve imports and context

**Implementation**:
```python
# apps/workers/ingestion-worker/src/chunkers/code_aware_chunker.py

import ast
from typing import List
import tiktoken

class CodeAwareChunker:
    """
    Chunks source code while preserving function/class boundaries.
    """

    def __init__(
        self,
        chunk_size: int = 1000,
        include_imports: bool = True,
    ):
        self.chunk_size = chunk_size
        self.include_imports = include_imports
        self.encoding = tiktoken.get_encoding("cl100k_base")

    def chunk_python(self, code: str, document_id: str = None) -> List[Chunk]:
        """
        Chunk Python code.
        """
        try:
            tree = ast.parse(code)
        except SyntaxError:
            # Fallback to line-based splitting
            return self._fallback_chunk(code, document_id)

        chunks = []
        imports = []

        # Extract imports
        if self.include_imports:
            for node in ast.walk(tree):
                if isinstance(node, (ast.Import, ast.ImportFrom)):
                    imports.append(ast.get_source_segment(code, node))

        import_text = "\n".join(imports) + "\n\n" if imports else ""

        # Extract functions and classes
        for node in tree.body:
            if isinstance(node, (ast.FunctionDef, ast.ClassDef, ast.AsyncFunctionDef)):
                segment = ast.get_source_segment(code, node)

                # Add docstring metadata
                docstring = ast.get_docstring(node)

                content = import_text + segment
                token_count = len(self.encoding.encode(content))

                chunks.append(
                    Chunk(
                        content=content,
                        index=len(chunks),
                        token_count=token_count,
                        char_count=len(content),
                        metadata={
                            "document_id": document_id,
                            "chunking_strategy": "code_aware",
                            "language": "python",
                            "element_type": "function" if isinstance(node, ast.FunctionDef) else "class",
                            "element_name": node.name,
                            "docstring": docstring,
                        },
                    )
                )

        return chunks

    def _fallback_chunk(self, code: str, document_id: str) -> List[Chunk]:
        """
        Fallback: split code by lines.
        """
        lines = code.split('\n')
        chunks = []
        current_chunk = []
        current_tokens = 0

        for line in lines:
            line_tokens = len(self.encoding.encode(line))

            if current_tokens + line_tokens > self.chunk_size and current_chunk:
                chunk_text = "\n".join(current_chunk)
                chunks.append(
                    Chunk(
                        content=chunk_text,
                        index=len(chunks),
                        token_count=current_tokens,
                        char_count=len(chunk_text),
                        metadata={
                            "document_id": document_id,
                            "chunking_strategy": "code_fallback",
                        },
                    )
                )
                current_chunk = []
                current_tokens = 0

            current_chunk.append(line)
            current_tokens += line_tokens

        if current_chunk:
            chunk_text = "\n".join(current_chunk)
            chunks.append(
                Chunk(
                    content=chunk_text,
                    index=len(chunks),
                    token_count=current_tokens,
                    char_count=len(chunk_text),
                    metadata={"chunking_strategy": "code_fallback"},
                )
            )

        return chunks
```

## Chunk Quality Scoring

**Criteria**:
1. **Length**: Not too short, not too long (Gaussian distribution)
2. **Coherence**: Semantic similarity within chunk
3. **Information Density**: Unique words / total words
4. **Boundary Quality**: Starts/ends at natural boundaries

**Implementation**:
```python
# apps/workers/ingestion-worker/src/chunkers/quality_scorer.py

import numpy as np
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity
import tiktoken

class ChunkQualityScorer:
    """
    Scores chunk quality (0.0 to 1.0).
    """

    def __init__(self):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.encoding = tiktoken.get_encoding("cl100k_base")

        # Ideal chunk size (tokens)
        self.ideal_size = 600
        self.std_dev = 200

    def score(self, chunk: Chunk) -> float:
        """
        Calculate overall quality score.
        """
        length_score = self._score_length(chunk.token_count)
        coherence_score = self._score_coherence(chunk.content)
        density_score = self._score_information_density(chunk.content)
        boundary_score = self._score_boundary(chunk.content)

        # Weighted average
        overall_score = (
            0.3 * length_score +
            0.3 * coherence_score +
            0.2 * density_score +
            0.2 * boundary_score
        )

        return round(overall_score, 3)

    def _score_length(self, token_count: int) -> float:
        """
        Score based on Gaussian distribution around ideal size.
        """
        # Gaussian probability
        exponent = -((token_count - self.ideal_size) ** 2) / (2 * self.std_dev ** 2)
        return np.exp(exponent)

    def _score_coherence(self, text: str) -> float:
        """
        Score based on semantic coherence (sentence similarity).
        """
        sentences = [s.strip() for s in text.split('.') if s.strip()]

        if len(sentences) < 2:
            return 1.0  # Single sentence is perfectly coherent

        embeddings = self.model.encode(sentences)

        # Calculate pairwise similarities
        similarities = []
        for i in range(len(embeddings) - 1):
            sim = cosine_similarity(
                embeddings[i].reshape(1, -1),
                embeddings[i + 1].reshape(1, -1)
            )[0][0]
            similarities.append(sim)

        # Average similarity
        return np.mean(similarities)

    def _score_information_density(self, text: str) -> float:
        """
        Score based on unique words vs total words.
        """
        words = text.lower().split()
        if not words:
            return 0.0

        unique_ratio = len(set(words)) / len(words)

        # Normalize (ideal range: 0.5 to 0.8)
        if unique_ratio < 0.3:
            return 0.5  # Too repetitive
        elif unique_ratio > 0.9:
            return 0.7  # Too sparse
        else:
            return 1.0  # Good density

    def _score_boundary(self, text: str) -> float:
        """
        Score based on natural boundaries (starts/ends cleanly).
        """
        score = 0.5  # Base score

        # Check start
        if text[0].isupper():
            score += 0.2

        # Check end
        if text.rstrip().endswith(('.', '!', '?', '\n')):
            score += 0.3

        return min(score, 1.0)
```

## Metadata Injection

Each chunk should include rich metadata:

```python
chunk_metadata = {
    "document_id": "uuid",
    "chunk_index": 0,
    "chunking_strategy": "recursive",
    "parent_heading": "Introduction",
    "section_title": "Background",
    "page_number": 1,
    "quality_score": 0.85,
    "token_count": 650,
    "char_count": 2600,
    "created_at": "2024-01-15T10:30:00Z",
    "language": "en",
    "keywords": ["AI", "machine learning", "neural networks"],
    "entities": ["GPT-4", "OpenAI"],
}
```

## Deduplication Logic

After chunking, deduplicate similar chunks:

```python
def deduplicate_chunks(chunks: List[Chunk], threshold: float = 0.95) -> List[Chunk]:
    """
    Remove near-duplicate chunks based on cosine similarity.
    """
    from sentence_transformers import SentenceTransformer
    from sklearn.metrics.pairwise import cosine_similarity

    if len(chunks) < 2:
        return chunks

    model = SentenceTransformer('all-MiniLM-L6-v2')
    embeddings = model.encode([c.content for c in chunks])

    # Calculate pairwise similarities
    similarity_matrix = cosine_similarity(embeddings)

    # Mark duplicates
    to_remove = set()
    for i in range(len(chunks)):
        if i in to_remove:
            continue
        for j in range(i + 1, len(chunks)):
            if similarity_matrix[i][j] > threshold:
                # Keep the longer chunk
                if chunks[i].token_count > chunks[j].token_count:
                    to_remove.add(j)
                else:
                    to_remove.add(i)
                    break

    # Return unique chunks
    return [c for i, c in enumerate(chunks) if i not in to_remove]
```

## Chunking Strategy Selection

**Decision Tree**:
```
if file_type == "html" or file_type == "md":
    use HTMLAwareChunker
elif file_type in ["py", "js", "ts", "java"]:
    use CodeAwareChunker
elif document_length > 10000 tokens and detect_topic_shifts(text):
    use SemanticChunker
else:
    use RecursiveChunker (default)
```

## Performance Optimization

- **Batch Processing**: Chunk multiple documents in parallel
- **Caching**: Cache sentence embeddings for semantic chunking
- **Incremental**: Only re-chunk modified sections (future)

## Next: Embedding Pipeline

The chunks will be passed to the embedding worker for vector generation.
