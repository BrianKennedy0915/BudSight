# 8. Scoring Algorithms

All scoring is deterministic TypeScript in `src/domain/scoring/`, operating on already-computed
evidence/metrics — never on raw AI output (brief §50 principle #2). AI may *classify* inputs
(e.g., "is this a critical-severity issue") but the weighted math itself is plain code so it's
unit-testable and reproducible per audit version.

## A. Opportunity Score (0–100)

Weights from brief §6:

| Factor | Weight | 0–100 sub-score definition |
|---|---|---|
| Business Impact | 35% | Revenue-estimate midpoint relative to a location-size-normalized benchmark (see note below); falls back to severity-based heuristic when no revenue estimate exists |
| Competitive Gap | 20% | `100 × clamp((competitorBest - clientValue) / competitorBest, 0, 1)` for the relevant metric (e.g., review count, review velocity), averaged across competitors with available data |
| Affected Demand | 20% | Estimated share of qualified traffic/customers touched by the issue (e.g., % of visits that land on a broken page) |
| Severity | 15% | Direct mapping from the AI/deterministic severity classification: critical=100, high=70, medium=40, low=15 |
| Ease of Improvement | 10% | Inverse-effort heuristic from the recommendation's `effort` field: low=100, medium=55, high=20 |

```ts
export function computeOpportunityScore(input: {
  businessImpact: number;       // 0-100
  competitiveGap: number;       // 0-100
  affectedDemand: number;       // 0-100
  severity: number;             // 0-100
  easeOfImprovement: number;    // 0-100
}): number {
  const score =
    0.35 * input.businessImpact +
    0.20 * input.competitiveGap +
    0.20 * input.affectedDemand +
    0.15 * input.severity +
    0.10 * input.easeOfImprovement;
  return Math.round(clamp(score, 0, 100));
}
```

**Business Impact sub-score, concretely:** since it can't always be a revenue number (some
findings — e.g., a broken menu link — are real and severe before a dollar estimate exists),
Business Impact is computed as:
- If a `RevenueEstimate` exists and is not `insufficientEvidence`: scale the estimate's midpoint
  against a conservative reference ceiling (e.g., 10% of a rough category revenue proxy derived
  from location size/market — itself an `Estimated` value, documented as an assumption) and clamp
  to 0–100.
- Else: fall back to the severity-derived value (same scale as the Severity factor) so a
  finding without a dollar figure yet can still be ranked sensibly.

## B. Confidence Score (0–100)

Weights from brief §6:

| Factor | Weight | 0–100 sub-score definition |
|---|---|---|
| Source Quality | 35% | Average `reliabilityScore` across the finding's evidence rows (brief §9's `reliabilityScore`, itself set per source type — see table below) |
| Data Recency | 20% | `100` if all evidence `capturedAt` within 7 days, decaying linearly to `0` at the evidence's `expiresAt` (or a default 90-day horizon if no explicit expiry); averaged across evidence |
| Calculation Quality | 25% | `100` if purely deterministic/`observed`+`calculated`; `60` if it involved one `estimated` input; `30` if multiple compounding `estimated` inputs; reflects how many assumption layers sit under the number |
| Cross-Validation | 20% | **Explicit rule (ambiguity #9 — confirm):** `100` if ≥2 independent source types agree on the underlying fact, `50` if a single reliable (`reliabilityScore ≥ 70`) source, `0` if the finding rests only on an inferred/estimated value with no corroborating observation |

```ts
export function computeConfidenceScore(input: {
  sourceQuality: number;
  dataRecency: number;
  calculationQuality: number;
  crossValidation: number;
}): number {
  const score =
    0.35 * input.sourceQuality +
    0.20 * input.dataRecency +
    0.25 * input.calculationQuality +
    0.20 * input.crossValidation;
  return Math.round(clamp(score, 0, 100));
}
```

**Default `reliabilityScore` by source type** (overridable per finding if cross-validated):

| Source type | Default reliability |
|---|---|
| `business_profile` (Google Places, first-party verified listing) | 90 |
| `website` (client's own site, direct HTML extraction) | 85 |
| `review` (aggregated platform rating/count, not sentiment) | 75 |
| `competitor` (same sources, applied to a competitor) | 70 (one notch down — less control over data freshness/accuracy for a business BudSight doesn't operate) |
| `search` (SERP position/visibility signal) | 65 |
| `user_input` (client-provided) | 95 (highest — first-party, but still logged with its own provenance, never silently treated as "observed" fact about the market) |

Scores are never confused (brief §6's hard requirement): `computeOpportunityScore` and
`computeConfidenceScore` are separate pure functions with disjoint input types — there's no
code path where one leaks into the other's weighting.

## C. BudSight Score (overall, 0–100) — **not specified by the brief; proposed default**

Brief §25 shows a single overall score plus 6 category scores but never defines how the
category scores aggregate into the headline number (ambiguity #8). Proposed default:
**equal-weighted average of the six category scores** (visibility, conversion, competitive,
promotion, reputation, customer_experience), each itself computed as:

```ts
categoryScore = 100 - weightedAverage(findingsInCategory.map(f => 
  f.opportunityScore * (f.confidenceScore / 100)   // low-confidence issues pull the score less
))
```

i.e., a category starts at 100 and is pulled down by its findings, proportional to both how
big the opportunity is and how confident BudSight is that it's real — a severe but
low-confidence finding shouldn't tank a category score the way a severe, well-evidenced one
should.

```ts
export function computeBudSightScore(categoryScores: Record<FindingCategory, number>): number {
  const values = Object.values(categoryScores);
  return Math.round(values.reduce((a, b) => a + b, 0) / values.length);
}
```

This equal-weighting is a placeholder pending the Week 4 real-world validation pass (brief
§43–45) — once 10–20 real audits are manually scored, category weights should be recalibrated
if any category turns out to be systematically over/under-influential on perceived report
credibility. The weighting constant lives in one named export
(`BUDSIGHT_SCORE_CATEGORY_WEIGHTS`) specifically so that recalibration is a one-line, versioned
change (bumps `scoring_version`).

## Testing contract (ties to brief §34)

Every function above is pure and takes plain numeric/object input — no network, no DB, no AI —
so the brief's example style of test applies directly:

```ts
test("opportunity score weights business impact at 35%", () => {
  const score = computeOpportunityScore({
    businessImpact: 80, competitiveGap: 50, affectedDemand: 50, severity: 70, easeOfImprovement: 90,
  });
  expect(score).toBe(Math.round(0.35*80 + 0.20*50 + 0.20*50 + 0.15*70 + 0.10*90)); // 67
});
```
