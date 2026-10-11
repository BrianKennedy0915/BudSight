# 2. System Architecture

## Components

```
                              ┌─────────────────────────┐
                              │        Browser           │
                              │  Next.js client (RSC +    │
                              │  client components)       │
                              └────────────┬──────────────┘
                                           │ HTTPS
                                           ▼
                      ┌─────────────────────────────────────┐
                      │   Next.js app (Vercel)                │
                      │   - Server Components / pages         │
                      │   - /api/* route handlers              │
                      │   - Auth via Supabase SSR helpers      │
                      └───────┬───────────────────┬───────────┘
                              │                   │
              reads/writes via                enqueue job event
              RLS-scoped client                      │
                              │                   ▼
                              │        ┌───────────────────────┐
                              │        │   Inngest (job engine) │
                              │        │  /api/inngest route    │
                              │        │  step functions, one   │
                              │        │  per pipeline stage,   │
                              │        │  each independently    │
                              │        │  retryable              │
                              │        └──────────┬─────────────┘
                              │                   │ service-role client
                              ▼                   ▼
                      ┌─────────────────────────────────────┐
                      │          Supabase project              │
                      │  - Postgres (RLS enforced)             │
                      │  - Auth (JWT, magic link, email/pw)    │
                      │  - Storage (PDF reports)               │
                      │  - Realtime (audit_events → UI)        │
                      └─────────────────────────────────────┘
                              ▲                   │
                              │                   │ outbound (SSRF-guarded)
                              │                   ▼
                      ┌─────────────────────────────────────┐
                      │   External services (all optional/   │
                      │   swappable behind DataProvider)       │
                      │  - Target + competitor websites        │
                      │  - Google Places API                   │
                      │  - SERP API provider                    │
                      │  - OpenAI API                           │
                      │  - Stripe                                │
                      └─────────────────────────────────────┘
```

## Request paths

**Synchronous (HTTP request/response):**
- Auth, CRUD on organizations/locations/competitors, reading audit status/findings/evidence,
  billing checkout/portal session creation, Stripe webhook ingestion.
- All synchronous routes are expected to return in well under Vercel's function timeout; no
  pipeline work happens inline with a request.

**Asynchronous (job engine):**
- Creating an audit (`POST /api/audits`) writes an `audits` row with `status = 'queued'`,
  then sends one Inngest event (`audit/created`). Everything after that — collection through
  report generation — runs as Inngest step functions.
- The client polls `GET /api/audits/:id/status` (cheap read of the `audits` row +
  latest `audit_events`) and/or subscribes to Supabase Realtime on `audit_events` for the
  progress checklist in brief §4 Step 3. Progress text in the UI is driven directly by rows in
  `audit_events`, so it's impossible for the UI to show a step that hasn't actually run.

## Why a job engine, and why Inngest specifically

The brief deliberately leaves the queue choice open ("appropriate for the chosen hosting
environment," §21/§46) but is explicit that:
- the full audit must not run inside one HTTP request (too slow, no retry granularity),
- every stage must be independently retryable (§21, §22),
- stages need durable state and must not double-create rows on retry (§22).

Three realistic options and the tradeoff:

| Option | Pros | Cons |
|---|---|---|
| **Inngest** (chosen) | Runs as ordinary Next.js API routes on Vercel — no new infra to host or pay for separately at low volume; built-in per-step retry, backoff, concurrency limits, and a visual run log that directly satisfies the "why did this audit fail" debugging need in brief §31; step functions map 1:1 to the pipeline stages. | Third-party dependency; vendor lock-in for orchestration (mitigated — each step is a plain async function, so migrating the orchestration layer later doesn't require rewriting business logic). |
| Supabase `pg_cron` + a `pgmq` queue table | No new vendor; data stays entirely inside Supabase. | You end up hand-building retry/backoff/concurrency/observability that Inngest gives for free; worker needs to run somewhere (Supabase Edge Functions have their own execution-time limits) — more work for no clear benefit at this scale. |
| Dedicated worker service (e.g., a small Node service on Fly.io/Railway consuming a Redis/SQS queue) | Full control, no execution-time ceiling. | A second deployable to operate, monitor, and pay for from day one — premature for an MVP whose per-audit compute is a few seconds of network calls plus a handful of LLM calls. |

**Decision:** Inngest. Revisit only if/when audit volume or per-stage runtime grows enough that
its pricing or execution model becomes a constraint — nothing in the domain/jobs split above
makes that migration expensive later.

## Data flow for one audit (narrative)

1. User submits the audit-setup form → `POST /api/audits` validates input (Zod), creates
   `locations` row if new, creates `audits` row (`status='queued'`), writes `audit_events`
   row, emits `audit/created` Inngest event, returns `{ auditId }` immediately.
2. Inngest orchestrator function picks up the event, transitions status via the state machine
   (`06-audit-pipeline.md`), and runs stages as steps — each step writes its own `audit_events`
   row on start/success/failure and is retried by Inngest automatically on transient failure
   (with the retry budget defined per stage).
3. Collection steps call `DataProvider` implementations (SSRF-guarded website fetch, Places,
   SERP) and write raw results to `data_snapshots`.
4. Normalization turns raw snapshots into typed `Evidence` rows (deterministic code, no AI).
5. Competitor discovery is deterministic (Places Nearby Search + category/distance filtering)
   with an AI relevance-classification pass on top (never AI-invented competitors) — writes
   `competitors`.
6. AI analysis steps (`lib/ai/structured-call.ts`) call OpenAI with the versioned prompts,
   validate output against Zod schemas, retry-with-correction once on validation failure, and
   persist `ai_usage` rows for cost tracking regardless of success/failure.
7. Scoring and revenue estimation are deterministic domain code operating on
   evidence + AI-classified findings — never on free-form AI numbers (brief §2, §50 principle 2).
8. Report generation renders the final HTML→PDF (self-hosted Chromium via Playwright) and
   uploads to Supabase Storage; `audits.status = 'completed'`.
9. Any stage failure sets `audits.status = 'failed'` with `error_message` +
   structured `audit_events` detail, and exposes a user-facing retry action scoped to the
   failed stage (not a full re-run), per brief §20.

## Environments

Three fully separate stacks (dev/staging/prod), each with its own Supabase project, Vercel
project/env, Inngest environment, and Stripe account mode (test vs live) — see
`14-deployment.md`.
