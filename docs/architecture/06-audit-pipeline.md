# 6. Audit Pipeline / State Machine

## States

```
   QUEUED
      │
      ▼
   COLLECTING ───────────────┐
      │                       │
      ▼                       │
   NORMALIZING                │
      │                       │
      ▼                       │
   COMPETITOR_DISCOVERY       │  any stage can transition to FAILED
      │                       │  (see "required vs degradable" below)
      ▼                       │
   ANALYZING                  │
      │                       │
      ▼                       │
   SCORING                    │
      │                       │
      ▼                       │
   REVENUE_ESTIMATION         │
      │                       │
      ▼                       │
   GENERATING                 │
      │                       │
      ▼                       ▼
   COMPLETED               FAILED
                       (error_code, error_message,
                        error_details, retryable: bool)
```

This matches brief §20 exactly, with one addition the brief doesn't resolve: **not every
sub-step failure should fail the whole audit.** An all-or-nothing model means a single flaky
review-API call throws away an otherwise-complete audit, which directly hurts the "handles
failures gracefully" success criterion in §39. So each stage is classified:

| Stage | Required? | Behavior if it ultimately fails (after its own retry budget) |
|---|---|---|
| `COLLECTING` — client website | **Required** | Audit → `FAILED`, `error_code='WEBSITE_UNREACHABLE'`. Without the client's own site, nothing downstream is meaningful. |
| `COLLECTING` — business profile (Places) | Degradable | Logged as `audit_events` warning; affected findings get lower confidence / fewer `OBSERVED` facts; audit continues. |
| `NORMALIZING` | **Required** | Pure deterministic code operating on already-collected data; a failure here is a bug, not a flaky dependency — surfaced loudly, audit → `FAILED`. |
| `COMPETITOR_DISCOVERY` | Degradable (floor: 0 competitors allowed) | If 0 competitors found, `competitive` category findings are skipped with an explicit "insufficient evidence" note rather than fabricated; audit continues. |
| `COLLECTING` — competitor data | Degradable per-competitor | A single competitor's site/profile failing doesn't block the others; that competitor is marked `data_incomplete` in its evidence and excluded from comparisons needing the missing field. |
| `ANALYZING` (AI) | **Required**, but see note | If the model call fails validation after its retry budget, that specific prompt's output degrades to "insufficient evidence" for its findings rather than failing the whole audit — **unless** the Website Analyst (the foundational prompt) itself fails, in which case the audit can't proceed and → `FAILED`. |
| `SCORING` | **Required** | Deterministic; a failure is a bug. |
| `REVENUE_ESTIMATION` | **Required**, but gracefully degrades to "insufficient evidence to produce a reliable revenue estimate" (brief §7) rather than erroring, when inputs are too thin |
| `GENERATING` (PDF) | **Required for report delivery**, but audit itself is already `COMPLETED` by this point — report generation failure sets `reports.status='failed'` independently and is retryable via `POST /api/reports/:id/regenerate` without re-running the audit |

This means `audits.status` only ever reaches `FAILED` for the stages marked "Required" above,
and the brief's binary state machine is preserved at the top level while degradation is
tracked at the finding/evidence level (`confidence_score`, and a literal "insufficient
evidence" finding where applicable) — satisfying both §20 (simple state machine) and §39
(graceful failure handling, credible-over-impressive numbers).

## Idempotency per stage (brief §22)

| Stage | Idempotency mechanism |
|---|---|
| Website/competitor collection | `data_snapshots` unique on `(location_id, source_type, source_identifier, hash)` — re-running with identical fetched content is a no-op insert; changed content creates a new snapshot (preserving history) without touching the old one |
| Competitor discovery | `competitors` unique on `(location_id, place_id)` — re-discovery upserts by `place_id`, never duplicates |
| Normalization / AI analysis / scoring / revenue estimation | `findings` unique on `(audit_id, dedupe_key)` where `dedupe_key` is a deterministic hash of `(category, normalized subject)` e.g. `"competitive:review_velocity_gap:competitor_<id>"` — re-running a stage **updates** the existing finding row (new scores/evidence links) rather than inserting a duplicate |
| Report generation | `reports` unique on `audit_id` — regeneration overwrites `storage_path`/`generated_at` on the same row |

## Retry semantics

- Inngest provides step-level automatic retry with exponential backoff for transient failures
  (network timeouts, 5xx from a provider, rate-limit responses) — default 3 attempts per step.
- A stage's **final** failure (retry budget exhausted) is what triggers the required/degradable
  logic above, not the first failure.
- User-initiated retry (`POST /api/audits/:id/retry`) resumes the *same* `audits` row from its
  recorded failed stage — it does not create a new audit, and relies on the idempotency keys
  above so already-completed stages' side effects aren't redone or duplicated.
- "Run a new audit" (from the UI, not a retry) always creates a brand-new `audits` row with a
  fresh `audit_version`, even for the same location — see ambiguity #11 in `00-overview.md`.
  This is what makes "a second audit produces updated data" (§39) and full historical
  reproducibility (§23) both true simultaneously: old audits are never mutated by new runs.

## Versioning (brief §23)

`scoring_version`, `prompt_version`, `data_provider_version` are stamped onto the `audits` row
**at creation time** from constants defined in `domain/scoring/`, `prompts/`, and
`lib/providers/` respectively (e.g., `SCORING_VERSION = "2026.01"`). Bumping any of these
constants in code only affects audits created after the bump; existing audits keep referencing
the version they were computed under, which is what makes "why did BudSight calculate $18,000
for this audit six months ago" answerable — the exact scoring logic and prompt text for that
version stays in git history, tagged/branchable by version string.

## `audit_events` as the single source of truth for UI progress

Every stage transition (`started` / `succeeded` / `failed` / `skipped`) writes one row. The
brief's progress checklist (§4 Step 3) is rendered directly from these rows — there is no
separate "fake progress" animation; a step shows a checkmark only once its `succeeded`
`audit_events` row exists. This also makes `audit_events` the primary artifact for debugging
failed audits per brief §17's note that this table is "extremely important."
