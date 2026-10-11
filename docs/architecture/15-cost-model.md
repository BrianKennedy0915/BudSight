# 15. Estimated Cost Per Audit

Target from brief §33: **< $2–5/audit, preferably substantially below.** Figures below are
estimates based on typical current pricing tiers for the provider categories in
`11-data-providers.md`; exact numbers depend on the final vendor picks (SERP provider
especially) and should be re-measured empirically from `ai_usage` + provider billing once
real audits run (brief §33's own requirement — this table is a pre-build estimate, not a
substitute for the real per-audit cost tracking the schema already supports).

## AI cost (OpenAI, server-side)

| Prompt | Approx. input tokens | Approx. output tokens | Model tier |
|---|---|---|---|
| Website Analyst | 8,000–15,000 (several pages of extracted text/structure) | 1,000–1,500 | small/cheap |
| Competitive Analyst | 10,000–20,000 (client + 3–5 competitors' evidence) | 1,000–2,000 | small/cheap |
| Revenue Analyst | 2,000–4,000 (deterministic metrics only, no page text) | 300–600 | small/cheap |
| Executive Analyst | 4,000–8,000 (finalized findings/scores/evidence summaries) | 800–1,500 | mid-tier |
| **Total per audit** | **~25,000–47,000 input** | **~3,100–5,600 output** | mixed |

At representative current small-model pricing (roughly $0.15–$0.50 per 1M input tokens,
$0.60–$2.00 per 1M output tokens for the cheap tier; a mid-tier model perhaps 3–10× that for
the Executive Analyst's smaller share) plus one retry budget on ~10–15% of calls (validation
failures), estimated AI cost per audit: **$0.03–$0.20**.

## External data-provider cost

| Provider | Calls per audit | Approx. cost per call | Subtotal |
|---|---|---|---|
| Google Places (Place Details + Nearby Search) — client + ~4 competitors | ~8–10 calls | $0.017–$0.032 | $0.15–$0.30 |
| Google Geocoding (if address needs geocoding) | 1–5 calls | ~$0.005 | $0.00–$0.03 |
| SERP/keyword provider | 5–15 queries (visibility check + keyword-volume lookups for traffic proxy) | $0.001–$0.01 | $0.01–$0.15 |
| **Subtotal** | | | **$0.16–$0.48** |

## Compute cost

- Website fetching, HTML parsing/sanitization, PDF rendering (self-hosted Playwright/Chromium):
  Vercel/Inngest function execution time, effectively a fraction of a cent per audit at this
  volume — folded into fixed platform costs, not metered per-audit in any provider's own
  billing.

## Total estimated cost per audit

| Component | Low | High |
|---|---|---|
| AI (OpenAI) | $0.03 | $0.20 |
| Data providers (Places + Geocoding + SERP) | $0.16 | $0.48 |
| Compute (amortized) | ~$0.01 | ~$0.05 |
| **Total** | **~$0.20** | **~$0.73** |

This is comfortably under the $2–5 target, with real margin for the SERP provider turning out
more expensive than estimated, for retries, or for adding a licensed cannabis market-data
subscription (Headset/BDSA) to ground revenue-estimation assumptions per `09-revenue-
estimation.md` — that subscription is typically a flat monthly/annual fee rather than a
per-audit charge, so it doesn't change this per-audit table directly but is a real fixed cost
to budget for separately.

## What actually enforces the cost target in code

- `ai_usage` rows are written on every call (success, failure, and retry) — see
  `10-ai-prompt-architecture.md` — so `estimated_cost` sums to a real, queryable per-audit
  total (`select sum(estimated_cost) from ai_usage where audit_id = ...`), not an estimate.
- A per-audit AI cost ceiling (e.g., hard-abort the analysis stage if cumulative `ai_usage` for
  this audit exceeds a configured threshold, logging an alert) is a cheap safety net worth
  adding in Week 2 — flagged here as a recommended guardrail rather than an already-decided
  requirement, since the brief doesn't ask for a hard ceiling explicitly, only tracking and a
  target.
- Model tier selection (`11-data-providers.md`'s table) is config (`src/lib/ai/models.ts`), so
  a cost regression from an accidental model upgrade is a one-line diff to catch in review, not
  a scattered search-and-replace across prompt call sites.
