# Example: Healthcare / Medical Information Chatbot
# ═══════════════════════════════════════════════════
# High-risk domain example showing stricter requirements.

## Filled Context Block

```
═══ CHATBOT CONTEXT BLOCK (paste into Phase 2) ═══

CHATBOT_TOPIC: Healthcare Clinical Guidelines
CHATBOT_DESCRIPTION: AI assistant for healthcare professionals to quickly look up clinical guidelines, drug information, treatment protocols, and diagnostic criteria
DOMAIN_RISK_LEVEL: critical
FAILURE_CONSEQUENCES: Wrong dosage information or contraindication miss could harm patients; incorrect diagnostic criteria could lead to misdiagnosis

USERS_PRIMARY: Physicians, nurses, pharmacists (highly technical in domain)
USERS_SECONDARY: Medical students, residents
USERS_TECH_LEVEL: mixed (domain experts but not tech experts)

CONTENT_TYPES: Clinical practice guidelines (PDF), drug databases (structured JSON), ICD-10 codes (database), journal articles (PDF), hospital protocols (DOCX)
CONTENT_VOLUME: 2,000 guidelines, 15,000 drug monographs, 70,000+ ICD codes
CONTENT_STRUCTURE: structured (guidelines have numbered sections, drug info has fixed schemas)
CONTENT_LANGUAGES: English
CONTENT_VERSIONED: yes, guidelines update annually, drug info updates monthly
CONTENT_METADATA: guideline_id, publishing_body (WHO, AHA, NICE), year, evidence_level, drug_name, NDC, ICD_code, specialty

STACK_FRAMEWORK: LlamaIndex
STACK_LLM: Claude 3.5 Sonnet
STACK_VECTORDB: Weaviate
STACK_EMBEDDING: voyage-large-2
STACK_HOSTING: GCP Cloud Run
STACK_CHUNKING: by section heading, ~400 tokens
STACK_RETRIEVAL: hybrid (vector + keyword), top-10
STACK_RERANKING: Cohere Rerank v3
STACK_CITATIONS: yes, guideline name + section number

PAIN_POINTS: occasional hallucinated dosages, can't handle drug-drug interactions well, slow for multi-hop questions, no automated eval
EXISTING_EVAL: 50 test questions reviewed by clinical team, manual testing only

PRIORITY_1: accuracy (zero tolerance for wrong dosages or contraindications)
PRIORITY_2: citation traceability (every claim must be verifiable)
PRIORITY_3: speed (clinicians need answers fast)
TARGET_LATENCY: 2 seconds
BUDGET: $2,000/month
COMPLIANCE: HIPAA, FDA 21 CFR Part 11
TEAM_SIZE: 3 developers, 1 clinical advisor, 1 compliance officer
TIMELINE: 12 weeks

REQUESTED_FEATURES: drug interaction checker, guideline comparison (old vs new), clinical decision support

═══ END CONTEXT BLOCK ═══
```

## Domain-Specific Features (auto-generated)

For a healthcare chatbot, the optimizer would generate:

1. **Drug Interaction Checker** — Cross-reference multiple medications for interactions, contraindications, and dosage adjustments; severity classification (major/moderate/minor)
2. **Evidence Level Tagging** — Every recommendation tagged with evidence quality (Level I–IV) and recommendation strength (A–D) from the source guideline
3. **Clinical Decision Support** — Given symptoms/lab values, suggest relevant guidelines and diagnostic pathways with confidence levels
4. **Guideline Version Diff** — "What changed in the 2024 AHA hypertension guidelines vs 2023?" — structured comparison with clinical significance
5. **Formulary Awareness** — Flag if recommended medications are on/off the hospital formulary, suggest alternatives
6. **Patient-Specific Adjustments** — Adjust recommendations based on patient factors (renal function, age, pregnancy, allergies) when provided
7. **Audit Trail** — Every answer logged with full provenance for regulatory compliance (HIPAA/FDA); immutable trace
```

## Key Differences from Low-Risk Chatbots

Because this is **critical risk**, the optimizer would enforce:
- 200+ golden eval questions, reviewed by clinical staff
- Citation on EVERY claim (guideline + section + evidence level)
- Post-generation faithfulness check with regeneration on failure
- Mandatory disclaimer: "This is clinical decision support, not a substitute for clinical judgment"
- Double retrieval verification for drug dosages
- HIPAA-compliant logging (PII encrypted, access audited)
- Stricter hallucination threshold: refuse rather than guess
