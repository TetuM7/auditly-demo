# Auditly — engineering case study

**Built by [Tetu Mbah](https://github.com/TetuM7)** · Full-Stack & Applied AI Engineer
GitHub: [github.com/TetuM7](https://github.com/TetuM7) · Email: tetum7@gmail.com

- **Live demo:** https://auditly.147.135.113.6.sslip.io/start (one-click "doors" — no signup)
- **Landing page:** https://tetum7.github.io/auditly-demo/
- **Application source:** private — walkthrough / read access available to reviewers on request.

This repo hosts the public landing page and this engineering write-up. The
application itself is a private monorepo; this document is the sanitized
technical case study so you can evaluate the work without the proprietary code.

> **Data honesty.** The extraction engine is validated against a corpus of
> **real, publicly-available bank-statement PDFs** (public templates / sample
> statements — real formats, no private data). The interactive demo uses
> **entirely synthetic** companies, people, and statements. No real, private, or
> customer financial data is exposed anywhere in the public demo.

---

## 1. What it is

Auditly is an **audit-prep platform**. It moves the painful front of a financial
audit — collecting, classifying, and tying out a client's evidence — to the
start, and automates the parts a machine should own. Two roles share one
engagement: the **firm** authors and sends a request list; the **client**
responds, has their statements extracted and reconciled on upload, and gets an
honest readiness verdict before fieldwork begins.

The distinctive engineering claim is **trust**: an extracted financial figure is
never accepted until it reconciles against its own source, and the LLM sits
*after* that gate, not in front of it.

## 2. The hardest problem, and how it's solved

**Problem: how do you use an LLM on financial documents without ever trusting a
number it might have hallucinated?**

The naïve approach — "ask the model to read the statement" — is unacceptable for
audit evidence: a plausible-but-wrong figure is worse than no figure. My design
inverts the usual order:

1. **Deterministic-first.** Layout parsing (docling) plus bank-specific
   anchor/regex extractors and a line-by-line register parser do the primary
   work. Most documents never call an LLM at all.
2. **A reconciliation gate is the trust boundary.** A figure is only accepted
   when the statement's own roll-forward ties out to the penny:
   `beginning + deposits − withdrawals − fees − checks = ending`, within a small
   tolerance. If it doesn't close, the value is **flagged for review, never
   silently trusted.**
3. **The LLM is break-glass, downstream of the gate.** For the few fields the
   deterministic layers abstain on, a hard-capped model call fills the gap — and
   its output is re-validated the same way. It fills; it doesn't decide.
4. **Provenance on every field.** The output records exactly which layer
   produced each value (deterministic tier, bank specialist, register parser, or
   LLM break-glass), so you can always answer "how was this number produced?".

This is the defensible version of "the AI never invents": **LLM output is never
accepted as authoritative without deterministic validation and source
provenance.**

## 3. Architecture & data flow

```
Upload (client workspace)
   ↓
API + auth              Express · JWT · Postgres RLS scopes every read/write
   ↓
Object storage          S3-compatible (MinIO); job enqueued (Redis/BullMQ)
   ↓
Async worker            scan → docling layout parse → classify → deterministic extract
   ↓
Reconciliation gate     ── TRUST BOUNDARY ── figures must tie out or are flagged
   ↓
LLM fallback            break-glass only, hard-capped, then re-validated at the gate
   ↓
Postgres + evidence     reconciled transactions + append-only, signed audit trail
   ↓
Auditor · auditee       role-aware workspaces read the same substrate through RLS
```

**Stack** (typed end-to-end, Turborepo + pnpm monorepo — `apps/{web,api,workers}`,
`packages/{shared,ui,…}`):

| Layer            | Technology |
|------------------|------------|
| Frontend         | Next.js 16 · React 19 |
| API              | Express · JWT · Postgres RLS |
| Async / pipeline | Workers · Redis / BullMQ |
| Data             | Postgres · Drizzle ORM · pgvector |
| Object storage   | MinIO (S3-compatible) |
| Authorization    | SpiceDB (relationship-based) |
| Document text    | docling (layout-aware OCR/parse) |
| LLM              | Anthropic Haiku (capped) · local model fallback |

## 4. Engineering highlights (my personal contribution)

Auditly is a **solo build** — I designed, implemented, tested, and deployed all
of it. The load-bearing pieces:

- **DB-enforced multi-tenancy.** Postgres row-level security on every tenant
  table, applied through a single scoped-query helper so isolation can't be
  forgotten per-endpoint. A cross-tenant read returns **404, not 403** (existence
  isn't leaked). Behind the reverse proxy, `trust proxy` is set so per-IP rate
  limits key on the real client, not the proxy.
- **Async document pipeline with real failure handling.** Idempotent worker
  stages; an unrecoverable error lands the document in a terminal `failed` state
  with a reason instead of hanging in `processing` forever. The
  extraction→transactions bridge is keyed on **content hash**, so a re-upload of
  identical bytes can't double-count the population.
- **Deterministic reconciliation gate** (section 2) — the core trust mechanism.
- **Grounded-AI layer.** One reusable copilot spine across four surfaces; the
  model only ever sees an **allow-listed grounding bundle** and defers to the
  engine. A safety-eval suite checks for context leakage and fabrication.
- **Signed, append-only evidence.** Control-test records are **Ed25519-signed
  per tenant**, so a forged or modified historical record fails signature
  verification — genuine tamper-*evidence*, not just an append-only claim.
- **Containerized deployment.** The whole stack runs from one Compose file
  behind Caddy auto-TLS on a VPS, with per-IP rate limiting, a hard LLM spend
  cap, a local-model fallback, and an **unattended nightly reset** (stop → drop →
  migrate → seed → restart, with a loud failure trap) that restores a clean
  seeded demo.
- **Observability.** Every model call is logged with provider, model, tokens,
  latency, and USD cost; every capture emits to the tagged audit trail.

## 5. Testing & validation

- **Extraction** is validated against a labeled corpus of real, public
  bank-statement PDFs with golden expected fields; the reconciliation tie-out is
  a self-checking invariant on every run (a wrong number usually fails its own
  arithmetic).
- **API / data layer**: integration tests run against a real Postgres instance
  (migrations replayed), exercising the RLS boundary, the PBC state machine, and
  the evidence lifecycle — plus unit tests across the shared engine.
- **Grounded AI**: a safety-eval suite for the copilots checks the two failure
  modes that matter — leaking out-of-scope context, and fabricating figures the
  engine didn't produce.

## 6. Security & operational design

- Tenant isolation enforced at the database (RLS), not just the app layer.
- CORS restricted to an allowlist; auth throttled per-IP with a separate, looser
  budget for token refresh so a mistyped login can't lock a user out.
- Every infra port bound to loopback; only 80/443 face the internet, via Caddy.
- LLM spend hard-capped with a free local-model fallback so the demo can't
  overspend or break.
- Append-only audit trail + per-tenant Ed25519-signed control records.

## 7. What's simulated vs production-ready

Honesty matters more than polish here:

- **Production-style, not production-scale.** Real architecture, real
  reconciliation, real deployment — on a single small VPS with synthetic demo
  data. Not carrying live traffic or regulated data.
- The **local-LLM fallback** is wired but the demo runs primarily on capped
  Haiku; the box is too small to serve a large local model under load.
- Demo **engagements and statements are seeded and synthetic.**
- Reliable transaction extraction from *arbitrary* real statements is an ongoing
  extraction-quality effort (deterministic-first covers common layouts; the
  LLM-fallback for messy summary tables is a measured, in-progress improvement).

## 8. Known limitations / next steps

- Custom demo domain (currently a raw-IP `sslip.io` host).
- A short recorded walkthrough video.
- Broader per-bank deterministic coverage + a measured prompt-eval loop for the
  LLM-fallback fields.

## 9. Run the demo

Open **https://auditly.147.135.113.6.sslip.io/start** and pick a door:

- **Client** — a live engagement with pending requests and a real readiness score.
- **Firm** — the same engagement from the auditor's side.
- **Start fresh** — the onboarding wizard from an empty account.

---

*Built by Tetu Mbah. For a code walkthrough, the engineering deep-dive, or to
discuss the design decisions above, reach me at tetum7@gmail.com.*
