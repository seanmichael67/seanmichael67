# Parlami SaaS audit — what each agent can really do today

**Date:** 2026-10-27
**Audited:** `seanmichael67/parlami-ads-saas` at commit `fc14d16` (2026-10-26), plus the two
side repos `seanmichael67/geogrid-tool` and `seanmichael67/faccia-capi`.
**Method:** six code-level reviews, one per agent lane. Every claim below was traced to a
file in the repo, not to a README. Where the docs and the code disagree, the code wins and
the disagreement is called out.

---

## 1. The short version

Parlami is much further along than the README says in some places, and much less far along
than the marketing names suggest in others. The honest shape of the product today is:

| Layer | State |
|---|---|
| **Pulling data in** (Meta Ads, Google Ads, GA4, Search Console, Google Maps rankings, Places reviews) | **Strong.** Nightly crons, 7 Search Console grains, 5 Meta breakdown families, ~30 Google Ads query types. |
| **Finding problems** (rule engines) | **Strong.** 19 Faccia rules, 61 Annunci rules, 7 Spia rules + 53 on-page checks, 6 Penna, 5 Architetto, 6 Bussola, 5 Zona, 10 Catena, 5 Musa, 5 Regista, 7 Stella. |
| **Suggesting fixes** (approval queue) | **Works.** Every agent writes evidence-backed proposals a human approves or rejects. |
| **Doing the fix** (executors) | **Built, dark.** Pause ad, remove placement, +20% budget, negative keyword, paused campaign draft, sitelinks all exist in code. Every one is behind a kill switch that ships OFF, and the nightly cron is hard-coded dry-run. Nothing has ever changed a live ad from inside the SaaS. |
| **Making creative** (images, video, blog copy) | **Weak.** Image generation exists but is switched off. Video generation lives on your workstation ("Spark"), not in the SaaS. Blog drafts are empty skeletons, not written posts. Campaign Studio ad copy is hard-coded per vertical, not AI-written. |
| **Closing the loop** (which ad produced which enrollment) | **Half built.** Native booking and enrollment forms capture first-touch and click IDs. But the script that records first touch on the school website, and the ad URL tagging, are both still gated on you. Meta form leads link to an ad but not to a booking. The `leads` table has no UTM or click-ID columns at all. |

Two findings that cut across everything:

1. **The product runs on three machines, and only one of them is the SaaS.** The Vercel app
   does sync, rules, approvals, and dashboards. Your "Spark" workstation does the competitor
   Ad Library scrape, video transcription, the ad-variation copywriting, and all video
   rendering, through cron jobs that write into the SaaS database. A separate Python server
   on Render (`faccia-capi`) sends live Meta conversion events for two schools and is not
   referenced anywhere in the SaaS. If Spark is off, Musa's competitor drafts stop, and
   nothing in the dashboard says why.
2. **Several agents are snapshot-fed.** Musa, Regista, Piazza (Voce) and Catena run their
   rules only over JSON files an operator imports by command line
   (`scripts/import-governance-snapshot.ts`, `scripts/import-catena-snapshot.ts`). They do not
   fetch the client's own creative inventory or backlinks. On a fresh tenant they report
   "no snapshot ingested."

---

## 2. Your questions, answered one by one

### Q1. Can Faccia read all Facebook ads, campaigns, and performance, and make true suggestions?

**Yes for reading. Yes for suggesting. No for acting. Partial for "true."**

**What it reads (nightly, `apps/web/src/lib/meta/`):**
- Every campaign, ad set (including learning-phase status, targeting, budgets) and ad in the
  selected ad account. Account-wide, no filter.
- Ad-level insights: impressions, reach, frequency, clicks, CTR, CPC, CPM, spend, leads,
  conversion values, video funnel (3-second views, 25/50/75/95/100% completion), Meta's
  quality / engagement / conversion rankings, three attribution windows.
- Five breakdowns: placement, age and gender, device, region, hour of day.
- Creative library, images, videos, custom and saved audiences, pixel health, account status,
  spend cap, rejected ads.
- Leads from lead forms, polled hourly plus a real-time webhook.

**What it suggests:** 19 rules produce findings (creative fatigue, zero-lead spend, placement
waste, weekday waste, weak video hook, poor quality ranking, cost-per-lead anomalies, and so
on). A second layer classifies each ad (winner, fatigued, zombie, testing) and writes five
proposal types to the approval queue: pause ad, remove placement, request new creative,
review objective, raise budget up to 20%.

**Why "true" is only partial:**
- Only the anomaly rules use real statistics (`packages/stats-core`). The other 14 rules are
  fixed thresholds: cost per lead over $50, frequency over 3.0, CTR under 1%. Those numbers
  are the same for a preschool and a personal-injury firm unless a strategy pack overrides
  them.
- Four of the five breakdowns (age/gender, device, region, hourly) are synced and stored
  but **no rule, verdict, or screen reads them.** Only placement is used.
- Insights are pulled for "yesterday" only and never re-pulled. Meta attributes 7-day-click
  conversions late, so those leads are lost from the stored data.
- `campaignOwnershipScope` (the rule that stops Beibei's dashboard showing Amici's campaigns
  on a shared ad account) is applied in the cockpit but **not** in Faccia's rule input, so
  findings can fire on a sibling business's campaigns.

**Acting:** The executor (`apps/web/src/lib/faccia/executor.ts`) supports pause ad, remove
placement, budget increase (max 20%, max $100/day) and paused campaign draft (max $50/day).
Gates that all ship OFF:

```
FACCIA_EXECUTOR_ENABLED=true
FACCIA_EXECUTOR_ACCOUNT_ALLOWLIST=act_xxx
COCKPIT_EXECUTOR_LIVE=true      # cockpit button, pause-ad only
```

Even with those on, the nightly executor cron is hard-coded to dry-run. The only live path is
the cockpit "Fix and approve" button, and only for pause-ad. No bid changes, no moving budget
between ad sets, no unpausing.

**Competitor ads:** Faccia's rules never read competitor data. See Q2.

**Outcome feedback:** `compute-faccia-outcomes` measures every approved proposal against real
bookings and enrollments 7/14/30 days later. Good. But nothing feeds that back into the rule
thresholds automatically (by design in v1). Meta Conversions API upload is built but dark
(`META_CAPI_UPLOAD_ENABLED`, `META_CAPI_PIXEL_ALLOWLIST`), and it sends custom event names
(`lead_qualified`, `lead_converted`) rather than Meta's standard `Lead` / `Schedule` events,
so even when switched on it will not train a standard Lead campaign without extra setup in
Events Manager.

### Q2. Can Musa make new creatives using competitors from the Facebook Ads Library?

**Partly, and the part that works does not run in the SaaS.**

**Three different things are called "Musa":**

1. **Five governance rules** (unapproved asset in use, missing placement, low resolution,
   wrong aspect ratio, creative fatigue). They only run over a JSON snapshot you import by
   hand. No automatic pull of the client's creative inventory.
2. **Campaign Studio "Musa review."** Produces three copy + image-concept variants
   (1:1, 4:5, 9:16). The copy is **hard-coded per vertical** ("Explore a place to learn and
   grow" for every preschool). The provenance record literally says `model: null`.
3. **Competitor-inspired drafts on the dashboard.** This is the real thing you are asking
   about. It works like this:
   - A cron on your **Spark workstation** (`scripts/knowledge-ingest-nightly.sh`, 02:17) opens
     the public Facebook Ad Library in headless Chromium, scrapes up to 6 competitors × 30 ads
     per tenant, and stores them in `competitor_ad_snapshots`.
   - For ads running 21+ days, an LLM distills "copy angle" lessons, then a second LLM pass
     writes ad-variation drafts grounded in the client's brand brief, with anti-plagiarism
     guards (no 40-character run copied, no competitor named).
   - A third script downloads competitor videos and transcribes them with Whisper to extract
     the spoken hook.
   - The SaaS dashboard (`MusaCreativeCarousel`) shows up to four of those drafts with the
     cited competitor ads, a "Draw" button, and a "Launch" button that creates a Campaign
     Studio draft.
   - The official Meta `ads_archive` API is also wired (for Spia's weekly intel) but, by the
     code's own comment, returns almost nothing for US non-political advertisers.

**Image generation:** real OpenAI `gpt-image-1.5` integration exists
(`apps/web/src/lib/musa/openai-image-provider.ts`) with consent tokens, spend claims, and
private storage. It ships OFF:

```
MUSA_IMAGE_GENERATION_ENABLED=true
OPENAI_API_KEY=...
```

Even on, the images are **text-free backgrounds**. Copy is never composited onto them, so there
is no finished ad unit. Preschool tenants are hard-blocked from any AI people imagery (correct
for your schools, but it means Musa can never draw a classroom scene with children).

**Getting a creative into Meta:** Campaign Studio can create a paused campaign / ad set / ad,
but only pointing at a `creative_id` that **already exists** in the ad account. Nothing uploads
a generated image to Meta.

**Rights tracking:** `media_assets` has a `client_upload` source and provenance flags, but
there is no upload UI and no code ever writes a `client_upload` row. Parental media-release is
a policy flag on the tenant, not a per-asset record.

**Missing for a real "creative studio":** client media upload with release records, copy
compositing, brand kit beyond tone and two colors, upload of generated assets to Meta, hosted
scraping (so it does not depend on Spark), LLM-written Studio copy.

### Q3. Can Regista make new videos?

**Not in the SaaS. On your workstation, yes, in a limited way.**

- **In the SaaS:** five rules (missing captions, overlong, unapproved video in use, video
  fatigue) over a hand-imported snapshot, plus a storyboard generator whose hook line is a
  hard-coded string per vertical. The Campaign Studio page says approval "cannot render,
  publish, activate, or spend." There is no ffmpeg, no TTS, no captioning, no video model
  anywhere under `apps/` or `packages/`.
- **On Spark (`registra/` folder, Python, cron every 5 min):**
  - `registra_render.py` takes existing client footage, trims it, reframes to 16:9 / 1:1 /
    9:16 / 4:5, burns a title in with Pillow, exports H.264 with ffmpeg. That is a real
    render, but it is a reframe of footage you already have, with no voiceover or captions.
  - `registra_suggest_titles.py` asks Claude for title options.
  - `generative/render_lane.py` (merged Oct 13) generates beats with **Higgsfield
    `seedance_2_5`** and stitches them with **HyperFrames**. This is the only AI-video path in
    the whole codebase. It reads hand-written lane JSON and has no input from, or output to,
    the SaaS.
  - `piazza_publish.py` can post organic video to Facebook / Instagram. README says it has
    never been live-fired.
- **Wiring:** none. No API route, cron, queue, or table connects the SaaS storyboard to the
  Spark renderer. The `MusaToRegistraVideoRequest` contract exists only in a doc.

### Q4. Can Google Search Console, Spia and Bussola give up-to-date information on client webpage visits, organic vs ads, and track first touch → booking → enrollment?

**Search Console and GA4: yes, nightly and real. Organic vs ads: partial. First touch to
enrollment: built, but the two pieces that make it work are still gated on you.**

**Search Console (`sync-gsc-queries`, 03:40 daily):** 30-day top 250 queries, plus 56 days of
daily data across seven grains (query, page, query×page, device, country, and combinations).
A 16-month history backfill exists behind `GSC_HISTORY_BACKFILL_ENABLED`. Beibei has had
rows since June 12.

**GA4 (`sync-ga4-metrics`, 03:30 daily):** 14 trailing days of sessions, engaged sessions,
users, key events, by channel group, source, medium, campaign name, and landing page. Not
pulled: device, geography, campaign ID, anything per person.

**"X visits this week, Y organic, Z from Meta, W from Google Ads":** The Bussola panel shows
a **28-day** channel breakdown (Organic Search, Paid Social, Paid Search, Direct) and a
three-way comparison of what Meta claims vs what Google claims vs what GA4 saw. There is no
weekly toggle on the client dashboard, and "from Meta" is inferred from GA4's channel
grouping, never joined to a campaign ID.

**First touch → booking → enrollment, traced end to end:**

| Step | Built? | Works in production? |
|---|---|---|
| Script on the school website that records first touch (UTMs, gclid, fbclid, referrer) and carries it to the booking page | Yes: `apps/web/public/parlami-attribution.js` and a Beibei-specific keeper | **Gated on you** (spec item R-1). As of Aug 23 prod had 5 bookings, 0 with any click ID. |
| Ads tagged with UTMs and campaign IDs in the landing URL | Documented | **Gated on you** (P-1). |
| Native tour booking (`/book/<slug>`) stores first and last touch, click IDs, Meta/Google campaign IDs | Yes: `tour_booking_attribution` | Yes |
| Enrollment form (`/enroll/<slug>`) stores the same, plus "how did you hear about us" | Yes: `enrollment_submissions` | Yes |
| Booking automatically linked to enrollment | Yes: `enrollment-tour-link.ts`, matched by browser session or parent email | Yes |
| `/journeys` page with first-touch / first-marketing / last-touch model switch and "unknown source %" | Yes | Yes, but mostly "unknown" until R-1 and P-1 land |
| Send the enrollment back to Google as an offline conversion | Yes, bookings with gclid only | Dark: `ANNUNCI_OFFLINE_UPLOAD_ENABLED` |
| Send it back to Meta via Conversions API | Yes | Dark: `META_CAPI_UPLOAD_ENABLED` |

**Where the chain breaks today:**
- The `leads` table has **no UTM, gclid, fbclid or referrer columns.** Meta form leads carry an
  ad ID (good) but reach a booking only if the parent later books with the same email.
  Website-pixel leads, CSV imports and YouCanBookMe bookings carry no source at all (YCBM
  actually parses gclid and UTMs, then drops them into `raw_payload`).
- The **client** funnel (`FunnelSummary`) answers "are they booking?" with GA4 key events and
  "are they enrolling?" with manually marked conversions. It does not read `tour_bookings` or
  `enrollment_submissions`, which the system already stores. The internal cockpit funnel does.
  So a school with 132 registrations and no manual marks shows "—" on stage 4.
- Three Bussola rules that would compare platform claims to site truth (`attribution-gap`,
  `cross-source-conversion-gap`, `source-freshness`) are exported but not in the default
  runner. The enrollment-attribution rule has no data producer.

**Spia:** seven nightly rules over Search Console (brand visibility, ranking decay,
striking distance, rising queries, cannibalization, low CTR, AI-referral traffic from ChatGPT /
Perplexity / Gemini). Weekly competitor intel (keyword gaps, domain overview, backlink
count) and AI-Overview / ChatGPT citation probes use DataForSEO and OpenAI, all behind
flags: `SPIA_AI_OVERVIEW_ENABLED`, `SPIA_AEO_QUESTIONS_ENABLED`, `SPIA_AEO_CHATGPT_ENABLED`,
`SPIA_WEEKLY_BOX_ENABLED`, plus a $10/week per-tenant spend cap.

### Q5. Can Zona give geo-grid tracking for the ZIP codes a client markets to, and check community pages for keywords?

**Geo-grid: yes, it works and the README is wrong. ZIP codes: no. Community pages: no.**

- **The README and agent catalog say Zona is "Blocked — needs a local-rank data source."
  That is stale.** `POST /api/zona/scan` runs a real Local-Falcon-style scan: DataForSEO
  Google Maps SERP at every grid cell (about $0.006 per cell), Google Places for the center,
  results into `zona_scans`, map with colored pins on `/agent/zona` and the cockpit, top-3
  share, found share, average rank, trend history, and five rules. A real scan ran on Beibei
  on Aug 9. Needs `DATAFORSEO_LOGIN` / `DATAFORSEO_PASSWORD` (not in `.env.example`) and
  `GOOGLE_PLACES_API_KEY`.
- **No schedule.** There is no Zona cron. Scans only happen when someone clicks "Run scan,"
  one keyword at a time. There is no tracked-keyword list.
- **No ZIP input anywhere.** The grid is a square lattice (default 5×5, 3 miles; cockpit
  hardcodes 5 miles) around the business's Google place ID. There is no postal-code field in
  the Zona code, the schema, or the UI. Onboarding collects a free-text "geography" answer
  into `tenants.target_market` and nothing parses it. The rules engine keys on a generic
  `grid_id`, so ZIP centroids could be dropped in without rewriting the rules.
- **Community / neighborhood pages:** nothing audits or generates them. Architetto's five
  rules and Penna's six do not look at location pages. The only local signal on pages is the
  generic LocalBusiness-schema check inside Spia's 53-point on-page audit. A spec
  (`docs/specs/zona-engine-port-v1.md` §1.2) describes exactly what you asked for, landmark
  density, geo signals, local JSON-LD, and it is not built. The legacy
  `amici-community-pages-generator.py` was deliberately excluded from the port.
- **Citations (Catena):** a real 45-directory catalog with a childcare pack (Care.com,
  Winnie, GreatSchools, Niche), monthly prefilled submission drafts, weekly verification, a
  `/citations` page. Actual form submission needs a Make/Zapier webhook
  (`CATENA_SUBMIT_WEBHOOK_URL`) or a human. NAP-mismatch findings fire only on snapshots you
  import.
- **Google Business Profile:** read-only client built; **API access is still pending with
  Google**, so `GOOGLE_GBP_SCOPE_ENABLED` stays off. Rating, review count and five review
  texts come through the Places API instead. GBP post drafts go to the approval queue weekly
  but there is no publisher.
- **The old `geogrid-tool` repo** (Playwright scraping Google Maps directly, DuckDuckGo
  citation checks, two hard-coded schools) is orphaned. Its grid math was ported; its scraper
  was correctly rejected as not SaaS-safe.

### Q6. Can Annunci run Google Ads effectively, using Google keyword research?

**Analysis: strong. Keyword research: none. Running campaigns: narrow and dark.**

- **Reads** (nightly): ~30 Google Ads query types over 30 days: campaigns, ad groups, ads
  (ad strength, RSA headlines), keywords with quality score components, search terms
  (including PMax and DSA), landing pages, device, hour of day, geo (region and country only,
  **not ZIP**), change history, conversion actions, sitelinks, Google's own recommendations,
  Local Services leads.
- **61 rules:** wasted search terms with the negative pre-drafted, winners capped by budget,
  device / geo / time-of-day leaks, duplicate keywords, silent Google changes, landing-page
  dead ends, n-gram waste themes, conversion-quality inflation, a scorecard against agency
  benchmarks.
- **Executes** (dark by default): add campaign-level negative keyword, create a **paused**
  Search campaign (one ad group, up to 20 keywords, one RSA, max $50/day, English only), create
  or replace a sitelink. Gates:

  ```
  ANNUNCI_EXECUTOR_ENABLED=true
  ANNUNCI_EXECUTOR_CUSTOMER_ALLOWLIST=1234567890
  # plus a per-tenant annunci_autonomy_policies row (default: manual)
  ```

  Budget changes, bid-strategy changes, pausing keywords or ads, editing RSAs, ZIP-level geo
  bids, and dayparting edits are all **proposal-only, "apply manually."**
- **Keyword research:** there is no call to Google's Keyword Planner
  (`generateKeywordIdeas`) and no volume, CPC, or competition data anywhere in Annunci. New
  keyword ideas come only from the account's own converting search terms and from Search
  Console queries. DataForSEO search-volume calls exist in Spia only.
- **Offline conversions:** uploads gclid-based tour bookings to Google. Enrollments and plain
  leads are not enqueued. Dark behind `ANNUNCI_OFFLINE_UPLOAD_ENABLED`.
- **Practical blocker:** FOCUS.md says Beibei's Google connection has been `pending` since
  July 24, and the rule runner skips tenants without a fresh active connection. The repo does
  not record whether the production OAuth consent screen is verified or the developer token
  has Basic access. Until that is fixed, the whole Annunci loop is idle for that tenant.

### Q7. Website optimization, SEO, AEO, geo ranking, blog writing, creative studio: where are we?

| Agency service | Parlami today | Gap |
|---|---|---|
| **Technical site audit** | Architetto fetches up to 50 WordPress-listed pages nightly with a plain HTTP GET: broken, slow (HTML fetch time only), mixed content, missing GA4/GTM tag, duplicate titles. Spia adds 53 on-page checks per page (title, meta, canonical, H1, schema, alt text, internal links, "answer-first" AI checks). | No Core Web Vitals / Lighthouse, no JavaScript rendering, no link-following crawl, no robots/sitemap, no Meta Pixel detection, no crawl history table, WordPress only. |
| **On-page optimization** | Findings with a ready-to-paste LocalBusiness JSON-LD block. | Advisory only. The SaaS cannot write a title, meta description, or schema to WordPress. |
| **AEO / GEO** | FAQPage and LocalBusiness schema builders, an `llms.txt` draft proposal, weekly AI-Overview citation probe and ChatGPT probe, six AI-readability on-page checks, AI-referral traffic rule. | All probes flag-gated. Nothing deploys. Raw AI answers are discarded. |
| **Keyword research + content calendar** | Topics come only from the tenant's own Search Console striking-distance queries (position 5–20). | No calendar, no volume data in Penna, no Keyword Planner. |
| **Blog writing** | Weekly content briefs (opt-in per tenant) with a strong editorial spec. WordPress draft push exists. | **No LLM writes a post.** The "draft" is an H2 skeleton with the literal placeholder `[Answer first in 40–60 words...]`. Draft push gated by `PENNA_WP_DRAFTS_ENABLED` (not even in `.env.example`) plus a tenant flag. |
| **WordPress control** | Verify credentials (Application Password), read pages/posts, create a draft post. | Cannot publish, update, edit meta/Yoast, install tracking, or add schema. |
| **Backlinks** | 10 Catena rules, outreach email drafts from competitor gaps. | Data is a manual CSV import. The DataForSEO backlinks adapter is complete and has **zero callers**. No send path for outreach. |
| **Local SEO / geo ranking** | Zona grid scan (see Q5), Places reviews, citation drafts. | No schedule, no ZIP, no location pages, no GBP publishing. |
| **Creative studio** | Competitor-inspired copy drafts, gated image generation, storyboard text. | See Q2 and Q3. |
| **Client reporting** | Weekly report covers Faccia, Annunci and leads. | The entire SEO lane (Penna, Architetto, Catena, Spia on-page, Zona) is absent from the weekly client report. |

---

## 3. Things the docs get wrong (fix these so you stop re-deciding them)

| Doc says | Code says |
|---|---|
| README, agent catalog: Zona "Blocked, needs local-rank data source" | Zona scans are live on DataForSEO since Aug 9 (`packages/db/drizzle/0087_zona_scans.sql` header says so). Flip `truthTag`, update README. |
| `PROJECT-STATE.md`: "Stella and Zona remain package-only" | Both are wired. |
| Agent catalog: "Production Google OAuth credentials pending (Session G)" on Annunci, Bussola, Spia | GSC and GA4 have had live rows for Beibei since June. Consent-screen verification status is simply undocumented. |
| `docs/faccia-rules.md`: 14-day lookback | Code default is 30 (`FACCIA_RULE_LOOKBACK_DAYS`). |
| `ANNUNCI_STATUS.md`: "no tracked Google code exists" | 4,665-line rule engine, executor, sync. |
| `.env.example` | Missing `DATAFORSEO_LOGIN`, `DATAFORSEO_PASSWORD`, `PENNA_WP_DRAFTS_ENABLED`, `FACCIA_EXECUTOR_*`, `SPIA_*_ENABLED`. |
| Zona agent page "Key numbers" cards | Read `governance_snapshots` for `zona`, which nothing writes. Always empty. |

---

## 4. What to do, in order

### A. Switches and connections (days, not weeks, no new code)

These are the highest-value moves because the code is already written and tested.

1. **Reconnect Google for Beibei** so Annunci, Bussola, Penna, Spia wake up. Document the
   OAuth consent-screen status once.
2. **Deploy the first-touch script** (R-1) on both school sites and **tag every ad URL**
   (P-1). Until this is done `/journeys` reads "unknown source" and every question about
   "which ad produced this enrollment" is unanswerable.
3. **Turn on `MUSA_IMAGE_GENERATION_ENABLED`** with an OpenAI key for one tenant. The
   consent-token and spend-claim safety is already built.
4. **Add `DATAFORSEO_*` to `.env.example`** and confirm it is set in Vercel. Then turn on
   `SPIA_AI_OVERVIEW_ENABLED` and `SPIA_WEEKLY_BOX_ENABLED` for one tenant.
5. **Flip `FACCIA_EXECUTOR_ENABLED` + allowlist** for one tenant in **dry-run** and let the
   executions page fill for two weeks before anything live.
6. **Fix the Zona docs and catalog** so the agent reads "Live, on demand."

### B. Small builds with large effect (one to three days each)

1. **Add UTM / gclid / fbclid / referrer columns to `leads`** and write them from the Meta
   webhook, YCBM webhook (already parsed, just dropped), and website capture.
2. **Make the client funnel read `tour_bookings` and `enrollment_submissions`** for stages
   3 and 4, the way the cockpit already does.
3. **Apply `campaignOwnershipScope` inside Faccia's rule input** so shared ad accounts stop
   leaking.
4. **Re-pull the last 7 days of Meta insights nightly** instead of yesterday only, so
   late-attributed leads are not lost.
5. **Add a Zona cron** with a per-tenant tracked-keyword list (3 to 5 keywords, weekly).
6. **Wire the DataForSEO backlinks adapter** into `run-catena-rules` so Catena stops
   depending on CSV imports.
7. **Add the SEO lane to the weekly client report.**

### C. Real builds (one to three weeks each)

1. **ZIP-code geo-grid.** New `zona_tenant_config` with a target-ZIP list, ZIP centroids as
   grid cells, per-ZIP rank and share-of-voice. The rules engine already accepts any cell ID.
2. **Community / location page audit and generation.** Build the spec in
   `zona-engine-port-v1.md` §1.2: for each target ZIP or neighborhood, does a page exist, does
   it rank, does it carry local schema and landmarks. Then an Architetto draft generator for
   missing pages (the Architetto landing-page spec is already locked, just unbuilt).
3. **Penna writes the post.** Route the approved brief through the existing Nick LLM provider
   and push real copy as the WordPress draft, not a skeleton.
4. **Keyword research.** Either call Google Keyword Planner through the existing Ads
   connection or reuse the DataForSEO search-volume call Spia already has, and surface it in
   Penna briefs and Annunci campaign drafts.
5. **Finished ad units.** Composite headline, CTA and logo onto Musa backgrounds, upload to
   Meta `adimages` / `adcreatives`, and let Campaign Studio reference the new creative ID.
6. **Bring video into the SaaS.** A queue table plus an adapter that turns a Regista
   storyboard into a Spark ticket, and a callback that stores the rendered MP4 as a
   `media_assets` row. Add TTS and captions on the Spark side.
7. **Client media upload with per-asset release records**, so the parental-release rule is
   data, not a policy flag.
8. **Hosted competitor scraping**, so Musa's drafts do not depend on a workstation cron.

---

## 5. Appendix: the kill switches, all in one place

Every one of these ships OFF. Nothing customer-visible changes until it is set to the literal
string `true` and, in most cases, an allowlist is also set.

```
FACCIA_EXECUTOR_ENABLED            FACCIA_EXECUTOR_ACCOUNT_ALLOWLIST    COCKPIT_EXECUTOR_LIVE
META_EXECUTOR_PROVIDER_WRITES_ENABLED   META_EXECUTOR_LAUNCH_ENABLED
META_CAPI_UPLOAD_ENABLED           META_CAPI_PIXEL_ALLOWLIST
ANNUNCI_EXECUTOR_ENABLED           ANNUNCI_EXECUTOR_CUSTOMER_ALLOWLIST  ANNUNCI_OFFLINE_UPLOAD_ENABLED
MUSA_IMAGE_GENERATION_ENABLED      PENNA_WP_DRAFTS_ENABLED              PIAZZA_EXECUTOR_ENABLED
GOOGLE_GBP_SCOPE_ENABLED           GSC_HISTORY_BACKFILL_ENABLED
SPIA_AI_OVERVIEW_ENABLED           SPIA_AEO_QUESTIONS_ENABLED           SPIA_AEO_CHATGPT_ENABLED
SPIA_WEEKLY_BOX_ENABLED            CATENA_SUBMIT_WEBHOOK_URL            CATENA_INBOUND_EMAIL_SECRET
NICK_CHAT_ENABLED                  VERIFIER_ROUTING_ENABLED             RENEWAL_RECEIPT_EMAIL_ENABLED
```

Source files for the six lane reviews, if you want to go deeper:
`packages/faccia-rules`, `apps/web/src/lib/faccia`, `apps/web/src/lib/meta` (Faccia);
`apps/web/src/lib/musa`, `packages/campaign-studio-core`, `registra/` (Musa, Regista);
`apps/web/src/lib/google`, `apps/web/src/lib/bussola`, `apps/web/src/lib/spia`,
`packages/db/src/journey-funnel.ts` (analytics and attribution);
`apps/web/src/lib/zona`, `packages/zona-rules`, `apps/web/src/lib/catena` (local);
`packages/annunci-rules`, `apps/web/src/lib/annunci` (Google Ads);
`packages/penna-rules`, `packages/architetto-rules`, `apps/web/src/lib/wordpress` (SEO).
