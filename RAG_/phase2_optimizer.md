# ═══════════════════════════════════════════════════════════════════════════════
# PHASE 2: UNIVERSAL CHATBOT OPTIMIZER
# ═══════════════════════════════════════════════════════════════════════════════
# Paste the CONTEXT BLOCK from Phase 1 at the top, then feed this entire
# prompt to your LLM. It will adapt every checklist item to YOUR chatbot.
# ═══════════════════════════════════════════════════════════════════════════════

## ── PASTE YOUR CONTEXT BLOCK HERE ──────────────────────────────────────────
# (Replace this section with the output from Phase 1)
#
# CHATBOT_TOPIC: ...
# CHATBOT_DESCRIPTION: ...
# ... etc ...
#
## ── END CONTEXT BLOCK ──────────────────────────────────────────────────────


---

## ROLE

You are a senior ML/RAG engineer and production systems architect. The user has a working RAG-based chatbot described in the CONTEXT BLOCK above. Your job is to audit the codebase and upgrade it to industrial grade.

**Adapt every recommendation to the user's specific domain, risk level, content types, and tech stack.** A medical chatbot needs different grounding rules than a customer support bot. A high-risk legal chatbot needs stricter citation requirements than a casual FAQ bot.

Work through the checklist below IN ORDER. For each item:
1. **Assess**: Tell the user what likely exists now (based on the context block and common patterns for their stack)
2. **Gap**: What's missing or suboptimal
3. **Implement**: Provide the fix — actual code, config, or architecture change tailored to their stack
4. **Verify**: Show how to test/verify the fix works

Don't skip items. If something doesn't apply to this chatbot's domain, say why and move on.

---

## MASTER CHECKLIST

### 1. 📋 EVALUATION SET (Do This First — Everything Else Depends On It)

**Why first**: You can't improve what you can't measure. Every change below should be validated against this eval set.

```
[ ] Build a golden evaluation set of 100+ questions tailored to {{CHATBOT_TOPIC}}
    Categories to include:
    - Direct factual lookups (single document/section)
    - Cross-document / cross-reference questions
    - "Which [document/product/policy] applies to [X]?" (routing questions)
    - Table/structured data lookups
    - Process/procedure questions ("how do I...")
    - Comparison questions ("difference between X and Y")
    - Out-of-scope questions (should be refused gracefully)
    - Adversarial/trick questions (misleading, outdated info, prompt injection attempts)
    - Multi-hop reasoning (need info from 2+ chunks)
    - {{DOMAIN-SPECIFIC CATEGORIES based on CHATBOT_TOPIC}}

[ ] Each test item must include:
    | Field             | Description                                        |
    |-------------------|----------------------------------------------------|
    | question          | The user's question                                |
    | expected_answer   | Gold-standard correct answer                       |
    | expected_source   | Document ID + section/page that contains the answer|
    | difficulty        | easy / medium / hard                               |
    | category          | From the categories above                          |
    | requires_refusal  | true/false (should the bot refuse to answer?)      |

[ ] Metrics to track:
    - Retrieval: recall@k, MRR (Mean Reciprocal Rank), precision@k
    - Generation: faithfulness (is every claim grounded?), answer relevance,
      citation accuracy, completeness
    - Safety: refusal accuracy (refuses when it should, answers when it should)
    - Latency: p50, p95, p99 per stage

[ ] Automated eval script:
    - Runs full eval set against the live system
    - Compares scores to baseline thresholds (configurable)
    - Fails CI if any metric drops below threshold
    - Stores results with timestamp + git hash for version comparison
    - Outputs a comparison table: "before vs after" for every change

[ ] Eval storage: JSON/CSV per run, with a dashboard or CLI tool to compare runs
```

**Adapt for risk level**:
- If `DOMAIN_RISK_LEVEL` is **high/critical**: Require 200+ golden questions, include adversarial and edge cases heavily, citation accuracy must be ≥ 95%
- If `DOMAIN_RISK_LEVEL` is **low**: 50+ questions may suffice, citation can be optional

---

### 2. 📄 INGESTION & CHUNKING

```
[ ] Document parsing — adapt to {{CONTENT_TYPES}}:
    - PDFs: Use structure-aware parser (pymupdf4llm, Unstructured, Docling, or
      cloud-based like Azure Document Intelligence / Google Document AI)
    - Tables: Preserve as markdown tables or structured JSON, NOT flattened text
    - Scanned/image PDFs: OCR pipeline (Tesseract, EasyOCR, or cloud OCR)
    - Web pages: Clean HTML extraction (BeautifulSoup, Trafilatura)
    - Databases/APIs: Structured extraction with schema preservation
    - Spreadsheets: Sheet-aware parsing, preserve formulas context
    - Code docs: Language-aware parsing with symbol extraction

[ ] Structured chunking — adapt to {{CONTENT_STRUCTURE}}:
    - If well-structured: Chunk by logical sections (chapters, clauses, headings)
      with parent-child hierarchy preserved
    - If semi-structured: Recursive chunking with heading detection
    - If unstructured: Semantic chunking (by topic coherence) or sliding window
      with generous overlap
    - Keep parent context: each chunk should reference its parent section/document
    - Target chunk sizes: 200-500 tokens for precise retrieval, with optional
      larger "context chunks" (1000-2000 tokens) for generation

[ ] Rich metadata per chunk:
    | Field           | Source                        | Required? |
    |-----------------|-------------------------------|-----------|
    | document_id     | Filename or assigned ID       | Yes       |
    | document_title  | Extracted from content        | Yes       |
    | section/heading | Parsed from structure         | Yes       |
    | page_number     | From parser                   | If applicable |
    | date_published  | From metadata or content      | If available |
    | date_ingested   | System timestamp              | Yes       |
    | version         | Document version/edition      | If applicable |
    | status          | active/deprecated/superseded  | If applicable |
    | language        | Detected or tagged            | Yes       |
    | {{DOMAIN-SPECIFIC METADATA based on CONTENT_METADATA}} |

[ ] Version & update handling:
    - If {{CONTENT_VERSIONED}} is yes:
      - Track document versions in metadata
      - Default retrieval: prefer latest active version
      - If user asks about old version: retrieve it but flag it clearly
      - On re-ingestion: mark old chunks as superseded, don't delete (audit trail)
    - If no: simple overwrite with re-index

[ ] Cleanup pipeline:
    - Remove headers, footers, page numbers, watermarks
    - Normalize whitespace, fix encoding issues
    - Deduplicate identical or near-identical chunks
    - Validate: spot-check 20 random chunks for quality after every ingest
```

---

### 3. 🔍 RETRIEVAL

```
[ ] Hybrid search — essential for {{CHATBOT_TOPIC}}:
    - Dense vectors: semantic similarity for natural language questions
    - Sparse/keyword (BM25): exact match for IDs, codes, names, technical terms
    - Fusion: Reciprocal Rank Fusion (RRF) or weighted combination
    - Tune weights based on eval results (some domains need more keyword, others more semantic)

[ ] Metadata filtering:
    - Pre-filter by: {{available metadata fields from CONTENT_METADATA}}
    - Auto-detect filters from query (e.g., "2023 policy" → date_published=2023)
    - Allow explicit filters via UI or structured queries

[ ] Query understanding & rewriting:
    - Spelling correction and normalization
    - Abbreviation expansion (domain-specific glossary for {{CHATBOT_TOPIC}})
    - Multi-language handling if {{CONTENT_LANGUAGES}} has multiple languages:
      - Detect query language
      - Translate to content language for retrieval OR use multilingual embeddings
    - Query decomposition for complex questions (break into sub-queries)
    - HyDE (Hypothetical Document Embedding) for abstract questions

[ ] Reranking:
    - Stage 1: Retrieve top-N (N=20-50) via hybrid search
    - Stage 2: Rerank with cross-encoder (Cohere Rerank, bge-reranker, or
      ColBERT) → top-k (k=3-8)
    - Measure: recall@k before and after reranking
    - Cost check: reranking adds latency; if {{TARGET_LATENCY}} is very tight,
      consider lightweight rerankers or skip if retrieval is already strong

[ ] Advanced retrieval (if needed based on {{PAIN_POINTS}}):
    - Multi-hop: iterative retrieval for complex questions
    - Agentic retrieval: LLM decides when to search, what to search, and when
      it has enough context
    - Knowledge graph overlay: for relationship-heavy domains
```

---

### 4. 🧮 EMBEDDING CONSISTENCY

```
[ ] Model alignment:
    - Document embedding model: {{STACK_EMBEDDING}} (version: ___)
    - Query embedding model: MUST be the same model AND version
    - Log model name + version in vector DB index metadata
    - If using an API-based model: pin the model version (don't use "latest")

[ ] Startup safety check:
    - On app startup: verify query-side embedding model matches the index metadata
    - If mismatch: BLOCK startup with clear error message
    - This prevents silent failures from model upgrades

[ ] Safe model upgrade path:
    - Create new index with new model (versioned name: index_v2_modelname)
    - Re-embed all documents into new index
    - Run eval suite against new index — compare scores to old index
    - If scores improve: switch traffic; keep old index as rollback
    - If scores drop: investigate before switching
    - Script: `reindex.py --model=new_model --index=index_v3`
```

---

### 5. 🛡️ GROUNDING & HALLUCINATION CONTROL

**Adapt strictness to {{DOMAIN_RISK_LEVEL}}**:

```
[ ] System prompt — grounding rules:
    ┌─────────────────────────────────────────────────────────────────────┐
    │ CORE RULES (adapt tone and specifics to {{CHATBOT_TOPIC}}):        │
    │                                                                     │
    │ 1. Answer ONLY from the retrieved context provided below.           │
    │ 2. If the context doesn't contain the answer, say:                  │
    │    "I couldn't find this in the available [{{CHATBOT_TOPIC}}]       │
    │     documents. [Suggest where to look or who to ask]"               │
    │ 3. NEVER make up facts, numbers, dates, IDs, or references.        │
    │ 4. For every factual claim, cite the source:                        │
    │    [document_id, section/page].                                     │
    │ 5. If information is from a superseded/old version, flag it.        │
    │ 6. For [{{DOMAIN-SPECIFIC SENSITIVE TOPICS}}], add disclaimer:      │
    │    "This is for informational purposes. [Appropriate disclaimer     │
    │     for {{CHATBOT_TOPIC}} domain]"                                  │
    └─────────────────────────────────────────────────────────────────────┘

[ ] Retrieval confidence gating:
    - Set a minimum relevance score threshold
    - If best retrieved chunk is below threshold → refuse to answer
    - Tune threshold using the eval set (maximize accuracy while minimizing
      false refusals)

[ ] Citation enforcement:
    - If DOMAIN_RISK_LEVEL is high/critical:
      → Every claim MUST have [document_id + section/page]
      → Post-process: strip any sentence that lacks a citation
    - If DOMAIN_RISK_LEVEL is medium:
      → Cite source document for each paragraph
    - If DOMAIN_RISK_LEVEL is low:
      → Cite source document at end of answer

[ ] Post-generation faithfulness check:
    - Option A (fast): LLM-as-judge — ask a cheap model "is each claim in
      the answer supported by the provided context? List unsupported claims."
    - Option B (thorough): NLI model (e.g., TRUE/AlignScore) on each
      claim-chunk pair
    - If unfaithful claims found:
      → If DOMAIN_RISK_LEVEL is high: regenerate or refuse
      → If DOMAIN_RISK_LEVEL is low: flag in metadata for review

[ ] Domain-specific disclaimers — auto-generate based on {{CHATBOT_TOPIC}}:
    - Legal: "This is not legal advice. Consult a qualified attorney."
    - Medical: "This is not medical advice. Consult a healthcare professional."
    - Financial: "This is not financial advice. Consult a licensed advisor."
    - Compliance/regulatory: "Verify with the official [regulatory body] publication."
    - Technical/safety: "Follow manufacturer guidelines and safety standards."
    - General: "Information is based on available documents and may not be current."
```

---

### 6. ✨ DOMAIN-SPECIFIC FEATURES

**These are auto-generated based on {{CHATBOT_TOPIC}}. The LLM should identify 5-10 high-value features specific to this domain.**

```
[ ] Common cross-domain features (include any that apply):

    📌 Document/Entity Finder:
    - "[User describes a need] → [Which documents/products/policies apply?]"
    - Maps user intent to relevant documents via metadata + semantic search

    📌 Comparison Engine:
    - Compare two documents, versions, products, or policies
    - Structured diff output: what changed, what's new, what's removed

    📌 Process/Workflow Guide:
    - Step-by-step guidance for processes in {{CHATBOT_TOPIC}}
    - Checklist generation based on user's specific situation

    📌 Multilingual Support (if {{CONTENT_LANGUAGES}} > 1):
    - Query in any supported language → answer in the same language
    - Cross-lingual retrieval (query in language A, retrieve from language B)
    - Translation of key terms with domain-specific glossary

    📌 Source Viewer:
    - Click-through to the original document
    - Highlight the exact section/paragraph used in the answer
    - If PDFs: embedded viewer with page jump

    📌 Feedback Loop:
    - Thumbs up/down on every answer
    - Optional: reason for downvote (wrong answer, incomplete, wrong source, etc.)
    - Feed downvoted Q&As into the eval set for regression testing
    - Monthly review: top-N failure modes → targeted improvements

    📌 Personalization (if applicable):
    - User profiles: role, preferences, history
    - Personalized answers based on user context
    - Saved searches / bookmarks / favorites

[ ] Domain-specific features (LLM: generate 3-7 based on {{CHATBOT_TOPIC}}):
    - [Auto-generated feature 1]
    - [Auto-generated feature 2]
    - [Auto-generated feature 3]
    - ... (the LLM identifies what's uniquely valuable for this domain)
```

---

### 7. ⚡ PERFORMANCE & COST

```
[ ] Latency optimization:
    - Target: p95 ≤ {{TARGET_LATENCY}} (default: 2-3 seconds)
    - Measure per stage:
      | Stage     | Target   | How to Measure              |
      |-----------|----------|-----------------------------|
      | Embedding | < 100ms  | Timer around embed call     |
      | Retrieval | < 200ms  | Timer around vector search  |
      | Reranking | < 300ms  | Timer around reranker       |
      | LLM gen   | < 2s TTFT| Timer to first token        |
    - If over budget: identify bottleneck, optimize that stage

[ ] Streaming responses:
    - Stream LLM output token-by-token to the UI
    - Show retrieval status while searching ("Searching documents...")
    - Progressive rendering: show sources before answer is complete

[ ] Semantic caching:
    - Cache: question embedding → answer (with source metadata)
    - Similarity threshold: 0.95+ cosine similarity = cache hit
    - TTL: based on content update frequency
    - Invalidation: clear cache on re-index or content update
    - Cache hit rate target: 20-40% for typical workloads
    - Tools: GPTCache, Redis with vector search, or custom

[ ] Token budgeting & cost control:
    - Context window management:
      - Calculate: system_prompt + retrieved_chunks + conversation_history ≤ model limit
      - If over: trim oldest conversation turns first, then reduce chunks
    - Model routing (if {{BUDGET}} is a concern):
      | Query Type        | Model          | Why                      |
      |-------------------|----------------|--------------------------|
      | Simple factual    | Small/fast     | Cheap, sufficient        |
      | Complex reasoning | Large/powerful  | Needs capability        |
      | Classification    | Small/fast     | Simple task              |
      - Router: lightweight classifier or heuristic (query length, keywords)
    - Track cost per query:
      - Input tokens × input_price + output tokens × output_price
      - Log with every trace
      - Alert if daily cost exceeds budget

[ ] Additional optimizations:
    - Connection pooling for vector DB and LLM API
    - Async/parallel: run embedding + BM25 in parallel
    - Batch operations where possible (embedding multiple queries)
    - CDN for static assets (PDF viewer, UI)
```

---

### 8. 📊 OBSERVABILITY

```
[ ] Per-query tracing — log everything:
    | Field              | Description                              |
    |--------------------|------------------------------------------|
    | trace_id           | Unique ID for the query                  |
    | timestamp          | When the query was received               |
    | user_id            | Anonymized user identifier               |
    | question           | The user's question                      |
    | detected_language  | Language of the query                    |
    | query_rewritten    | Rewritten/expanded query (if any)        |
    | filters_applied    | Metadata filters used                    |
    | chunks_retrieved   | List of chunk IDs with scores            |
    | reranked_chunks    | Chunk IDs after reranking with scores    |
    | prompt_sent        | Full prompt sent to LLM (or hash)        |
    | answer             | Generated answer                         |
    | sources_cited      | Sources referenced in the answer         |
    | tokens_in          | Input token count                        |
    | tokens_out         | Output token count                       |
    | cost               | Estimated cost                           |
    | latency_total      | Total response time                      |
    | latency_retrieve   | Retrieval time                           |
    | latency_rerank     | Reranking time                           |
    | latency_generate   | Generation time                          |
    | cache_hit          | Was this served from cache?              |
    | faithfulness_score | Post-gen faithfulness check result       |
    | user_feedback      | Thumbs up/down (when received)           |

[ ] Dashboards:
    - Real-time: latency p50/p95, error rate, active users
    - Daily: cost, query volume, cache hit rate, top queries
    - Weekly: faithfulness trend, refusal rate, user satisfaction
    - Monthly: eval scores over time, improvement tracking

[ ] Alerts:
    - Faithfulness score drops below threshold → Slack/email alert
    - Latency p95 exceeds {{TARGET_LATENCY}} → alert
    - Daily cost exceeds budget → alert
    - Error rate spikes (5xx, timeouts) → alert
    - Refusal rate spikes (might indicate retrieval degradation) → alert

[ ] Tools (choose based on {{STACK_FRAMEWORK}}):
    - Langfuse: Great for LangChain/LlamaIndex, open-source option
    - LangSmith: Native for LangChain, managed
    - Phoenix (Arize): Good for any framework, open-source
    - Custom: Structured logging to BigQuery/PostgreSQL + Grafana
    - Minimum viable: structured JSON logs + simple dashboard
```

---

### 9. 🔒 SECURITY & RELIABILITY

```
[ ] Prompt injection defense:
    - Treat ALL user input as untrusted
    - Treat ALL retrieved document content as untrusted
    - Sandboxing: system prompt → [DOCUMENTS: ...] → [USER QUERY: ...]
      with clear delimiters
    - Input validation: max length, character filtering, pattern detection
    - Output validation: check for instruction leakage, system prompt exposure
    - Regular testing with known prompt injection attacks

[ ] Access control & auth:
    - Authentication: API keys, JWT, OAuth (based on {{COMPLIANCE}})
    - Rate limiting: per-user and global
      - Default: 10 queries/minute per user, 1000/minute global
      - Adjust based on expected usage
    - Role-based access if different users should see different content

[ ] Data protection:
    - PII detection and redaction in logs (names, emails, phone numbers)
    - If {{COMPLIANCE}} includes GDPR/HIPAA: encryption at rest and in transit,
      data retention policies, right to deletion
    - Don't log full prompts in production if they might contain PII
      (hash or redact)

[ ] Reliability:
    - Timeouts: LLM call (30s), vector DB (5s), total request (45s)
    - Retries: exponential backoff for transient failures (429, 503)
    - Fallbacks:
      - LLM down → "Service temporarily unavailable, please try again"
      - Vector DB down → keyword-only search if possible, else graceful error
      - Embedding API down → queue and retry
    - Circuit breaker pattern for external dependencies
    - Health check endpoint: /health (checks LLM, vector DB, embedding service)
    - Graceful degradation: clearly communicate reduced functionality

[ ] Input validation:
    - Max query length (e.g., 500 characters)
    - Reject or sanitize special characters / code injection attempts
    - Language detection: if query is in unsupported language, say so politely
```

---

### 10. 🚀 DEPLOYMENT

```
[ ] Containerization:
    - Dockerfile: multi-stage build, minimal base image
    - docker-compose for local dev (app + vector DB + cache + monitoring)
    - Environment-based config: .env file, NO secrets in code
    - Secrets management: cloud secret manager or Vault

[ ] CI/CD pipeline:
    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
    │   Build    │───▶│   Test     │───▶│   Eval     │───▶│  Deploy    │
    │            │    │ Unit tests │    │ Run golden  │    │ If eval    │
    │ Docker     │    │ Integration│    │ eval set    │    │ passes     │
    │ build      │    │ tests      │    │ Compare to  │    │            │
    │            │    │            │    │ baseline    │    │ Staging →  │
    │            │    │            │    │ Fail if ↓   │    │ Production │
    └────────────┘    └────────────┘    └────────────┘    └────────────┘

[ ] Environment management:
    - Development: local Docker, mock LLM for unit tests
    - Staging: full stack, real LLM, subset of data
    - Production: full stack, all data, monitoring
    - Rollback: one-command rollback to previous version

[ ] Documentation:
    - README with:
      - Architecture diagram (Mermaid or image)
      - Setup instructions (< 5 minutes to run locally)
      - Configuration reference
      - API documentation
      - Eval results summary
    - Runbook: common issues and how to fix them
    - Contributing guide if team project
```

---

## OUTPUT FORMAT

Produce your audit in this exact structure:

### A. Gap Analysis Table

| # | Item | Status | Priority | Effort | Impact |
|---|------|--------|----------|--------|--------|
| 1.1 | Golden eval set | ✅ Done / 🟡 Partial / ❌ Missing | P0-P3 | S/M/L | High/Med/Low |
| ... | ... | ... | ... | ... | ... |

**Priority guide**:
- **P0**: Blocking — fix immediately (e.g., no eval set, hallucination issues)
- **P1**: Critical — fix this week (e.g., no reranking, no caching)
- **P2**: Important — fix this month (e.g., multilingual, advanced features)
- **P3**: Nice to have — backlog (e.g., PDF viewer, voice input)

### B. Implementation Plan

Implement fixes in priority order. For each fix:
```
## Fix: [Item Name]
### What exists: ...
### Gap: ...
### Implementation:
[Code / config / architecture — ACTUAL implementation, not pseudocode]
### Verify:
[How to test this specific fix]
### Eval impact:
[Expected change in eval metrics]
```

### C. Before vs After

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Retrieval recall@5 | X% | Y% | +Z% |
| Faithfulness | X% | Y% | +Z% |
| p95 Latency | Xs | Ys | -Zs |
| Cost per query | $X | $Y | -Z% |
| ... | ... | ... | ... |

### D. Demo Script

A script or sequence of commands that demonstrates the improved chatbot for stakeholders:
```
1. Run: [command to start the system]
2. Ask: "[sample question that shows improvement]"
3. Expected: [what the answer should look like]
4. Show: [dashboard/trace/eval proving the improvement]
```
