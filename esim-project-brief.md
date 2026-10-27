# eSIM Reseller — Project Brief for Claude Code

> **Audience:** Claude Code (the builder) and the founder (non-developer, building everything through Claude Code).
> **Status:** Pre-build. Numbers marked **[verify]** are research benchmarks from Oct 2026, not quotes. Items marked **[assumption]** are working guesses to confirm.
> **Companion files:** `esim-wholesaler-briefs.md` (wholesaler research), `influencer-discovery-strategy.md` (finding creators).

---

## 1. Business summary

| | |
|---|---|
| **What we sell** | Prepaid **data-only travel eSIMs** (single-country, regional, global), plus top-ups |
| **Model** | **API reseller.** A wholesaler (SM-DP+ operator) issues eSIM profiles through an API; we add the brand, storefront, app, support, and markup. We hold no telecom network, IMSIs, or SM-DP+. |
| **Customers** | Travelers (broad), reached mainly through influencers and affiliates |
| **Acquisition** | **Affiliate- and influencer-led. No Google Ads.** Per-creator codes/links, commissions, and influencer perk eSIMs (§5) |
| **Team** | One founder: builds everything with Claude Code, writes the help center, handles support (not 24/7) |
| **Revenue** | Markup on data: retail ≈ 2.5–4× wholesale per GB |

**Why the reseller model works:** the profile belongs to one "home" operator (often multi-IMSI) with roaming deals worldwide. One profile covers many countries, and each "plan" is just a billing rule (allowed countries, GB cap, validity), so new plans cost almost nothing to add.

**Out of scope:** running our own SM-DP+, GSMA certification, buying IMSI ranges, carrier negotiations, and in-app direct eSIM install (needs a carrier entitlement from Apple that resellers generally don't get).

---

## 2. Unit economics (reference numbers)

### 2.1 Retail benchmarks (big brands, 2026) **[verify]**

| Plan | Price | Per GB |
|---|---|---|
| Japan 1GB (Airalo) | ~$4 | $4.00 |
| Japan 5GB (Airalo) | ~$16 | $3.20 |
| USA 1GB (Saily) | $3.99 | $3.99 |
| USA 5GB/30d (Airalo/Saily) | $13.50–13.99 | ~$2.75 |
| Europe 10GB/30d (Nomad / Airalo / Saily) | $23 / $31 / $35.99 | $2.30–3.60 |
| "Unlimited" 7d (Holafly, EU/USA) | ~$27–30 | slowed after a few GB/day |

Rule of thumb: **$3–4/GB on small plans, $2.30–3.60/GB on larger ones.**

### 2.2 Wholesale benchmarks **[verify with chosen wholesaler]**

| Region | $/GB |
|---|---|
| Western Europe | ~$0.60–1.10 |
| North America | ~$0.80–1.40 |
| Examples from one price list | USA $0.97, UK $0.61, Canada $2.56 |
| Volume tiers | ~15–25% off around 500 activations/mo; ~$0.50–0.85/GB at 10k+/mo |

**Planning assumption for a new reseller: ~$1–1.50/GB** in popular destinations. Canada, parts of Asia, LatAm, and regional/global bundles cost more.

### 2.3 Worked margin: Europe 10GB at $25 (wholesale ≈ $9)

| | Non-EU customer | EU customer (VAT) |
|---|---|---|
| Price | $25.00 | $25.00 |
| Affiliate discount code (10%) | −$2.50 | −$2.50 |
| VAT (~20%, included in price) | — | −$3.75 |
| Card fees (~3% + $0.30) | −$1.00 | −$1.00 |
| Refunds / failed installs / fraud reserve (~5%) | −$1.10 | −$1.10 |
| Wholesale data | −$9.00 | −$9.00 |
| **Contribution per order** | **~$11.40** | **~$7.65** |

Other per-order costs: **Stripe dispute fee ~$15** (charged win or lose), **FX ~1–2%**, income/corporate tax on profit.
Small plans are weak: a $3.99 1GB plan leaves **~$2**.

### 2.4 Fixed costs (~$100–300/month) **[assumption: depends on country]**

| Item | Approx. |
|---|---|
| Company + accountant/bookkeeping | ~$50–200/mo (biggest variable) |
| Apple Developer $99/yr + Google Play $25 one-time | ~$10/mo |
| Domain + transactional email | ~$10–20/mo |
| Affiliate tool (if bought, §4.9) | ~$0–50/mo |
| Test eSIMs for QA | ~$10–30/mo |
| ToS/privacy lawyer review | one-time, a few hundred $ |
| Hosting (Vercel/Supabase free/low tiers) | ~$0–25/mo at start |

Break-even on fixed costs: **~$200/mo ÷ ~$9 ≈ 22 orders/month.**

### 2.5 Acquisition cost targets

- **Break-even CAC ≈ contribution per order × orders per customer.** One order: ~$7–11. At ~1.5 orders (repeat trips/top-ups): ~$11–17.
- **Target CAC: ≤ $5–7 per customer.**
- **Default affiliate payout: $3–5 per sale** (or the % equivalent). Pay-per-sale deals can't lose money; flat-fee sponsorships can.
- A $500 flat sponsorship must produce **~70–100 purchases** to pay for itself.
- **Levers:** push 5–20GB plans and bundles (order size matters more than markup), avoid unlimited plans at launch, and don't race to undercut the big brands.

---

## 3. Architecture & stack

### 3.0 Core design principle: every external vendor is swappable

**Anything we don't build ourselves, we depend on someone else for, so it must be replaceable without a rewrite.** This applies to every integration: wholesalers, payments, tax, email, support, affiliate tracking, auth, analytics, monitoring, hosting. Vendors raise prices, change terms, have outages, freeze accounts or shut down; switching should be a contained project, not an emergency.

**Rules (apply to every vendor)**
1. **Adapter per vendor category.** The app calls our own interface (`payments.createCheckout()`, `email.send()`, `support.identifyUser()`, `affiliates.recordConversion()`, …), never the vendor SDK directly. Vendor SDK imports live only inside `/integrations/<category>/<vendor>/`.
2. **Our own types and events.** Vendor webhooks are verified and translated into internal events (`payment.succeeded`, `refund.created`, `esim.activated`, `ticket.created`) at the edge. Business logic only handles internal events.
3. **We own the data that matters.** Orders, customers, attributions, commissions, perk grants, support-relevant context and audit logs live in **our** database. A vendor may hold a copy, never the only copy. (Exception by design: card/billing details stay in the payment processor, §7.1 — we keep the IDs.)
4. **Vendor IDs are references, not keys.** Store `stripe_customer_id`, `supplier_order_id` etc. as columns next to our own IDs; never use a vendor ID as our primary key.
5. **Config, not code.** Which vendor is active per category is set by environment/config (and per-destination routing for wholesalers). Credentials per vendor in env vars.
6. **Mock adapter for every category** for tests and local dev. Shared contract tests every real adapter must pass.
7. **Prefer open standards and exportable data:** Postgres (not a proprietary DB), standard SQL migrations, SMTP-compatible email, OAuth/OIDC-style auth, data exports on a schedule.
8. **Vendor register** (`docs/vendors.md`): per vendor — purpose, adapter path, data it holds, cost, DPA status (GDPR), how to export data, **exit plan** and rough switch effort, alternative candidates.

**Swappability levels (don't over-engineer)**
| Level | What it means | Applies to |
|---|---|---|
| **A. Multi-active with routing** | Several vendors live at once, chosen per order by rules, automatic failover | **Wholesalers** (§3.7) |
| **B. Swappable via adapter** | One active vendor; switching = write a new adapter + config change + data migration | Payments, tax, email, support, affiliate tool, SMS/push, analytics, error monitoring |
| **C. Portable by standards** | No adapter needed; portability comes from standard tech | Hosting (Next.js runs on any Node host), database (Postgres), DNS/domain |

Build **the interface and a mock** for each level-B category from the start; build a **second real adapter only when needed**.

**Known hard spots (be honest about these)**
- **Payments:** saved payment methods and subscriptions don't move freely between processors (Stripe can export card data to another PCI-compliant processor on request [verify]). Stripe-specific features (Checkout, Radar, Tax, promotion codes) need equivalents elsewhere. Keep the adapter thin and keep promo codes and prices defined in **our** DB, synced to Stripe.
- **Auth:** if Supabase Auth is used, keep our own `customers` table as the source of truth and treat the auth provider's user ID as a reference; passwordless login (§7.1) means no password hashes need migrating.
- **Affiliate tool (Rewardful etc.):** record every attribution and commission in our DB too (via its webhooks/API), so leaving the tool doesn't lose history (§4.9).
- **Support tool:** export tickets regularly; customer context is pushed from our DB, so a new tool can be fed the same way.

### 3.1 Build order
1. **Backend + website** (money logic lives here)
2. **Affiliate + influencer system** (on the same backend)
3. **Expo mobile app** (thin client on the same backend)

Influencer traffic goes to the **website**: link → pay converts better than link → install app → sign up → pay. The app is for repeat customers and top-ups.

### 3.2 Stack (recommended; equivalents fine — every vendor sits behind an adapter, §3.0)

| Layer | Choice | Notes |
|---|---|---|
| Web + API | **Next.js on Vercel** | Server routes for webhooks and wholesaler calls |
| DB + auth | **Supabase** (Postgres + Auth) | Row-level security on customer data |
| Payments | **Stripe Checkout (hosted)** | No card data on our servers; Radar, receipts, refunds built in |
| Tax | **Stripe Tax** (or accountant-defined VAT setup) | EU VAT on digital/telecom; confirm rules with accountant |
| Promo codes | **Stripe promotion codes** | Mapped 1:1 to affiliates |
| Email | **Postmark or Resend** | SPF/DKIM/DMARC on our domain |
| eSIM issuing | **Wholesaler APIs behind a supplier adapter layer** (§3.7) | Multiple wholesalers from day one; per-destination routing; hot-swappable |
| Jobs/scheduling | Vercel Cron / Supabase scheduled functions / queue | Retries, daily influencer top-ups, balance checks |
| Monitoring | Error tracking (e.g. Sentry) + alerts to founder (email/push) | |
| Mobile | **Expo (React Native) + EAS cloud builds** | Founder is on **Windows**; EAS builds iOS without a Mac |

### 3.3 Core data model (starting point)
`customers`, `products` (what the customer buys: destination, GB, validity, our price — supplier-independent), `suppliers`, `supplier_offers` (each wholesaler's packages, synced: cost, countries, features), `product_offers` (which offers can fulfil which product), `routing_rules`, `supplier_health` (§3.7), `orders` (incl. chosen supplier, routing reason, actual cost), `payments` (Stripe IDs), `esims` (**supplier_id**, supplier order/ICCID refs, status, activation code/LPA string, install links), `usage_snapshots`, `topups`, `affiliates`, `affiliate_links`, `promo_codes`, `clicks`, `attributions`, `commissions`, `payouts`, `influencer_perks`, `perk_grants`, `webhook_events` (for idempotency), `refunds`, `audit_log`.

### 3.4 Customer flow
1. Browse plans by destination → 2. Stripe Checkout (with optional code) → 3. `checkout.session.completed` webhook → 4. **route the order to a supplier (§3.7) and issue the eSIM exactly once** → 5. email with QR + iOS install link + manual code + install guide → 6. "My eSIM" page: install, usage/data left, expiry, top-up.

### 3.5 eSIM install methods
- **iOS (17.4+):** Apple's universal eSIM setup link (opens the native install flow) + QR fallback.
- **Android:** QR code, or a copyable activation code / SM-DP+ address. Direct-link support varies by phone maker **[verify per device family]**.
- Always show a device compatibility check and "is your phone carrier-unlocked?" before payment.

### 3.6 Mobile app rules
- **Payments:** Apple guideline **3.1.3(e)** (services used outside the app) → use Stripe/Apple Pay via our processor, **not in-app purchase**. Re-check the current guideline text at submission.
- **Apple:** if users can create an account, they must be able to **delete it in the app**. If we offer Google/other social login, **Sign in with Apple** is required. Expect at least one review rejection.
- **Google Play:** new **personal** developer accounts need a **closed test with 12+ testers for 14 days** before production. An organization account (needs a company + D-U-N-S number) avoids this.
- Costs: Apple $99/yr, Google $25 one-time.

### 3.7 Multi-wholesaler architecture (required from day one)

**Requirement:** wholesalers must be **hot-swappable**, and we must be able to route per destination by price or reliability — e.g. *Japan → wholesaler A, Germany → wholesaler B, Europe regional → A with fallback to B*. Even if we launch with one real wholesaler, the code is built for many. Retrofitting this later is expensive; building it in from the start is cheap.

**1. Supplier adapter layer (one module per wholesaler)**
- Every wholesaler is wrapped in an adapter implementing one common interface, e.g.:
  `syncOffers()`, `issueEsim(offer, idempotencyKey)`, `getEsimStatus(ref)`, `getUsage(ref)`, `topUp(ref, offer, idempotencyKey)`, `cancelOrRefund(ref)`, `getBalance()`, `verifyAndParseWebhook(request)` → normalized events (`activated`, `usage_low`, `expired`, `issue_failed`, …).
- **No wholesaler-specific code outside its adapter.** The rest of the app only knows our own types (product, offer, eSIM, usage).
- A **mock adapter** (fake supplier) exists from day one for tests and local development, so the app is never built against just one real supplier.
- Shared **contract tests** that every adapter must pass (issue once with the same idempotency key, usage parsing, webhook verification, error mapping).
- Webhooks per supplier: `/api/webhooks/suppliers/[supplier]`.

**2. Products are ours; offers belong to suppliers**
- A `product` ("Japan 5GB / 30 days, $16") is what we sell and price. It does not reference a wholesaler directly.
- `supplier_offers` are synced from each wholesaler (package ID, cost, countries covered, GB, validity, features: top-up, daily plans, throttling, 5G, operator names, IP/breakout country, install/APN notes).
- `product_offers` maps each product to **one or more** equivalent offers from different suppliers (must match or exceed: countries, GB, validity).

**3. Routing rules (editable in admin, no deploy needed)**
| Field | Examples |
|---|---|
| Scope | a specific product, a destination country, a region, or global default |
| Strategy | `fixed` (always supplier A) · `priority` (A, then B if A unavailable) · `cheapest_valid` · `weighted` (e.g. 80/20 split to A/B-test reliability) |
| Constraints | required features (e.g. daily plans for influencer perks, top-ups), minimum health score, minimum margin |
| Override | manual "force supplier X" / "disable supplier X for country Y" |

Every order stores **which supplier was chosen and why** (rule ID, strategy, fallback used, cost at time of order).

**4. Reliability & health**
- Track per supplier **and per country**: API success rate, issuance latency, activation failures, "no connection" support tickets (tag them), refund rate.
- A `supplier_health` score feeds routing; below a threshold → skip that supplier for that country and alert the founder.
- **Circuit breaker / automatic failover:** if a supplier's API errors or times out repeatedly, or its **balance** is too low, new orders go to the next supplier in the priority list automatically.
- **Kill switch:** disable a supplier globally or per country instantly from admin.

**5. Rules that keep this safe**
- **An issued eSIM stays with its supplier forever.** Usage checks, top-ups and webhooks for that eSIM always go to the supplier that issued it (`esims.supplier_id`). Hot-swapping affects **new orders only**. If that supplier drops out, existing customers get a new eSIM (or refund), not a migrated one.
- **Customer price never depends on routing.** Price lives on the product. Routing must never pick an offer that drops margin below a configured floor; if no offer qualifies → hide the product and alert.
- **Catalog sync per supplier** (daily): detect price changes, removed packages and coverage changes → alert; auto-disable products with no valid offer left.
- **Balances per supplier**, each with its own low-balance alert.
- Supplier-specific install instructions/APN settings are stored per offer and shown on the customer's eSIM page and email.

**6. Build scope by phase**
- **Phase 1:** adapter interface + mock adapter + primary wholesaler adapter + routing rules table (even if every rule points to one supplier) + per-order supplier/cost recording.
- **Phase 4:** second (and further) real adapters, health scoring, automatic failover, weighted A/B routing.

---

## 4. Affiliate program (TOP PRIORITY)

Goal: a robust, self-serve program where creators earn reliably, the founder sees cost per creator vs. revenue, and fraud and refunds can't produce losses.

### 4.1 Affiliate identity & assets
- Each affiliate gets: a **unique promo code** (e.g. `MARIA10`), **unique referral links** (`/r/maria`, plus deep links to specific destinations like `/r/maria/japan`), and optional **campaign tags** (`?c=yt-japan-vlog`) to compare videos and posts.
- Code and link always map to the same affiliate.
- Self-serve sign-up with **manual approval** by the founder (default), plus an admin "invite affiliate" flow.

### 4.2 Attribution rules
| Rule | Default | Notes |
|---|---|---|
| Cookie/attribution window | **30 days** [assumption] | Stored first-party cookie + server-side record; app installs carry the ref via deep link/user record |
| Model | **Last click** | Most recent affiliate link wins |
| **Code beats link** | Yes | A code entered at checkout overrides any cookie attribution |
| Customer lock-in | Referred customer stays tied to that affiliate for recurring commission (§4.3) for **N months** [assumption: 6–12] | |
| Self-referral | Not allowed | See fraud controls |

Store **why** each order was attributed (code / link / cookie age) for disputes.

### 4.3 Commission models (configurable per affiliate)
| Model | Example | Use for |
|---|---|---|
| **Flat per sale** | $3–5 per order | Default; simple, safe |
| **% of net revenue** | 15–25% of (price − discount − VAT − fees) [assumption] | Bigger creators; scales with order size |
| **Tiered by volume** | 0–50 sales/mo: $3; 51–200: $4; 200+: $5 | Rewards top performers |
| **Recurring** | X% on repeat purchases & top-ups by referred customers for N months | Ongoing creator incentive |
| **Bonuses** | One-off milestone bonus (e.g. 100 sales) | Campaigns |

Always calculate commission on **net revenue after refunds**, never on gross. Keep commission rates configurable in the DB with history (a rate change must not rewrite past commissions).

### 4.4 Customer discount
- Code gives the customer a discount (default **10%** [assumption]), configurable per affiliate, with an optional max discount and min order size.
- Implemented as Stripe promotion codes tied to one coupon per discount level.
- The margin math in §2.3 already assumes a 10% discount. Show the founder a warning if discount + commission exceeds a configured share of contribution.

### 4.5 Affiliate dashboard
- Clicks, conversions, conversion rate, revenue driven, commission: **pending / approved / paid**.
- Per-campaign-tag breakdown, top destinations sold.
- Copy-ready links, codes, QR codes, and a **media kit** (logos, screenshots, approved claims, sample scripts).
- Payout settings and history; tax info collection.
- Notifications: "you earned a commission", "payout sent".

### 4.6 Payouts
- Method: **Stripe Connect (Express)** preferred, or **PayPal / Wise** batch payouts.
- **Minimum payout threshold** (e.g. $25–50) [assumption].
- **Hold period:** commission stays **pending until the refund window closes** (e.g. 30 days [assumption, match the refund policy]) → then **approved** → paid on a monthly schedule.
- **Clawback:** a refund or chargeback on an attributed order reverses its commission. If the commission was already paid, it becomes a negative balance offset against future earnings.
- Collect tax forms/info as needed per country [verify with accountant].

### 4.7 Fraud & abuse controls
- **Self-referral:** block when the buyer email, payment fingerprint, device, or IP matches the affiliate's.
- **Coupon-site leakage:** detect codes appearing on coupon aggregators (traffic spikes with no referrer or with coupon-site referrers). The founder can **rotate or disable a code** and issue the creator a new one, and the terms allow withholding commission on leaked-code sales.
- **Stolen cards:** commissions only become payable after the hold period. Stripe Radar plus velocity limits apply to referred orders too. Flag affiliates whose referred orders have high dispute or refund rates.
- **Brand bidding:** affiliates may not bid on our brand in paid search (terms).
- Admin can **suspend an affiliate** (freeze pending commissions).

### 4.8 Affiliate terms (needs lawyer review)
Commission rates and changes, hold period, clawbacks, payout threshold, prohibited promotion (spam, coupon sites, brand bidding, misleading claims like "truly unlimited"), disclosure requirements (#ad / FTC / ASA-style rules), termination, and the right to withhold on fraud.

### 4.9 Build vs. buy — recommendation
| | Buy (Rewardful / FirstPromoter) | Build our own |
|---|---|---|
| Speed | Days | Weeks |
| Cost | ~$0–50+/mo [verify] | Dev time |
| Stripe-native tracking | Yes | We build it |
| Custom rules (perks tied to performance, per-destination links, our own clawback logic) | Limited | Full control |

**Recommendation:** **Phase 0–1: buy** (Rewardful or FirstPromoter on top of Stripe) to launch quickly and validate creators. **Phase 2: build our own** once the perk system (§5) needs to read affiliate performance, or when tool fees or limits start to hurt. Design the DB (`affiliates`, `attributions`, `commissions`) from Phase 1 so the migration is easy. Codes are defined in **our DB** and synced to Stripe promotion codes (§3.0); a bought affiliate tool is used behind the `affiliates` adapter, and every attribution/commission it records is mirrored into our DB so switching tools loses nothing.

---

## 5. Influencer perk plans

**Founder's idea:** give active influencers a personal eSIM with a small free daily allowance (e.g. **1GB/day**) so they keep using the product and keep shouting it out. It's capped low enough that they can't do heavy work like uploading a full day's footage. Low cost, high goodwill, real "I actually use this" content.

### 5.1 Implementation options (depends on wholesaler)
| Option | How | Notes |
|---|---|---|
| **A. Native daily-reset plan** | Wholesaler offers "X GB/day" plans | Cleanest. Check availability and price in the wholesaler research |
| **B. Scheduled daily top-ups** | Our scheduler buys a 1GB top-up to the same ICCID each day | Works if top-ups are supported; watch per-top-up minimums and costs |
| **C. Monthly bundle + soft cap** | Grant ~30GB/30 days | Simplest, but lets someone spend 30GB on one upload day, which defeats the purpose |

Preferred: **A**, else **B**. Avoid C unless throttling is available.

### 5.2 Cost math
- Worst case: **1GB/day × 30 = 30GB/month**. At ~$1/GB (cheap regions) that's **≈ $30/month per influencer**.
- **Flag:** influencers travel. **Regional/global coverage costs more per GB** (often well above $1, e.g. Canada $2.56/GB), so a global perk can cost 2–3×+ that. Restrict perk coverage, or budget per region.
- Most days they won't use the full 1GB [assumption], so the real cost is likely lower. Track it.

### 5.3 Rules & limits
- **Daily cap** (1GB default, configurable per influencer: 500MB–2GB).
- **Throttle** after the cap if the wholesaler supports it; otherwise hard stop.
- **Country/region restrictions** per perk (e.g. "Europe only", or "where they're traveling this month").
- **Validity:** perk runs in cycles (e.g. monthly) and renews only if eligibility rules pass.

### 5.4 Eligibility tied to performance
- Configurable per influencer: e.g. **≥ X posts/mentions per month** and/or **≥ Y attributed sales per month**.
- Sales come from attribution data automatically. Posts are logged by the influencer or founder (link submission) [assumption: manual at first].
- **Auto-pause** when a cycle ends below threshold (with a warning email a few days before), and **auto-resume** when the threshold is met again.
- Grace period for new influencers (e.g. first month free of conditions).

### 5.5 Admin tooling
- Grant / pause / resume / revoke perk; change cap, region, rules.
- Per-influencer **P&L**: perk data cost + commissions paid vs. net revenue and contribution from attributed sales → **ROI**. Highlight influencers with negative ROI.
- Audit log of all perk changes.

### 5.6 Abuse risks & mitigations
| Risk | Mitigation |
|---|---|
| Sharing the eSIM/QR with others | One ICCID per influencer; the profile can usually only be installed once; usage anomaly alerts |
| Hotspot use to bypass the "phone-only" spirit | The daily cap limits the damage; mention it in perk terms |
| Inactive influencer still costing money | Performance-based auto-pause |
| Perk used in expensive regions | Region restrictions; per-perk monthly cost cap with auto-pause |

### 5.7 Suggestions (not yet decided by founder)
- **Suggestion:** tiered perks: 500MB/day for new creators, 1GB/day for proven ones, 2GB/day for top performers.
- **Suggestion:** a monthly **cost ceiling per perk** (e.g. $40) as a hard guardrail.
- **Suggestion:** give the perk eSIM a **visible branded label** in the app/dashboard and supply creators with ready "my daily eSIM" content prompts. Content is more valuable than the data.
- **Suggestion:** a "trip boost" request flow: the influencer asks for a temporary higher cap for a sponsored trip, and the founder approves.

---

## 6. Money-critical engineering rules (put these in CLAUDE.md)

1. **Issue each eSIM exactly once.** Stripe webhooks can arrive more than once. Store each `event.id` in `webhook_events`; issuance is keyed by order ID with a DB uniqueness constraint; use wholesaler idempotency keys if offered.
2. **Paid-but-not-issued is never silent.** Wholesaler failure → automatic retries with backoff → alert the founder after N failures → customer gets a "we're on it" email → auto-refund after a timeout.
3. **Secrets stay server-side.** The wholesaler API key, Stripe secret key, and service-role DB key are never sent to the browser or app.
4. **Low-balance alerts** on the prepaid wholesaler balance (thresholds + daily check). Optionally stop selling plans that would fail.
5. **Fraud limits:** Stripe Radar on, rules for velocity (orders per card/email/IP/day), 3-D Secure for higher-risk or larger orders, block disposable emails if abused.
6. **Verify webhook signatures** (Stripe and wholesaler).
7. **Prices come from our DB**, never from the client. Recheck the plan price server-side when creating a Checkout session.
8. **Tests for money paths:** duplicate webhooks, wholesaler timeout, refund → commission clawback, promo code + affiliate attribution, perk daily top-up job running twice.
9. **After every feature:** run a "how could this lose money?" review and fix what it finds.
10. **Test mode only** until launch: Stripe test keys + wholesaler sandbox.
11. **Wholesalers only through adapters** (§3.7). No supplier-specific code outside its adapter; every issued eSIM records its `supplier_id`, and all later calls for it go to that supplier.
12. **No payment or billing details in our database** (§7.1). Store Stripe IDs and order metadata only; never log card or billing data; support tools receive context, never payment data.
13. **Every vendor is swappable** (§3.0). Vendor SDKs only inside `/integrations/<category>/<vendor>/`; business logic uses our interfaces and internal events; our DB holds the data that matters; every new vendor gets a mock adapter and an entry in `docs/vendors.md`.

---

## 7. Support & help center

Founder handles support (not 24/7). Urgent tickets come from someone who just landed with no internet, often at night in the founder's time zone. Design to prevent those.

- **Tell people to install before the trip**, at checkout, in the order email, and on the help center. This alone prevents most emergencies.
- **Order email works offline:** include the top fixes (turn on data roaming for the eSIM line, select it as the data line, APN settings, restart) and the help center link.
- **Automatic refund policy:** e.g. unused and never activated within N days = self-serve refund button [assumption: define N]. Fast refunds prevent chargebacks.
- **Auto-reply** with realistic response times + top 3 troubleshooting steps.
- **Help center topics:** compatible devices, carrier-locked phones, install (iOS/Android), activation timing, roaming/data line settings, APN, hotspot, top-ups, refunds, "connected but no internet", why the IP is in another country.
- **Chargeback prevention:** keep disputes **below ~0.75–1%** of transactions (networks/Stripe act around there; each dispute costs ~$15). Clear statement descriptor, receipts, easy refunds.

### 7.1 Authenticated support & data minimization

**Principle:** Stripe holds all payment and billing details. Our database stores **only references and metadata**, never card data or full billing details. Support always knows *who* the customer is because they're logged in, without us storing anything sensitive.

**What we store vs. what stays in Stripe**
| Ours (DB) | Stays in Stripe only |
|---|---|
| Internal customer ID, email, country (for VAT evidence) | Card number, expiry, CVC (we never see these) |
| Stripe customer ID, Checkout session ID, payment intent ID | Full billing name/address (unless VAT rules require it [verify with Steuerberater]) |
| Order amount, currency, VAT amount, status, refund status | Payment method details |
| Optionally card brand + last 4 (display only, for "which card did I use?") | Disputes evidence (managed in Stripe dashboard) |

Rule: when unsure whether to store a field, **don't**; fetch it from Stripe on demand in the admin page.

**Help center: public content, authenticated contact**
- **Help articles stay public** (no login): people check device compatibility and install steps *before* buying, articles bring in search traffic, and a traveler on airport Wi-Fi shouldn't hit a login wall to read "turn on data roaming".
- **Contacting support requires login.** "Contact us" / "Report a problem" / chat → prompts login if not logged in → opens the support widget already identified.
- **Login method:** passwordless (email magic link or one-time code) so no one gets stuck on a forgotten password. Works on Wi-Fi when the eSIM isn't working.
- **Fallback for people who can't log in** (lost access to email, pre-sales questions): a limited guest contact form that asks for the order number + email and is verified against the order before any account details are shown or changed.

**Support tool integration** (tool TBD: e.g. Crisp, Help Scout, Intercom, Plain [verify pricing/features])
- Use the tool's **identity verification**: our server signs the user's internal ID/email (e.g. HMAC "user hash") so the widget can't be spoofed by someone typing another person's email.
- Pass **context, not payment data**, to the support tool: internal customer ID, email, plan(s), order IDs, eSIM status, **supplier** (§3.7), activation time, data used/left, expiry, device model if known, app/web, affiliate who referred them. This lets most tickets be answered without asking the customer anything.
- **Admin deep link** from each ticket to that customer's page in our admin (orders, eSIMs, refund/re-send buttons).
- Tag tickets by type ("no connection", "install failed", "refund", …) and country/supplier so they feed supplier health scoring (§3.7).
- Support tool and email provider need a **GDPR data processing agreement (DPA)**; list them in the privacy policy.

**Phase placement:** public help center + logged-in contact form in **Phase 1**; widget with identity verification and customer context by **end of Phase 1 or Phase 2**; same authenticated support inside the mobile app in **Phase 3**.

---

## 8. Compliance / pre-build checklist

Do these before writing production code:

- [ ] **Ask Stripe support** whether "reselling prepaid travel data eSIMs from a licensed wholesaler" needs extra review. Stripe's restricted list has no eSIM entry, but "prepaid products" are restricted, so get a written answer.
- [ ] **Backup processor** identified (e.g. Adyen, Checkout.com, Paddle/Lemon Squeezy *if they accept the category* [verify]).
- [ ] **Ask the wholesaler** about licensing: some EU countries require a **notification to the national telecom regulator** for electronic communications services, and reselling may count. Ask what their resellers in our country do.
- [ ] **Ask the wholesaler** which processors their resellers use and what reseller documentation Stripe may need (wholesaler agreement).
- [ ] **Steuerberater (Germany, §8.1):** regular VAT vs. Kleinunternehmer, eSIM VAT + OSS, affiliate payments/§50a, Gewerbe setup; plus US sales tax / Universal Service Fund (likely N/A for data-only eSIMs used abroad, plus the Internet Tax Freedom Act — **unconfirmed, verify**).
- [ ] **Bundesnetzagentur:** check whether the §5 TKG notification applies; file it if so.
- [ ] **ToS, privacy policy (GDPR), refund policy, affiliate terms, influencer perk terms**, reviewed by a lawyer.
- [ ] **Cash buffer** for the prepaid wholesaler balance and possible **processor payout holds/reserves** for a new "telecom/digital goods" merchant.

The pasted "AI overview" about telecom billing engines (Billingbooth etc.) applies to VoIP/MVNO billing, **not** this model. Stripe + Stripe Tax is normal for eSIM resellers.

### 8.1 Germany: tax & legal setup

> General information from planning conversations, **not tax advice**. Confirm everything with a Steuerberater who knows e-commerce / digital services. Items marked **[verify]** are open questions for that meeting.

**Founder situation**
- Lives in Germany; already runs a **sole-proprietor Gewerbe** that invoices **only US companies** for **location-independent services** (no German or EU clients, no services tied to a physical place). Those invoices carry no German VAT (place of supply abroad, §3a(2) UStG).
- The eSIM business will be a **second Gewerbe** (or an extension of the existing one via Gewerbeummeldung) under the same person.

**Legal form**
- **Start as Einzelunternehmen** (sole proprietorship): Gewerbeanmeldung (~€20–60) + *Fragebogen zur steuerlichen Erfassung* via ELSTER (or update the existing registration). Simple cash-basis bookkeeping (**EÜR**), one Anlage EÜR + Anlage G per business.
- **GmbH/UG later**, only when profits are high and can stay in the company (~30% corporate + trade tax vs. personal marginal rate). Costs notary, full accounting, ~€2–5k/yr advisor. Downside: eSIM losses could no longer offset the existing business's profit.
- **No foreign company.** Living in and running the business from Germany means Germany taxes it regardless of where a company is registered (*Ort der Geschäftsleitung*, CFC rules / AStG; US LLC profit lands on the German return). Estonian e-residency etc. adds cost without legal tax savings.

**Income & trade tax**
- Income tax is progressive and applies to the **combined** profit of both businesses plus any other income. A second Gewerbe does **not** create separate brackets; eSIM profit is taxed at the founder's marginal rate.
- **Losses offset**: start-up losses in the eSIM business (test eSIMs, software, devices; a prepaid wholesaler deposit only becomes a cost as it is used) reduce tax on the existing business's profit.
- **Gewerbesteuer** is per business: genuinely different activities can count as separate businesses, **each with its own €24,500 allowance** [verify]. Largely credited against income tax (§35 EStG) anyway.

**VAT (Umsatzsteuer): the main ongoing work**
- **Do not use the Kleinunternehmerregelung.** Foreign wholesalers (e.g. eSIM Access, Singapore) and foreign SaaS (Rewardful, etc.) trigger **reverse charge (§13b UStG)**: we owe 19% German VAT on those purchases. As a regular VAT payer it's reclaimed immediately; as a Kleinunternehmer it's a real 19% cost on wholesale.
- VAT status covers **all businesses of the person**. Switching to regular VAT also applies to the existing business, which is harmless here: US B2B invoices stay VAT-free (note "nicht im Inland steuerbar / place of supply abroad"), and the existing business can now **reclaim input VAT** on its costs. Cost: Umsatzsteuer-Voranmeldungen (monthly/quarterly) for both.
- Customer VAT for eSIM sales:

| Customer resides in | VAT |
|---|---|
| Germany | 19% German VAT |
| Other EU country | That country's rate, reported via **OSS** (One-Stop-Shop, BZSt), one quarterly return |
| Outside the EU | Generally no German VAT; some countries (UK, CH, NO, AU…) require foreign-seller registration above thresholds [verify when volume grows] |

- **eSIM-specific question:** "use and enjoyment" rules (where data is actually used vs. where the customer lives) [verify].
- Implementation: Stripe Tax calculates and records rates; collect customer country evidence at checkout (billing country + IP/card country); OSS returns are filed by us/advisor, not by Stripe.

**Paying affiliates & creators**
- German creators invoice us (with or without VAT depending on their own status).
- Foreign creators: **reverse charge** again (neutral for a regular VAT payer, but must be booked correctly).
- **§50a EStG withholding (15%)** may apply when paying foreign creators for **usage rights to their content** (e.g. re-running their videos as our ads / whitelisting). Plain affiliate commissions are usually not affected [verify before licensing creator content].
- Keep each affiliate's country, VAT status and invoice/self-billing (*Gutschrift*) records in the affiliate system.

**Telecom regulation**
- **§5 TKG**: providers of publicly available telecommunications services must **notify the Bundesnetzagentur**. Reselling eSIM data may count; it's a free notification, not a license [verify].

**Legal ways to optimize within Germany**
- Deduct all business costs: test eSIMs, software, hardware, share of home office, travel used for testing/content.
- Switch to GmbH once the numbers justify it.
- Trade tax rate (Hebesatz) varies by municipality, but only a real place of business counts.

**Questions for the Steuerberater (Erstberatung)**
1. Confirm regular VAT (no Kleinunternehmer) across both businesses, given reverse charge on foreign wholesale; any impact on the existing US-only business.
2. VAT treatment of travel eSIMs (place of supply, use-and-enjoyment), OSS setup, and which non-EU registrations to watch.
3. Paying foreign affiliates/creators: invoices vs. Gutschrift, reverse charge, §50a when licensing content.
4. Does the §5 TKG notification apply?
5. Are the two activities separate businesses for Gewerbesteuer?
6. At what profit level does a GmbH make sense?

---

## 9. Consolidated pitfalls

| # | Pitfall | Mitigation |
|---|---|---|
| 1 | Duplicate webhook → two eSIMs issued for one payment | Idempotency (§6.1) |
| 2 | Wholesaler down → customer paid, got nothing | Retries, alerts, auto-refund (§6.2) |
| 3 | Leaked API key drains the wholesaler balance | Server-side secrets, key rotation |
| 4 | Balance hits zero → sales silently fail | Low-balance alerts |
| 5 | Card testers / stolen cards on instant digital goods | Radar, velocity limits, 3DS |
| 6 | Chargebacks > ~1% → Stripe account at risk; $15 per dispute | Easy refunds, clear descriptor, install-before-travel |
| 7 | Stripe declines or freezes the account (restricted category) | Ask first; backup processor; cash buffer |
| 8 | EU VAT eats ~⅓ of contribution | Price per market; Stripe Tax; accountant |
| 9 | Small plans ($3–4) barely profitable | Push 5–20GB plans and bundles |
| 10 | Unlimited plans: heavy users wipe out margin | Avoid at launch, or use strict fair-use caps |
| 11 | Undercutting big brands destroys margin | Compete on the creator's recommendation + discount code, not lowest price |
| 12 | Flat-fee sponsorships that don't convert | Default to pay-per-sale; measure CAC per creator |
| 13 | Affiliate code leaks to coupon sites | Rotate codes, leakage detection, terms |
| 14 | Commissions paid on refunded or fraudulent orders | Hold period + clawback |
| 15 | Perk eSIMs in expensive regions / inactive creators | Region limits, cost ceilings, performance auto-pause |
| 16 | Night-time "no internet on arrival" tickets | Install-before-travel, offline-readable email, auto-reply |
| 17 | App store friction (Google 12 testers/14 days, Apple rejections) | Plan time; web first |
| 18 | Expecting one-tap in-app install | Use iOS setup link + QR; Android QR/code |
| 19 | Using IAP and losing 15–30% | 3.1.3(e): external payments |
| 20 | Wholesaler price change, outage or dropped country | Adapter layer + per-destination routing from day one (§3.7); second wholesaler (Phase 4); sync alerts |
| 23 | Code hard-wired to one wholesaler → painful switch later | Adapter interface + mock adapter in Phase 1 (§3.7) |
| 24 | Top-up or usage call sent to the wrong supplier after a routing change | `esims.supplier_id` is permanent; routing applies to new orders only |
| 25 | Any vendor (payments, support, email, affiliate tool…) becomes a lock-in | Adapters, internal events, our DB as source of truth, vendor register with exit plans (§3.0) |
| 21 | Device not eSIM-capable or carrier-locked | Compatibility check before payment |
| 22 | Regulatory notification missed (some EU countries) | Pre-build checklist |

---

## 10. Phased roadmap

### Phase 0 — No-code validation (weeks 0–2)
Launch on the wholesaler's **white-label store or Shopify integration**; run 1–2 influencer deals on pay-per-sale.
**Done when:** ≥ 1 creator deal run; real orders placed; measured CAC per customer and conversion; Stripe/processor question answered.

### Phase 1 — Backend + website (≈ 3–6 weeks part-time)
Supplier adapter layer (§3.7: interface, mock adapter, primary wholesaler adapter, routing rules table, per-order supplier/cost recording); products mapped to supplier offers with our pricing; destination pages; Stripe Checkout + Tax; idempotent issuance; order email (QR + iOS link + code + guide); "My eSIM" page with usage; admin page (orders, refunds, re-send, balance); help center; refund flow; alerts. Affiliate tracking through a bought tool (Rewardful/FirstPromoter) + Stripe promo codes.
**Done when:** all §6 tests pass in test mode; duplicate-webhook test issues once; wholesaler-failure test triggers retry → alert → refund; a real purchase installs on iOS and Android; affiliate code purchase shows in the affiliate tool; routing a test product to the mock adapter instead of the real one works by changing a rule in admin, with no code change.

### Phase 2 — Own affiliate system + influencer perks (≈ 2–4 weeks) [assumption]
Own attribution, commissions, hold/clawback, payouts (Stripe Connect or PayPal/Wise), affiliate dashboard, fraud controls, media kit; perk eSIMs (§5) with daily allowance, region limits, performance rules, auto-pause, per-influencer ROI.
**Done when:** refund → commission reversed; self-referral blocked; payout batch runs in test mode; perk daily job is idempotent (running twice ≠ 2GB); perk auto-pauses below threshold; ROI report matches manual calculation.

### Phase 3 — Expo mobile app (≈ 2–4 weeks + 2–4 weeks store review)
Login, buy (external payment), install (iOS setup link/QR, Android QR/code), usage, top-up, push notifications (low data, expiry), in-app account deletion, deep links carrying affiliate refs.
**Done when:** EAS builds for both platforms; tested on real devices; Google closed test (12 testers/14 days, if personal account) done; approved on both stores.

### Phase 4 — Resilience & growth
Second (and further) real wholesaler adapters, supplier health scoring per country, automatic failover, weighted A/B routing, top-ups everywhere, bundles, multi-currency, volume-tier renegotiation.
**Done when:** failover tested by disabling the primary wholesaler in staging; `cheapest_valid` routing picks the cheaper qualifying supplier; a per-country rule (e.g. Japan → A, Germany → B) routes correctly; existing eSIMs keep using their original supplier for usage and top-ups.

---

## 11. Working with Claude Code

**Conventions**
- **Plan mode for every feature.** Claude explains the plan in plain language before coding; the founder asks "why" until it's clear.
- **Commit to git after each working step.** Small commits, clear messages.
- **CLAUDE.md** holds the §6 rules plus: never expose secrets client-side; every order is issued exactly once; prices are server-side only; commissions are calculated on net after refunds; test mode only until launch.
- **Test-mode keys only** (Stripe test mode, wholesaler sandbox) until go-live; separate env files per environment, never committed.
- **Tests for money paths** with every feature + a "how could this lose money?" review.
- Ask Claude to explain each new external service (what it does, what it costs, what breaks if it's down).

**Open decisions for the founder**
1. **Wholesaler** (primary + backup): see `esim-wholesaler-briefs.md`. Must have a sandbox, top-ups, usage API, webhooks, and ideally daily-reset plans (for §5).
2. **Company setup**: decided in principle: **Germany, Einzelunternehmen (second Gewerbe), regular VAT** (§8.1). Confirm with Steuerberater. Note: a Google Play organization account needs a D-U-N-S number; check whether a sole proprietorship qualifies [verify].
3. **Currencies** to sell in (USD only vs. USD + EUR + others).
4. Affiliate defaults: commission model, customer discount %, attribution window, hold period, payout threshold.
5. Perk defaults: daily cap, regions, eligibility thresholds, cost ceiling.
6. Refund policy window and conditions.
7. Launch plan set: which destinations and plan sizes (favor 5–20GB).
8. Whether to offer unlimited plans at all.
