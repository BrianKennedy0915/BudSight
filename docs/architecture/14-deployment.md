# 14. Deployment Architecture

## Stack

| Layer | Choice | Notes |
|---|---|---|
| Frontend/backend | Next.js on **Vercel** | Per brief §46 |
| Database/Auth/Storage/Realtime | **Supabase** (Postgres) | Per brief §46; Storage holds generated PDFs, Realtime drives progress UI |
| Job engine | **Inngest**, served via `/api/inngest` on the same Vercel deployment | See `02-system-architecture.md` for rationale |
| AI | **OpenAI API**, server-side only | Per brief §46 |
| Payments | **Stripe** | Per brief §46; see risk #2 in `00-overview.md` re: eligibility |
| SERP / Places data | SerpApi-class vendor + Google Maps Platform | See `11-data-providers.md` |
| Error monitoring | Sentry | Captures both Next.js and Inngest function errors |
| Logging | Structured JSON logs to Vercel's log drain → a log store (e.g., Axiom/Logtail/Datadog, vendor TBD by budget, not architecturally significant) | |
| Rate limiting store | Upstash Redis | Low-latency, serverless-friendly, pairs naturally with Vercel |

## Environments

Three fully isolated stacks, matching brief §46's minimum requirement:

| Environment | Supabase project | Vercel target | Stripe mode | Inngest env | Purpose |
|---|---|---|---|---|---|
| `development` | local Supabase (CLI) or a dedicated dev project | local `next dev` + Vercel preview deployments | test | dev | Everyday work; migrations authored and tested here first |
| `staging` | dedicated Supabase project | Vercel "staging" branch deployment | test | staging | Pre-prod validation, Week 4's real-world audit runs against real public sites but isolated data |
| `production` | dedicated Supabase project | Vercel production deployment (`main` branch) | live | production | Real customers |

Never developed directly against production (brief §46's explicit rule) — enforced by: no
developer holding production Supabase service-role credentials locally (only CI/CD and the
production Vercel env have them), and branch protection requiring a passing CI run (migrations
+ full test suite) before merge to `main`.

## Migrations

- Source of truth: `supabase/migrations/*.sql`, applied via `supabase db push` (or the
  Supabase CLI's migration runner) as a CI step on merge to `main` (prod) and on merge to a
  `staging` branch (staging) — never applied by hand through the Supabase dashboard.
- Numbered sequentially per brief §48; each migration is committed alongside the code that
  depends on it in the same PR, so schema and application code never drift out of sync in git
  history.
- `supabase/seed.sql` provides local dev fixtures only — never run against staging/production.

## CI/CD pipeline (on every PR)

1. Lint + typecheck
2. Unit tests
3. Spin up local Supabase (via CLI in CI), apply migrations, run integration + API tests
4. AI contract tests (replayed cassettes — no live API cost on every PR; a separate scheduled
   job re-records cassettes weekly against live OpenAI to catch drift)
5. Build Next.js app (catches type errors in RSC boundaries, etc.)
6. On merge to `main`: deploy to Vercel production, apply pending migrations, run a smoke-test
   E2E pass against the live staging environment first, then promote

## Monitoring

- Sentry captures unhandled exceptions from both the Next.js app and Inngest step functions,
  tagged with `requestId`/`auditId` for correlation with structured logs.
- Inngest's own dashboard provides per-stage run history, retry counts, and latency — this is
  the primary debugging surface for pipeline failures, complementing `audit_events` (which is
  the user/support-facing debugging surface, and remains in Postgres regardless of job-engine
  choice so it survives even if Inngest is swapped later).
- A simple uptime check (e.g., a scheduled ping of `/api/health`) alerts if the deployment is
  down.
- Weekly cost report (from `ai_usage` + provider billing dashboards) surfaced internally — see
  `15-cost-model.md`.

## Secrets management

- Vercel environment variables, scoped per environment (dev/staging/prod each have their own
  values — never shared).
- `SUPABASE_SERVICE_ROLE_KEY`, `OPENAI_API_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`
  are server-only Vercel env vars, never prefixed `NEXT_PUBLIC_*`.
- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`
  are the only values intentionally exposed client-side, matching brief §47 exactly.
