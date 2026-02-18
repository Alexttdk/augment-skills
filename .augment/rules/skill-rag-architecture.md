---
type: agent_requested
description: Design and implement RAG pipelines: chunking strategies, embedding model selection, vector DB comparison, hybrid retrieval, reranking, HyDE, and RAGAS evaluation.
---

# RAG Architecture

## When to Use
Activate when asked to:
- Build Q&A or chat systems over documents, codebases, or private knowledge bases
- Implement semantic search or hybrid search pipelines
- Connect LLMs to proprietary/private data to reduce hallucinations
- Choose and configure vector databases, embedding models, or chunking strategies
- Improve retrieval quality with reranking, query expansion, or HyDE
- Evaluate RAG systems with RAGAS, TruLens, or custom metrics

## RAG Pipeline Overview

```
Documents → Chunking → Embedding → Vector Store
                                        ↓
User Query → Query Processing → Retrieval (hybrid) → Reranking → Prompt Augmentation → LLM → Answer
```

| Stage | Component | Common Tools |
|-------|-----------|--------------|
| Ingestion | Load + clean docs | LangChain loaders, Unstructured.io |
| Chunking | Split semantically | RecursiveCharacterTextSplitter, semantic splitter |
| Embedding | Encode to vectors | OpenAI, Cohere, BGE |
| Storage | Index + persist | Pinecone, Qdrant, Weaviate, pgvector |
| Retrieval | Fetch top-k | Cosine similarity + BM25 hybrid |
| Reranking | Re-score relevance | Cohere Rerank, cross-encoder |
| Generation | Grounded answer | Any LLM via context-stuffed prompt |

## Chunking Strategies

| Strategy | Size | Overlap | Best For | Avoid When |
|----------|------|---------|----------|------------|
| Fixed-size | 256–512 tok | 50–100 tok | Generic text baseline | Structured docs |
| Recursive character | 200–800 tok | 10–15% | General purpose (default) | — |
| Sentence-based | 1–5 sentences | 1 sentence | Factual Q&A, news | Long dense paragraphs |
| Semantic | 100–500 tok | None | Technical dense docs | Need fast ingestion |
| Markdown-aware | Per section | 0–50 tok | Docs, READMEs | Non-structured text |
| Code-aware (AST) | Function/class | 0 | Codebase search | Non-code content |
| Parent-child | Small/large pair | N/A | Legal, long-form docs | Simple FAQs |

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,
    separators=["\n\n", "\n", ". ", " ", ""]
)
chunks = splitter.split_documents(documents)
```

**Size guidelines:** Q&A → 100–200 tok (sentences). Tech docs → 300–512 tok (recursive). Legal/medical → 512–800 tok + parent-child. Code → function-level AST.

## Embedding Model Selection

| Model | Dimensions | Max Tokens | Hosted | Best For |
|-------|-----------|------------|--------|----------|
| `text-embedding-3-large` | 3072 | 8191 | OpenAI API | High-accuracy production |
| `text-embedding-3-small` | 1536 | 8191 | OpenAI API | Cost-efficient production |
| `cohere-embed-v3` | 1024 | 512 | Cohere API | Multilingual, search-optimized |
| `voyage-large-2` | 1536 | 16000 | Voyage API | Long documents, code |
| `BAAI/bge-large-en-v1.5` | 1024 | 512 | Self-hosted | Best free English quality |
| `BAAI/bge-m3` | 1024 | 8192 | Self-hosted | Multilingual self-hosted |
| `nomic-embed-text` | 768 | 8192 | Self-hosted | Local dev, good quality |

**Selection logic:**
```
Multilingual required? → bge-m3 or cohere-embed-v3
Self-hosted required?  → bge-large-en-v1.5 or nomic-embed-text
Documents > 4K tokens? → voyage-large-2
Cost-sensitive?        → text-embedding-3-small
Default production:    → text-embedding-3-large
```

## Vector Database Comparison

| DB | Deployment | Metadata Filter | Hybrid Search | Best For |
|----|-----------|----------------|---------------|----------|
| Pinecone | Managed SaaS | Yes | Sparse+dense | Managed production |
| Qdrant | Self/Cloud | Rich filters | BM25+vector | High-perf OSS |
| Weaviate | Self/Cloud | Yes | BM25+vector | Full-feature OSS |
| Chroma | Local/server | Basic | No | Dev/prototyping |
| pgvector | PostgreSQL | Full SQL | With pg_search | Existing PG stack |
| Milvus | Self/Cloud | Yes | Yes | Billion-scale |

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

client = QdrantClient(":memory:")  # or url="http://localhost:6333"
client.create_collection("docs", vectors_config=VectorParams(size=1536, distance=Distance.COSINE))
client.upsert("docs", points=[
    PointStruct(id=i, vector=emb, payload={"text": chunk, "source": url})
    for i, (emb, chunk, url) in enumerate(zip(embeddings, chunks, sources))
])
results = client.search("docs", query_vector=query_emb, limit=5, with_payload=True)
```

**Decision:** Managed + simple → Pinecone. OSS + rich filtering → Qdrant. Already on Postgres → pgvector. Prototyping → Chroma.



## Retrieval Patterns

### Naive Retrieval (baseline)
```python
results = vector_db.search(query_embedding, limit=5)
context = "\n\n".join(r.payload["text"] for r in results)
```

### Hybrid Search (Semantic + Keyword)
```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25     = BM25Retriever.from_documents(docs, k=5)
semantic = vector_db.as_retriever(search_kwargs={"k": 5})
hybrid   = EnsembleRetriever(retrievers=[bm25, semantic], weights=[0.4, 0.6])
results  = hybrid.invoke(query)
```

**Reciprocal Rank Fusion (RRF)** — merge ranked lists without score normalization:
```python
def rrf_score(rank: int, k: int = 60) -> float:
    return 1 / (k + rank)

def rrf_merge(lists: list[list[str]]) -> list[str]:
    scores: dict[str, float] = {}
    for ranked_list in lists:
        for rank, doc_id in enumerate(ranked_list, start=1):
            scores[doc_id] = scores.get(doc_id, 0) + rrf_score(rank)
    return sorted(scores, key=scores.get, reverse=True)
```

### Reranking
```python
import cohere
co = cohere.Client(api_key)
reranked = co.rerank(
    model="rerank-english-v3.0",
    query=query,
    documents=[r.payload["text"] for r in initial_results],
    top_n=3
)
top_chunks = [initial_results[r.index] for r in reranked.results]
```
**When to use:** Always in production. Adds ~50–200 ms but improves precision@3 by 15–30%.
**Alternatives:** `cross-encoder/ms-marco-MiniLM-L-12-v2` (free, self-hosted cross-encoder).

### HyDE (Hypothetical Document Embedding)
```python
# Generate a hypothetical answer, embed it, use that embedding to search
hyde_prompt = f"Write a paragraph that would answer: {query}"
hypothetical_doc = llm.invoke(hyde_prompt)
hyde_embedding   = embed(hypothetical_doc)
results = vector_db.search(hyde_embedding, limit=5)
```
**Best for:** Short/keyword queries that use different vocabulary than stored documents.

### Query Expansion
```python
expansion_prompt = f"""Generate 3 distinct search queries to find information about: {query}
Return as a numbered list, one query per line."""
expanded_queries = llm.invoke(expansion_prompt).split("\n")
# Run all queries, deduplicate results, rerank combined set
```

## Evaluation Metrics

### Retrieval Metrics
| Metric | What It Measures | Target |
|--------|-----------------|--------|
| Precision@k | (relevant in top-k) / k | > 0.70 |
| Recall@k | (relevant in top-k) / total relevant | > 0.80 |
| MRR | 1 / rank of first relevant result | > 0.60 |
| NDCG@k | Normalized discounted cumulative gain | > 0.70 |

### Generation Metrics (RAGAS)
```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_recall, context_precision

results = evaluate(
    dataset=eval_dataset,   # HF Dataset with: question, contexts, answer, ground_truth
    metrics=[faithfulness, answer_relevancy, context_recall, context_precision]
)
# faithfulness:        answer grounded in context?   (0–1, target > 0.90)
# answer_relevancy:   answer addresses the question? (0–1, target > 0.80)
# context_recall:     retrieval found needed info?   (0–1, target > 0.80)
# context_precision:  retrieved context is relevant? (0–1, target > 0.70)
```

### Production Monitoring
- Log: query, chunk IDs retrieved, scores, latency, final answer, user feedback
- Track: avg faithfulness, answer relevancy per category per week
- Alert on: retrieval latency > 500 ms, faithfulness < 0.70, thumbs-down rate > 10%
- Sample 5–10% of production queries weekly for offline RAGAS evaluation

## Common Pitfalls

| Problem | Symptom | Fix |
|---------|---------|-----|
| Chunks too large | Off-topic context, low precision | Reduce size; add reranker |
| Chunks too small | Truncated answers, missing context | Increase size; use parent-child |
| Wrong embedding model | Semantically unrelated results | Benchmark models on your domain data |
| Missing metadata filter | Wrong document version retrieved | Add date/source filter at query time |
| No reranking | Top-1 result often irrelevant | Add Cohere Rerank or cross-encoder |
| Context window overflow | LLM ignores later chunks | Reduce top-k; use contextual compression |

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/rag-architect
- RAGAS: https://docs.ragas.io
- LangChain RAG: https://python.langchain.com/docs/use_cases/question_answering
- Qdrant: https://qdrant.tech/documentation
