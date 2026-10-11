# BudSight Architecture Review — Week 1

**Status:** Draft for approval. No application code has been written. This package is the
Week 1 "Product Definition + Architecture" deliverable called for in `BUDSIGHT_MASTER_BRIEF.md`
§40 and §54.

## What's in this package

| # | Document | Covers |
|---|---|---|
| 1 | [01-repo-structure.md](./01-repo-structure.md) | Repository layout |
| 2 | [02-system-architecture.md](./02-system-architecture.md) | System architecture, job engine choice |
| 3 | [03-database-schema.md](./03-database-schema.md) | ERD + full schema |
| 4 | [04-rls-strategy.md](./04-rls-strategy.md) | Supabase RLS strategy |
| 5 | [05-api-contract.md](./05-api-contract.md) | API contract |
| 6 | [06-audit-pipeline.md](./06-audit-pipeline.md) | Audit state machine, retries, idempotency |
| 7 | [07-domain-models.md](./07-domain-models.md) | TypeScript domain models |
| 8 | [08-scoring-algorithms.md](./08-scoring-algorithms.md) | Opportunity / Confidence / BudSight Score |
| 9 | [09-revenue-estimation.md](./09-revenue-estimation.md) | Revenue estimation methodology |
| 10 | [10-ai-prompt-architecture.md](./10-ai-prompt-architecture.md) | Prompt architecture, schemas, versioning |
| 11 | [11-data-providers.md](./11-data-providers.md) | External data-provider requirements |
| 12 | [12-security-ssrf.md](./12-security-ssrf.md) | Security / SSRF strategy |
| 13 | [13-testing-architecture.md](./13-testing-architecture.md) | Testing architecture |
| 14 | [14-deployment.md](./14-deployment.md) | Deployment architecture |
| 15 | [15-cost-model.md](./15-cost-model.md) | Estimated cost per audit |

Each document makes a specific, opinionated engineering decision rather than leaving options
open — but several of those decisions depend on facts or choices only you can confirm. Those
are collected below. **Nothing in this package should be treated as final until the open
questions are resolved**, because several of them change the schema, the RLS model, or the
revenue methodology itself.

---

## Top risks (read this section even if nothing else)

### 1. Revenue estimation has no grounded source for its core benchmarks — this is the single biggest threat to the product's core promise

The formula in brief §7 (`traffic × conversion × AOV × capture probability`) is sound *as a
formula*, but the brief does not identify where conservative, defensible **conversion rate**,
**AOV**, and **capture probability** ranges for cannabis retail actually come from. Without a
licensed source, these become AI-generated or engineer-guessed numbers wearing the costume of
an "ESTIMATED" category — which is exactly the hallucination brief §2 prohibits, just one layer
removed. See [09-revenue-estimation.md](./09-revenue-estimation.md) for the proposed mitigation
(a versioned, citable **Assumption Library**, sourced from licensed cannabis market-data
providers such as Headset/BDSA where budget allows, client-provided numbers where available,
and a hard "insufficient evidence" fallback everywhere else). **This needs a budget decision
and a provider decision before Week 2.**

### 2. Stripe and cannabis-adjacent businesses

Stripe's acceptable-use policy has historically restricted or prohibited accounts in the
cannabis industry, including ancillary B2B software that serves cannabis retailers, depending
on underwriting review at account creation time. Building the entire billing architecture
(brief §29, §46) around Stripe without first confirming BudSight's account will be approved is
a real risk of having to rebuild billing mid-project. **Action needed before Week 3:** submit
BudSight for Stripe underwriting (or get written confirmation of eligibility) early, and
identify a cannabis-tolerant backup processor (e.g., Aeropay, PayQwick, or a payment facilitator
that explicitly serves ancillary cannabis software) in case Stripe declines.

### 3. Weedmaps / Leafly are the most relevant competitive data sources and are also the riskiest to touch

For a cannabis retailer, Weedmaps and Leafly menu/review/ranking data is almost certainly more
decision-relevant than generic SEO signals — but neither offers a public API for third-party
competitive intelligence use, and brief §11/§35 explicitly rules out scraping that violates
terms of service. Treat these platforms as **out of scope for MVP data collection**. This
means "Promotion Opportunity" and parts of "Competitive Position" will often have to report
degraded confidence or "insufficient evidence" for dispensaries whose competitive activity
lives mainly on those platforms — which is a real product-quality gap, not just a technicality.
A future licensing conversation with Weedmaps/Leafly (or a client-permissioned
screen-share/manual-entry workaround) is the only clean path around this.

### 4. Google's own data has real coverage limits for *competitor* intelligence

Google Business Profile Performance API (impressions, clicks, calls) requires the profile
owner's OAuth consent — BudSight can get this for the client (if they connect their own GBP)
but **never** for competitors. For competitors, only the public Places API surface (rating,
review count, category, hours, address) is available. "Search visibility" and "local search
presence" findings about competitors will be inferred from SERP-position and review-signal
proxies, not real impression data. This should be stated explicitly in the report's methodology
section so it isn't mistaken for something it isn't.

### 5. Role model is inconsistent in the brief itself

§17 lists roles as `owner, admin, analyst, viewer`; §30 (Authentication) lists `owner, admin,
viewer`. I've designed around the 4-role superset (§17) since it's more specific, and treated
`analyst` as create/edit-but-not-billing. **Confirm this is right** — see
[04-rls-strategy.md](./04-rls-strategy.md).

### 6. The "free audit" flow before signup is unspecified and has real RLS/anti-abuse consequences

Brief §4 implies a user can run a free audit directly from the landing page before creating an
account, but §18/§30 build the entire authorization model around organization membership. I've
resolved this by requiring a lightweight account (email, no password needed yet — magic link)
*before* the audit pipeline starts, which keeps RLS clean and gives rate-limiting something to
key on. A fully anonymous audit is possible but would need its own unauthenticated-write policy
surface and much more aggressive bot/abuse controls (CAPTCHA, email verification gating result
delivery) — I'd avoid it for MVP. **Confirm this tradeoff is acceptable** — it does add one
small step of friction to the funnel described in §37.

### 7. No defined source for "Qualified Incremental Traffic"

The revenue formula's first term needs an actual traffic number. MVP has no first-party
analytics for the client (unless they connect GA/Search Console, which is out of scope) and
no license to scrape competitor traffic. The only defensible path without a paid traffic API
is: keyword-volume (from a SERP/keyword API) × a published, citable average CTR-by-SERP-position
curve. This is workable but must be in the architecture explicitly so it doesn't quietly become
an AI guess. See [09-revenue-estimation.md](./09-revenue-estimation.md).

---

## Full list of ambiguities / risks requiring a decision

Numbered for reference in follow-up discussion:

1. **Role model**: 4 roles (`owner/admin/analyst/viewer`) vs 3 (`owner/admin/viewer`) — brief is internally inconsistent (§17 vs §30).
2. **Anonymous audit vs required signup** before the pipeline runs — affects RLS, abuse control, and the funnel.
3. **Revenue benchmark sourcing** — no identified source for conversion/AOV/capture-probability ranges specific to cannabis retail. Needs a budget/provider decision (Headset, BDSA, or client-supplied-only with aggressive "insufficient evidence" fallback).
4. **Traffic estimation source** — no first-party or licensed traffic data identified; proposing keyword-volume × CTR-curve as the defensible proxy.
5. **Stripe eligibility for cannabis-adjacent SaaS** — needs confirmation before billing is built; identify a backup processor.
6. **Weedmaps/Leafly exclusion** — confirm MVP explicitly excludes these as data sources (ToS risk), accepting the resulting confidence/coverage gap on Promotion and Competitive Position findings.
7. **Competitor GBP/Places data limits** — competitor "search visibility" is a proxy, not real impression data; confirm the report's methodology section should say so explicitly.
8. **Category → BudSight Score weighting** — brief gives weights for Opportunity Score and Confidence Score sub-factors but not for aggregating the 6 category scores into the single "BudSight Score." Proposing equal weighting (16.67% each) pending Week 4 calibration.
9. **Cross-validation definition** for the Confidence Score's 20% "Cross-Validation" factor — proposing a concrete rule (≥2 independent sources agree = 100, single reliable source = 50, inferred only = 0); needs sign-off.
10. **Displaying competitor-scraped content verbatim** (e.g., full review text) in a client-facing report — recommend showing only normalized metrics (counts, ratings, dates) for competitors, never verbatim third-party text, to limit copyright exposure.
11. **Audit re-run semantics** — proposing every "Run Audit" click creates a new immutable `audits` row (full history, supports reproducibility goals in §23); "Retry" only applies to a stage within a `FAILED` audit. Confirm this matches the intended UX of "a second audit produces updated data" (§39).
12. **Multi-location orgs** — schema supports it, but dashboard aggregation across locations is undefined for MVP. Proposing location-scoped dashboards only for MVP; org-level rollups are a V2 item.
13. **`expiration_at` on provenance** (§10) isn't reflected in the §17 `data_snapshots`/`evidence` table definitions — added `expires_at` to both in the schema so snapshot staleness can actually be computed and feed the Confidence Score's "Data Recency" factor.
14. **PDF rendering approach** — proposing self-hosted HTML→PDF (Playwright/Chromium) rather than a third-party PDF API, to avoid sending client/competitor data to another vendor. Confirm acceptable.
15. **Job/queue engine** — brief leaves this open ("appropriate for the chosen hosting environment"). Proposing **Inngest** (step functions, per-step retry, runs inside Next.js on Vercel, no extra infra to operate) over a self-hosted queue or Supabase pg_cron/pgmq. See [02-system-architecture.md](./02-system-architecture.md) for the tradeoff.

---

## What I need from you before Week 2 starts

- A decision on risks **#1–#6** above (they change schema/RLS/legal posture, not just code).
- Sign-off (or correction) on ambiguities **#8, #9, #11, #12, #14, #15**, where I've made a
  specific default choice that's reasonable but reversible-with-cost once code exists.
- Confirmation that the repo structure, schema, API contract, state machine, domain models,
  scoring model, prompt architecture, security model, testing plan, deployment plan, and cost
  model in the linked documents are acceptable as the basis for Week 2 implementation.

I have **not** written any application code, migrations, or infrastructure yet, per your
instruction. Once you approve (or amend) this package, I'll proceed to Week 2.
