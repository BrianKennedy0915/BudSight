# 11. External Data-Provider Requirements

Every row in this table implements `DataProvider` (see `07-domain-models.md`). "Swappable"
means a new provider for the same purpose is a new file in `src/lib/providers/`, not a change
to `jobs/` or `domain/`.

| Purpose | Recommended provider (MVP) | Why | Licensing/ToS posture | Cost tier |
|---|---|---|---|---|
| Client/competitor website content | Self-hosted fetcher (Node `fetch` + Readability/Cheerio extraction, SSRF-guarded) | Not scraping a third-party platform — it's fetching the dispensary's own public marketing site, which is categorically different from scraping a review/menu aggregator | Low risk: fetching a business's own public website for a tool that business or a prospective vendor requests analysis of is standard practice; still respects `robots.txt` for non-essential crawling and sets a descriptive, honest user-agent | Compute only, ~$0 marginal |
| Business profile (hours, address, category, rating, review count) | **Google Places API** (Place Details + Nearby Search) | Official, licensed, documented ToS; works for both client and competitors since it's public listing data Google licenses out | Licensed — must comply with Places API terms (e.g., caching limits, attribution requirements) | ~$0.017–$0.032 per call depending on SKU/fields requested |
| Competitor discovery (candidate generation) | **Google Places Nearby Search** by category + radius | Deterministic, licensed, matches brief §12's "deterministic where possible" requirement directly | Same as above | Included in Places cost above |
| Search visibility / SERP position / keyword volume | **A licensed SERP API** (e.g., SerpApi, DataForSEO, or ValueSerp — final vendor selection pending a quick cost/feature bake-off, not architecturally significant) | Scraping Google's SERP directly violates Google's ToS (brief §11 explicitly rules this out: "do not build the MVP around a scraping strategy that violates website terms") | Licensed, vendor-dependent terms | ~$0.001–$0.01 per query depending on vendor/volume tier |
| Reviews — Google | Covered by Google Places API (`rating`, `user_ratings_total`) | Same listing above | Licensed | Included above |
| Reviews — Yelp | **Yelp Fusion API** (optional, only if Yelp is materially relevant for Arizona dispensaries — needs a quick market check) | Official API exists with documented terms; rate-limited and may require partner approval for some endpoints | Licensed, conditional on approval | Free tier available, paid tiers for volume |
| Reviews/menu — Weedmaps, Leafly | **Excluded from MVP data collection** | No public third-party API; scraping would violate ToS and directly contradicts brief §11/§35. See `00-overview.md` risk #3. | N/A — explicitly out of scope until a licensing/partnership path exists | N/A |
| Geocoding (address → lat/lng, for distance ranking) | Google Geocoding API (same Google Maps Platform account as Places) | Same account, same licensing terms, no new vendor relationship | Licensed | ~$0.005 per call |
| AI analysis/classification/narrative | **OpenAI API** | Specified explicitly in brief §46 | Standard commercial API terms | See `15-cost-model.md` |
| PDF report rendering | **Self-hosted** (Playwright + headless Chromium rendering the report's HTML template) | Avoids sending client/competitor data to a third-party rendering service; no per-document vendor cost | N/A | Compute only |
| Payments | **Stripe** | Specified in brief §29/§46 | See `00-overview.md` risk #2 re: cannabis-adjacent eligibility | Standard Stripe processing fees |
| Error monitoring | Sentry (or equivalent) | Brief §46 calls for "application error monitoring and structured logging" without naming a vendor; Sentry is the standard Next.js/Vercel-compatible choice | Standard commercial terms | Free/low tier sufficient at MVP volume |

## Explicitly out of scope for MVP (revisit in V2+ per brief §51)

- Weedmaps/Leafly data (no license path yet)
- Any traffic-estimation API (SEMrush/Ahrefs/Similarweb) — MVP uses the keyword-volume × CTR
  proxy from `09-revenue-estimation.md` instead; revisit if budget allows a direct license
- Google Business Profile **Performance** API (impressions/clicks) for the *client's own*
  listing — requires the client to OAuth-connect their GBP, which is a real, valuable, cheap
  addition but is a scope decision (small amount of new onboarding UX), not purely an
  architecture one. Flagging as a strong V1.1 candidate rather than true V2.
- POS/inventory integrations (explicitly V2 per brief §51)

## Rate limits / quotas to plan for

Each provider call is wrapped with a per-provider rate limiter (brief §32) sized to the
provider's own API quota, independent of BudSight's user-facing rate limits — e.g., the Places
and SERP providers get a shared token-bucket limiter per API key so a burst of concurrent
audits doesn't trip the vendor's own throttling and cause unrelated audits to fail.
