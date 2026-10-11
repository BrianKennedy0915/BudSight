# 12. Security / SSRF Strategy

## SSRF-hardened fetcher (`src/lib/security/ssrf-guard.ts`)

The core risk: BudSight accepts a user-supplied URL (the dispensary's website) and the server
fetches it. Without hardening, this is a classic SSRF vector — a malicious submitter could
point "their website" at an internal service or cloud metadata endpoint.

**Validation pipeline, applied both at submit time (`POST /api/locations`) and again
immediately before every actual fetch (collection stage), since DNS can change between the two
— this double-check is what closes the DNS-rebinding gap:**

1. **Scheme check** — only `http://`/`https://` allowed; reject everything else
   (`file://`, `ftp://`, `gopher://`, etc.).
2. **Hostname resolution** — resolve DNS explicitly (don't let the HTTP client do it
   implicitly), and validate **the resolved IP**, not just the hostname string:
   - Reject loopback (`127.0.0.0/8`, `::1`)
   - Reject private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `fc00::/7`)
   - Reject link-local (`169.254.0.0/16`, `fe80::/10`) — this specifically blocks the AWS/GCP/
     Azure metadata endpoint `169.254.169.254`
   - Reject other reserved/special ranges (`0.0.0.0/8`, multicast, etc.)
   - Reject known internal hostnames outright as a defense-in-depth belt-and-suspenders check
     (`localhost`, `*.internal`, `*.local`) even before DNS resolution
3. **Pin the validated IP for the actual connection** — don't re-resolve the hostname when
   making the real request (which would reopen the DNS-rebinding window between check and
   use). Use an HTTP client option to connect directly to the validated IP while still sending
   the original `Host` header/SNI, so TLS and virtual-hosting still work correctly.
4. **Redirect handling** — disable automatic redirect-following at the HTTP-client level;
   handle redirects manually, re-running the full validation pipeline (steps 1–3) on every
   redirect target, capped at a small max-hop count (e.g., 5). A redirect to a blocked target
   fails the fetch with `SSRF_BLOCKED_URL`, it does not silently follow then fail later.
5. **Response limits** — hard timeout (e.g., 10s connect, 20s total), hard max response size
   (e.g., 5MB) read via a streaming cap, not after full buffering — protects against
   slow-loris-style resource exhaustion from a malicious or broken target.
6. **Port restrictions** — only default ports (80/443) unless explicitly allowed; blocks
   probing of internal services running on nonstandard ports.

```ts
export async function safeFetch(url: string): Promise<Response> {
  const parsed = new URL(url); // throws on malformed input
  assertAllowedScheme(parsed);
  let currentUrl = parsed;
  for (let hop = 0; hop <= MAX_REDIRECTS; hop++) {
    const ip = await resolveAndValidate(currentUrl.hostname); // throws SsrfBlockedError
    const res = await fetchPinnedIp(currentUrl, ip, { redirect: "manual", timeoutMs: 20_000, maxBytes: 5_000_000 });
    if (isRedirect(res.status)) {
      currentUrl = resolveRedirectTarget(currentUrl, res.headers.get("location"));
      continue;
    }
    return res;
  }
  throw new SsrfBlockedError("Too many redirects");
}
```

## HTML sanitization

All extracted HTML/text from target and competitor sites is passed through a sanitizer
(DOMPurify-equivalent server-side, or a strict allowlist text extractor) before being stored in
`data_snapshots.data` or shown anywhere in the UI — strips `<script>`, event handlers, and any
executable content, since this is third-party content being displayed back inside BudSight's
own app (report view, evidence drawer).

## Secrets

- No API key, service-role key, or Stripe secret ever ships in client bundles — enforced by
  convention (`NEXT_PUBLIC_*` prefix reserved exclusively for truly public values: Supabase URL,
  Supabase anon key, Stripe publishable key) and checked in CI via a simple grep/lint rule
  scanning the client bundle output for known secret-key prefixes (`sk_`, service-role JWT
  pattern) as a backstop.
- `SUPABASE_SERVICE_ROLE_KEY` is only imported in `lib/supabase/server-client.ts`, which itself
  is only imported from `jobs/`, webhook handlers, and a small number of explicitly-audited API
  routes that need it for cross-tenant-safe writes (findings status changes, etc.) — never from
  any file under `app/(marketing)` or any client component.

## Input validation

Every API route validates its body/query with Zod before touching the database — rejecting
malformed input with a 400 and a specific error code rather than letting it reach SQL or
business logic. URL fields additionally go through `safeFetch`'s validation pipeline (step 1–3
above) synchronously at submission time so bad input is rejected immediately with a clear
message, not discovered later during the async collection stage.

## Rate limiting (brief §32)

Implemented via a Redis-backed (Upstash) sliding-window limiter, keyed appropriately per
endpoint class:

| Endpoint class | Key | Limit |
|---|---|---|
| Audit creation | `organization_id` | Per plan entitlement (see `PLAN_LIMITS`), backstopped by 10/hour/org hard ceiling |
| Auth (sign-in/sign-up/magic-link) | IP + email | 10/hour per IP, 5/hour per email |
| Report generation/regeneration | `organization_id` | 10/hour |
| AI calls (internal, not user-facing but still bounded) | per-audit stage | bounded by the pipeline itself (each stage calls each prompt at most once + one retry) rather than a separate limiter — the pipeline design is the rate limit |
| Public endpoints (marketing pages, example report) | IP | standard reverse-proxy-level limit, generous, mainly bot/abuse protection |

## Webhook security

Stripe webhook handler verifies the `Stripe-Signature` header against `STRIPE_WEBHOOK_SECRET`
before parsing the body at all; invalid signatures are rejected with 400 and logged, never
processed.

## Logging discipline (brief §31)

Structured logger (`src/lib/logging/logger.ts`) attaches `{ requestId, auditId, userId,
organizationId, stage, provider }` to every log line. User-facing error responses are always a
short, specific, non-technical message (per the error-code table in `05-api-contract.md`);
`error_details` (stack trace, provider response body, etc.) is written only to
`audits.error_details` (internal column, never returned by any API route) and to the structured
log stream — never surfaced in an HTTP response body.
