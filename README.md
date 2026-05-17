<!--
README.md — v0.11.0 (current codebase, project in standby)
For the full architectural deep dive, see ARCHITECTURE.md.
-->

# ATLAS HS Classifier

> AI-assisted customs tariff classification for freight forwarders and customs brokers.

> **Status: Project in standby (May 2026).**
> Documentation public, codebase private, beta access available on request.

---

## What it is

HS (Harmonised System) code classification is the first step in every cross-border shipment, and for SME exporters and freight forwarders it is still largely manual: an operator reads a commercial invoice line by line, cross-references the Combined Nomenclature, and either commits to a code or escalates. The work is repetitive, time-pressured, and unforgiving — a wrong code can trigger fines, customs holds, or duty reassessment months later.

ATLAS HS Classifier is an AI-assisted pipeline that takes a commercial invoice in and produces an EDI-ready customs declaration out. Under the hood it is a 7-node LangGraph state machine: OCR ingestion, OCR human review, RAG enrichment over ~244k EU EBTI and US CROSS rulings, GRI 1-6 chain-of-thought classification, deterministic and semantic evaluation, classification human review, and multi-format export (XML AIDA 2.0, XML EUCDM, JSON, CSV).

What sets it apart from a generic LLM wrapper: human-in-the-loop is architecturally mandatory, not optional — two `interrupt()` gates sit inside the orchestration graph and cannot be bypassed. GRI reasoning is preserved end-to-end so every classification carries its justification chain. A per-tenant memory store accumulates HITL-confirmed corrections and feeds them back into RAG, so reviewer load decreases as the tenant uses the system.

## Demo

- LinkedIn carousel walkthrough: *[link forthcoming]*

## Architecture at a glance

- 7-node LangGraph pipeline with two HITL gates (OCR review + classification review)
- RAG over ~244k rulings (EBTI 52,259 + CROSS 191,654) on pgvector with HNSW indexing
- Three-layer cost guardrails: circuit breaker (max 3 LLM cycles) + per-tenant token budget + monthly provider spending caps
- Encrypted ephemeral document storage (Fernet, 30-day default retention) for GDPR compliance
- Multi-format export: XML AIDA 2.0, XML EUCDM, JSON, CSV
- Full audit trail per classification decision

**→ See [ARCHITECTURE.md](./ARCHITECTURE.md) for full C4 diagrams, 8 ADRs, and tech stack rationale.**

## Tech stack summary

| Layer | Technology |
|---|---|
| Orchestration | LangGraph (StateGraph, AsyncPostgresSaver, interrupt/resume) |
| LLM | Groq (Llama 3.3 70B) primary, Anthropic Claude optional fallback |
| Embeddings | OpenAI text-embedding-3-small (1536 dims) |
| Vector DB | PostgreSQL 16 + pgvector (HNSW indexing) on Supabase |
| Backend | FastAPI + Jinja2 + HTMX |
| Deploy | Railway + Cloudflare |

Full table with rationale is in [ARCHITECTURE.md](./ARCHITECTURE.md).

## What I learned building this

- **HITL by architecture, not by policy** (ADR 1, ADR 2): compliance-grade AI tooling can't bolt human review on top — it must be a structural node in the orchestration graph. We placed two HITL gates (OCR review before any LLM cost is spent, classification review before any export is emitted) inside the LangGraph pipeline itself. `auto_approve_enabled` defaults to `False` and requires per-tenant legal sign-off to flip. Customs misclassification carries criminal liability under Art. 303 TULD; the human gate is a legal shield, not a quality knob.

- **Cost guardrails need three independent layers** (ADR 6): a circuit breaker on the generator-evaluator feedback loop (`max_validation_cycles = 3`, checked *before* the evaluator's verdict so a corrupted verdict cannot bypass it), a per-tenant daily token budget enforced before each LLM call, and a monthly spending cap on the provider side. Each layer defends against a different failure mode — runaway loop, runaway tenant, runaway month — and each fails closed.

- **Documentation drift accumulates faster than expected**: by mid Phase 2, our architecture document was six concrete facts out of sync with the codebase (wrong node count, Stripe still rendered as a container, `users` instead of `tenant_users`, etc.). Treating `ARCHITECTURE.md` as a code artifact with the same refactoring discipline as the source — and re-deriving it from the running code, not from memory — is non-negotiable. The current revision was rewritten from scratch against verified file references.

- **RAG over a compliance corpus is not generic RAG**: we deduplicate by `ruling_id`, translate non-English entries to English at ingestion time, strip GDPR-sensitive fields (`NAME_AND_ADDRESS`), tune HNSW separately per corpus, and shard into three stores (shared EBTI, shared CROSS, tenant-isolated memory) with the tenant store filtered by `tenant_id` on every query. Off-the-shelf vector pipelines fall apart on any one of these constraints.

- **`interrupt()` + `Command(resume=...)` is the right primitive for human-in-the-loop**: LangGraph's pause/resume model checkpoints state to Postgres at every node boundary via `AsyncPostgresSaver`, which means a pipeline can sit paused for hours or days waiting for an operator and resume cleanly. The same primitive serves both HITL gates without bespoke state management.

- **GDPR-friendly defaults shrink the blast radius** (ADR 4): PDFs land on the Railway container filesystem under a per-tenant, per-session directory, Fernet-encrypted with a single key, and never reach object storage. A separate Railway cron deletes files past `DATA_RETENTION_DAYS` and offuscates PII in derived line items. The data-at-rest surface for a deletion request is one provider (Supabase EU) plus one ephemeral disk.

- **Concierge onboarding validates willingness-to-pay before billing infrastructure exists**: skipping Stripe for the beta saved roughly a month of premature engineering. The signal we needed wasn't "people *can* pay" but "people *want* to pay". The architecture document still shows no `subscriptions` table by design — it would have been speculative scaffolding.

## Why this is paused

After the initial LinkedIn announcement, the demand signal from the target audience (Italian freight forwarders and customs brokers) was insufficient to justify the next investment phase — self-service onboarding, Stripe billing, and the customer success motion required to convert pilot conversations into paid seats. Other projects in the portfolio took priority. The product is functional and the concierge beta remains open for anyone in the customs space who wants to evaluate it on real shipments.

## Beta access

- Contact: andrea.tierno21@yahoo.com
- What "beta" means: concierge onboarding, manual account provisioning, no self-service signup
- No credit card, no commitment — feedback-only

## Repository contents

```
.
├── README.md              # This file
├── ARCHITECTURE.md        # C4 diagrams, 8 ADRs, tech stack rationale
└── LICENSE                # CC BY 4.0 (documentation)
```

The source code is in a separate private repository.

## License & legal

- **Documentation in this repository**: licensed under [CC BY 4.0](./LICENSE)
- **Source code**: private, all rights reserved
- **Disclaimer**: the classifier output is an aid to classification, not customs advice. Final declaration responsibility rests with the declarant under Art. 15 of the Union Customs Code (Regulation EU 952/2013).
- **GDPR**: ephemeral encrypted document storage with configurable retention (default 30 days), tenant data isolation by `tenant_id`. Full data lifecycle in [ARCHITECTURE.md](./ARCHITECTURE.md).

---

*This README documents a project in extended standby. Last reviewed: 2026-05-17.*
