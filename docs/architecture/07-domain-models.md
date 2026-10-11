# 7. TypeScript Domain Models

These live in `src/domain/` and `src/types/`, imported by both `jobs/` and `app/api/`. The
central addition beyond brief §8/§9 is a generic `DataPoint<T>` wrapper that encodes the
OBSERVED / CALCULATED / ESTIMATED / AI_INTERPRETATION distinction from brief §2 **in the type
system**, not just in prose — so it's structurally impossible to assign an AI-generated string
to a field typed as `DataPoint<"observed", number>`.

```ts
// src/types/data-point.ts

/** The four provenance categories from brief §2. Every quantitative or factual
 *  claim in the system must be wrapped in one of these — never a bare number/string. */
export type DataCategory = "observed" | "calculated" | "estimated" | "ai_interpretation";

export interface DataPoint<C extends DataCategory, V> {
  category: C;
  value: V;
  /** Required for observed/calculated/estimated. Optional only for ai_interpretation,
   *  which cites the evidence IDs it was derived from instead of being a source itself. */
  evidenceIds?: string[];
  /** Human-readable explanation of how `value` was produced — mandatory for
   *  calculated/estimated so a GM can see the math, per brief §7. */
  basis?: string;
  confidence: number; // 0-100
}

export type Observed<V> = DataPoint<"observed", V>;
export type Calculated<V> = DataPoint<"calculated", V>;
export type Estimated<V> = DataPoint<"estimated", V>;
export type AiInterpretation = DataPoint<"ai_interpretation", string>;
```

```ts
// src/domain/organizations/types.ts

export type Role = "owner" | "admin" | "analyst" | "viewer";

export interface Organization {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
}

export interface Profile {
  id: string;            // == auth.users.id
  organizationId: string;
  role: Role;
  name: string | null;
  createdAt: string;
}
```

```ts
// src/domain/locations/types.ts

export interface Location {
  id: string;
  organizationId: string;
  name: string;
  website: string;
  address: string | null;
  city: string | null;
  state: string | null;
  zip: string | null;
  latitude: number | null;
  longitude: number | null;
  createdAt: string;
  updatedAt: string;
}
```

```ts
// src/domain/competitors/types.ts

export interface Competitor {
  id: string;
  organizationId: string;
  locationId: string;
  name: string;
  website: string | null;
  address: string | null;
  city: string | null;
  state: string | null;
  zip: string | null;
  latitude: number | null;
  longitude: number | null;
  selectionReason: string;
  confidenceScore: number; // 0-100
  source: "google_places" | "serp" | "manual";
  placeId: string | null;
  createdAt: string;
}

export interface CompetitorRankingInput {
  distanceMeters: number;
  sameCategory: boolean;
  searchVisibilityOverlap: number;   // 0-1, keyword/SERP overlap with target
  websiteSimilarity: number;         // 0-1
  customerIntentOverlap: number;     // 0-1
}
```

```ts
// src/domain/audit/types.ts

export type AuditStatus =
  | "queued" | "collecting" | "normalizing" | "competitor_discovery"
  | "analyzing" | "scoring" | "revenue_estimation" | "generating"
  | "completed" | "failed";

export interface Audit {
  id: string;
  organizationId: string;
  locationId: string;
  status: AuditStatus;
  auditVersion: number;
  scoringVersion: string;
  promptVersion: string;
  dataProviderVersion: string;
  startedAt: string | null;
  completedAt: string | null;
  overallScore: number | null;       // "BudSight Score", 0-100
  opportunityLow: number | null;
  opportunityHigh: number | null;
  confidenceScore: number | null;
  errorCode: string | null;
  errorMessage: string | null;       // user-facing only
  retryOfAuditId: string | null;
  createdAt: string;
}

export interface AuditEvent {
  id: string;
  auditId: string;
  stage: string;
  status: "started" | "succeeded" | "failed" | "skipped";
  message: string | null;
  metadata: Record<string, unknown> | null;
  createdAt: string;
}
```

```ts
// src/domain/findings/evidence.ts

export type EvidenceSourceType =
  | "website" | "search" | "review" | "competitor" | "business_profile" | "user_input";

export interface Evidence {
  id: string;
  findingId: string;
  sourceType: EvidenceSourceType;
  sourceUrl: string | null;
  capturedAt: string;
  expiresAt: string | null;
  observation: string;
  rawValue: string | number | null;
  normalizedValue: string | number | null;
  reliabilityScore: number; // 0-100
}
```

```ts
// src/domain/findings/finding.ts

export type FindingCategory =
  | "visibility" | "conversion" | "competitive" | "promotion" | "reputation" | "customer_experience";

export type Severity = "critical" | "high" | "medium" | "low";
export type FindingStatus = "new" | "accepted" | "dismissed" | "resolved";

export interface RevenueAssumption {
  /** e.g. "estimated_conversion_rate" */
  key: string;
  /** e.g. [0.05, 0.08] */
  range: [number, number];
  /** source of the assumption — must be traceable, per brief §7 */
  source: "client_provided" | "licensed_benchmark" | "observed_derived" | "default_conservative";
  sourceDetail: string;      // e.g. "Headset AZ dispensary benchmark, Q3 2025" or "client POS export"
  assumptionLibraryVersion: string;
}

export interface RevenueEstimate {
  low: number;
  high: number;
  assumptions: RevenueAssumption[];
  /** set true when the range was too wide to be useful and the system
   *  fell back to "insufficient evidence" per brief §7 */
  insufficientEvidence: boolean;
}

export interface Recommendation {
  id: string;
  title: string;
  description: string;
  estimatedImpact: string | null;
  effort: "low" | "medium" | "high" | null;
  priority: number;
  status: "pending" | "in_progress" | "done" | "dismissed";
}

export interface Finding {
  id: string;
  auditId: string;
  category: FindingCategory;
  severity: Severity;
  title: string;
  summary: string;
  evidence: Evidence[];
  opportunityScore: number;      // 0-100, see 08-scoring-algorithms.md
  confidenceScore: number;       // 0-100
  revenueEstimate?: RevenueEstimate;
  recommendations: Recommendation[];
  status: FindingStatus;
  dedupeKey: string;
}
```

```ts
// src/domain/revenue/types.ts

/** Pure input to the revenue formula — every field is a DataPoint so its
 *  provenance travels with it through the calculation. */
export interface RevenueEstimateInputs {
  qualifiedIncrementalTraffic: Estimated<[number, number]>;
  conversionRate: Estimated<[number, number]>;
  averageOrderValue: Estimated<[number, number]> | Observed<[number, number]>; // observed if client-provided
  capturProbability: Estimated<[number, number]>;
}
```

```ts
// src/lib/providers/types.ts  (brief §11's DataProvider abstraction)

export interface CollectionInput {
  locationId: string;
  url?: string;
  placeQuery?: { name: string; lat: number; lng: number };
}

export interface CollectionResult {
  sourceType: EvidenceSourceType;
  sourceIdentifier: string;
  capturedAt: string;
  expiresAt: string | null;
  data: unknown;          // raw provider payload, persisted to data_snapshots.data
  hash: string;
}

export interface DataProvider {
  name: string;
  version: string;
  collect(input: CollectionInput): Promise<CollectionResult>;
}
```

```ts
// src/domain/billing/plans.ts

export type Plan = "free" | "monitor" | "optimize" | "intelligence";

export const PLAN_LIMITS: Record<Plan, { auditsPerMonth: number; priceCents: number }> = {
  free:         { auditsPerMonth: 1,  priceCents: 0 },
  monitor:      { auditsPerMonth: 4,  priceCents: 29900 },
  optimize:     { auditsPerMonth: 8,  priceCents: 59900 },
  intelligence: { auditsPerMonth: 20, priceCents: 99900 },
};
```

All Zod schemas used to validate AI output and API input mirror these types 1:1 and live
alongside the prompts (`src/prompts/*/schema.ts`) and API handlers
(`src/lib/validation/schemas.ts`) respectively — see `10-ai-prompt-architecture.md`.
