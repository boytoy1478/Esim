# Architecture

> **Status:** Draft v1, 2026-10-27. Pre-build.
> **Audience:** Claude Code (the builder) and the founder (non-developer). Written so both can follow it.
> **Precedence:** Where this document and `esim-project-brief.md` disagree, **this document wins** (e.g. hosting is Railway, not Vercel; the browser never talks to the database). Business rules, unit economics and the roadmap still live in the brief.
> **Related:** `docs/decisions.md` (what was decided and why), `esim-project-brief.md`, `esim-wholesaler-briefs.md`, `influencer-discovery-strategy.md`.
> **Keep it current:** any change to the architecture updates this file in the same pull request, and adds an entry to `docs/decisions.md`.

Markers: **[verify]** = fact or price to confirm. **[decision needed]** = open, see §27. **[default]** = Claude's recommended default, change if needed.

---

## 1. What we're building

A brand that sells prepaid travel data eSIMs. Wholesalers issue the eSIMs through their APIs; we own the brand, website, mobile app, customers, pricing, affiliate program and support.

```
                         ┌──────────────── Customers ────────────────┐
                         │  website (web)        mobile app (Phase 3)│
                         └───────┬───────────────────────┬──────────┘
                                 │ HTTPS                 │ HTTPS (/api/v1)
   Team (owner, support,   ┌─────▼──────────────────────▼─────┐
   creator mgr, bookkeeper)│   web service (Next.js, Railway) │◄── Stripe / wholesaler /
          │                └─────┬───────────────────────┬────┘    support webhooks
          │ HTTPS                │ enqueue jobs          │ read/write (limited DB user)
   ┌──────▼───────────────┐      │                       │
   │ admin service        │      │      ┌────────────────▼──────────┐
   │ (Next.js, Railway,   ├──────┼─────►│ Postgres (Supabase, EU)   │
   │  own subdomain, 2FA) │      │      │ our data + job queue      │
   └──────────────────────┘      │      └────────────────▲──────────┘
                                 │                       │
                          ┌──────▼───────────────────────┴──┐
                          │ worker service (Node, Railway)  │── holds the powerful keys
                          │ runs jobs: issue eSIM, refunds, │   (wholesalers, Stripe write,
                          │ sync, reconciliation, perks     │    email, ad conversions)
                          └──────┬──────────────────────────┘
                                 ▼
     Wholesalers · Stripe · Email · Support tool · PostHog · GA4/Meta/TikTok · Alerts
```

**The core idea:** the web and admin services *ask* for things to happen (by writing to the database and enqueuing jobs). The **worker** *does* the risky things (issuing eSIMs, refunds, payouts) and is the only service holding the keys for them. See §14.

---

## 2. Principles

These six principles decide every design question. When something isn't covered here, choose the option that best fits them.

### 2.1 Every vendor is swappable
Anything we don't build, we depend on someone for, so it must be replaceable without a rewrite. We call our own interfaces (`payments.createCheckout()`, `email.send()`, `support.identifyUser()`); vendor SDKs live only in `packages/integrations/src/<category>/<vendor>/`. Vendor webhooks are verified and translated into our own internal events at the edge. Each category has a mock adapter. Full rules: brief §3.0. Details: §6.

### 2.2 Our database is the source of truth
Orders, customers, attributions, commissions, perks, support context and audit logs live in **our** Postgres. A vendor may hold a copy, never the only copy. Vendor IDs (`stripe_customer_id`, `supplier_order_id`) are stored as references next to our own IDs, never used as our primary keys. **The one exception by design:** card and billing details stay in Stripe (§20).

### 2.3 Collect everything useful
Strong analytics, error logging and data collection are core to how the company makes decisions. The working rule:

> **Collect everything useful, linked to an internal customer ID, with consent where the law requires it. Payment card data stays in Stripe.**

In practice: track behavior, funnels, errors, devices, destinations, referring creators and support history; tie it to our internal `customer_id` (never the email) in vendor tools; get consent where required (§15.2); be able to export or delete any customer's data on request (§8.6).

### 2.4 Boring code
- **One language (TypeScript, strict mode) and mainstream, well-documented patterns.** No custom frameworks, no "magic", no clever abstractions that save lines but cost understanding.
- **Explicit beats clever.** Plain functions, obvious names, small files, one way to do each thing.
- **The test:** a developer hired in two years could understand any part in an afternoon.

### 2.5 Don't roll your own
Solved problems use established, well-maintained libraries or services, **above all anything security-related**: authentication, sessions, 2FA, cryptography, signatures, payments, input validation, SQL, rate limiting, sanitization. Hand-writing these is forbidden. The approved list and the rules for adding libraries are in §23.

### 2.6 Contained, noticed, recoverable
> **Assume any single part will eventually fail or be compromised: a library, the runtime, a vendor, a key, an account. Design so that when it happens, the damage is contained, noticed quickly, and recoverable.**

Every design decision is checked against three questions: *If this part were compromised tomorrow, how much damage could it do? How fast would we know? How fast could we recover?*

| | Measures |
|---|---|
| **Contained** | Separate services; keys scoped per service; limited database users; separate admin service; money caps; few dependencies (§14) |
| **Noticed** | Anomaly alerts; daily reconciliation; error tracking; audit log (§16, §10.8) |
| **Recoverable** | Tested backups; key-rotation runbook; swappable vendors; supplier kill switch; fast patching (§24, §23.5) |

---

## 3. Stack

| Layer | Choice | Why | If it goes down / must be replaced |
|---|---|---|---|
| Language | **TypeScript (strict)** everywhere | One language for web, admin, worker, mobile; Claude writes it best | n/a |
| Web + admin framework | **Next.js** | Most common React framework, best AI/tutorial support, shares React with the mobile app | Runs on any Node host |
| Hosting | **Railway** (EU region) | Website, admin, worker and jobs on one host; deploys from GitHub; PR environments | Next.js/Node run anywhere (Render, Fly.io…) |
| Database + customer login | **Supabase** (EU region, Frankfurt), **Pro plan** | Real Postgres (portable), built-in auth incl. 2FA, backups + point-in-time restore | Postgres dump → any Postgres; auth is behind an adapter |
| Job queue | **pg-boss** (queue stored in our Postgres) [default] | No queue vendor; job history in our DB | It's a library; alternative: Graphile Worker |
| Payments | **Stripe Checkout (hosted)** + Stripe Tax + Radar | Industry standard; card data never touches us | Adapter; backup processor on the pre-build checklist |
| Email | **Postmark or Resend** [decision needed at build] | Reliable transactional email | Adapter, SMTP-compatible |
| Analytics | **PostHog (EU cloud)** + GA4, Meta Pixel, TikTok Pixel | Product analytics, funnels, replay, flags; ad platforms build history from day one | Our tracking layer fans out; raw data exportable |
| Error tracking | **PostHog error tracking** (plan B: Sentry) | One tool, errors linked to replays | Adapter; switching ≈ 1 day (+ app release) |
| Support | **Intercom or Zendesk** [decision needed] | Founder knows both; outsourcing firms use them | Adapter; tickets exported regularly |
| Alerts | **Slack or Telegram** channel + phone push | Team sees routine alerts; owner gets critical ones loudly | Adapter |
| Mobile (Phase 3) | **Expo (React Native) + EAS** | Builds iOS in the cloud from Windows | n/a |
| Package manager | **pnpm** | Blocks dependency install scripts by default; supports release-age delay | n/a |

**Regions:** Railway services and Supabase in the EU. Vendors that hold personal data need a GDPR data processing agreement (DPA); track in `docs/vendors.md`.

**Monthly cost estimate:** see §25.

---

## 4. Services & environments

### 4.1 Services (all on Railway)

| Service | What it does | Public? |
|---|---|---|
| `web` | Customer website, checkout start, "My eSIM", help center, customer API for the app (`/api/v1`), webhook receivers | Yes, `yourbrand.com` |
| `admin` | Team admin panel | Yes, but on `admin.yourbrand.com` behind 2FA + an extra access gate (§13.3) |
| `worker` | Runs all background jobs (§12) | **No public URL** |

Postgres (and the job queue inside it) lives in Supabase.

### 4.2 Environments

| Environment | Data & keys | Who uses it |
|---|---|---|
| **Local** | Local/dev database, mock adapters | Claude while building |
| **PR preview** | Temporary Railway environment per pull request, test keys, mock or sandbox suppliers | Automated checks, quick look |
| **Staging** | Own Supabase project, **Stripe test mode**, wholesaler test credits | Every change is tried here before production |
| **Production** | Live keys | Customers |

Staging and production share **nothing**: separate databases, separate keys, separate webhooks. Until launch, production also runs in test mode (brief §6.10).

---

## 5. Code layout

One repository, organized as a **pnpm workspace** (a standard "monorepo": several apps sharing code).

```
apps/
  web/            Next.js: customer website + customer API + webhook receivers
  admin/          Next.js: admin panel
  worker/         Node: job runner (pg-boss)
  mobile/         Expo app (Phase 3)
packages/
  core/           Business logic: orders, pricing, routing, attribution, commissions, perks.
                  Pure TypeScript. Knows only OUR types and interfaces, never a vendor SDK.
  db/             Drizzle schema, migrations, database access, DB roles
  integrations/   Adapters: src/<category>/<vendor>/ (+ src/<category>/mock/)
  events/         Internal event types (payment.succeeded, esim.issued, …)
  config/         Typed config + env var validation (Zod)
  ui/             Shared UI components
content/
  help/           Help center articles (Markdown/MDX), English
docs/             architecture.md, decisions.md, vendors.md, runbooks/
```

**Import rules (enforced by lint):**
- Apps import from packages. Packages never import from apps.
- `core` never imports a vendor SDK or `integrations/<vendor>` directly; it receives adapters through its interfaces.
- Vendor SDKs (`stripe`, wholesaler clients, `posthog-node`…) may only be imported inside `packages/integrations/src/<category>/<vendor>/`.
- Only `worker` (and `admin` for read-only lookups) may load supplier and Stripe-write adapters.

---

## 6. Vendor adapter pattern

| Category | Interface (examples) | Active vendor | Swappability (brief §3.0) |
|---|---|---|---|
| `suppliers` | `syncOffers`, `issueEsim`, `getUsage`, `topUp`, `cancelOrRefund`, `getBalance`, `verifyAndParseWebhook` | eSIM Access (pending final choice) | **A: several live at once, routed per order** (§7) |
| `payments` | `createCheckout`, `refund`, `getPaymentDetails`, `verifyAndParseWebhook` | Stripe | B |
| `tax` | handled inside Stripe Checkout; exports for accounting | Stripe Tax | B |
| `email` | `send(template, to, data)` | Postmark/Resend | B |
| `support` | `identifyUser`, `pushContext`, `createTicket`, `verifyAndParseWebhook` | Intercom/Zendesk | B |
| `analytics` | `track`, `identify`, `alias`, `serverConversion` | PostHog + GA4 + Meta + TikTok (fan-out) | B |
| `errors` | `capture`, `setUser`, `addContext` | PostHog (plan B Sentry) | B |
| `alerts` | `notify(severity, message, context)` | Slack/Telegram + push | B |
| `affiliates` (Phase 1) | `recordConversion`, `syncCodes` | Rewardful/FirstPromoter (brief §4.9) | B, replaced by our own in Phase 2 |
| `auth` | `sendLoginLink`, `verifySession`, `requireMfa` | Supabase Auth | B |
| `payouts` (Phase 2) | `createPayout` | Stripe Connect or PayPal/Wise | B |
| `ai` (support assistant) | `answer(conversation, tools)` | [decision needed] | B |

Rules for every category: an interface in `packages/integrations/src/<category>/types.ts`, a **mock adapter**, shared **contract tests** every real adapter must pass, the active vendor chosen by config, credentials in env vars, and an entry in `docs/vendors.md` (purpose, data held, cost, DPA status, export method, exit plan).

---

## 7. Supplier layer (wholesalers)

Full specification: **brief §3.7**. Summary of what is binding:

- **Products are ours, offers are the supplier's.** A `product` ("Japan 5GB / 30 days, $16") never references a supplier. `supplier_offers` are synced daily from each wholesaler. `product_offers` maps a product to one or more equivalent offers.
- **Routing rules live in the database**, editable in admin with no deploy: scope (product/country/region/global), strategy (`fixed`, `priority`, `cheapest_valid`, `weighted`), constraints (features, health, margin floor), overrides and kill switches.
- **Every order stores which supplier was chosen and why** (rule, strategy, fallback used, cost at the time).
- **An issued eSIM stays with its supplier forever** (`esims.supplier_id`). Usage, top-ups and webhooks always go to that supplier. Routing changes affect new orders only.
- **Customer price never depends on routing.** If no offer meets the margin floor, hide the product and alert.
- **Per-supplier balances** with low-balance alerts, and a **maximum balance target of ~2 weeks of sales** (limits loss if a supplier fails or the balance is stolen).
- Phase 1 builds the interface, mock adapter, one real adapter and the routing tables. Health scoring, automatic failover and weighted routing come in Phase 4.

---

## 8. Data model

### 8.1 Main tables (grouped)
- **Customers & identity:** `customers`, `customer_devices`, `team_members`, `team_roles`, `role_permissions`
- **Catalog:** `products`, `product_prices` (per currency), `suppliers`, `supplier_offers`, `product_offers`, `routing_rules`, `supplier_health`
- **Orders & money:** `orders`, `order_items`, `payments`, `refunds`, `esims`, `topups`, `usage_snapshots`, `fx_rates`
- **Affiliates & creators:** `affiliates`, `affiliate_links`, `promo_codes`, `clicks`, `attributions`, `commission_rates` (with history), `commissions`, `payouts`, `creators` (prospects, see influencer strategy §3.2)
- **Perks:** `influencer_perks`, `perk_grants`, `perk_usage`
- **Plumbing:** `webhook_events` (idempotency), `outbox` / jobs (pg-boss tables), `notifications`, `audit_log`, `analytics_events` (money/lifecycle events we must own), `reconciliation_runs`, `consents`

### 8.2 Conventions
- **IDs:** our own UUIDs as primary keys. Vendor IDs are separate, indexed columns.
- **Money:** always **integer minor units** (cents) **plus a currency column**. Never floating point. Formatting via `Intl.NumberFormat`.
- **Time:** always stored and computed in **UTC** (`timestamptz`). Converted to local time only for display. Daily resets (perks, limits) define their reference time explicitly.
- **Status fields** use explicit enums and only change through the state machines in §9.
- **Every money-relevant change writes to `audit_log`** (who/what, before/after, reason).
- **Migrations:** SQL migrations generated with Drizzle, reviewed in the PR, applied to staging before production. Never edit the production schema by hand.

### 8.3 Currency
- **USD is the default currency.** The schema supports several from day one; adding EUR later is configuration plus prices, not a rebuild.
- Prices are **set per currency** in `product_prices` (e.g. $24.99, later €22.99), never auto-converted.
- Each order stores: price paid (amount + currency), wholesale cost (amount + supplier currency), and a **EUR value** for German bookkeeping (from `fx_rates`, daily rate).
- Stripe pays out to the founder's **Wise USD account**; wholesalers are paid in USD.

### 8.4 What we never store
Card numbers, CVC, expiry, full payment method details (stay in Stripe). Billing address only if VAT rules require it [verify with Steuerberater]. Passwords (login is passwordless).

### 8.5 Sensitive fields
- **eSIM activation codes / LPA strings / QR data are treated like passwords:** anyone holding one can install the eSIM. Encrypted at rest (Supabase column encryption or app-level with a managed key [default]), never logged, never sent to analytics/error tools, shown only to the owning customer and to roles with permission, masked for outsourced support.
- Emails and names are personal data: sent to vendors only where needed (email provider, support tool), never to analytics as identifiers (hashed for ad conversions only with consent, §15.3).

### 8.6 Deletion & retention
- A customer can delete their account (web and in-app; Apple requires in-app deletion). Deletion **anonymizes** personal data and pseudonymizes analytics, but **keeps order and invoice records** for the legal retention period (Germany ~10 years, GoBD [verify with Steuerberater]).
- Data export per customer (GDPR access request) as a single admin action.
- Retention periods per data type [decision needed: lawyer/Steuerberater].

---

## 9. State machines

All status changes go through one function per entity that checks the transition is allowed and writes `audit_log`. Anything else is a bug.

### 9.1 Order
```
checkout_open ──► paid ──► fulfilling ──► fulfilled ──► (refund_requested ──► refunded | partially_refunded)
      │                        │
      ▼                        ▼ (retries exhausted)
  expired               fulfillment_failed ──► refund_requested ──► refunded
```
- `checkout_open → paid` only via a verified Stripe webhook, never via the browser's redirect to the "thank you" page.
- `paid → fulfilling` enqueues exactly one issuance job (unique per order).
- `fulfillment_failed` alerts the owner and emails the customer ("we're on it"); auto-refund after a timeout (brief §6.2).

### 9.2 eSIM
`issuing → issued → installed → active → depleted | expired`, plus `suspended`, `cancelled` (unused, refunded at supplier). Driven by supplier webhooks and the usage sync job.

### 9.3 Commission
`pending → approved → paid`, with `reversed` from any state on refund/chargeback (paid ones become a negative balance). Approval only after the hold period (brief §4.6).

### 9.4 Perk grant
`active → paused → active`, `revoked`. Eligibility evaluated per cycle (brief §5.4).

---

## 10. Key flows

### 10.1 Purchase → eSIM
1. Customer picks a plan; compatibility + "carrier-unlocked?" check shown.
2. `web` creates the order (`checkout_open`) with the price **from our DB**, and a Stripe Checkout session (restricted key: create sessions only). Email only, no account needed (§13.1).
3. Stripe sends `checkout.session.completed` → `web` verifies the signature, stores the event in `webhook_events` (duplicate → ignore), marks the order `paid`, enqueues `issue_esim(order_id)`, returns 200 **immediately**. No supplier call inside the webhook.
4. `worker` runs `issue_esim`: routes the order (§7), calls the supplier with an idempotency key, stores the eSIM, marks `fulfilled`, emits `esim.issued`.
5. Notification module sends the email (QR, iOS install link, manual code, install guide, offline-readable troubleshooting).
6. Customer account is created/linked by email; "My eSIM" page shows install, usage, expiry, top-up.

### 10.2 Issuance failure
Retries with exponential backoff (e.g. 1 min → 5 → 15 → 60). After N failures: `fulfillment_failed`, critical alert, customer email, try the next supplier if the routing rule allows, auto-refund after the timeout.

### 10.3 Top-up
Same as purchase, but the job goes to the eSIM's **original supplier** (`esims.supplier_id`).

### 10.4 Usage & low-data
Supplier webhooks (e.g. ≤100MB left, 1 day left) + a periodic usage sync job → `usage_snapshots` → events → "running low, top up?" email/push.

### 10.5 Refund
Admin or the customer's self-serve button creates a `refund_requested` record (checked against role limits, §13.3) → `worker` executes it with Stripe (and the supplier, if the eSIM is unused) → `refunded` → commission reversed (§9.3).

### 10.6 Affiliate attribution
Click on `/r/<creator>` → first-party cookie (consent rules, §15.2) + server-side click record. At checkout: **code beats link**, else last click within 30 days. The reason is stored on the order. Phase 1 mirrors the bought affiliate tool's data into our tables; Phase 2 replaces it.

### 10.7 Perk daily job
Runs once per perk per day, keyed by `(perk_id, date)` so running twice never grants twice. Checks region, cap, monthly cost ceiling and eligibility.

### 10.8 Daily reconciliation
Nightly job compares:
- Stripe successful payments ↔ our `paid`/`fulfilled` orders
- our fulfilled orders ↔ supplier orders
- supplier balance change ↔ sum of recorded costs
- refunds/chargebacks ↔ reversed commissions

Any mismatch → alert with the list. Results stored in `reconciliation_runs`.

---

## 11. Webhooks & internal events

- **Receivers** at `/api/webhooks/<vendor>` (e.g. `/api/webhooks/stripe`, `/api/webhooks/suppliers/<supplier>`, `/api/webhooks/support`).
- Every receiver: **verify signature** (official SDK method) → **store** in `webhook_events` keyed by the vendor's event ID (unique constraint = idempotency) → **translate** into an internal event → **enqueue** follow-up jobs → return 200. Receivers do no slow work.
- **Internal events** (`packages/events`): `payment.succeeded`, `payment.refunded`, `dispute.created`, `esim.issued`, `esim.installed`, `esim.usage_low`, `esim.expiring`, `order.fulfillment_failed`, `commission.reversed`, `ticket.created`, … Business logic, notifications and analytics subscribe to these, never to raw vendor payloads.
- Processing is **at-least-once**, so every handler must be safe to run twice.

---

## 12. Background jobs

Run by `worker` using **pg-boss** (jobs stored in Postgres).

| Job | Trigger |
|---|---|
| `issue_esim`, `topup_esim` | After payment |
| `execute_refund` | Refund requested |
| `send_notification` | Internal events |
| `sync_supplier_offers` | Daily per supplier |
| `sync_usage` | Periodic for active eSIMs |
| `check_supplier_balances` | Hourly |
| `reconcile` | Nightly |
| `perk_daily_grant`, `perk_cycle_evaluate` | Daily / per cycle |
| `approve_commissions` | Daily (after hold period) |
| `server_conversions` | After purchase (ad platforms, consent permitting) |
| `fx_rates_update` | Daily |
| `support_context_push` | On customer/order changes |

Rules: every job is **idempotent** (unique keys per order/date), has retries with backoff and a dead-letter state that alerts. The worker writes a **heartbeat** every minute; if it stops, a critical alert fires (a dead worker means no eSIMs are issued).

---

## 13. Identity, login & roles

### 13.1 Customers
- **No account needed to buy:** email at checkout; the account is created automatically after payment.
- **Passwordless login** (email magic link or one-time code) via Supabase Auth. Works on airport Wi-Fi when the eSIM doesn't.
- Our `customers` table is the source of truth; the Supabase auth user ID is a reference.

### 13.2 Team members
- Separate `team_members` (never mixed with customers). Login via Supabase Auth with **mandatory 2FA** (authenticator app/passkey; never SMS).
- Sessions expire; re-authentication required for sensitive actions (refunds above a limit, payouts, role changes, data export).

### 13.3 Admin service & roles
- Runs as its own service on `admin.yourbrand.com`, with 2FA and an **extra access gate** in front (e.g. Cloudflare Access or IP allowlist [decision needed at build]).
- **Permission checks on the server for every action**, based on `role_permissions`. Every action written to `audit_log`.

| Role | Can | Cannot |
|---|---|---|
| **Owner** | Everything | n/a |
| **Support agent** | View customers, orders, eSIM status; resend eSIM; refund up to a limit/day | Finances, payouts, settings, exports, roles |
| **Creator/outreach manager** | Creator pipeline, affiliate codes, perks; sales per creator | Customer personal details, refunds, finances |
| **Outsourced support** | Like support agent, but emails and activation codes partly masked, lower refund limit | Exports, unmasked personal data, finances, settings |
| **Bookkeeper** | Read-only finance reports and accounting exports | Any change, customer contact |

Limits (refund amounts, daily caps) are configuration, changeable by the owner.

---

## 14. Security architecture

### 14.1 Keys per service (least privilege)
| Key | `web` | `admin` | `worker` |
|---|---|---|---|
| Stripe | Restricted: create Checkout sessions | Restricted: read-only | Full (refunds, payouts) |
| Wholesaler API keys | none | none | yes |
| Email provider | none | none | yes |
| Support tool secret | Identity-verification signing only | Read | Context push |
| Ad platform server-side tokens | none | none | yes |
| Database | `app_web` user | `app_admin` user | `app_worker` user |

Separate keys for staging and production. All secrets in Railway environment variables, never in the repo, never sent to the browser or app.

### 14.2 Database users
- `app_web`: read catalog; create orders/clicks/consents; read only the logged-in customer's own data (enforced in code); **cannot** mark orders fulfilled, issue refunds, or touch commissions.
- `app_admin`: reads broadly; writes only through requests (refund requests, rule changes) that the worker executes; permission-checked per role.
- `app_worker`: writes orders, eSIMs, payments, commissions.
- Supabase's browser access (public API) is **disabled or deny-all** as a safety net, since the browser never talks to the database.

### 14.3 Money caps (limit what any failure can cost)
- Supplier balance kept at **≤ ~2 weeks of sales**.
- **Issuance caps** per hour and per day (global and per customer/card/IP); hitting one pauses sales and sends a critical alert.
- **Refund limits** per role per day.
- **Perk cost ceiling** per perk per month (brief §5.7).
- Payout batches require owner approval with re-authentication.

### 14.4 Abuse protection
- **Rate limits** on public endpoints: checkout creation, login links, compatibility check, contact form, webhook receivers (library, §23).
- Stripe Radar, velocity rules and 3-D Secure for risky orders (brief §6.5).
- All input validated with Zod at the edge; prices and amounts always computed server-side.
- Customer messages (support, AI assistant) and creator content are **untrusted data**: never executed or treated as instructions.

### 14.5 Security zones
Changes touching these areas get an **automatic AI security review** before merge, a plain-language summary for the owner, and need the owner's OK for any new library (§23.4):

`auth`, `payments`, `webhooks`, `suppliers` adapters, `admin` roles/permissions, `payouts`/commissions, anything reading or writing personal data or activation codes, secrets/config, CI/CD and dependency settings.

### 14.6 Founder accounts (the biggest target)
GitHub, Stripe, Railway, Supabase, domain registrar, DNS, email, Wise, support tool: **passkeys or hardware keys** (never SMS 2FA), unique passwords in a password manager, emergency access for one trusted person (§24.3). Whoever controls these controls everything.

### 14.7 External audit
A paid human security review/penetration test **once revenue justifies it**. Trigger: [decision needed: e.g. steady $X/month revenue or before the mobile app launch, whichever comes first].

---

## 15. Analytics & tracking

### 15.1 Tracking layer
- One internal call: `analytics.track(event, properties)` / `analytics.identify(customer_id)`. The layer **fans out** to PostHog, GA4, Meta Pixel and TikTok Pixel, respecting consent.
- **Event naming:** `object_action`, lowercase (`checkout_started`, `esim_installed`, `topup_purchased`). Every event and its properties are listed in `docs/tracking-plan.md` (created in Phase 1). Untracked events don't exist.
- **Money and lifecycle events** (purchase, refund, fulfillment, attribution) are recorded in **our** `analytics_events` table from the server, not only in vendor tools, so the numbers that matter never depend on a browser or an ad blocker.
- Client-side (browser/app) tracking for behavior; server-side tracking for anything money-related.

### 15.2 Consent (by region)
- **US visitors:** full analytics and ad pixels by default, with a visible opt-out ("Do not sell or share my info") [verify per state, e.g. California].
- **EU/UK visitors:** consent banner. Before consent: cookieless, server-side basics only. After consent: full tracking per accepted category (analytics, marketing).
- **Unknown location → EU rules** (safe default). Location from IP geolocation [decision needed at build: library vs. edge header].
- Consent choices stored in `consents`, with timestamp and version of the banner text.
- The affiliate cookie's legal category (necessary vs. consent-required) [decision needed: lawyer]. "Code beats link" keeps attribution working either way.

### 15.3 Ad platforms
- GA4, Meta Pixel and TikTok Pixel installed from day one (consent-gated in the EU), so ad platforms build conversion history before any ads run.
- **Server-side conversions** (Meta Conversions API, Google Enhanced Conversions, TikTok Events API) sent by the worker after purchase. Email/phone only **hashed** and only with marketing consent where required.

### 15.4 Identity & profiles
- Anonymous visitor ID → linked to `customer_id` at purchase or login. Analytics, errors, support and orders all link to that one internal ID.
- Vendor tools receive the internal ID, not the email (exceptions: email provider, support tool, hashed ad conversions).

### 15.5 Session replay
- PostHog session replay on the website and app, **all text inputs and marked personal fields masked**, activation codes/QR areas blocked. EU visitors only after consent. **Never** on the admin service.

---

## 16. Errors, logging & alerts

### 16.1 Error tracking
- All errors go through `errors.capture()` → PostHog (adapter; Sentry is the documented plan B, switching ≈ 1 day plus a mobile app release).
- Every error carries context: request ID, `customer_id`, `order_id`, service, release version. Never activation codes, card data or secrets.

### 16.2 Customer-facing errors
- Customers see a friendly message plus a **short reference code** (e.g. `ERR-7F3K2`) that support can look up to find the exact error, request and customer.
- No stack traces or internal details ever shown to customers.

### 16.3 Logging
- Structured JSON logs (pino) with a **request ID** carried across web → job → worker → vendor call, so one order's full story can be followed.
- Secrets, activation codes and personal data are redacted by the logger configuration.

### 16.4 Alerts
| Severity | Where | Examples |
|---|---|---|
| **Critical** | Team channel **+ loud phone push to the owner** | Issuance failing, worker heartbeat lost, supplier balance near zero, issuance cap hit, reconciliation mismatch on money, admin login from a new country |
| **Warning** | Team channel | Error spike, supplier sync changes, low balance, refund spike, dispute received |
| **Info** | Daily digest | Sales summary, new creators, jobs completed |

Alerts are deduplicated so one problem doesn't send 500 messages.

### 16.5 Health
Health-check endpoint per service; Railway restarts unhealthy services; uptime monitor on the public site and checkout.

---

## 17. Support

- **Tool:** Intercom or Zendesk [decision needed], behind the `support` adapter.
- **Contact requires login** (help articles stay public); fallback guest form verified by order number + email (brief §7.1).
- **Identity verification:** our server signs the customer's internal ID so the widget can't be spoofed.
- **Context pushed to the tool** (never payment data): plans, order IDs, eSIM status, supplier, activation time, data used/left, expiry, device, web/app, referring creator, plus an **admin deep link**.
- Tickets tagged by type and country/supplier; tags feed supplier health (§7).
- **AI first-line assistant from launch** [provider decision tied to the support tool: our own Claude-based assistant or the tool's built-in AI]. Architecture is the same either way:
  - Reads the help center (`content/help`) and the **logged-in customer's own** order/eSIM/usage data through a scoped, read-only API.
  - Cannot issue refunds, change data, or promise anything outside policy; can create a handoff to a human with a summary.
  - Always offers a human; every conversation logged and reviewable.
  - Treats customer messages as untrusted input.
- **Outsourced team (later):** works in the support tool with the restricted role (§13.3). Data leaving the EU needs GDPR transfer contracts [lawyer].

---

## 18. Notifications

One `notifications` module subscribes to internal events and decides what to send, on which channel, in which language. Email via the `email` adapter, templates in **React Email**; push (Phase 3) via Expo. Every sent message recorded in `notifications`. Transactional emails (order, install, usage) are separate from marketing emails, which require consent.

---

## 19. Content & languages

- **English only at launch**, but all user-facing text goes through **next-intl** message files, so adding a language is translation work, not a rebuild.
- **Help center:** Markdown/MDX files in `content/help/`, rendered on our domain (best for search traffic, design control, and readable by the AI assistant). Helpers suggest edits; changes go through a pull request.
- Destination/plan pages generated from the catalog in our DB.

---

## 20. Payments, tax & invoices

- Stripe Checkout (hosted): card data never touches our servers. Apple Pay/Google Pay via Stripe.
- **Stripe Tax** calculates VAT; country evidence (billing country + IP/card country) stored on the order. OSS returns are filed by us/advisor, not by Stripe.
- **Promo codes are defined in our DB** and synced to Stripe promotion codes.
- **Invoices** for customers that satisfy German requirements [decision needed: Stripe invoices vs. a separate invoicing tool; confirm with Steuerberater].
- **Accounting export** for the Steuerberater (orders, refunds, VAT per country, EUR values, fees) [decision needed: format, e.g. DATEV-compatible CSV].
- Backup payment processor: on the pre-build checklist (brief §8). Also ask a merchant-of-record provider (Paddle, Lemon Squeezy) whether they accept eSIMs; a yes would change the tax setup.

---

## 21. Mobile app (Phase 3)

- Thin client on the same backend via a **versioned API** (`/api/v1`). Breaking changes go to `/api/v2`; old versions keep working until a minimum supported version is enforced.
- The app checks a **minimum supported version** at startup and asks users to update when needed.
- External payments via Stripe (Apple guideline 3.1.3(e)), in-app account deletion, deep links carrying affiliate refs. Details: brief §3.6, §10 Phase 3.

---

## 22. Workflow: from change to production

1. Every change is made on a branch and opened as a **pull request**.
2. **Automatic checks must pass:** TypeScript type check, lint (including import rules from §5), unit + contract tests, money-path tests (brief §6.8), dependency vulnerability scan, secret scan, and an **AI code review**. Changes in security zones also get the **AI security review** (§14.5).
3. PR deploys to a preview environment; after merge to `main`, deploys to **staging**.
4. Promotion to **production** after checks on staging (database migrations applied to staging first).
5. Every PR description ends with a plain-language summary for the owner and a "how could this lose money?" note for money-related changes.

`main` is protected: no direct pushes, no merging with failing checks.

---

## 23. How the code is written

### 23.1 Principles
Boring code (§2.4), don't roll your own (§2.5), contained/noticed/recoverable (§2.6).

### 23.2 Approved libraries & services
| Job | Use |
|---|---|
| Customer & team login, sessions, 2FA | **Supabase Auth** (`@supabase/ssr`) |
| Payments, refunds, webhook signatures | **`stripe`** official SDK |
| Input & config validation | **Zod** |
| Database queries & migrations | **Drizzle ORM** (parameterized queries; no hand-built SQL strings) |
| Job queue | **pg-boss** |
| Rate limiting | **rate-limiter-flexible** (Postgres store) [default] |
| Emails | **React Email** + the email adapter |
| Translations | **next-intl** |
| Forms | **React Hook Form** + Zod |
| UI | **Tailwind CSS** + **shadcn/ui** |
| Dates | **date-fns** (UTC) |
| Logging | **pino** |
| Analytics & errors | Official **PostHog** SDKs (`posthog-js`, `posthog-node`, `posthog-react-native`) |
| Testing | **Vitest** (unit/contract), **Playwright** (end-to-end) |
| Mobile | **Expo** + EAS |
| Supplier APIs | The supplier's official SDK if one exists and passes the checklist; otherwise a thin client in its adapter using `fetch` |
| Encryption of sensitive fields | Supabase/Postgres column encryption or Node's built-in `crypto` via a reviewed helper; no custom algorithms |

### 23.3 Checklist before adding any new library
1. It **really exists** under exactly that name (check the npm page; AI tools sometimes invent package names that attackers then register).
2. Widely used (meaningful weekly downloads) and **actively maintained** (recent releases, issues answered).
3. License allows commercial use (MIT, Apache-2.0, BSD, ISC).
4. It does something substantial; no packages for things that take a few lines.
5. Not overlapping with a library already on the approved list.
6. Named in the PR description with one line on why.

### 23.4 Who approves
- **Outside security zones:** Claude may add a library that passes the checklist; it's listed in the PR summary.
- **Inside security zones (§14.5):** Claude explains the library in plain words and **waits for the owner's yes** before adding it.
- Approved additions go into the table in §23.2.

### 23.5 Updates & patching
- Versions locked via `pnpm-lock.yaml`; Node version pinned in one place.
- Automated update PRs (Dependabot or Renovate) for libraries **and the Node runtime**.
- **Normal updates wait ~7 days after release** (most poisoned releases are caught within days).
- **Security fixes skip the wait: applied within 24–48 hours**, including the runtime.
- Dependency install scripts are blocked (pnpm default); exceptions listed explicitly.
- Owner subscribes to Node.js security announcements and Railway/Supabase status pages.

### 23.6 Code conventions
- TypeScript strict; no `any` without a comment explaining why.
- Business logic in `packages/core` as plain, testable functions; apps stay thin.
- One way to do each thing: errors via `errors.capture`, events via `analytics.track`, config via `packages/config`, money via the money helpers, status changes via the state-machine functions.
- Tests for every money path; contract tests for every adapter.

---

## 24. Backups, recovery & continuity

### 24.1 Backups
- Supabase Pro daily backups + **point-in-time recovery** on production.
- Weekly logical export (`pg_dump`) to separate storage outside Supabase [default].
- **Restore test every quarter** into staging, documented in `docs/runbooks/`.

### 24.2 Runbooks (`docs/runbooks/`)
Short step-by-step guides for: key leaked (rotate each key in minutes), supplier down, Stripe account frozen, worker dead, database restore, admin account compromised, chargeback spike.

### 24.3 If the founder is unavailable
- Every account and key in a **password manager**, with **emergency access** for one trusted person.
- A one-page "keep things running for a week" note (pause sales, answer support, who to call).

---

## 25. Monthly running costs (estimate)

| Item | Approx. / month |
|---|---|
| Railway (web, admin, worker; staging + prod) | ~$15–50 [verify] |
| Supabase Pro (prod) + staging project | ~$25–40 [verify] |
| PostHog (EU) | $0–50 at start [verify] |
| Support tool: Intercom or Zendesk | ~$20–100+ per seat; Intercom AI charged per resolution [verify] |
| AI assistant (if our own, Claude API) | ~$10–50 [estimate] |
| Email provider | ~$0–15 |
| Alerts (Slack/Telegram), GitHub | ~$0 |
| Password manager | ~$5 |
| Domain, DNS | ~$2 |
| **Infrastructure + tools total** | **~$150–400** (up from the brief's $100–300) |

Company, accountant, app store and legal costs: brief §2.4.

---

## 26. What gets built when

| Phase (brief §10) | Architecture scope |
|---|---|
| **0: No-code validation** | None of this; wholesaler's white-label store |
| **1: Backend + website** | Monorepo, `web` + `admin` + `worker`, Supabase, staging, CI + checks, supplier layer (interface, mock, primary adapter, routing tables), Stripe, state machines, webhooks, jobs, notifications, help center, analytics + consent + pixels, errors, alerts, reconciliation, roles (owner + support at least), AI support assistant, support tool, backups, runbooks; affiliate tool behind adapter |
| **2: Own affiliate system + perks** | Attribution, commissions, payouts, creator manager role, `creators` pipeline, perks |
| **3: Mobile app** | `apps/mobile`, `/api/v1` hardening, push notifications |
| **4: Resilience & growth** | Second+ supplier adapters, health scoring, automatic failover, weighted routing, multi-currency pricing |

---

## 27. Open questions

| # | Question | Owner |
|---|---|---|
| 1 | Primary wholesaler (eSIM Access pending tests) and backup | Founder |
| 2 | Support tool: Intercom vs. Zendesk | Founder |
| 3 | AI assistant: our own (Claude-based) vs. the support tool's built-in AI | Founder, with #2 |
| 4 | Email provider: Postmark vs. Resend | Claude at build |
| 5 | Admin extra access gate (Cloudflare Access, IP allowlist, …) | Claude at build |
| 6 | IP geolocation method for consent rules | Claude at build |
| 7 | Affiliate cookie: necessary or consent-required? US opt-out rules per state | Lawyer |
| 8 | Data retention periods; GoBD vs. GDPR deletion | Lawyer + Steuerberater |
| 9 | Customer invoices: Stripe invoices or separate tool | Steuerberater |
| 10 | Accounting export format (e.g. DATEV) | Steuerberater |
| 11 | GDPR transfer contracts for outsourced support outside the EU | Lawyer (later) |
| 12 | Merchant of record (Paddle/Lemon Squeezy): would they accept eSIMs? | Founder |
| 13 | External security audit trigger (revenue amount) | Founder |
| 14 | Business defaults from brief §11 (affiliate, perks, refund window, launch destinations, unlimited plans) | Founder |
