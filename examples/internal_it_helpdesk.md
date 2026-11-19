# Example: Internal IT Knowledge Base Chatbot
# ═══════════════════════════════════════════════
# Low-to-medium risk example for internal company use.

## Filled Context Block

```
═══ CHATBOT CONTEXT BLOCK (paste into Phase 2) ═══

CHATBOT_TOPIC: Internal IT Helpdesk & Knowledge Base
CHATBOT_DESCRIPTION: AI assistant for employees to troubleshoot IT issues, find internal docs, understand company policies, and submit tickets
DOMAIN_RISK_LEVEL: low
FAILURE_CONSEQUENCES: Employee wastes time with wrong troubleshooting steps; may need to escalate to human agent anyway

USERS_PRIMARY: Company employees (all departments, mixed technical level)
USERS_SECONDARY: IT support team (uses it as a co-pilot)
USERS_TECH_LEVEL: mixed

CONTENT_TYPES: Confluence wiki pages (HTML), Jira knowledge base articles, IT runbooks (Markdown), company policies (PDF), video transcripts (TXT)
CONTENT_VOLUME: 5,000 wiki pages, 800 KB articles, 200 runbooks, 50 policy docs
CONTENT_STRUCTURE: semi-structured (wiki pages have headings but inconsistent formatting)
CONTENT_LANGUAGES: English
CONTENT_VERSIONED: yes, wiki pages update frequently, policies update annually
CONTENT_METADATA: page_id, space/category, last_updated, author, tags, department

STACK_FRAMEWORK: Custom Python (FastAPI + OpenAI SDK)
STACK_LLM: GPT-4o-mini
STACK_VECTORDB: ChromaDB
STACK_EMBEDDING: text-embedding-3-small
STACK_HOSTING: Internal Kubernetes cluster
STACK_CHUNKING: fixed-size, 1000 tokens, 200 overlap
STACK_RETRIEVAL: vector similarity, top-3
STACK_RERANKING: no
STACK_CITATIONS: partial (links to wiki page but not specific section)

PAIN_POINTS: outdated info returned (old wiki pages), can't find recently added content, no eval, retrieval misses for exact error messages
EXISTING_EVAL: none
EXISTING_FEEDBACK: Slack channel where users complain

PRIORITY_1: retrieval accuracy (find the right document)
PRIORITY_2: freshness (prefer recent content)
PRIORITY_3: cost (keep it minimal)
TARGET_LATENCY: 5 seconds (internal tool, less pressure)
BUDGET: $200/month
COMPLIANCE: SOC2 (internal data stays internal)
TEAM_SIZE: 1 developer (part-time)
TIMELINE: 4 weeks

REQUESTED_FEATURES: auto-ticket creation, freshness scoring, feedback buttons

═══ END CONTEXT BLOCK ═══
```

## Domain-Specific Features (auto-generated)

1. **Smart Ticket Creation** — If the chatbot can't resolve, auto-draft a Jira/ServiceNow ticket with context from the conversation
2. **Freshness-Weighted Retrieval** — Boost recently updated documents, flag answers from pages older than 6 months
3. **Error Message Search** — Exact-match index for error codes/messages (BM25 heavily weighted for these queries)
4. **Guided Troubleshooting** — Step-by-step diagnostic flows: "Is the VPN connected? → Yes/No → Next step"
5. **Department-Aware Routing** — Answers tailored to user's department (HR policies for HR, dev tools for engineering)
6. **Related Articles** — "People who asked this also found these helpful" based on click/feedback data

## Key Differences from High-Risk Chatbots

Because this is **low risk**, the optimizer would relax:
- 50 golden eval questions is sufficient
- Citations: link to source page (no need for section-level)
- No post-generation faithfulness check needed (save cost)
- No mandatory disclaimer
- Simpler monitoring: basic error rate + user feedback only
- Budget-focused: aggressive caching, smallest model possible, minimal infrastructure
