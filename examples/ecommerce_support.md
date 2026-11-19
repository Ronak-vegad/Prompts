# Example: E-Commerce Product Support Chatbot
# ═══════════════════════════════════════════════
# This shows how the CONTEXT BLOCK would look for an e-commerce chatbot,
# and the domain-specific features the optimizer would generate.

## Filled Context Block

```
═══ CHATBOT CONTEXT BLOCK (paste into Phase 2) ═══

CHATBOT_TOPIC: E-Commerce Product Support
CHATBOT_DESCRIPTION: AI assistant that helps customers find products, understand specs, compare items, track orders, and handle returns/refunds
DOMAIN_RISK_LEVEL: medium
FAILURE_CONSEQUENCES: Wrong product recommendations lead to returns; wrong return policy info causes customer frustration and support escalation

USERS_PRIMARY: Online shoppers (non-technical)
USERS_SECONDARY: Customer support agents (semi-technical)
USERS_TECH_LEVEL: non-technical

CONTENT_TYPES: Product catalog (JSON/CSV), FAQ pages (HTML), policy documents (PDF), shipping info (database), user manuals (PDF)
CONTENT_VOLUME: 50,000 products, 200 policy/FAQ pages, 500 user manuals
CONTENT_STRUCTURE: semi-structured
CONTENT_LANGUAGES: English, Spanish
CONTENT_VERSIONED: yes, product catalog updates daily, policies update quarterly
CONTENT_METADATA: product_id, category, brand, price, SKU, policy_type, effective_date

STACK_FRAMEWORK: LangChain
STACK_LLM: GPT-4o-mini
STACK_VECTORDB: Pinecone
STACK_EMBEDDING: text-embedding-3-small
STACK_HOSTING: AWS ECS
STACK_CHUNKING: recursive, 500 tokens, 50 overlap
STACK_RETRIEVAL: vector similarity only, top-5
STACK_RERANKING: no
STACK_CITATIONS: no

PAIN_POINTS: hallucinations on pricing, can't handle product comparisons, slow on complex queries, no evaluation, costs growing
EXISTING_EVAL: 20 manual test questions, no automation

PRIORITY_1: accuracy (especially pricing and policy info)
PRIORITY_2: cost reduction
PRIORITY_3: comparison features
TARGET_LATENCY: 3 seconds
BUDGET: $500/month LLM costs
COMPLIANCE: CCPA (California users)
TEAM_SIZE: 2 developers, 1 PM
TIMELINE: 6 weeks

REQUESTED_FEATURES: product comparison, multilingual (Spanish), feedback loop

═══ END CONTEXT BLOCK ═══
```

## Domain-Specific Features (auto-generated)

For an e-commerce chatbot, the optimizer would generate:

1. **Product Comparison Table** — Side-by-side comparison of 2-4 products with specs, pricing, reviews, and pros/cons
2. **Smart Product Finder** — "I need a laptop for video editing under $1000" → filtered, ranked recommendations with reasoning
3. **Real-Time Price & Availability** — Live inventory/pricing API integration with staleness detection
4. **Return/Refund Policy Navigator** — Step-by-step return guidance based on product category, purchase date, and condition
5. **Order Status Integration** — Connect to order management system for tracking, estimated delivery, and issue resolution
6. **Review Summarizer** — Aggregate customer reviews into pros/cons/common issues for any product
7. **Size/Compatibility Checker** — "Will this case fit my iPhone 15?" — cross-reference dimensions and compatibility data
