# ⚖️ Legal HR RAG Bot — Internal Legal/HR Assistant (R&D, n8n + Pinecone + GPT)

> **Status: R&D prototype, not deployed to production.** Built as an internal proof-of-concept for a company's HR/legal team; the internal stakeholder froze the project for prioritization reasons before a production rollout. Shared here as a case study in iterative architecture decisions — not as a live product.

A Telegram-based RAG assistant that answers employees' HR/legal questions (labor law, internal policies, migration, personal data, e-signature, etc.) by retrieving from a curated FAQ and, when needed, escalating to a full corpus of legal texts — instead of letting an LLM answer from general knowledge.

---

## Why this project exists

HR/legal teams get the same questions repeatedly ("how many vacation days am I entitled to", "can we hire on a GPH contract", "can we share employee data with a partner company"). Many of these have a correct, pre-approved answer; some are genuinely risky and need a human lawyer. The goal was a bot that:

- answers instantly from **pre-approved positions**, not from the model's own legal knowledge,
- is honest when it doesn't know, instead of guessing,
- can fall back to the **full legal corpus** when the curated FAQ has no match,
- keeps every law/policy update outside the model — the model is a *retriever + formatter*, never the source of truth.

---

## How the approach evolved (3 iterations)

This repo intentionally keeps artifacts from all three phases — the point of the case study is the *trade-offs*, not just the final diagram.

### 1. Careful chunking experiments (Python / Colab)
Early work focused on getting retrieval quality right before worrying about orchestration speed:
- law texts split **by article boundaries** ("Статья N"), not naive fixed-size chunks, to avoid cutting a legal provision mid-sentence;
- chunk size/overlap tuned per source type;
- structured metadata per chunk (law code, article number, law name) to make citations traceable back to a specific article.

→ `load_to_pinecone.py`

### 2. Full production-grade architecture (designed, not fully shipped)
Once retrieval worked, the design was extended toward something defensible enough to put in front of a legal team:
- a **topic classifier** LLM call (10 legislative acts: labor code, civil code, tax code, AML law, migration law, personal data law, e-signature law, etc.);
- a dedicated **risk-triage LLM call** checking 8 "RED" categories (dismissal, AML, migration, personal data, penalties, policy/law conflicts, e-signature with legal consequences, staff outsourcing) — any hit forces escalation regardless of what the main answer-generation call says;
- a strict **GREEN / YELLOW / RED** status model with automatic escalation to a human lawyer on RED;
- a full **PostgreSQL audit schema** (`db_schema.sql`) — per-request log, escalation lifecycle tracking, SLA views, feedback scoring;
- full deployment documentation (`docs.html`) covering credentials, environment variables, data formats, test cases, and a go-live checklist.

→ `n8n_workflow.json`, `db_schema.sql`, `docs.html`, `Пайплайн_для_ФАК.docx`

This is the "do it properly" version — designed to be audit-ready and safe enough for a legal/compliance context. It was the reference architecture, not what ended up running.

### 3. Fast MVP — what was actually deployed and tested
Speed of iteration became the priority: validate whether the *core retrieval idea* works for real employee questions before investing further in the risk-classification and audit layers above. The result is deliberately leaner:

- **no separate topic-classification call** and **no separate RED-trigger call** — a single retrieval-and-answer agent per tier;
- **no Postgres** — request logging moved to **Google Sheets** (zero extra hosting, good enough for a pilot);
- **no lawyer-escalation workflow** — the bot either answers or tells the user it found nothing;
- retrieval split into **two Pinecone namespaces** inside one index instead of two separate classification-driven searches: `faq` (curated, high-trust positions) tried first, `new2` (full raw document corpus) as a fallback.

→ `legal-hr-rag-bot.n8n.json` *(sanitized export of the actual running workflow — see `rag_law_view.jpeg` for a real execution trace)*

**Net result:** the MVP is *not* a scaled-down copy of the full design — the risk-triage and audit layers were deliberately engineered first, then set aside in favor of shipping something that could be pointed at real users quickly and cheaply. That governance layer design is kept in this repo precisely because it shows the harder architecture was already solved; it just wasn't needed to validate the product hypothesis.

---

## Final (deployed MVP) architecture

```
Telegram (message)
   ├──► Append/Update row (Google Sheets) — audit log
   └──► Edit Fields (text, sessionId)
            │
            ▼
        AI Agent1  ──tool──► Vector Store Tool ──► Pinecone[faq] (topK=4)
            │
            ├──► Send to Telegram (instant reply to user)
            │
            └──► Detect empty result (JS) ──► If(hasResults?)
                                                 │
                                    false ───────┤
                                                 ▼
                              Send to Telegram (holding message: "checking legal DB")
                                                 │
                              ┌──────────────────┼───────────────────┐
                              ▼                                      ▼
                  Normalize Query (JS)                        AI Agent (fallback)
                        │                                      │ tool
                        ▼                                      ▼
             Pinecone[new2] (semantic load)          Vector Store Tool1 → Pinecone[new2]
                        │                                      │
                        ▼                                      ▼
             Code: format source links               Send to Telegram (text answer)
                        │
                        ▼
             Send to Telegram (list of sources)
```

**Stack:** n8n (orchestration) · Telegram Bot API · OpenAI `gpt-4.1-mini` (2 agent nodes) · Pinecone (1 index, 2 namespaces, 512-dim embeddings) · Google Sheets (logging)

### Key engineering decisions worth calling out
- **Anti-hallucination guardrail in the prompt itself.** The agent's system prompt hard-codes an exact refusal phrase for "no relevant data found" — no hedging, no answering from general knowledge. This gives the pipeline a single, reliably parseable signal for routing.
- **Empty-result detection without a second LLM call.** A cheap JS node scans the agent's own output for known refusal phrases instead of spending another model call to judge confidence.
- **Asymmetric response timing.** The FAQ-tier answer is sent to the user immediately, before the empty-result check even finishes running — the common case (FAQ has the answer) feels instant; the deeper corpus search only costs extra latency when it's actually needed.
- **Namespace partitioning, not separate indexes.** `faq` and `new2` live in the same Pinecone index — simpler to operate, while still keeping the trust hierarchy between "vetted positions" and "raw corpus" intact.

---

## What's in this repo

| File | What it is |
|---|---|
| `legal-hr-rag-bot.n8n.json` | **Sanitized** export of the actual running workflow (credentials/IDs/sheet links replaced with placeholders — see note below) |
| `n8n_workflow.json` | The fuller reference architecture (classification + risk-triage + Postgres + lawyer escalation) — designed, not fully deployed |
| `load_to_pinecone.py` | Article-aware chunking + embedding/upload script for laws and FAQ |
| `db_schema.sql` | Full audit-log schema for the reference architecture (Postgres/Supabase) |
| `docs.html` | Deployment documentation for the reference architecture |
| `Пайплайн_для_ФАК.docx` | Original concept/architecture doc (Russian) |
| `rag_law_view.jpeg` | Screenshot of a real successful execution of the deployed workflow |

> **Note on the sanitized JSON:** credential IDs, webhook IDs, and the Google Sheet ID/URL have been replaced with `YOUR_*` placeholders. To run it yourself, re-create credentials for Telegram, OpenAI, Pinecone and Google Sheets in your own n8n instance, point the Google Sheets node at your own spreadsheet, and create a Pinecone index named `legis` with `faq` and `new2` namespaces (or rename to match your own).

---

## Why it was frozen, not what killed it

The prototype worked as intended on test queries (see execution screenshot) — it wasn't shelved for a technical failure. The internal stakeholder paused the initiative for resourcing/prioritization reasons before a production decision was made. The risk-triage/audit architecture above was designed in anticipation of a production rollout that didn't happen yet.

## What I'd do differently next time
- Validate the MVP retrieval hypothesis *before* building out the full risk-classification/audit layer, rather than in parallel — would have saved the design-ahead-of-need work.
- Add a lightweight feedback signal (👍/👎 per answer) even in the MVP — it was designed into the Postgres schema but never wired into the shipped version.
- Keep FAQ and full-corpus retrieval metrics separate from day one, to know how often the fallback tier is actually needed in practice.
