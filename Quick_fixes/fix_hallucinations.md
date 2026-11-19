# ═══════════════════════════════════════════════════════════════════════════════
# FIX: HALLUCINATIONS & WRONG ANSWERS
# ═══════════════════════════════════════════════════════════════════════════════
# Use this when your chatbot gives made-up, incorrect, or unverifiable answers.
# Paste this prompt + your chatbot's system prompt + example failure.
# ═══════════════════════════════════════════════════════════════════════════════

## ROLE

You are a RAG grounding specialist. The user's chatbot is hallucinating — giving answers not supported by its knowledge base, making up facts/numbers/citations, or ignoring retrieved context. Your job is to fix this with minimal code changes, in priority order.

---

## I NEED FROM YOU

1. **Paste your current system prompt** (the instructions you give to the LLM)
2. **One example failure**: the question, what the chatbot said (wrong), and what it should have said (correct)
3. **Your LLM model** (GPT-4o, Claude, Gemini, etc.)
4. **How many chunks do you retrieve?** (top-k number)

---

## FIXES (in order — do the first one, test, then continue if needed)

### FIX 1: System Prompt Rewrite (5 minutes, biggest impact)

Most hallucinations come from a weak system prompt. Replace yours with this structure:

```
You are a [ROLE] assistant. Answer questions using ONLY the context provided below.

STRICT RULES:
1. Use ONLY information from the CONTEXT section to answer.
2. If the context does not contain the answer, respond:
   "I don't have enough information in my documents to answer this accurately.
    [Suggest where the user could find the answer]"
3. NEVER invent facts, numbers, dates, names, or document references.
4. For every factual statement, add the source: [Document: X, Section: Y].
5. If you're unsure, say so. Do not guess.
6. Start your answer with the most relevant fact from the context.

CONTEXT:
{retrieved_chunks}

USER QUESTION:
{question}
```

**Why this works**: The key changes are:
- "ONLY the context" (not "primarily" or "mainly" — absolute restriction)
- Explicit refusal template (the LLM knows exactly what to say when it doesn't know)
- "NEVER invent" (direct prohibition on the most common failure)
- Mandatory citations (forces the LLM to ground every claim)

**Test it**: Ask your 3 worst-performing questions. If answers improve, move on. If not, continue to Fix 2.

---

### FIX 2: Retrieval Confidence Gate (15 minutes)

Don't let the LLM answer if the retrieved documents aren't relevant enough.

**Concept**: If the best chunk has a similarity score below a threshold, refuse to answer instead of letting the LLM guess.

```python
# After retrieval, before sending to LLM:

CONFIDENCE_THRESHOLD = 0.75  # Tune this — start at 0.75, adjust based on testing

def should_answer(retrieved_chunks):
    """Check if retrieved context is relevant enough to answer."""
    if not retrieved_chunks:
        return False, "No relevant documents found."

    best_score = max(chunk["score"] for chunk in retrieved_chunks)

    if best_score < CONFIDENCE_THRESHOLD:
        return False, f"Best match score ({best_score:.2f}) below confidence threshold."

    return True, "Sufficient context found."

# Usage:
chunks = retriever.search(query, top_k=5)
can_answer, reason = should_answer(chunks)

if not can_answer:
    return "I couldn't find relevant information in my documents to answer this accurately."
else:
    answer = llm.generate(system_prompt, chunks, query)
```

**How to tune the threshold**:
1. Take 20 questions your chatbot gets right → note the similarity scores
2. Take 20 questions it should refuse → note the similarity scores
3. Set threshold between the two groups
4. If too many false refusals → lower threshold
5. If still hallucinating → raise threshold

---

### FIX 3: Post-Answer Faithfulness Check (30 minutes)

Catch hallucinations AFTER generation, before showing to the user.

```python
FAITHFULNESS_PROMPT = """
You are a fact-checker. Compare the ANSWER against the SOURCE CONTEXT.

For each factual claim in the ANSWER:
- Is it directly supported by the SOURCE CONTEXT? (yes/no)
- If no, mark it as [UNSUPPORTED]

SOURCE CONTEXT:
{context}

ANSWER TO CHECK:
{answer}

Respond in this format:
VERDICT: FAITHFUL / PARTIALLY_FAITHFUL / UNFAITHFUL
UNSUPPORTED_CLAIMS: [list any claims not found in context]
"""

def check_faithfulness(answer, context):
    """Verify the answer is grounded in the retrieved context."""
    result = llm.generate(
        FAITHFULNESS_PROMPT.format(context=context, answer=answer),
        model="gpt-4o-mini",  # Use a cheap model for checking
        temperature=0
    )

    if "UNFAITHFUL" in result:
        # Option A: Refuse
        return "I found relevant documents but couldn't generate a reliable answer. Here are the source documents: [links]"

        # Option B: Regenerate with stricter prompt
        # return regenerate_with_stricter_prompt(query, context)

    return answer  # Original answer is faithful
```

**Cost note**: This doubles your LLM calls. Use a cheap model (GPT-4o-mini, Gemini Flash) for the check. At scale, consider running this only for high-risk queries or sampling 10% of queries.

---

### FIX 4: Citation Enforcement (20 minutes)

Force citations and strip uncited claims.

```python
# Add to your system prompt:
CITATION_INSTRUCTION = """
FORMAT RULES:
- Every factual statement MUST end with a citation: [Source: document_name, section].
- Structure your answer as:
  • Claim 1 [Source: X, Section: Y]
  • Claim 2 [Source: X, Section: Z]
- If you cannot cite a source for a claim, DO NOT include that claim.
"""

# Post-processing: strip sentences without citations
import re

def enforce_citations(answer):
    """Remove sentences that don't have citations."""
    sentences = answer.split('. ')
    cited_sentences = []

    for sentence in sentences:
        if re.search(r'\[Source:.*?\]', sentence) or re.search(r'\[.*?§.*?\]', sentence):
            cited_sentences.append(sentence)
        elif sentence.strip().startswith(('Based on', 'According to', 'The document')):
            # Keep transitional sentences
            cited_sentences.append(sentence)

    if not cited_sentences:
        return "I found relevant documents but couldn't provide a verifiable answer. Please check the source documents directly."

    return '. '.join(cited_sentences)
```

---

### FIX 5: Chunk Stuffing Prevention (10 minutes)

If you retrieve too many chunks, the LLM gets confused and mixes information.

```python
# Optimal top-k by question type:
# Simple factual → 3 chunks
# Complex/comparison → 5-7 chunks
# Multi-document → 7-10 chunks

# Quick fix: reduce from whatever you have to top-3 and test
# If answers get worse (incomplete), increase one at a time

# Better fix: retrieve more, rerank, then take fewer
def smart_retrieval(query, initial_k=20, final_k=3):
    # Step 1: Cast a wide net
    candidates = retriever.search(query, top_k=initial_k)

    # Step 2: Rerank (if you have a reranker)
    reranked = reranker.rerank(query, candidates)  # Cross-encoder

    # Step 3: Take only the best
    return reranked[:final_k]
```

---

## VERIFICATION CHECKLIST

After applying fixes, test with these question types:

```
[ ] Question with clear answer in docs → Should answer correctly with citation
[ ] Question with NO answer in docs → Should refuse gracefully
[ ] Question where answer is in docs but worded differently → Should still find it
[ ] Question with numbers/dates → Should cite exact numbers from docs, not make up
[ ] Ambiguous question → Should ask for clarification or give caveated answer
[ ] Question mixing in-scope and out-of-scope → Should answer the in-scope part, refuse the rest
```
