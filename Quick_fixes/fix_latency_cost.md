# ═══════════════════════════════════════════════════════════════════════════════
# FIX: SLOW RESPONSES — "My Chatbot Takes Too Long"
# ═══════════════════════════════════════════════════════════════════════════════
# Use this when responses take more than 3 seconds.
# Each fix targets a specific stage of the pipeline.
# ═══════════════════════════════════════════════════════════════════════════════

## ROLE

You are a performance optimization specialist for RAG pipelines. The user's chatbot is too slow. Diagnose which stage is the bottleneck and fix it.

---

## I NEED FROM YOU

1. **How slow is it?** (average response time, worst case)
2. **Your stack**: LLM model, vector DB, embedding model, hosting
3. **Do you stream responses** to the user? (yes/no)
4. **How many chunks do you retrieve and pass to the LLM?**

---

## STEP 1: FIND THE BOTTLENECK (do this first)

```python
import time

class LatencyTracker:
    """Wrap around your pipeline to find what's slow."""

    def timed_query(self, question):
        timings = {}

        # Stage 1: Embedding
        t0 = time.time()
        query_embedding = embedding_model.embed(question)
        timings["embed"] = time.time() - t0

        # Stage 2: Retrieval
        t0 = time.time()
        chunks = vector_store.search(query_embedding, top_k=10)
        timings["retrieve"] = time.time() - t0

        # Stage 3: Reranking (if used)
        t0 = time.time()
        ranked_chunks = reranker.rerank(question, chunks, top_k=3)
        timings["rerank"] = time.time() - t0

        # Stage 4: LLM generation
        t0 = time.time()
        answer = llm.generate(system_prompt, ranked_chunks, question)
        timings["generate"] = time.time() - t0

        timings["total"] = sum(timings.values())

        # Print breakdown
        print(f"\n⏱️ Latency Breakdown:")
        for stage, duration in timings.items():
            bar = "█" * int(duration * 20)
            print(f"  {stage:12s} {duration:6.2f}s {bar}")

        return answer, timings

# Run this on 10 queries and find the pattern
tracker = LatencyTracker()
for q in test_questions:
    answer, timings = tracker.timed_query(q)
```

**Typical breakdowns**:
| Bottleneck | Pattern | Fix |
|-----------|---------|-----|
| LLM generation | 70-80% of total time | Streaming, model routing, caching |
| Retrieval | Slow vector DB or too many chunks | Index optimization, reduce top-k |
| Reranking | 500ms+ | Lighter reranker or skip for simple queries |
| Embedding | 200ms+ | Cache embeddings, use a faster model |

---

## FIX BY BOTTLENECK

### If LLM generation is slow → ADD STREAMING (15 minutes)

The user doesn't need to wait for the full answer. Stream it token by token.

```python
# FastAPI streaming endpoint
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import openai

app = FastAPI()

async def stream_answer(question, context):
    """Stream LLM response token by token."""
    response = openai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {question}"}
        ],
        stream=True
    )

    for chunk in response:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content

@app.post("/chat")
async def chat(question: str):
    # Retrieve happens normally (non-streaming)
    chunks = retriever.search(question)
    context = "\n\n".join([c["text"] for c in chunks])

    # Generation is streamed
    return StreamingResponse(
        stream_answer(question, context),
        media_type="text/plain"
    )
```

**Perceived latency**: Even if total time is the same, the user sees the first word in ~500ms instead of waiting 3-5s for the complete answer. This alone makes the chatbot feel 5x faster.

---

### If repeat questions are common → ADD SEMANTIC CACHE (30 minutes)

```python
import hashlib
import json
import numpy as np
from datetime import datetime, timedelta

class SemanticCache:
    def __init__(self, embedding_model, similarity_threshold=0.95, ttl_hours=24):
        self.embedding_model = embedding_model
        self.threshold = similarity_threshold
        self.ttl = timedelta(hours=ttl_hours)
        self.cache = []  # In production, use Redis with vector search

    def get(self, question):
        """Check if a similar question was already answered."""
        query_embedding = self.embedding_model.embed(question)

        for entry in self.cache:
            # Check TTL
            if datetime.now() - entry["timestamp"] > self.ttl:
                continue

            # Check similarity
            similarity = self._cosine_similarity(query_embedding, entry["embedding"])
            if similarity >= self.threshold:
                print(f"🎯 Cache hit! Similarity: {similarity:.3f}")
                return entry["answer"]

        return None  # Cache miss

    def set(self, question, answer):
        """Store a question-answer pair."""
        self.cache.append({
            "question": question,
            "embedding": self.embedding_model.embed(question),
            "answer": answer,
            "timestamp": datetime.now()
        })

    def invalidate_all(self):
        """Call this after re-indexing or content updates."""
        self.cache.clear()

    def _cosine_similarity(self, a, b):
        return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# Usage:
cache = SemanticCache(embedding_model, similarity_threshold=0.95)

def answer_question(question):
    # 1. Check cache first
    cached = cache.get(question)
    if cached:
        return cached  # ~0ms instead of ~3s

    # 2. Normal pipeline
    chunks = retriever.search(question)
    answer = llm.generate(system_prompt, chunks, question)

    # 3. Cache for next time
    cache.set(question, answer)
    return answer
```

**Tune the threshold**:
- 0.98+ → Very strict, only near-exact matches (safe)
- 0.95 → Good default, catches rephrased versions
- 0.90 → Aggressive, might return wrong cached answers
- **Start at 0.95**, lower if cache hit rate is too low

---

### If costs are high → MODEL ROUTING (20 minutes)

Don't use your expensive model for simple questions.

```python
class ModelRouter:
    """Route queries to cheap or powerful models based on complexity."""

    def __init__(self):
        self.cheap_model = "gpt-4o-mini"       # ~$0.15/1M tokens
        self.powerful_model = "gpt-4o"          # ~$2.50/1M tokens

    def classify_complexity(self, question, chunks):
        """Simple heuristic — no LLM call needed."""
        is_simple = True

        # Complex signals:
        if any(word in question.lower() for word in
               ["compare", "difference", "versus", "vs", "pros and cons",
                "explain why", "analyze", "summarize all"]):
            is_simple = False

        if len(question.split()) > 30:  # Long questions are usually complex
            is_simple = False

        if len(chunks) > 5:  # Needs to synthesize many sources
            is_simple = False

        return "simple" if is_simple else "complex"

    def get_model(self, question, chunks):
        complexity = self.classify_complexity(question, chunks)
        model = self.cheap_model if complexity == "simple" else self.powerful_model
        print(f"📊 Routing: {complexity} → {model}")
        return model

router = ModelRouter()

def answer_question(question):
    chunks = retriever.search(question)
    model = router.get_model(question, chunks)
    return llm.generate(system_prompt, chunks, question, model=model)
```

**Expected savings**: If 60-70% of queries are simple (typical), you cut LLM costs by ~50-60%.

---

### If retrieval is slow → OPTIMIZE VECTOR DB (varies)

```python
# Quick wins for common vector DBs:

# ChromaDB: Add HNSW parameters
collection = client.create_collection(
    name="documents",
    metadata={
        "hnsw:space": "cosine",
        "hnsw:M": 32,              # Higher = better recall, more memory
        "hnsw:ef_construction": 200  # Higher = better index quality, slower build
    }
)

# Pinecone: Use serverless + metadata filtering
index = pinecone.Index("documents")
results = index.query(
    vector=query_embedding,
    top_k=5,
    filter={"status": "active"},  # Pre-filter reduces search space
    include_metadata=True
)

# pgvector: Add HNSW index
# CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
# WITH (m = 16, ef_construction = 200);
# SET hnsw.ef_search = 100;  -- Higher = better recall, slower

# General: Reduce top-k (retrieve fewer chunks)
# Going from top-20 to top-5 can cut retrieval time by 50%+
```

---

## QUICK REFERENCE: Expected Latency Targets

| Stage | Good | Acceptable | Needs Fix |
|-------|------|------------|-----------|
| Embedding | < 50ms | < 100ms | > 200ms |
| Retrieval | < 100ms | < 300ms | > 500ms |
| Reranking | < 200ms | < 500ms | > 1s |
| LLM (TTFT) | < 500ms | < 1s | > 2s |
| **Total (streamed)** | **< 1s TTFT** | **< 2s TTFT** | **> 3s TTFT** |
| **Total (non-streamed)** | **< 2s** | **< 3s** | **> 5s** |
