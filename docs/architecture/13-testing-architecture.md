# 13. Testing Architecture

Pyramid shape, weighted toward the layers that are cheap to run and protect the highest-risk
logic (scoring, revenue estimation, SSRF guard, AI contract adherence).

## Unit tests (`src/tests/unit/`, Vitest)

Pure functions in `src/domain/`, no I/O, no mocks needed since there's nothing external to mock:

- `computeOpportunityScore`, `computeConfidenceScore`, `computeBudSightScore` — weight
  correctness, boundary clamping (0/100), and that the two scores can never be computed from
  each other's inputs (brief §6's "never confuse these scores" — enforced by disjoint input
  types, verified by a test that asserts the function signatures don't overlap in practice via
  a representative fixture).
- `computeRevenueEstimate` — interval arithmetic correctness (brief §34's literal example:
  1000 visits × 5% × $80 → $4,000, generalized to range math), and the
  `insufficientEvidence` fallback triggering correctly on a too-wide range.
- Competitor ranking (`domain/competitors/ranking.ts`) — deterministic ordering given fixed
  inputs; same input twice produces the same ranked order (determinism requirement, brief §12).
- Normalization functions (raw provider payload → `Evidence`) — golden-file tests per provider
  payload shape.
- Assumption Library — a test that **fails the build** if any constant lacks a non-empty
  `sourceDetail`/citation string, enforcing brief §7's "every variable must have a source or
  documented assumption" at the type/lint level, not just by convention.

## Integration tests (`src/tests/integration/`)

Run against a local Supabase instance (`supabase start`, via the Supabase CLI, same
migrations as production) and mocked `DataProvider`/`LlmProvider` implementations (fixed
canned responses, no real network calls):

- Full happy-path pipeline: create organization → create location → create audit → run every
  Inngest step in sequence (Inngest's local dev server supports this) → assert findings,
  evidence, scores, revenue estimates, and a generated report row all exist and are internally
  consistent (e.g., every `finding.evidence[].id` actually resolves, every `revenue_low ≤
  revenue_high`).
- Idempotency: run the same stage twice against the same audit, assert row counts don't
  double (`data_snapshots`, `competitors`, `findings` unique constraints hold).
- Retry: force a stage to fail once (mock throws on first call, succeeds on second), assert
  Inngest's retry recovers without manual intervention and `audit_events` shows the
  failed→succeeded sequence.
- Degraded-but-complete path: force competitor discovery to return zero results, assert the
  audit still reaches `COMPLETED` with an explicit "insufficient evidence" competitive finding
  rather than failing outright (per `06-audit-pipeline.md`'s required/degradable table).

## API tests (`src/tests/api/`)

Against a running Next.js instance + local Supabase, using real (test) auth sessions for
different roles:

- Auth required/rejected appropriately per route.
- Authorization: a `viewer` role gets 403 on mutating endpoints; a user from org A gets 404 (not
  403 — avoid leaking existence) when requesting org B's resources, verifying RLS actually
  blocks it even if an application-level check were accidentally missing (defense in depth —
  this test is specifically designed to catch "frontend-only" authorization regressions, brief
  §18).
- Invalid input → 400 with the expected error code.
- Rate limits → 429 after the configured threshold, using a fast-forwarded/test rate-limit
  window.
- Duplicate requests (e.g., double-clicking "Run Audit") → either idempotent no-op or a clear
  409, never two audits silently created from one click (debounced client-side too, but the
  server test doesn't rely on that).

## AI contract tests (`src/tests/ai-contract/`)

Fixed evidence fixtures per prompt (`fixtures/website-analyst/*.json`, etc.), run against the
real `OpenAiProvider` in CI but with responses **recorded and replayed** (VCR-style cassette)
after an initial real run, so CI doesn't burn API budget or flake on model nondeterminism on
every commit — a real (uncached) run is triggered manually or on a schedule to detect drift
when OpenAI updates a model version:

- Valid JSON per the Zod schema.
- No unsupported claims — specifically, every `evidenceRefs` entry resolves to an id actually
  present in the fixture's supplied evidence (a fixture is constructed with a few evidence ids
  that *shouldn't* be cited, as a negative check).
- Required fields present, score fields within `[0,100]`.
- A fixture deliberately constructed with thin/contradictory evidence produces the explicit
  "insufficient evidence" / low-confidence output rather than a confident answer — this is the
  single most important test in the whole suite relative to brief §2's core principle, and
  should be run with multiple fixture variants (zero evidence, conflicting evidence, stale
  evidence) to pressure-test the guardrail from different angles.

## End-to-end tests (`src/tests/e2e/`, Playwright)

Using the pre-installed Chromium:

- Landing page → sign up (magic link captured via a test-mode email intercept) → create
  location → create audit → poll/observe progress checklist updating in real time → view
  completed report → open a finding's evidence drawer → change a finding's status → view
  billing page → Stripe test-mode checkout → webhook (simulated via Stripe CLI test event) →
  subscription status reflected in UI.
- Negative path: submit a private/internal URL at the audit-setup step, assert the
  user-facing SSRF-blocked message appears (brief §31's "good" error example) rather than a
  generic failure.

## Golden-audit regression suite

A small, fixed set of real (consented, or synthetic-but-realistic) location fixtures with
pinned provider responses, run whenever `scoring_version`, `prompt_version`, or the Assumption
Library version bumps. Asserts category scores, BudSight Score, and revenue ranges stay within
an expected tolerance band — this is the mechanism that catches an unintended scoring-logic
regression ("why did this location's score jump 20 points after an unrelated code change")
before it reaches a real customer, and is the practical answer to brief §23's reproducibility
goal at the test-suite level, not just the schema level.
