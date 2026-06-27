# Vector Search Guide

## Overview

Couchbase Vector Search stores and queries float32 embedding vectors alongside JSON documents. It enables semantic similarity search, recommendation systems, RAG (Retrieval-Augmented Generation), and hybrid text+vector queries.

Supported similarity metrics:
- `dot_product` — fastest; requires normalized (unit) vectors (recommended for most embedding models)
- `cosine` — cosine similarity; normalized internally
- `l2_norm` (Euclidean distance) — for unnormalized embeddings or coordinate spaces

## Vector Index Setup

### 1. Add Embedding Field to Documents

```python
import openai  # or any embedding provider

def embed(text: str) -> list[float]:
    resp = openai.embeddings.create(model="text-embedding-3-small", input=text)
    return resp.data[0].embedding  # 1536-dimensional vector

# Store document with embedding
collection.upsert("product::p1", {
    "type": "product",
    "name": "Wireless Headphones",
    "description": "Premium noise-cancelling over-ear headphones",
    "price": 199.99,
    "embedding": embed("Wireless Headphones Premium noise-cancelling over-ear headphones")
})
```

### 2. Create a Vector-Enabled Search Index

Include a `vector` field in the index mapping:

```json
{
  "name": "products-vector-index",
  "type": "fulltext-index",
  "params": {
    "doc_config": {
      "mode": "scope.collection.type_field",
      "type_field": "type"
    },
    "mapping": {
      "default_mapping": { "enabled": false },
      "types": {
        "my-scope.products": {
          "enabled": true,
          "properties": {
            "name": {
              "fields": [{ "type": "text", "analyzer": "en", "index": true, "store": true }]
            },
            "description": {
              "fields": [{ "type": "text", "analyzer": "en", "index": true, "store": true }]
            },
            "category": {
              "fields": [{ "type": "keyword", "index": true, "store": true }]
            },
            "embedding": {
              "fields": [{
                "type": "vector",
                "dims": 1536,
                "similarity": "dot_product",
                "index": true,
                "xattr": false
              }]
            }
          }
        }
      }
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "my-bucket"
}
```

**Key vector field options:**
- `dims` — must match your embedding model's output dimension (e.g., 1536 for `text-embedding-3-small`, 3072 for `text-embedding-3-large`, 768 for `all-MiniLM-L6-v2`)
- `similarity` — `dot_product` (best for OpenAI normalized embeddings), `cosine`, or `l2_norm`
- `index` — must be `true` to enable vector search

## Vector Search (Python SDK)

```python
from couchbase.vector_search import VectorQuery, VectorSearch
from couchbase.search import SearchRequest, SearchOptions

def semantic_search(scope, query_text: str, k: int = 10):
    # 1. Embed the query
    query_vec = embed(query_text)

    # 2. Run vector search
    result = scope.search(
        "products-vector-index",
        SearchRequest.create(
            VectorSearch.from_vector_query(
                VectorQuery(
                    field_name="embedding",
                    vector=query_vec,
                    num_candidates=k * 2  # over-fetch then rank — higher = better recall, slower
                )
            )
        ),
        SearchOptions(
            limit=k,
            fields=["name", "description", "category", "price"]
        )
    )

    return [
        {"id": row.id, "score": row.score, **row.fields}
        for row in result.rows()
    ]

results = semantic_search(scope, "comfortable wireless headphones for commuting")
```

**`num_candidates`** — the HNSW algorithm pre-selects this many candidates then ranks. A good default is `limit * 2` to `limit * 5`. Higher values improve recall at the cost of latency.

## Hybrid Search (Text + Vector)

Combines BM25 text relevance with vector similarity — provides best-of-both-worlds:

```python
from couchbase.vector_search import VectorQuery, VectorSearch
import couchbase.search as search
from couchbase.search import SearchRequest, SearchOptions

def hybrid_search(scope, query_text: str, k: int = 10):
    query_vec = embed(query_text)

    result = scope.search(
        "products-vector-index",
        SearchRequest.create(
            search.MatchQuery(query_text)  # FTS part
        ).with_vector_search(
            VectorSearch.from_vector_query(
                VectorQuery("embedding", query_vec, num_candidates=k * 3)
            )
        ),
        SearchOptions(limit=k, fields=["name", "description"])
    )

    return [{"id": row.id, "score": row.score, **row.fields} for row in result.rows()]
```

Scores are normalized and merged. Hybrid search generally outperforms either alone for user-facing search.

## Multiple Vector Queries (OR)

```python
# Search for multiple query vectors — useful for multi-modal or expanded queries
result = scope.search(
    "products-vector-index",
    SearchRequest.create(
        VectorSearch([
            VectorQuery("embedding", embed("headphones"), num_candidates=20),
            VectorQuery("embedding", embed("earphones"), num_candidates=20),
        ])
    ),
    SearchOptions(limit=10)
)
```

## Pre-filtering with Vector Search

Apply a filter alongside the vector search to restrict the search space:

```python
# Search for similar products only within the "electronics" category
result = scope.search(
    "products-vector-index",
    SearchRequest.create(
        search.TermQuery("electronics", field="category")  # pre-filter
    ).with_vector_search(
        VectorSearch.from_vector_query(
            VectorQuery("embedding", query_vec, num_candidates=50)
        )
    ),
    SearchOptions(limit=10, fields=["name", "category"])
)
```

## RAG (Retrieval-Augmented Generation) Pattern

```python
from couchbase.vector_search import VectorQuery, VectorSearch
from couchbase.search import SearchRequest, SearchOptions
import anthropic  # or openai

def rag_answer(scope, question: str) -> str:
    # 1. Retrieve relevant chunks
    query_vec = embed(question)
    result = scope.search(
        "docs-vector-index",
        SearchRequest.create(
            VectorSearch.from_vector_query(VectorQuery("embedding", query_vec, num_candidates=10))
        ),
        SearchOptions(limit=5, fields=["text", "source"])
    )

    context_chunks = [row.fields.get("text", "") for row in result.rows()]
    context = "\n\n---\n\n".join(context_chunks)

    # 2. Generate answer with LLM
    client = anthropic.Anthropic()
    message = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": f"Context:\n{context}\n\nQuestion: {question}\n\nAnswer based only on the context."
        }]
    )
    return message.content[0].text
```

## Embedding Model Dimensions Reference

| Model | Dims | Similarity |
|-------|------|-----------|
| `text-embedding-3-small` (OpenAI) | 1536 | dot_product |
| `text-embedding-3-large` (OpenAI) | 3072 | dot_product |
| `text-embedding-ada-002` (OpenAI) | 1536 | dot_product |
| `all-MiniLM-L6-v2` (sentence-transformers) | 384 | cosine |
| `all-mpnet-base-v2` (sentence-transformers) | 768 | cosine |
| `embed-english-v3.0` (Cohere) | 1024 | dot_product |
| Gemini `embedding-001` | 768 | dot_product |

Always use the same model for indexing documents and querying — mixing models produces incorrect results.

## Bulk Embedding and Indexing

```python
from concurrent.futures import ThreadPoolExecutor
import openai

def embed_batch(texts: list[str]) -> list[list[float]]:
    resp = openai.embeddings.create(model="text-embedding-3-small", input=texts)
    return [item.embedding for item in sorted(resp.data, key=lambda x: x.index)]

def index_documents(collection, documents: list[dict], batch_size=100):
    for i in range(0, len(documents), batch_size):
        batch = documents[i:i + batch_size]
        texts = [f"{d['name']} {d.get('description', '')}" for d in batch]
        embeddings = embed_batch(texts)
        for doc, emb in zip(batch, embeddings):
            doc["embedding"] = emb
            collection.upsert(f"product::{doc['sku']}", doc)
        print(f"Indexed {min(i + batch_size, len(documents))}/{len(documents)}")
```
