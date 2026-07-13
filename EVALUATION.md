# Auditly extraction engine — evaluation methodology & results

*By [Tetu Mbah](https://github.com/TetuM7) · tetum7@gmail.com*

This is a sanitized summary of how the Auditly bank-statement extraction engine
is measured. The full result files (~60 benchmark runs) live in the private
application repo under `apps/api/scripts/outputs/`; the numbers below are the
canonical full-engine baseline and are reproducible with a fresh, no-cache run.

## What "correct" means

Extraction is scored **per field** against a hand-labeled ground-truth manifest
for each statement (institution, statement period, account number, ending
balance, and the movement totals). A field is a **pass** only on an exact match
(numbers to the penny). "Full pass" means *every* field on a statement is exact —
a deliberately strict bar.

On top of field accuracy, the engine enforces a **reconciliation tie-out** as a
self-check: `beginning + deposits − withdrawals − fees − checks = ending`. This
means a wrong numeric extraction usually **fails its own arithmetic**, so it's
flagged rather than silently trusted — the metric and the product share the same
gate.

## Methodology

- **Fresh, 0-cache runs.** The canonical benchmark is a full end-to-end run with
  no cached results — real docling parsing, real extraction, real (capped) LLM
  calls where the deterministic layers abstain.
- **Holdback / protected fixtures.** A subset of fixtures is held back and
  tracked run-over-run to catch regressions and guard against overfitting to the
  visible corpus (`manifest_holdback.json` per bank).
- **Cost + latency recorded.** Every run records per-document USD cost and
  latency distributions (min / p50 / p95 / max), not just accuracy.
- **Deterministic contribution logged.** Each field records which layer produced
  it (Tier-1 deterministic, bank specialist, register parser, or LLM
  break-glass), so "how much does the LLM actually do?" is a measured number.
- **Corpora.** Real, publicly-available bank-statement PDFs across multiple
  institutions (Chase, Bank of America, Wells Fargo, Capital One, Citi, US Bank)
  plus a staging failure-category set. No private or customer data.

## Canonical baseline (full-engine, fresh run)

| Metric | Value |
|---|---|
| Statements benchmarked | 160 (6 docling-skipped) |
| Runtime errors | **0** |
| Avg cost / document | **$0.094** (p50 $0.046 · p95 $0.33) |
| Avg latency / document | 10.7 s (p50 9.1 s) |

**Field-level accuracy (v3.1 corpus, 97 statements):**

| Field | Accuracy |
|---|---|
| Institution | **85.6%** |
| Ending balance | **84.5%** |
| Account # (last 4) | **81.4%** |
| Period start / end | **76.3%** |
| All-fields-exact ("full pass") | 49.5% |

**Deterministic contribution:** the institution field is resolved **100%
deterministically** (0 LLM calls) — the "LLM only when needed" thesis, measured.
Account #, period, and ending balance mix deterministic + LLM-fallback depending
on how cleanly the bank's layout exposes them.

## Reading the numbers honestly

- **Field-level accuracy (76–86%)** is the useful headline: most fields, most of
  the time, extracted exactly from real-world statements.
- **All-fields-exact (49.5%)** is intentionally strict — one wrong field on a
  statement fails the whole thing. It's the number I optimize against precisely
  *because* it's unforgiving, and it's the target of the ongoing per-bank and
  LLM-fallback improvement work (the `dls-*`, `tpl-*`, and `phase-5b-d-*` runs).
- The engine is **deterministic-first by design**: the goal isn't "the LLM reads
  the statement," it's "the deterministic layers do as much as possible, the
  reconciliation gate validates it, and the LLM only fills what's left — never
  authoritatively."

## Ongoing work

- Broader per-bank deterministic coverage for messy summary-table layouts.
- A measured prompt-eval loop for the LLM-fallback fields (deposits/withdrawals
  totals on statements where docling splits labels from values).
- Grounded-copilot safety evals (context-leak / fabrication) alongside extraction.

---

*Want the raw result JSON or a walkthrough of the harness? Reach me at
tetum7@gmail.com.*
