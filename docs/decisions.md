# Decision Log

What was decided, why, and what was turned down. Newest decisions are added at the bottom.

**Why this file exists:** the reasoning behind decisions otherwise lives only in chat history, which future Claude sessions can't see. Before proposing to change anything listed here, read its entry. Re-open a decision only with a new reason, and record the change as a new entry that **supersedes** the old one (don't edit history).

**Statuses:** `Decided` · `Open` (choice pending) · `Superseded by D-xxx`

**Entry format:**
```
## D-xxx — Title
- Date / Status / Decided by
- Decision:
- Why:
- Rejected:
- Revisit when:
```

---

## D-001: Every vendor sits behind an adapter
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder (confirmed from brief §3.0)
- **Decision:** All external services are called through our own interfaces; vendor SDKs live only in `packages/integrations/src/<category>/<vendor>/`. Our database holds the data that matters. Wholesalers are multi-active with per-destination routing.
- **Why:** Vendors change prices, freeze accounts or shut down; switching must be a contained project, not an emergency.
- **Rejected:** Calling vendor SDKs directly throughout the code (faster at first, painful lock-in).
- **Revisit when:** Never in principle; individual adapters can be simplified if a category never needs switching.

## D-002: TypeScript + Next.js for web and admin
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder (confirmed Claude's proposal)
- **Decision:** TypeScript (strict) everywhere; Next.js for the website and admin panel.
- **Why:** Most mainstream React framework, which Claude writes most reliably; shares React with the Expo mobile app; any future developer will know it.
- **Rejected:** SvelteKit/Remix (simpler but less common); Django/Rails (built-in admin, but a second language next to the mobile app).
- **Revisit when:** Next.js complexity causes repeated bugs.

## D-003: Hosting on Railway (replaces Vercel)
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Website, admin, worker and job queue all run on Railway, EU region.
- **Why:** Background jobs need an always-running worker, which Vercel isn't built for. One host and one bill for everything.
- **Rejected:** Vercel + Inngest (Claude's original recommendation: nothing to keep alive, good job dashboard, but job history at a vendor and two hosts); Vercel + Railway worker (two hosts); Render/Fly.io (less smooth / more technical); Hetzner VPS (cheapest, but self-managed server).
- **Revisit when:** Railway reliability or pricing becomes a problem. Next.js and Node run on any host.

## D-004: Supabase for database and logins; only our server talks to it
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Supabase (EU region, Pro plan from launch) for Postgres and passwordless login. The browser and app never access the database directly.
- **Why:** Standard Postgres (portable), built-in auth with 2FA, backups and point-in-time restore. Server-only access avoids the most common Supabase data leak (a wrong row-level security rule).
- **Rejected:** Railway Postgres + separate auth service (one bill, but backups and auth become DIY); browser-direct access with row-level security (faster to build, riskier).
- **Revisit when:** Supabase pricing or limits hurt.

## D-005: Background jobs in our own Postgres (pg-boss) on a Railway worker
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder (follows from D-003)
- **Decision:** An always-running worker service on Railway processes jobs stored in our Postgres (pg-boss by default). The worker sends a heartbeat; if it stops, a critical alert fires.
- **Why:** No queue vendor; job history stays in our database. Stripe webhooks return immediately and issuing happens in a job, avoiding timeouts and duplicate issuance.
- **Rejected:** Inngest, Upstash QStash, Vercel Cron only.
- **Revisit when:** Job volume or visibility needs outgrow pg-boss.

## D-006: Stripe for payments
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder (confirmed from brief)
- **Decision:** Stripe Checkout (hosted), Stripe Tax, Radar; Stripe Connect or PayPal/Wise for affiliate payouts later.
- **Why:** Industry standard, best documentation, used by most eSIM resellers; card data never touches our servers.
- **Rejected (for now):** Merchant-of-record providers (Paddle, Lemon Squeezy). They would handle all VAT, but cost more, give less control, and may not accept eSIMs. Asking them is on the open list.
- **Revisit when:** Stripe refuses or freezes the account, or a merchant of record accepts eSIMs and the VAT workload becomes heavy.

## D-007: USD is the default currency
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Sell in USD by default; the data model supports multiple currencies from day one; prices set per currency (never auto-converted); each order also stores a EUR value for bookkeeping. Stripe pays out to the founder's existing Wise USD business account.
- **Why:** Most customers expected in the US; wholesalers bill in USD, so margins don't move with exchange rates; USD revenue funds the USD wholesaler balance without conversion.
- **Rejected:** EUR default (Claude's first proposal, matching the German setup, but mismatched with US customers and USD costs).
- **Revisit when:** EU customers become a large share; add EUR prices.

## D-008: Full staging environment
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** A complete staging copy (own database, Stripe test mode, wholesaler test credits) where every change is tried before production.
- **Why:** With no human developer, a realistic test environment is the main way to catch problems before customers do.
- **Rejected:** Preview links only, sharing one test database.

## D-009: Custom admin panel as a separate service, with roles
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Admin built in our codebase, run as its own service on `admin.<domain>` with mandatory 2FA and an extra access gate. Roles: owner, support agent, creator/outreach manager, outsourced support (restricted, masked data), bookkeeper (read-only). Every action logged.
- **Why:** Uses our own money rules (refund limits, clawbacks), no per-seat fees, and prepares for helpers and an outsourced support team. Running it separately keeps attacks on the public site away from admin powers.
- **Rejected:** Off-the-shelf internal tool (e.g. Retool: per-seat costs, can bypass our rules); Supabase dashboard only (no roles, easy to corrupt data).

## D-010: PostHog for analytics, plus GA4, Meta and TikTok pixels from day one
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** PostHog (EU cloud) is the main analytics tool. GA4, Meta Pixel and TikTok Pixel are installed from launch (consent-gated in the EU), plus server-side purchase conversions to the ad platforms. All tracking goes through our own tracking layer; money events are also stored in our database.
- **Why:** Strong analytics are a priority. Ads aren't planned but remain a plan B; ad platforms need conversion history to target well, so tracking starts before any ads run.
- **Rejected:** GA4 only (weaker for product analytics); Plausible (too shallow); building our own (too much work); adding ad pixels only when ads start (no history).
- **Revisit when:** Pixels hurt page speed or consent rates noticeably.

## D-011: Tracking consent by region
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** US visitors: full tracking by default with an opt-out. EU/UK visitors: consent banner; cookieless basics until they accept. Unknown location: EU rules.
- **Why:** Collects the most data the law allows; the US is the main market.
- **Rejected:** One banner for everyone (loses US data); cookieless only (too little data).
- **Revisit when:** Lawyer review, or US state laws change.

## D-012: Session replay with masking
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Record visitor sessions in PostHog with typed text, personal fields and activation codes masked; EU visitors only after consent; never on the admin panel.
- **Why:** Shows exactly where customers get stuck and what happened before an error.
- **Rejected:** Error sessions only; no replay.

## D-013: PostHog for error tracking, Sentry as plan B
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Errors reported through our own `errors.capture()` wrapper to PostHog. Customers see a friendly message with a reference code.
- **Why:** One tool fewer; errors link to session replays. The wrapper makes switching to Sentry about a day of work (plus a mobile app release).
- **Rejected (for now):** Sentry (Claude's recommendation: more mature for backend and mobile).
- **Revisit when:** PostHog misses backend or mobile errors, or lacks needed features.

## D-014: Full customer profiles on an internal ID
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** After purchase or login, analytics, errors, support and orders all link to one internal customer ID. Vendor tools get that ID, not the email. Customers can export or delete their data.
- **Why:** Customer-level insight (lifetime value, referring creator, destinations, devices) for better decisions.
- **Rejected:** Anonymous aggregates only.

## D-015: Data principle: collect everything useful
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder (wording proposed by Claude)
- **Decision:** "Collect everything useful, linked to an internal customer ID, with consent where the law requires it. Payment card data stays in Stripe."
- **Why:** The founder places a high value on data for decision-making. This wording replaces the brief's "when unsure, don't store" while staying within GDPR.
- **Rejected:** "As much data as possible" without limits (conflicts with GDPR); strict data minimization (too little insight).

## D-016: Checkout without an account
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Customers buy with just an email; the account is created automatically; login later by email link.
- **Why:** Influencer traffic converts best with the fewest steps.
- **Rejected:** Sign-up before paying.

## D-017: Help center as files in our website
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Help articles are Markdown/MDX files in `content/help/`, shown on our domain.
- **Why:** Best for search traffic and design; the AI assistant reads them directly; changes are versioned.
- **Rejected:** The support tool's help center (helpers edit directly, but less SEO/design control); a separate CMS (another vendor).
- **Revisit when:** Helpers need to edit often without going through Claude.

## D-018: AI first-line support from launch
- **Date:** 2026-10-27 · **Status:** Decided (provider Open, see D-019) · **By:** Founder
- **Decision:** An AI assistant answers customers 24/7 using the help center and the customer's own order/eSIM data (read-only), hands off to a human when unsure or for refunds, and cannot promise refunds.
- **Why:** Covers the night-time gap (travelers landing at night in the founder's time zone) without 24/7 staff.
- **Rejected:** Humans only; AI later.

## D-019: Support tool (and AI provider): Intercom vs. Zendesk
- **Date:** 2026-10-27 · **Status:** Open · **By:** Founder
- **Options:** Intercom (leaning, for integrations) or Zendesk (to re-evaluate). The AI assistant provider (our own Claude-based assistant vs. the tool's built-in AI, e.g. Intercom Fin) is decided together with the tool.
- **Why these:** The founder has experience with both, and outsourcing companies already use them, so no forced switch later.
- **Rejected:** Crisp, Help Scout (cheaper, but would likely mean switching when support is outsourced).
- **Decide before:** Phase 1 support integration.

## D-020: English only at launch, ready for translation
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Launch in English; all text goes through translation files (next-intl).
- **Why:** US-first market, English-only support; adding languages later is content work, not a rebuild.
- **Rejected:** English + Spanish, English + German at launch.

## D-021: Alerts to a team channel plus phone push
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Routine alerts in a Slack or Telegram channel the team can see; critical alerts also push loudly to the owner's phone.
- **Rejected:** Email only; phone push only.

## D-022: Don't roll your own; approved libraries
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Security-critical and solved problems (auth, 2FA, crypto, payments, validation, SQL, rate limiting) always use established libraries from the approved list (architecture §23). New libraries must pass a checklist (real package, popular, maintained, license, substantial). Outside security zones Claude may add them and list them in the PR; inside security zones the owner must approve first.
- **Why:** No human developer reviews the code; hand-written security code is the most common source of serious vulnerabilities.
- **Rejected:** Owner approval for every library (rubber-stamping risk); checklist only (too little oversight in sensitive areas).

## D-023: Boring code
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Mainstream, explicit, well-documented patterns; no custom frameworks or clever abstractions. A future developer should understand any part in an afternoon.
- **Why:** Creative code fails more often and is harder to maintain, especially when maintained only by AI.

## D-024: Contained, noticed, recoverable
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Assume any single part (library, runtime, vendor, key, account) will eventually fail or be compromised; design so the damage is contained, noticed quickly and recoverable. Implemented through separate services, per-service keys, limited database users, money caps, anomaly alerts, reconciliation, backups and runbooks.
- **Why:** AI-generated, non-developer-maintained apps are attractive targets, and platform-level vulnerabilities (e.g. PHP2Shell, Oct 2026) are becoming more common. The goal is general resilience, not defenses against specific exploits.

## D-025: Patch policy
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Normal library updates wait ~7 days after release; security fixes (libraries and the Node runtime) are applied within 24–48 hours. Dependency install scripts are blocked.
- **Why:** Most poisoned releases are caught within days, but known vulnerabilities must not stay open for a week.

## D-026: Pull requests with automatic checks
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Every change goes through a pull request: type checks, lint, tests, vulnerability and secret scans, and an AI code review must pass; staging before production; `main` is protected.
- **Rejected:** Pushing straight to `main` with checks afterwards.

## D-027: Automatic AI security review in security zones
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** Changes touching auth, payments, webhooks, supplier adapters, admin roles, payouts, personal data, secrets or CI get a separate AI security review before merging.
- **Rejected:** Security reviews only on request.

## D-028: Human security audit once revenue justifies it
- **Date:** 2026-10-27 · **Status:** Decided (trigger Open) · **By:** Founder
- **Decision:** Launch on automated checks; buy a professional security audit/penetration test once revenue justifies it.
- **Rejected:** Audit before launch (Claude's recommendation); never.
- **Open:** Concrete trigger (e.g. steady $X/month revenue or before mobile app launch, whichever first).

## D-029: Expo for the mobile app
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder (confirmed from brief)
- **Decision:** Expo (React Native) + EAS cloud builds, Phase 3.
- **Why:** Builds iOS from Windows without a Mac; same language and React as the website.

## D-030: Keep a decision log
- **Date:** 2026-10-27 · **Status:** Decided · **By:** Founder
- **Decision:** This file, separate from `docs/architecture.md`.
- **Why:** The architecture doc describes the current state; this file keeps the history and reasons, so future sessions don't re-argue settled decisions.
- **Rejected:** A section inside the architecture doc; no log.
