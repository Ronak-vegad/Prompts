# ═══════════════════════════════════════════════════════════════════════════════
# FIX: NO EVALUATION — "I Have No Idea If My Chatbot Is Good"
# ═══════════════════════════════════════════════════════════════════════════════
# Use this when you have no testing, no metrics, no way to measure quality.
# This is ALWAYS Fix #0 — do it before anything else.
# ═══════════════════════════════════════════════════════════════════════════════

## ROLE

You are a QA engineer for RAG systems. The user has no evaluation framework. Help them build one in under 2 hours that they can run on every change.

---

## I NEED FROM YOU

1. **What does your chatbot do?** (topic, one line)
2. **5 example questions** your users commonly ask
3. **2 example failures** (questions where the chatbot gave a wrong or bad answer)

---

## STEP 1: Build a Starter Eval Set (30 minutes)

You don't need 100 questions on day one. Start with 25, then grow.

```python
# eval_set.json — start with this structure
eval_set = [
    # === EASY: Direct lookups (5+ questions) ===
    {
        "id": "easy_01",
        "question": "What is [specific fact from your docs]?",
        "expected_answer": "The answer from your docs, verbatim or close",
        "expected_source": "document_name, section X",
        "difficulty": "easy",
        "category": "factual_lookup",
        "should_refuse": False
    },

    # === MEDIUM: Needs some reasoning (5+ questions) ===
    {
        "id": "med_01",
        "question": "How do I [process/procedure from your docs]?",
        "expected_answer": "Step-by-step process...",
        "expected_source": "document_name, section Y",
        "difficulty": "medium",
        "category": "process",
        "should_refuse": False
    },

    # === HARD: Cross-document or complex (5+ questions) ===
    {
        "id": "hard_01",
        "question": "What's the difference between [X] and [Y]?",
        "expected_answer": "Comparison...",
        "expected_source": "doc_A section Z + doc_B section W",
        "difficulty": "hard",
        "category": "comparison",
        "should_refuse": False
    },

    # === SHOULD REFUSE: Out-of-scope (5+ questions) ===
    {
        "id": "refuse_01",
        "question": "What's the weather today?",
        "expected_answer": "[Should refuse — not in documents]",
        "expected_source": "none",
        "difficulty": "easy",
        "category": "out_of_scope",
        "should_refuse": True
    },

    # === ADVERSARIAL: Trick questions (5+ questions) ===
    {
        "id": "trick_01",
        "question": "[Question that sounds right but the answer in docs is different from common belief]",
        "expected_answer": "The correct answer from docs (not the common misconception)",
        "expected_source": "document_name, section X",
        "difficulty": "hard",
        "category": "adversarial",
        "should_refuse": False
    },
]

# LLM can help you generate more questions:
GENERATE_QUESTIONS_PROMPT = """
You are creating a test set for a chatbot about {topic}.

Here are some example documents from the knowledge base:
{sample_documents}

Generate {count} test questions in these categories:
- 5 easy factual lookups (answer is directly in one document)
- 5 medium process/how-to questions
- 3 hard cross-document questions
- 3 out-of-scope questions (the chatbot should refuse)
- 2 trick questions (seem simple but have surprising answers)

For each question, provide:
- question
- expected_answer (from the documents)
- expected_source (document name + section)
- difficulty (easy/medium/hard)
- category
- should_refuse (true/false)

Output as JSON array.
"""
```

---

## STEP 2: Automated Eval Script (45 minutes)

```python
# evaluate.py — run this on every change

import json
import time
from dataclasses import dataclass

@dataclass
class EvalResult:
    question_id: str
    question: str
    expected_answer: str
    actual_answer: str
    expected_source: str
    retrieved_sources: list
    retrieval_hit: bool      # Did we retrieve the right document?
    answer_correct: bool     # Is the answer factually correct?
    refusal_correct: bool    # Did it refuse when it should (or answer when it should)?
    latency: float
    tokens_used: int

def evaluate_single(test_case, chatbot):
    """Evaluate a single test case."""
    start = time.time()
    result = chatbot.query(test_case["question"])  # Your chatbot's query function
    latency = time.time() - start

    # Check retrieval: did we find the right document?
    retrieved_ids = [chunk["source"] for chunk in result.get("chunks", [])]
    retrieval_hit = test_case["expected_source"] in " ".join(retrieved_ids)

    # Check refusal accuracy
    is_refusal = any(phrase in result["answer"].lower() for phrase in [
        "i don't have", "i couldn't find", "not in my documents",
        "i'm not able to", "outside my knowledge"
    ])
    refusal_correct = (test_case["should_refuse"] == is_refusal)

    # Check answer correctness (LLM-as-judge)
    answer_correct = llm_judge(
        test_case["question"],
        test_case["expected_answer"],
        result["answer"]
    )

    return EvalResult(
        question_id=test_case["id"],
        question=test_case["question"],
        expected_answer=test_case["expected_answer"],
        actual_answer=result["answer"],
        expected_source=test_case["expected_source"],
        retrieved_sources=retrieved_ids,
        retrieval_hit=retrieval_hit,
        answer_correct=answer_correct,
        refusal_correct=refusal_correct,
        latency=latency,
        tokens_used=result.get("tokens", 0)
    )

def llm_judge(question, expected, actual):
    """Use a cheap LLM to judge if the answer is correct."""
    prompt = f"""
    Compare the EXPECTED answer to the ACTUAL answer for this question.
    Are they semantically equivalent? (The actual answer can use different
    words but must convey the same facts.)

    Question: {question}
    Expected: {expected}
    Actual: {actual}

    Respond with ONLY: CORRECT or INCORRECT
    """
    result = llm.generate(prompt, model="gpt-4o-mini", temperature=0)
    return "CORRECT" in result.upper()

def run_full_eval(eval_set_path, chatbot):
    """Run the complete evaluation."""
    with open(eval_set_path) as f:
        test_cases = json.load(f)

    results = [evaluate_single(tc, chatbot) for tc in test_cases]

    # Calculate metrics
    metrics = {
        "total": len(results),
        "retrieval_recall": sum(r.retrieval_hit for r in results) / len(results),
        "answer_accuracy": sum(r.answer_correct for r in results) / len(results),
        "refusal_accuracy": sum(r.refusal_correct for r in results) / len(results),
        "avg_latency": sum(r.latency for r in results) / len(results),
        "p95_latency": sorted([r.latency for r in results])[int(len(results) * 0.95)],
        "total_tokens": sum(r.tokens_used for r in results),
    }

    # Print results
    print("=" * 60)
    print("EVALUATION RESULTS")
    print("=" * 60)
    print(f"📊 Retrieval Recall:  {metrics['retrieval_recall']:.1%}")
    print(f"✅ Answer Accuracy:   {metrics['answer_accuracy']:.1%}")
    print(f"🚫 Refusal Accuracy:  {metrics['refusal_accuracy']:.1%}")
    print(f"⏱️  Avg Latency:      {metrics['avg_latency']:.2f}s")
    print(f"⏱️  P95 Latency:      {metrics['p95_latency']:.2f}s")
    print(f"🪙 Total Tokens:     {metrics['total_tokens']:,}")
    print("=" * 60)

    # Show failures
    failures = [r for r in results if not r.answer_correct]
    if failures:
        print(f"\n❌ FAILURES ({len(failures)}):")
        for f in failures:
            print(f"  [{f.question_id}] {f.question[:60]}...")
            print(f"    Expected: {f.expected_answer[:80]}...")
            print(f"    Got:      {f.actual_answer[:80]}...")
            print()

    # Save results for comparison
    timestamp = time.strftime("%Y%m%d_%H%M%S")
    output = {
        "timestamp": timestamp,
        "metrics": metrics,
        "results": [vars(r) for r in results]
    }
    with open(f"eval_results/{timestamp}.json", "w") as f:
        json.dump(output, f, indent=2)

    return metrics

# CI gate: fail if below thresholds
THRESHOLDS = {
    "retrieval_recall": 0.80,
    "answer_accuracy": 0.85,
    "refusal_accuracy": 0.90,
    "p95_latency": 3.0
}

metrics = run_full_eval("eval_set.json", my_chatbot)
for metric, threshold in THRESHOLDS.items():
    if metric == "p95_latency":
        assert metrics[metric] <= threshold, f"❌ {metric}: {metrics[metric]:.2f} > {threshold}"
    else:
        assert metrics[metric] >= threshold, f"❌ {metric}: {metrics[metric]:.1%} < {threshold:.1%}"

print("✅ All thresholds passed!")
```

---

## STEP 3: Compare Before vs After (10 minutes)

```python
# compare_evals.py — compare two eval runs

import json
import sys

def compare(before_path, after_path):
    with open(before_path) as f:
        before = json.load(f)
    with open(after_path) as f:
        after = json.load(f)

    print(f"\n📈 Comparison: {before['timestamp']} → {after['timestamp']}")
    print("=" * 65)
    print(f"{'Metric':<25} {'Before':>10} {'After':>10} {'Change':>10}")
    print("-" * 65)

    for key in before["metrics"]:
        b = before["metrics"][key]
        a = after["metrics"][key]
        if isinstance(b, float) and b < 1:
            change = f"{(a-b)*100:+.1f}%"
            print(f"{key:<25} {b:>9.1%} {a:>9.1%} {change:>10}")
        elif isinstance(b, float):
            change = f"{a-b:+.2f}s" if "latency" in key else f"{a-b:+.0f}"
            print(f"{key:<25} {b:>10.2f} {a:>10.2f} {change:>10}")
        else:
            print(f"{key:<25} {b:>10} {a:>10} {a-b:>+10}")

# Usage: python compare_evals.py eval_results/before.json eval_results/after.json
if __name__ == "__main__":
    compare(sys.argv[1], sys.argv[2])
```

---

## GROWING YOUR EVAL SET

After the initial 25, grow to 100+ over time:

```
Week 1: Start with 25 manually written questions
Week 2: Add 15 from real user questions (from logs)
Week 3: Add 15 from failure cases (questions users thumbs-downed)
Week 4: Add 20 generated by LLM + reviewed by you
Week 5: Add 25 edge cases and adversarial questions
= 100 questions in 5 weeks, organically grown from real usage
```

**Key rule**: Every time a user reports a bad answer, add it to the eval set. Your test suite grows from real failures, so it protects against real problems.
