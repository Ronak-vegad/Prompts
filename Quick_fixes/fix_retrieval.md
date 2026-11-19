# ═══════════════════════════════════════════════════════════════════════════════
# FIX: RETRIEVAL FAILURES — "Can't Find the Right Documents"
# ═══════════════════════════════════════════════════════════════════════════════
# Use this when your chatbot retrieves irrelevant chunks, misses obvious
# documents, or can't match user queries to the right content.
# ═══════════════════════════════════════════════════════════════════════════════

## ROLE

You are a retrieval optimization specialist. The user's chatbot is failing at the search stage — the right documents exist but aren't being found. Fix the retrieval pipeline with minimal changes, in priority order.

---

## I NEED FROM YOU

1. **An example failure**: a question where the chatbot retrieved the wrong documents (and what it should have found)
2. **Your vector DB and embedding model**
3. **Your current chunking** (strategy, chunk size, overlap)
4. **Current retrieval method** (vector only? hybrid? top-k number?)

---

## FIXES (in order — each one stacks on the previous)

### FIX 1: Add Hybrid Search (30 minutes, highest impact for retrieval)

**The problem**: Pure vector search fails for exact terms — product IDs, codes, names, technical terms, error messages. User searches "Error 503" and gets chunks about "server issues" instead of the exact error code doc.

**The fix**: Add BM25 keyword search alongside vector search, combine with Reciprocal Rank Fusion.

```python
# Using rank_bm25 library for keyword search
from rank_bm25 import BM25Okapi
import numpy as np

class HybridRetriever:
    def __init__(self, vector_store, documents, alpha=0.5):
        """
        alpha: weight for vector search (1-alpha = weight for BM25)
        Start at 0.5, tune based on eval results.
        """
        self.vector_store = vector_store
        self.alpha = alpha

        # Build BM25 index from document chunks
        tokenized_docs = [doc["text"].lower().split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized_docs)
        self.documents = documents

    def search(self, query, top_k=5):
        # 1. Vector search
        vector_results = self.vector_store.similarity_search_with_score(
            query, k=top_k * 3  # Retrieve more for fusion
        )

        # 2. BM25 keyword search
        tokenized_query = query.lower().split()
        bm25_scores = self.bm25.get_scores(tokenized_query)
        bm25_top_indices = np.argsort(bm25_scores)[::-1][:top_k * 3]
        bm25_results = [
            (self.documents[i], bm25_scores[i])
            for i in bm25_top_indices if bm25_scores[i] > 0
        ]

        # 3. Reciprocal Rank Fusion
        fused = self._rrf_fusion(vector_results, bm25_results)
        return fused[:top_k]

    def _rrf_fusion(self, vector_results, bm25_results, k=60):
        """Reciprocal Rank Fusion — combines rankings from multiple sources."""
        scores = {}

        for rank, (doc, _score) in enumerate(vector_results):
            doc_id = doc.get("id", doc["text"][:100])
            scores[doc_id] = scores.get(doc_id, 0) + self.alpha * (1 / (k + rank + 1))

        for rank, (doc, _score) in enumerate(bm25_results):
            doc_id = doc.get("id", doc["text"][:100])
            scores[doc_id] = scores.get(doc_id, 0) + (1 - self.alpha) * (1 / (k + rank + 1))

        # Sort by fused score
        sorted_docs = sorted(scores.items(), key=lambda x: x[1], reverse=True)
        return sorted_docs
```

**How to tune alpha**:
- More keyword-heavy content (codes, IDs, error messages) → alpha = 0.3 (more BM25 weight)
- More natural language content (articles, guides) → alpha = 0.7 (more vector weight)
- Run your eval set at different alpha values and pick the best recall@k

---

### FIX 2: Add a Reranker (20 minutes, second biggest impact)

**The problem**: Both vector and BM25 retrieve "roughly relevant" documents. A cross-encoder reranker picks the actually-best ones.

```python
# Option A: Cohere Rerank (API, easy, fast)
import cohere
co = cohere.Client("your-api-key")

def rerank(query, documents, top_k=3):
    results = co.rerank(
        query=query,
        documents=[doc["text"] for doc in documents],
        top_n=top_k,
        model="rerank-v3.5"
    )
    return [documents[r.index] for r in results.results]

# Option B: Open-source cross-encoder (free, self-hosted)
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("BAAI/bge-reranker-v2-m3")

def rerank(query, documents, top_k=3):
    pairs = [[query, doc["text"]] for doc in documents]
    scores = reranker.predict(pairs)
    ranked_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)
    return [documents[i] for i in ranked_indices[:top_k]]

# Full pipeline:
def retrieve_and_rerank(query, top_k=3):
    # Step 1: Broad retrieval (cast wide net)
    candidates = hybrid_retriever.search(query, top_k=20)

    # Step 2: Precision reranking (narrow to best)
    best = rerank(query, candidates, top_k=top_k)

    return best
```

---

### FIX 3: Query Rewriting (20 minutes)

**The problem**: Users ask questions in ways that don't match how the documents are written.

```python
REWRITE_PROMPT = """
You are a search query optimizer. Rewrite the user's question to improve
document retrieval. Generate 2-3 search queries that capture the same intent
but use different wordings.

Rules:
- Keep technical terms and IDs exactly as written
- Expand abbreviations
- Add synonyms for key concepts
- If the query is in another language, add an English translation as a variant

User question: {question}

Output as a JSON list of strings:
["rewritten query 1", "rewritten query 2", "rewritten query 3"]
"""

import json

def rewrite_query(question):
    """Generate multiple search queries from one user question."""
    result = llm.generate(
        REWRITE_PROMPT.format(question=question),
        model="gpt-4o-mini",  # Cheap model is enough
        temperature=0.3
    )
    queries = json.loads(result)
    return [question] + queries  # Always include the original

def multi_query_retrieve(question, top_k=5):
    """Retrieve using multiple query variants, deduplicate results."""
    queries = rewrite_query(question)
    all_results = []
    seen_ids = set()

    for query in queries:
        results = hybrid_retriever.search(query, top_k=top_k)
        for doc in results:
            doc_id = doc.get("id", doc["text"][:100])
            if doc_id not in seen_ids:
                all_results.append(doc)
                seen_ids.add(doc_id)

    # Rerank the combined results
    return rerank(question, all_results, top_k=top_k)  # Use ORIGINAL question for reranking
```

---

### FIX 4: Better Chunking (1-2 hours, requires re-ingestion)

**The problem**: Fixed-size chunks cut through the middle of important sections. The answer might be split across two chunks, and neither one makes sense alone.

```python
# BEFORE (bad — fixed size cuts mid-sentence/mid-section):
# "...the maximum load capacity is 500kg. For installations above"
# | CHUNK BOUNDARY |
# "3 meters, additional reinforcement is required per Section 4.2..."

# AFTER (good — chunk by document structure):

from langchain.text_splitter import MarkdownHeaderTextSplitter

# Option A: Split by headings (if your docs have structure)
headers_to_split_on = [
    ("#", "h1"),
    ("##", "h2"),
    ("###", "h3"),
]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
chunks = splitter.split_text(document_text)

# Option B: Semantic chunking (for unstructured text)
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

splitter = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=85
)
chunks = splitter.split_text(document_text)

# Option C: Parent-child chunking (best of both worlds)
# Small chunks for retrieval precision, but return the bigger parent for context
class ParentChildChunker:
    def __init__(self, small_chunk_size=300, large_chunk_size=1500):
        self.small_size = small_chunk_size
        self.large_size = large_chunk_size

    def chunk(self, document):
        # Create large parent chunks
        parents = self._split(document, self.large_size)

        # Create small child chunks, linked to parents
        children = []
        for i, parent in enumerate(parents):
            small_chunks = self._split(parent["text"], self.small_size)
            for child in small_chunks:
                child["parent_id"] = parent["id"]
                child["parent_text"] = parent["text"]
                children.append(child)

        return children  # Index children; on retrieval, return parent_text
```

---

### FIX 5: Metadata Filtering (30 minutes)

**The problem**: Searching the entire knowledge base when the user clearly wants a specific subset.

```python
def extract_filters(query):
    """Auto-detect metadata filters from the query."""
    FILTER_PROMPT = """
    Extract any metadata filters from this search query.
    Available filter fields: {metadata_fields}

    Query: {query}

    Return JSON: {{"field": "value", ...}} or {{}} if no filters detected.
    Examples:
    - "What does the 2023 policy say about returns?" → {{"year": 2023}}
    - "Show me section 4.2 of the safety manual" → {{"document": "safety manual", "section": "4.2"}}
    - "Latest update on product X" → {{"status": "active", "sort": "date_desc"}}
    """
    # Use a cheap/fast model for filter extraction
    result = llm.generate(FILTER_PROMPT.format(
        metadata_fields="category, year, status, document_type, section",
        query=query
    ), model="gpt-4o-mini", temperature=0)

    return json.loads(result)

# Apply filters before vector search
def filtered_search(query, top_k=5):
    filters = extract_filters(query)
    if filters:
        return vector_store.similarity_search(query, top_k=top_k, filter=filters)
    else:
        return hybrid_retriever.search(query, top_k=top_k)
```

---

## VERIFICATION

After each fix, test retrieval quality:

```python
def test_retrieval(test_cases):
    """
    test_cases: list of {"question": ..., "expected_doc_id": ..., "expected_section": ...}
    """
    hits = 0
    for case in test_cases:
        results = retriever.search(case["question"], top_k=5)
        retrieved_ids = [r["doc_id"] for r in results]

        if case["expected_doc_id"] in retrieved_ids:
            hits += 1
            rank = retrieved_ids.index(case["expected_doc_id"]) + 1
            print(f"✅ Found at rank {rank}: {case['question'][:60]}...")
        else:
            print(f"❌ MISSED: {case['question'][:60]}...")

    recall = hits / len(test_cases)
    print(f"\nRecall@5: {recall:.1%} ({hits}/{len(test_cases)})")
    return recall
```
