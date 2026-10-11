# 9. Revenue Estimation Methodology

This is the highest-risk piece of the entire system (see `00-overview.md` risk #1). The
formula in brief §7 is correct as math; the problem is that two of its four inputs
(**Qualified Incremental Traffic**, **Conversion Rate**) and part of a third
(**Capture Probability**) have no grounded, licensed, or first-party source identified
anywhere in the brief. Shipping the formula without solving that is just hallucination with
better formatting — a model (or an engineer) still has to invent the numbers that go into it.

## The formula

```
Estimated Revenue Opportunity =
  Qualified Incremental Traffic (range)
  × Estimated Conversion Rate (range)
  × Estimated Average Order Value (range)
  × Capture Probability (range)
```

Implemented as **interval arithmetic** over `[low, high]` ranges, not point estimates
multiplied together then "widened":

```ts
export function computeRevenueEstimate(inputs: RevenueEstimateInputs): RevenueEstimate {
  const [tLow, tHigh] = inputs.qualifiedIncrementalTraffic.value;
  const [cLow, cHigh] = inputs.conversionRate.value;
  const [aLow, aHigh] = inputs.averageOrderValue.value;
  const [pLow, pHigh] = inputs.capturProbability.value;

  const low  = tLow  * cLow  * aLow  * pLow;
  const high = tHigh * cHigh * aHigh * pHigh;

  const assumptions = [inputs.qualifiedIncrementalTraffic, inputs.conversionRate,
                        inputs.averageOrderValue, inputs.capturProbability]
    .map(toRevenueAssumption);

  if (rangeTooWideToBeUseful(low, high)) {
    return { low: 0, high: 0, assumptions, insufficientEvidence: true };
  }
  return { low: Math.round(low), high: Math.round(high), assumptions, insufficientEvidence: false };
}
```

`rangeTooWideToBeUseful` (brief §7's explicit escape hatch): proposed rule — if
`(high - low) / high > 0.85` **and** `high` exceeds a sanity ceiling tied to location size
(e.g., estimated opportunity implausibly exceeds a reasonable fraction of a plausible total
revenue proxy for a single dispensary location), return `insufficientEvidence: true` instead
of a number. This directly operationalizes brief §7's "insufficient evidence" fallback and
§44's rule ("if evidence only supports $8–15k, lower the number — don't round up for impact").

## Where each input actually comes from

### Average Order Value — the one input with a real first-party path
- **Best case:** client provides it (onboarding question or connected POS export, even just a
  typed-in number) → `Observed`, `source: "client_provided"`, high confidence.
- **Fallback:** licensed cannabis retail benchmark (see Assumption Library below) →
  `Estimated`, `source: "licensed_benchmark"`.
- **No fallback to "AI's best guess."** If neither is available, AOV defaults to the Assumption
  Library's conservative published range, never an unsourced number.

### Conversion Rate — no first-party source in MVP (client has no analytics integration)
- MVP has no GA/Shopify/POS access (out of scope per brief §51 V2). This is **always**
  `Estimated` from the Assumption Library, sourced from published e-commerce/cannabis-retail
  conversion benchmarks, not computed per-client.
- This is a real limitation: BudSight cannot currently know *this specific dispensary's*
  conversion rate — only a defensible industry range. The report's methodology section must
  say this plainly (brief §28 §10, "methodology increases trust" — it only does if it's honest
  about what's inferred vs. known).

### Qualified Incremental Traffic — the hardest input, no licensed traffic API in MVP budget assumed
Proposed defensible proxy, used **only** when a paid traffic-estimation API (SEMrush/Ahrefs/
Similarweb) is out of budget:

```
qualifiedIncrementalTraffic ≈ Σ over targeted keywords/categories of
    monthlySearchVolume(keyword)        -- from the SERP/keyword provider (licensed)
  × ctrByPosition(currentRank vs achievableRank)   -- published, citable CTR-by-position curve
  × localIntentFilter                    -- fraction of volume judged local/commercial intent
```

- `monthlySearchVolume` comes from the SERP provider's keyword-volume endpoint (an `Observed`
  value, with timestamp/source).
- `ctrByPosition` is a **documented, versioned, citable constant table** (industry-standard
  organic CTR curves, e.g., from published SEO industry studies) — stored in the Assumption
  Library with its own citation and version, never invented per-audit.
- The delta between the client's *current* SERP position and a plausible *achievable* position
  (used to compute the incremental, not absolute, traffic) is itself an `Estimated` judgment
  call and must be stated as an assumption in the finding, not asserted as fact.
- If a licensed traffic API is later budgeted, this entire sub-calculation becomes a single
  `Observed` provider call behind the same `DataProvider` interface — the formula and
  everything downstream doesn't change, only the source of one input does. This is exactly why
  the system is architected around the `DataProvider` abstraction (brief §11).

### Capture Probability — judgment-heavy by nature, kept conservative and bounded
- Derived from competitive-gap evidence (e.g., how much of the visibility/review gap is
  realistically closable given the recommended action's effort level) via a small, explicit,
  versioned lookup (not free-form AI judgment): e.g., low-effort fixes get a narrower/higher
  floor (10–25%), high-effort/competitive-displacement opportunities get a wider, lower range
  (5–15%). This table lives in the Assumption Library too.

## The Assumption Library

A single versioned module, `src/domain/revenue/assumption-library.ts`, exporting named,
citable constants:

```ts
export const ASSUMPTION_LIBRARY_VERSION = "2026.01";

export const DEFAULT_AOV_RANGE_USD: [number, number] = [70, 90]; // SOURCE: <citation>, confirm before use
export const DEFAULT_CONVERSION_RATE_RANGE: [number, number] = [0.05, 0.08]; // SOURCE: <citation>
export const CTR_BY_POSITION: Record<number, number> = { 1: 0.28, 2: 0.15, 3: 0.11, /* ... */ };
export const CAPTURE_PROBABILITY_BY_EFFORT: Record<"low"|"medium"|"high", [number, number]> = {
  low: [0.10, 0.25], medium: [0.07, 0.15], high: [0.05, 0.10],
};
```

Every value here **must** carry a real citation before Week 2 ships anything using it — the
placeholders above are structurally where the numbers go, not an endorsement of those specific
numbers. This is the concrete decision needed for risk #1 in `00-overview.md`: either budget
for a licensed cannabis market-data source (Headset, BDSA/Jane) to populate this table with
industry-specific numbers, or explicitly accept that MVP estimates are grounded in
general-retail/e-commerce benchmarks (weaker, but still sourced and better than invented) until
better data is budgeted.

`assumptionLibraryVersion` is stamped onto every `RevenueAssumption` so a later audit of "why
did this say $18,000" can see exactly which benchmark values were in effect.

## Revenue Analyst prompt's actual role

Per brief §15's Prompt 3, the AI's job here is **narrow**: given the deterministic findings and
calculated metrics, identify *which* revenue-estimation inputs apply to *this* finding and
flag when a required input is missing — it does not compute the dollar range itself. The Zod
schema for this prompt's output has no numeric revenue field at all; it outputs
`{ applicableAssumptionKeys: string[], additional_data_needed: string[], confidence: number }`,
and `computeRevenueEstimate` (plain TypeScript, shown above) does the actual arithmetic. This
is the literal enforcement of brief §50 principle #2 ("deterministic calculations before
language generation") for the single most risk-sensitive number in the product.
