# 5. API Contract

All routes are Next.js App Router route handlers under `src/app/api/`. All responses use a
consistent envelope:

```ts
// success
{ data: T }

// error
{
  error: {
    code: string;            // machine-readable, e.g. "SSRF_BLOCKED_URL"
    message: string;         // user-facing, per brief §31 — never a stack trace
    requestId: string;       // correlates to structured logs
  }
}
```

Auth: every route except `(marketing)` pages, `POST /api/auth/*`, and
`POST /api/webhooks/stripe` requires a valid Supabase session; the route handler reads the
user's JWT via the SSR helper and relies on RLS for row-level enforcement, with an explicit
role check added in the handler wherever the action is more than a plain read (per the role
matrix in `04-rls-strategy.md`).

Rate limits noted per-route; implementation in `12-security-ssrf.md`.

---

## Auth

Handled by Supabase Auth directly from the client SDK for sign-up/sign-in/magic-link in most
cases; these two server routes exist for flows that need server-side side effects:

### `POST /api/auth/complete-signup`
Creates the `organizations` + `profiles` row for a brand-new user (first-time org bootstrap).
Idempotent: calling twice for the same `auth.users.id` is a no-op if `profiles` already exists.

- **Body:** `{ organizationName: string }`
- **200:** `{ data: { organizationId: string } }`
- **Rate limit:** 5/hour per IP

---

## Organizations

### `GET /api/organizations/me`
Returns the caller's organization + subscription summary.

### `PATCH /api/organizations/me`
- **Body:** `{ name?: string }` — role: owner/admin

---

## Locations

### `GET /api/locations`
List locations in the caller's org.

### `POST /api/locations`
- **Body:** `{ name, website, city, state, zip, address? }` — Zod-validated; `website` run
  through SSRF/URL validation (`12-security-ssrf.md`) at submit time, not just at collection
  time, so bad input fails fast with a clear message.
- Role: owner/admin/analyst

### `GET /api/locations/:id` / `PATCH /api/locations/:id` / `DELETE /api/locations/:id`
Standard CRUD, scoped by RLS; delete cascades audits/competitors (confirm retention policy —
recommend soft-delete via `deleted_at` if audit history must survive location edits; omitted
from MVP schema pending that decision).

---

## Audits

### `POST /api/audits`
Creates and enqueues a new audit.

- **Body:**
  ```ts
  { locationId: string } // location must already exist and belong to caller's org
  ```
- **Behavior:** inserts `audits` row (`status='queued'`, versions stamped from current
  `scoring_version`/`prompt_version`/`data_provider_version` constants), writes initial
  `audit_events` row, emits `audit/created` Inngest event.
- **201:** `{ data: { auditId: string, status: "queued" } }`
- **429:** if org has exceeded plan's audit-creation rate limit (brief §32 — free plan capped
  hard, e.g. 1 audit; paid plans capped per billing period)
- **Rate limit:** enforced primarily via plan entitlement, backstopped by 10/hour/org absolute
  ceiling regardless of plan, to contain runaway client bugs

### `GET /api/audits`
List audits for a location (or org-wide), paginated, newest first.

### `GET /api/audits/:id`
Full audit summary: status, scores, opportunity range, confidence, timestamps, version stamps.

### `GET /api/audits/:id/status`
Lightweight polling endpoint — status + latest `audit_events` rows since an optional
`?since=<event_id>` cursor. Also exposed via Supabase Realtime subscription on `audit_events`
as the primary mechanism; this REST endpoint is the fallback for clients not using Realtime.

### `GET /api/audits/:id/findings`
All findings for the audit, with nested `recommendations`, sorted by `opportunity_score` desc.
Supports `?category=` and `?status=` filters.

### `PATCH /api/audits/:id/findings/:findingId`
- **Body:** `{ status: "accepted" | "dismissed" | "resolved" }`
- Role: owner/admin/analyst. Writes go through the service-role client inside the handler
  after an explicit role check (see `04-rls-strategy.md` — findings have no client-writable
  RLS policy by design).

### `GET /api/audits/:id/evidence`
All evidence for the audit (joined across its findings), supports `?findingId=` filter — this
backs the "View Evidence" drawer (brief §9, §26).

### `POST /api/audits/:id/retry`
- **Body:** `{ fromStage?: string }` — defaults to retrying from the failed stage recorded on
  the audit; only valid when `audits.status = 'failed'`.
- **Behavior:** does **not** create a new `audits` row; re-emits an Inngest event scoped to
  resume the same audit from its failed stage, relying on each stage's idempotency keys to
  avoid duplicate side effects (brief §22).
- **409:** if audit is not in `failed` status

---

## Competitors

### `GET /api/competitors?locationId=`
List competitors for a location, with `selection_reason`, `confidence_score`, `source`.

### `DELETE /api/competitors/:id`
Lets a user remove a competitor they judge irrelevant (feedback signal for future ranking
tuning — captured but not yet used to auto-tune in MVP). Role: owner/admin/analyst.

---

## Reports

### `GET /api/reports/:id`
Report metadata (status, generated_at) + structured data for the in-app report view (reuses
`findings`/`evidence`/`audits` data, assembled into report sections per brief §28).

### `GET /api/reports/:id/pdf`
Streams the generated PDF from Supabase Storage (signed URL redirect) or `202` with
`{ data: { status: "generating" } }` if not yet ready. Rate limit: 10/hour/org (PDF rendering
is the most compute-expensive synchronous-feeling action).

### `POST /api/reports/:id/regenerate`
Re-renders the PDF from current `findings`/`audits` data without re-running the pipeline
(e.g., after a finding status change). Role: owner/admin/analyst.

---

## Billing

### `POST /api/billing/checkout`
- **Body:** `{ plan: "monitor" | "optimize" | "intelligence" }`
- Creates a Stripe Checkout Session for the org, returns `{ data: { url } }` to redirect to.
  Role: owner/admin.

### `POST /api/billing/portal`
Creates a Stripe Billing Portal session for the org's existing customer. Role: owner/admin.

### `POST /api/webhooks/stripe`
- No user auth — verified via Stripe signature header (`STRIPE_WEBHOOK_SECRET`) instead.
- Deduplicates via `stripe_webhook_events` (insert-or-ignore on Stripe event id) before
  processing, so Stripe's at-least-once delivery can't double-apply a state change.
- Handles: `checkout.session.completed`, `customer.subscription.created/updated/deleted`,
  `invoice.payment_failed` — updates `subscriptions` using the service-role client exclusively.
- Always returns `200` quickly once the event is durably recorded (even if async
  post-processing is still finishing), per Stripe's retry/timeout expectations.

---

## Error codes (non-exhaustive, extend as needed)

| Code | Meaning | User-facing message example |
|---|---|---|
| `INVALID_URL` | Submitted website fails basic URL validation | "That doesn't look like a valid website address." |
| `SSRF_BLOCKED_URL` | URL resolves to a private/internal/metadata address | "We can't analyze that address. Please provide your dispensary's public website." |
| `WEBSITE_UNREACHABLE` | Fetch failed after retries (timeout, blocked, DNS) | "We couldn't reach your website. You can retry or check that the URL is correct." |
| `AUDIT_RATE_LIMIT` | Plan's audit quota exceeded | "You've reached your plan's audit limit for this period." |
| `INSUFFICIENT_EVIDENCE` | Not an error, but surfaced in findings/report where applicable | "There wasn't enough reliable data to produce this estimate." |
| `AI_VALIDATION_FAILED` | Model output failed schema validation after retry | "We hit a snag analyzing this section. It's been flagged for review and your audit will continue." |
