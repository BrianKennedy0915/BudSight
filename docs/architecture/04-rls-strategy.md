# 4. Supabase RLS Strategy

## Principles

- RLS is the enforcement layer, not the frontend (brief §18, §30). Every table holding
  tenant data has RLS **enabled and forced** (`alter table ... enable row level security;
  alter table ... force row level security;` so even the table owner role doesn't bypass it).
- The service-role key (which bypasses RLS entirely) is used **only** from Inngest job
  functions and the Stripe webhook handler — never from a user-facing route handler. Every
  service-role write must be explicitly scoped to the correct `organization_id`/`location_id`
  in application code, because the database will not stop a bug there.
- Denormalize `organization_id` onto every row below `locations` (see schema doc). This avoids
  multi-hop joins inside policies (`findings → audits → locations → organizations`), which are
  slower and, on `profiles`-adjacent tables specifically, can create recursive-policy
  evaluation problems in Postgres RLS.

## Resolving "who is the current user's org" without recursion or per-row subqueries

Two viable approaches; recommending the first:

**A. JWT custom claims (recommended).** Use Supabase's Auth Hook
(`custom_access_token_hook`) to inject `organization_id` and `role` into the JWT at
sign-in/token-refresh time, read from `profiles`. Policies then read
`(auth.jwt() -> 'organization_id')` directly — no subquery per row, no risk of a policy on
`profiles` recursively querying `profiles`. Tradeoff: a role/org change (e.g., admin changes a
user's role) doesn't take effect until the user's token refreshes; mitigate by forcing a token
refresh after role-changing admin actions.

**B. `security definer` helper function**, e.g. `auth_org_id()` that does
`select organization_id from profiles where id = auth.uid()`, marked `stable` and
`security definer` so it bypasses RLS on `profiles` internally and avoids recursion. Simpler to
keep in sync (always current) but costs a function call per row evaluated.

**Decision:** A for all tables except `profiles` itself (which has no choice but to use B,
since the JWT can't be used to read the table that produces the JWT's own claims without a
bootstrapping function). Both are defined in `002_rls_policies.sql`; the approach is
switchable without changing table structure.

## Policies by table

### `organizations`
- `select`: `id = (auth.jwt() ->> 'organization_id')::uuid`
- `update`: same, **and** role in `('owner','admin')`
- `insert`/`delete`: service role only (org creation happens at signup via a server action using
  the service-role client, not a direct client insert)

### `profiles`
- `select`: `organization_id = auth_org_id()` (own org, so teammates are visible)
- `update`: `id = auth.uid()` (self) OR (`auth_org_id() = organization_id` AND requester role
  in `('owner','admin')`) — lets owners/admins change teammate roles
- `insert`/`delete`: service role only (invite flow)

### `locations`
- `select`/`insert`/`update`: `organization_id = (auth.jwt() ->> 'organization_id')::uuid`
- `insert`/`update`/`delete` additionally require role in `('owner','admin','analyst')` —
  `viewer` is read-only everywhere
- `delete`: role in `('owner','admin')`

### `competitors`, `audits`, `findings`, `evidence`, `data_snapshots`, `recommendations`, `reports`, `ai_usage`, `audit_events`
- `select`: `organization_id = (auth.jwt() ->> 'organization_id')::uuid` — uniform read policy
- `insert`/`update`/`delete`: **service role only**. These tables are written exclusively by
  the job pipeline and never directly by a user's browser session — a user "accepting" or
  "dismissing" a finding goes through an API route that uses the service-role client after
  checking the user's role and org in application code, not a direct client-side write. This
  keeps the write path auditable in one place and avoids needing to express pipeline-stage
  write permissions as RLS policies.
- `viewer` role: read-only, enforced the same way as `locations` (role check lives in the API
  route for any mutating endpoint, e.g. finding status changes), since RLS alone can't see
  "this request is a PATCH to findings.status" vs. "this is the pipeline writing new evidence."

### `subscriptions`
- `select`: `organization_id = (auth.jwt() ->> 'organization_id')::uuid`
- `insert`/`update`/`delete`: service role only (Stripe webhook handler) — brief §29 "never
  trust frontend payment state" is enforced structurally: there is no client-writable path to
  this table at all.

### `stripe_webhook_events`
- No RLS-relevant access from clients; service role only, used purely for idempotency checks
  inside the webhook handler.

## Role matrix (confirm — see ambiguity #1 in 00-overview.md)

| Action | owner | admin | analyst | viewer |
|---|---|---|---|---|
| View dashboard/reports | ✅ | ✅ | ✅ | ✅ |
| Create/retry audits | ✅ | ✅ | ✅ | ❌ |
| Manage locations/competitors | ✅ | ✅ | ✅ | ❌ |
| Change finding status (accept/dismiss) | ✅ | ✅ | ✅ | ❌ |
| Manage billing | ✅ | ✅ | ❌ | ❌ |
| Invite/remove users, change roles | ✅ | ✅ | ❌ | ❌ |
| Delete organization | ✅ | ❌ | ❌ | ❌ |

## Anonymous / pre-signup audit flow

Per ambiguity #2 in `00-overview.md`: MVP requires a lightweight account (email, magic link —
no password needed to start) before `POST /api/audits` succeeds. This means:
- There is no "anonymous write" policy surface to design or secure.
- Every `organizations`/`locations`/`audits` row always has a real owner from creation, so the
  RLS model above needs no special-casing for anonymous actors.
- The signup step doubles as the anti-abuse gate for audit-creation rate limiting (keyed on
  `organization_id`, see `12-security-ssrf.md`), rather than needing CAPTCHA/IP heuristics for
  a fully anonymous flow.

If product decides a truly anonymous "no signup" free audit is required, this needs its own
design pass (temporary unauthenticated session token, email-gated result delivery, much
stricter per-IP rate limiting) — flagged, not designed here, since it's a materially different
authorization model.
