# 1. Repository Structure

Single Next.js (App Router) repo. Domain logic is isolated from framework code so scoring,
revenue estimation, and pipeline logic can be unit-tested without Next.js, Supabase, or network
access, and so the job engine (Inngest) and the HTTP layer both call into the same domain code.

```
budsight/
├── docs/
│   └── architecture/              # this package
├── supabase/
│   ├── migrations/                # numbered SQL migrations (source of truth for schema)
│   │   ├── 001_initial_schema.sql
│   │   ├── 002_rls_policies.sql
│   │   ├── 003_audit_pipeline.sql
│   │   ├── 004_evidence.sql
│   │   ├── 005_competitors.sql
│   │   ├── 006_ai_usage.sql
│   │   ├── 007_billing.sql
│   │   └── 008_audit_versioning.sql
│   ├── seed.sql                   # local dev seed data (fake org/location/audit)
│   └── config.toml
├── src/
│   ├── app/
│   │   ├── (marketing)/            # landing page, example report — public, no auth
│   │   │   └── page.tsx
│   │   ├── (auth)/
│   │   │   ├── sign-in/page.tsx
│   │   │   └── sign-up/page.tsx
│   │   ├── (app)/                  # authenticated shell
│   │   │   ├── layout.tsx
│   │   │   ├── dashboard/page.tsx
│   │   │   ├── audit/
│   │   │   │   ├── new/page.tsx
│   │   │   │   └── [auditId]/
│   │   │   │       ├── page.tsx           # progress/report view
│   │   │   │       └── findings/[findingId]/page.tsx
│   │   │   ├── competitors/page.tsx
│   │   │   ├── reports/page.tsx
│   │   │   ├── settings/page.tsx
│   │   │   └── billing/page.tsx
│   │   └── api/
│   │       ├── organizations/
│   │       ├── locations/
│   │       ├── audits/
│   │       │   ├── route.ts
│   │       │   └── [id]/
│   │       │       ├── route.ts
│   │       │       ├── status/route.ts
│   │       │       ├── findings/route.ts
│   │       │       ├── evidence/route.ts
│   │       │       └── retry/route.ts
│   │       ├── competitors/
│   │       ├── reports/
│   │       │   └── [id]/
│   │       │       ├── route.ts
│   │       │       └── pdf/route.ts
│   │       ├── billing/
│   │       │   ├── checkout/route.ts
│   │       │   └── portal/route.ts
│   │       ├── webhooks/
│   │       │   └── stripe/route.ts
│   │       └── inngest/route.ts    # Inngest serve handler (job engine entrypoint)
│   │
│   ├── components/
│   │   ├── dashboard/
│   │   ├── audit/
│   │   ├── report/
│   │   ├── competitors/
│   │   ├── billing/
│   │   └── ui/                     # design-system primitives
│   │
│   ├── domain/                     # pure business logic — no Next.js / Supabase imports
│   │   ├── audit/
│   │   │   ├── state-machine.ts
│   │   │   └── idempotency.ts
│   │   ├── findings/
│   │   │   ├── finding.ts
│   │   │   └── evidence.ts
│   │   ├── scoring/
│   │   │   ├── opportunity-score.ts
│   │   │   ├── confidence-score.ts
│   │   │   └── budsight-score.ts
│   │   ├── revenue/
│   │   │   ├── estimate.ts
│   │   │   └── assumption-library.ts   # versioned, sourced benchmark ranges
│   │   ├── competitors/
│   │   │   └── ranking.ts
│   │   ├── organizations/
│   │   └── billing/
│   │       └── plans.ts
│   │
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── server-client.ts        # service-role client (jobs/webhooks only)
│   │   │   ├── route-client.ts         # per-request client bound to user's JWT
│   │   │   └── browser-client.ts
│   │   ├── stripe/
│   │   │   └── client.ts
│   │   ├── ai/
│   │   │   ├── provider.ts             # LLM provider abstraction (OpenAI first impl)
│   │   │   ├── structured-call.ts      # Zod-validated call + retry-with-correction
│   │   │   └── usage-tracking.ts
│   │   ├── providers/                  # DataProvider implementations
│   │   │   ├── types.ts                # DataProvider interface
│   │   │   ├── website-fetcher.ts
│   │   │   ├── places-provider.ts
│   │   │   ├── serp-provider.ts
│   │   │   └── review-provider.ts
│   │   ├── validation/
│   │   │   └── schemas.ts              # shared Zod schemas
│   │   ├── security/
│   │   │   ├── ssrf-guard.ts
│   │   │   ├── html-sanitize.ts
│   │   │   └── rate-limit.ts
│   │   └── logging/
│   │       └── logger.ts
│   │
│   ├── jobs/                           # Inngest functions, one per pipeline stage
│   │   ├── client.ts                   # Inngest client + event schema registration
│   │   ├── audit-orchestrator.ts       # top-level function, calls steps below
│   │   ├── collection/
│   │   │   ├── collect-website.ts
│   │   │   └── collect-business-profile.ts
│   │   ├── normalize.ts
│   │   ├── discover-competitors.ts
│   │   ├── collect-competitors.ts
│   │   ├── analyze.ts                  # AI analysis step(s)
│   │   ├── score.ts
│   │   ├── estimate-revenue.ts
│   │   └── generate-report.ts
│   │
│   ├── prompts/
│   │   ├── website-analyst/
│   │   │   ├── v1.ts
│   │   │   └── schema.ts
│   │   ├── competitive-analyst/
│   │   ├── revenue-analyst/
│   │   └── executive-analyst/
│   │
│   ├── types/
│   │   └── database.ts                 # generated from Supabase schema
│   │
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── api/
│       ├── ai-contract/
│       │   └── fixtures/               # recorded evidence fixtures per prompt
│       └── e2e/
│
├── .env.example
├── package.json
└── next.config.ts
```

**Rules enforced by this layout:**

- `domain/` has zero imports from `app/`, `lib/supabase`, `lib/ai`, or `jobs/` — it receives
  plain data in and returns plain data out. This is what makes brief §34's unit-test examples
  (`1000 visits, 5% conversion, $80 AOV → $4,000`) possible without spinning up Supabase or
  OpenAI.
- `jobs/` is the only place that orchestrates multiple domain modules + providers + AI calls
  together. Each file under `jobs/` corresponds 1:1 to a stage in the state machine
  (`06-audit-pipeline.md`).
- `lib/providers/` implements the `DataProvider` interface from brief §11 — swapping a SERP
  vendor means adding a new file here, not touching `jobs/` or `domain/`.
- React components under `components/` render props/state only; no Supabase calls or scoring
  logic inside a component (brief §49/§50 principle #3).
