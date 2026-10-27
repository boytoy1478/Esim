# B2B eSIM Wholesale / Reseller Platforms: Briefs for a Solo, Influencer-Led Travel eSIM Brand

*Research date: 2026-10-26. Sources are mostly vendors' own sites and docs (fetched Oct 2026), plus a few third-party articles dated 2025-2026. Vendor statements are reported as **claims**. Where something could not be confirmed it is marked **unknown / not public**.*

**What the buyer needs:** a solo founder who isn't a developer and builds with Claude Code. They want to sign up without industry contacts, take payments on their own Stripe account through their own website and app, sell under their own brand through influencers and affiliates, and offer **daily-reset data plans** (for example 1GB/day).

**Method limits (please read):**
- Reddit could not be searched. The search and fetch tools are blocked from reddit.com, so reseller sentiment comes from Trustpilot, aggregators and vendor docs. This is a real gap, so read r/esim and r/eSIMs yourself before committing money.
- Most "comparison" articles about eSIM wholesalers are written by competing vendors (Omax/Bappy, TripoSIM, Celitech's AI-generated blog, zendit). They are used here only for facts that can be checked, never for rankings.
- **Wholesale price lists are almost never public.** They sit behind a portal login or a sales call. Where no price was found, the brief says so.

---

## TL;DR

| | |
|---|---|
| **Primary** | **eSIM Access** (Redtea Mobile group). You sign up yourself in minutes. Deposits start at $50 and there's no minimum order. It has real **daily plans** (500MB, 1, 2, 3 and 10GB per day), webhooks for low data and expiry, top-ups, Apple's one-tap install link, and an optional $19/mo branded store that charges through *your own* Stripe account. Main risks: its reputation is thin and mixed, the docs contradict themselves in places, and it's a China/Singapore-rooted supplier. |
| **Backup 1** | **zendit** (owned by IDT Corp, NYSE: IDT). You can register online. It claims about 3,900 "unlimited" plans, has a test-mode API and a large, credible parent company. eSIM-specific docs are thin, and whether it has daily-reset plans is unverified. |
| **Backup 2** | **CELITECH** (US). Its docs and webhooks are the cleanest, there are no setup or monthly fees, and it bills monthly in arrears. But onboarding goes through sales and daily plans are unconfirmed. **Airalo Partners** is the "quality" alternative, but you can't price below Airalo's own retail price. |
| **Avoid for now** | eSIM Go (needs **$10k/month for the first 3 months**, about $30k). 1GLOBAL and BICS (enterprise only). Truphone (absorbed into 1GLOBAL). Bitcall (new, unproven). Breathe (a consumer brand, not a wholesaler). |

---

## 1. eSIM Access — fit **5/5**

- **What it is and who's behind it:** A wholesale eSIM reseller platform. It calls itself a Redtea Mobile initiative. Redtea's HQ is given as Singapore in its Jan 2026 press release and its MWC listing, but GSMA lists the entity as Redtea Technology (Shanghai), Shenzhen-based. Redtea is a GSMA-member eSIM orchestrator and aggregator, and also runs the consumer brand RedteaGO. Who owns the IMSIs is **not public**. It is best treated as an aggregator of carrier wholesale deals.
- **Signup:** **Self-serve.** The site says: "Sign up free and get your API credentials in the Console under Developer", with "no contracts" and "No MOQ". Account **top-ups start at $50**. Payment is by card through Stripe, with optional auto-charge. Wire transfers are suggested above $3k for a first deposit or $5k a month. eSIM Access says you can ask for free test credits. On KYC, it says most travel markets need no end-user eKYC and that it tracks the few that do. Its business-verification requirements for resellers are **not public**.
- **API:** Public REST docs (docs.esimaccess.com). Requests are signed with HMAC-SHA256. Operations cover balance, package list, ordering (single and batch), query (install state, usage, expiry), usage, **top-up of an existing eSIM**, cancel/refund of unused eSIMs, suspend/revoke, SMS, and webhook configuration. They also publish an official "API Agent Skill" for AI coding agents and an MCP server (mcp.esimaccess.com), which is unusually helpful for building with Claude Code.
  - **Webhooks (official list):** `ORDER_STATUS` (eSIM ready), `ESIM_STATUS` (in use), `DATA_USAGE` (100MB or less remaining), `VALIDITY_USAGE` (1 day left). These are fixed thresholds, not configurable.
  - **Sandbox:** No formal sandbox found. You test by asking for free test credits. A third-party note says usage data lags 1-3 hours (**unverified**).
- **Plans:** Single-country, regional and global. Fixed data runs from 1 to 50GB. **Unlimited/Day Pass plans give 500MB, 1GB, 2GB, 3GB or 10GB of high-speed data per day** (10GB only in some countries). After the daily quota, speed drops to a "best-effort" 128-512 Kbps (the page also says 1-2 Mbps on some plans). You can pick any length from 1 to 365 days, and the price is days × daily rate, with **length discounts of 4% (1-4 days) up to 18% (30+ days)**. Hotspot is allowed. *Caveats:* the docs contradict themselves on when the daily allowance resets (24h from first connection vs the carrier's local midnight) and on whether day passes can be topped up ("Not available now" in one place, "top up for extending the days" in another). Get both answered in writing.
- **Example prices:** **Not public.** A price viewer exists at app.esimaccess.com/public/current-price, but it renders with JavaScript and could not be extracted. The full rate sheet is in the console. Their own Stripe-shop example uses a "$5.00 wholesale eSIM" sold at $8.99, and another uses a $5.70 wholesale price at a 75% markup ($9.98). Both are illustrations, not a rate card.
- **White label:**
  - **eSIM Access Express:** a branded store at $19/mo with the first 30 days free. **Payments go straight to your own Stripe account** (Stripe Connect / Direct Charge). You get your logo, colours and email templates, and either a custom domain or a yourname.esimqr.link subdomain. It takes 7 currencies, supports 11 languages, includes customer top-ups, and eSIM Access handles 24/7 eSIM support tickets. It was a public beta as of Aug 2025.
  - **Turnkey Shopify shop:** $1,390 one-time setup.
  - **Branded iOS/Android app projects:** "from $10k".
  - **Installs:** QR code, plus **Apple's one-tap install link** (esimsetup.apple.com, iOS 17.4+), and an SDK for installing inside your own app.
- **Coverage:** It claims 200+ countries and regions, 40+ multi-country plans and over 5,000 plans. Third-party lists give 100+ or 150+. Network quality and IP breakout: **unknown / not public**. Chinese-rooted aggregators often route data out through a third country (for example HK or SG), so test with an IP-check site in your top destinations.
- **Reputation:** Trustpilot gives it **2.6/5 from only 4 reviews** (Dec 2025 to Oct 2026, profile unclaimed). The complaints are a $50 top-up that never reached the account (unresolved for 5 months), no reply to a would-be reseller's questions for weeks, "none of the eSIMs worked", and "scammers". The sample is tiny but the warning is real. No independent reseller forum evidence could be found (Reddit was inaccessible).
- **Red flags:** Thin and negative public reputation. Contradictory docs on day passes. A China-linked supplier, which could matter for data-protection wording and for some markets. Webhook thresholds are fixed. No true sandbox.
- **Why 5/5:** It's the only supplier that combines instant self-serve signup, a $50 start, published daily plans, the webhooks you need for "low data → upsell" emails, and a branded store on your own Stripe account. Start with $50-100, run real eSIMs in 3-5 target countries, and only scale once support has answered a ticket.

## 2. eSIM Go — fit **2/5** (needs too much money up front)

- **What it is:** eSIM Go Ltd, a UK aggregator. It claims to own "a network core … end to end" and to cover 190+ countries. It has now expanded into UK MVNO plans on Vodafone.
- **Signup and money:** You open an account yourself ("Standard tier is live in minutes"), **but the pricing page sets monthly top-up commitments**:
  - **Standard:** $0 commitment on paper, but **$10k/mo for the first 3 months** (their FAQ says to budget **$30k for a 90-day test**). The money becomes credit you spend on plans.
  - Silver: $20k/mo. Gold: $100k. Platinum: $250k.
  - Custom brand profile: $2,000 per profile per month. Android direct install: $2,000 per app per month.
  - A "per eSIM: $0" line contradicts an "eSIM fees" row on the same page, so ask.
- **API:** Mature v2.5 REST API with an `X-API-Key` header. It has an organisation balance, bundle inventory, "apply bundle to eSIM" (which acts as a top-up), status and remaining data for applied bundles, a usage callback and balance notifications. Sandbox: not documented.
- **Plans:** Fixed bundles plus **Unlimited Lite / Essential / Plus**. Essential gives 1GB/day at full speed then 1.25 Mbps. Plus gives 2GB/day then 2 Mbps (from help-centre search snippets; the page itself returned 403). These are good daily plans, which makes the money barrier a pity.
- **Prices:** Rates are only in the portal and change monthly. The public page shows illustrative rates only.
- **White label:** Branded profile at $2k/mo. "Powered by Breeze" managed white label. Breeze affiliate programme. iOS universal-link install is documented.
- **Reputation:** No Trustpilot reviews for esim-go.com. It's frequently named in industry roundups as a major aggregator.
- **Red flags:** $30k of credit in 90 days is unworkable for a bootstrapped influencer start. The pricing page has internal contradictions.
- **Fit:** 2. Revisit at about $10k/mo revenue.

## 3. CELITECH — fit **3/5**

- **What it is:** CELITECH Inc., a US company. The site lists the HQ as both Santa Clarita CA and Los Angeles CA. It's a travel eSIM platform claiming Tier-1 partners (AT&T, Orange, Telefónica, Vodafone) and 215+ countries and regions (a competitor's table says 160+). It is an aggregator, and its "brandable networks" claim is marketing.
- **Signup:** **Sales-led** ("Schedule a Demo"). The sandbox is "enabled per account on request". It claims no setup fees and no capex. **Billing is invoiced monthly through Stripe**, with each purchase added as an invoice item, so it's effectively postpaid. Minimums: **unknown / not public**. Third-party competitor pages describe the model as a revenue share or backend margin, with an undisclosed percentage.
- **API:** Best-documented of the group. OAuth2 client credentials; destinations, packages, purchases, **top-up of an existing eSIM**, consumption/status, eSIM history and top-up eligibility; idempotency. SDKs plus iFrame, Zapier, MCP and Slack integrations (beta). A real sandbox.
  - **Webhooks:** `consumption.threshold.reached` at 50% and 80%, plus `esim.downloaded`, `installed`, `enabled`, `disabled` and `deleted`. Payloads are signed and retried.
- **Plans:** Fixed GB, for example "United States 1GB" or "Europe 5GB". **Daily / unlimited: not mentioned anywhere found.**
- **Prices:** **Not public.**
- **White label:** Branded landing pages and an embeddable iFrame. Install method: **unknown / not public**.
- **Reputation:** Almost no independent reseller feedback found. Its blog (ai.celitech.com) is mass-produced AI SEO content, so ignore its claims.
- **Red flags:** Sales gate. Opaque margins. No confirmed daily plans.
- **Fit:** 3. A strong backup *if* they accept you and confirm daily or unlimited plans.

## 4. Airalo Partners (Partner API / Partner Platform) — fit **3/5**

- **What it is:** Airalo (Singapore), the largest consumer travel eSIM brand, which aggregates many carriers. Partner options:
  - Partner Platform: a reseller portal.
  - REST API / SDK, plus a WooCommerce plugin.
  - Co-branded landing page.
  - Affiliate programme (via Impact).
- **Signup:** You fill in an online form and Airalo onboards you. There's no sign-up or subscription fee. Company-document requirements: **not public**. The docs mention a credit limit (there's a credit-limit webhook), but prepaid or minimum terms are **not public**.
- **API:** Good public docs. A sandbox that mirrors production prices but can't produce installable eSIMs (they suggest 1-2 refundable test orders in production). Orders, async bulk orders, top-ups (`GET /sims/:iccid/topups`, `POST /orders/topups`), installation instructions, brand settings.
  - **Low-data webhook:** levels `75%`, `90%`, `3days` and `1days` before expiry, HMAC-SHA512 signed, with a simulator.
  - Usage is available through Get eSIM, *except* on "prepaid-operator" packages, where usage is unavailable.
- **Plans:** Country, regional and global fixed GB. **Daily or unlimited packages through the Partner API: not confirmed.** Airalo markets "unlimited" plans to consumers, but nothing confirms they're in the API.
- **Pricing rule (important):** You must sell **at or above Airalo's own retail price** ("minimum selling price"). Your margin is that price minus `net_price`. You **cannot undercut Airalo**. For influencer codes this means your "discounted" price can't drop below airalo.com's price.
- **White label:** "Complete white label in most instances", but **some carriers show "Airalo" as the network name on the phone** (this can't be changed). Branded emails and brand settings are available.
- **Reputation:** Strong consumer brand. No reseller-specific complaints found.
- **Red flags:** The price floor and the possible "Airalo" network name undermine an independent brand. Onboarding is form-based.
- **Fit:** 3. Reliable and well documented, but commercially restrictive. Its affiliate programme is a zero-inventory fallback.

## 5. zendit (IDT Corporation) — fit **3/5**

- **What it is:** IDT Corp's (NYSE: IDT, US) "prepaid-as-a-service" platform, launched Feb 2023. It added eSIM in Feb 2024. It aggregates mobile top-ups, gift cards, utilities and eSIMs. It claims 193 carriers in 190+ countries and **about 1,500 standard plus about 3,900 "unlimited" eSIM plans** (zendit's own blog, Aug 2026).
- **Signup:** **Self-serve** ("register in just minutes", no contracts, pay-as-you-go). You fund a prepaid balance, but the minimum is **unknown / not public**.
- **API:** REST at api.zendit.io/v1 with a bearer API key. There's a test environment and test key in the dashboard. eSIM endpoints cover offers, purchases, purchase details and QR codes. Webhooks are said by an API directory to exist, but **zendit's eSIM-specific webhook events, usage endpoint and top-up of an existing eSIM are unknown / not public**. The eSIM docs are thin compared with eSIM Access, Celitech and Airalo.
- **Plans:** Fixed 1-20GB for 7-30 days, plus "unlimited" plans (claimed). **Daily-reset design: unverified.** Check in test mode.
- **Prices:** **Not public.**
- **White label:** A no-code branded storefront and widget (claimed). Install method: QR or manual.
- **Reputation:** No independent reviews found. The listicles ranking zendit first are its own or unsponsored-but-suspect content.
- **Red flags:** eSIM is a side product for a top-up company. Docs are sparse.
- **Fit:** 3. A credible, solvent, self-serve backup. Confirm daily plans, usage data and top-ups before relying on it.

## 6. Telna — fit **2/5**

- **What it is:** A US/Canada FCC-licensed carrier and GSMA member, using multi-IMSI and claiming 10+ points of presence for local breakout. Of everyone here it's the closest to owning its network and IMSIs.
- **Signup:** The main site is sales-led, **but "Connect Flex" offers "Free registration with self-onboarding"**: a prepaid account with top-ups, a standard reusable price list and an online-store option for eSIM resellers (claims, Mar 2025 blog plus product page). Minimums: **not public**.
- **API:** Ordering, renewals, profile retrieval and usage monitoring. Public reference docs: **not found**. Webhooks: **unknown**.
- **Plans:** Custom data packages. **Daily: not mentioned.**
- **Prices:** **Not public** ("best rates").
- **Fit:** 2. Worth one test signup on Connect Flex to see the price list. Probably better as a later-stage, network-grade supplier.

## 7. 1GLOBAL (formerly Truphone) — fit **1/5** (enterprise only)

- **What it is:** A London HQ'd GSMA-certified operator, MVNE and full MVNO in some markets. It **acquired Truphone's assets in Jan 2023**. Partners are names like Revolut, N26, KPN, freenet, Jazeera Airways and Jamf.
- **Signup:** A "Distribution Partnership Programme" for "trusted partners" via an enquiry form to sales. No self-serve signup, public prices or public reseller docs.
- **Fit:** 1. Not reachable for a solo founder today.

## 8. Truphone / "Gamma" — **dropped (defunct as a separate supplier)**

- Truphone's assets now belong to 1GLOBAL (Jan 2023), and truphone.com redirects to 1GLOBAL. No link between Truphone and Gamma Communications in the travel-eSIM wholesale space could be found. If you meant something else by "Gamma", it's unverified.

## 9. BICS — fit **1/5** (enterprise only)

- **What it is:** A Belgian (Proximus group) wholesale carrier with a travel eSIM programme for MNOs and MVNOs: API-first, data plus voice/SMS/numbers, 8-12 weeks to launch.
- **Signup:** Sales-led. Light Reading quotes BICS as targeting companies that can generate **more than 100,000 SIMs a year** (an older article). Some smaller white-label platforms resell BICS connectivity (Omax claims "BICS Tier-1").
- **Fit:** 1.

## 10. Maya Mobile — fit **2/5**

- **What it is:** Mobile Maya Inc., a US travel eSIM brand founded in 2021, with a reseller programme and a "Connect+" API.
- **Signup:** Contact form. **NDA required** (legal company name plus country of incorporation) before docs are shared. Third parties claim no application or monthly fees. Prices depend on volume.
- **API:** Provisioning, get eSIM, suspend and reactivate, real-time install status and usage, custom package templates. Docs are **not public**.
- **Plans:** Its consumer range is strong on **unlimited** plans (with a traffic/fair-use policy). Wholesale availability of daily plans: **unknown**. A third-party review mentions routing issues hurting performance.
- **Fit:** 2. Too opaque to start on.

## 11. Keepgo (and its B2B arm esimba.ai) — fit **2/5**

- **What it is:** Keepgo, a "global MVNO" (Wikipedia says Delaware, founded 2011; directories say Israel, 2009). Its model is **lifetime, non-expiring data** that stays valid if you top up once a year. 150+ countries (claim).
- **Signup:** Free partner account at myaccount.keepgo.com, with Free / Essential / Professional / Enterprise editions. Your discount tier depends on your latest deposit (tiers below $500 and up). Pay by card, wire or crypto. esimba claims "no deposit, no setup fee, no subscription" for API access, which contradicts the deposit-tier FAQ.
- **API:** REST: bundles, `POST /line/{ICCID}/refill`, usage. Auto-refill below 100MB. Webhooks: **unknown**.
- **Plans:** Lifetime and Travel lines. **No daily-reset plans found**, which is the wrong product for your idea.
- **White label:** esimba webstore, app, Telegram and WhatsApp bots, built for you.
- **Fit:** 2.

## 12. eSIMCard (partners) — fit **3/5**

- **What it is:** ESIMCARD LLC, Longwood, Florida (US). A consumer brand plus a wholesale programme. It's an aggregator. 200+ countries.
- **Signup:** Self-serve form, then **custom pricing within 24h**, then an **NDA** before API access. No setup fee, no minimum order, pay-as-you-go. KYC: not mentioned.
- **API:** Advertised: instant top-ups, real-time usage, add or remove packages on the same eSIM, SDKs. Docs: **not public**. Webhooks and sandbox: **not mentioned**.
- **Plans:** 1GB up to "unlimited" tiers, local, regional and global, plus data+voice and US-number plans. Daily: **unknown**.
- **Example prices (their own comparison, market unspecified):** 5GB/15 days at **$7.90**, 10GB/30 days at **$10.21**.
- **White label:** Ready-made white-label store or app ("10+ solutions"), plus an onboarding manager.
- **Reputation:** Consumer Trustpilot **4.5/5 from 340 reviews** (claimed, paid profile). Praise for support. Complaints about dropped connections and some country failures. Nothing reseller-specific.
- **Fit:** 3. A reasonable third option. Ask for daily plans and webhook docs under the NDA.

## 13. Citrus Mobile — fit **2/5**

- **What it is:** CitrusMobile (credits DialAnyone and Stripe). A **pay-as-you-go per-MB** data eSIM in 200-220 destinations. HQ: **not stated**.
- **Signup:** Self-serve consumer-style account. Developer portal and API docs are linked. The partner programme page has no detail.
- **Public per-GB prices:** UK, France, Germany, Italy, Spain and others at **$0.61/GB**; Switzerland $1.13; **USA $0.97/GB**; **Japan $1.54/GB**. Global: none.
- **Plans:** **No fixed or daily plans by design.** Balance never expires.
- **Red flags:** Contradictory fees on its own homepage ($4 vs $50 minimum top-up; $0 vs $25 vs $2.45 activation). No white-label programme described.
- **Fit:** 2. Interesting as a cheap per-GB benchmark, wrong for daily influencer plans.

## 14. Breathe — **dropped (not a wholesaler)**

- "BreatheSIM" appears to be a consumer eSIM app *built on* Mobilise's white-label platform (Mobilise case study). No wholesale or reseller programme was found. Mobilise itself is a white-label platform vendor (setup €10-25k plus €500-1,500/mo per a competitor's table), not a self-serve wholesaler.

## 15. Bitcall — fit **1/5**

- **What it is:** bitcall.io / "Bitcall OS", a white-label telecom panel (eSIM data, SIP voice, SMS, OTP, HLR). **Launched on Product Hunt in Feb 2026**, with **no G2 reviews**.
- **Pricing (aggregators, inconsistent):** Reseller panel €149/mo (Capterra), or $120 / $249 / $499 tiers (SaaSworthy). A "Global eSIM €3". Full API only in the top tier.
- **Red flags:** New, unproven, crypto-centric, no wholesale eSIM rate card or docs. Upstream supplier unknown.
- **Fit:** 1.

---

## Additional self-serve options found

| Provider | Facts (claims) | Fit |
|---|---|---|
| **MobiMatter** (Abu Dhabi, ADGM) | Apply by form and get approved. **$250 minimum wallet top-up.** Prepaid wallet (card, crypto, bank). API keys, white-label delivery with your logo. Aggregates many operators' products (even Airalo's). Daily plans and webhooks: unknown. | 3 |
| **Telnyx eSIM** (US) | Self-signup portal. **$0.70 per eSIM + $2 per active SIM per month + data.** 650+ networks, 180+ countries. Aimed more at IoT and "connectivity resellers". Daily plans not mentioned. The monthly per-SIM fee hurts short trips. | 2 |
| **TripoSIM** (BroadNet Technologies LLC, Dubai) | Register online, "no setup fee, no minimum order", prepaid wallet, free sandbox. Webhooks for provisioning, usage thresholds and expiry; usage and top-up endpoints. **But reseller price is only 10-30% off its retail** (e.g. US 5GB/30d $6.00 retail → $5.40 at the starter tier). Daily plans not mentioned. Also publishes vendor-biased "guides". | 2 |
| **Omax Telecom / Bappy** (London shell address) | Business accounts with KYC. **€500-1,000 setup credited to your wallet**, €0/mo, 48h to launch, claims BICS Tier-1 connectivity. Authors self-serving comparison articles. | 2 |
| **Roamic** (reseller.roamic.com) | A search snippet claims wholesale from about €0.30/GB (EU, Japan) and €0.40/GB (US). The page didn't render, so this is **unverified**. | ? |

---

## Comparison table

| Provider | Self-serve? | Min. to start | Public API docs | Sandbox | Webhooks (low data / expiry) | Top-up existing eSIM | **Daily plans** | Public prices | Branded store on *your* Stripe | Fit |
|---|---|---|---|---|---|---|---|---|---|---|
| **eSIM Access** | Yes, instant | $50 top-up | Yes | Test credits only | Yes: ≤100MB, 1 day left | Yes (day passes unclear) | **Yes, 0.5/1/2/3/10GB per day** | Portal only | **Yes, $19/mo Express** | **5** |
| eSIM Go | Yes, but… | **$10k/mo × 3 months** | Yes | Not documented | Usage callback | Yes (apply bundle) | Yes (Essential 1GB/day, Plus 2GB/day) | Portal only | No ($2k/mo brand profile) | 2 |
| CELITECH | No (sales) | Not public (postpaid invoice) | Yes, excellent | Yes, on request | Yes: 50%/80% | Yes | Not found | No | No (landing page / iFrame) | 3 |
| Airalo Partners | Form, then onboarding | Not public | Yes, good | Yes | Yes: 75%/90%, 3d/1d | Yes | Not confirmed in API | Portal; **floor = Airalo retail** | No (your own build or WooCommerce) | 3 |
| zendit (IDT) | Yes | Not public | Partial | Yes (test key) | Not documented for eSIM | Unknown | "Unlimited" claimed; daily unverified | No | Storefront (claimed) | 3 |
| eSIMCard | Form + NDA | None stated | No (NDA) | Unknown | Unknown | Yes (claimed) | Unknown ("unlimited" tiers) | 2 examples | White-label store | 3 |
| MobiMatter | Form + approval | $250 | Not seen | Unknown | Unknown | Yes (top-up packages) | Unknown | No | White-label delivery | 3 |
| Telna Connect Flex | Yes (claimed) | Not public | No | Unknown | Unknown | Renewals | Not mentioned | No | Online store (claimed) | 2 |
| Maya Mobile | No (NDA) | Not public | No | Unknown | Unknown | Unknown | Unlimited (retail) | No | Unknown | 2 |
| Keepgo / esimba | Yes (free account) | Deposit-tiered | Partial | Free tier | Unknown | Yes (refill) | **No (lifetime model)** | No | Built for you | 2 |
| Citrus Mobile | Yes | $4-50 (contradictory) | Linked | Unknown | Unknown | Pay-as-you-go balance | **No (per-MB)** | **Yes: $0.61/GB EU, $0.97 US, $1.54 JP** | No | 2 |
| Telnyx | Yes | None stated | Yes | Unknown | Unknown | Yes | No | $0.70/eSIM + $2/SIM/mo + data | No | 2 |
| TripoSIM | Yes | None | Yes | Yes | Yes | Yes | Not mentioned | Yes (10-30% off retail) | Unknown | 2 |
| 1GLOBAL | No | Enterprise | No | – | – | – | – | No | – | 1 |
| BICS | No | ~100k SIMs/yr | No | – | – | – | – | No | – | 1 |
| Bitcall | Yes | €149/mo panel | No | – | – | – | – | Inconsistent | Panel | 1 |
| Truphone | Dropped (now 1GLOBAL) | | | | | | | | | – |
| Breathe | Dropped (consumer app on Mobilise) | | | | | | | | | – |

---

## Recommended shortlist

### Primary: eSIM Access
1. **It's the only one that clears all three hard constraints at once:** instant self-serve signup with a $50 start, **published daily plans** (1GB/day and similar, with length discounts), and a webhook set that lets you automate "you're down to 100MB, top up?" and "expires tomorrow" messages. Those messages drive repeat revenue in an influencer-led business.
2. **It's friendly to building with Claude Code:** public docs, an official AI agent skill and an MCP server. You can start on the **$19/mo Express store charging your own Stripe account** to validate demand in days, then move to your own Next.js site and app through the API without changing supplier.
3. **Lower the risk before scaling:**
   - Deposit only $50-100 at first.
   - Order real eSIMs for your 3-5 hero destinations and check speeds and the country your IP appears in.
   - Open one support ticket and time the reply.
   - Get written answers on (a) when the daily allowance resets, (b) whether day passes can be extended or topped up, and (c) refund rules for uninstalled eSIMs.
   - Never keep more than about 2 weeks of sales in the balance (given the Trustpilot balance complaint).

### Backup 1: zendit (IDT)
- Self-serve, with a financially solid listed parent and claimed unlimited plans. Integrate it behind the same internal "provider" interface so you can fail over or price-compare per country. Confirm daily-reset offers, usage data and top-ups in test mode first.

### Backup 2: CELITECH (or Airalo Partners if Celitech won't take you)
- CELITECH has the best API and webhooks and US billing with no setup fees, but you'll need to sell yourself on a demo call and confirm daily plans.
- Airalo is the dependable fallback for coverage and quality, but you **can't undercut Airalo's retail price**, and some eSIMs show "Airalo" as the network name. Its **affiliate programme** is a zero-risk way to monetise the audience while the branded stack is being built.

**When to look again:** revisit **eSIM Go** once you can commit roughly $10k a month. Its unlimited daily plans are attractive, but the $30k first-90-days credit is out of reach today.

---

## Sources

*Accessed 2026-10-26 unless stated.*

**eSIM Access**
- https://esimaccess.com/
- https://esimaccess.com/esim-pricing-coverage/
- https://esimaccess.com/pricing/
- https://app.esimaccess.com/public/current-price
- https://esimaccess.com/docs/what-webhook-notifications-do-you-send/
- https://esimaccess.com/esim-access-api-agent-skill/
- https://mcp.esimaccess.com/
- https://esimaccess.com/docs/unlimited-day-pass-plans/
- https://app.esimaccess.com/faq
- https://esimaccess.com/sell-esims-with-your-stripe-account/ (Aug 7, 2025)
- https://esimaccess.com/docs/turnkey-shopify-esim-shop/
- https://esimaccess.com/new-apple-universal-link-for-esim-install/
- https://esimaccess.com/mapping-the-esim-reseller-journey-api-integration-beyond/ (Jul 25, 2024)
- https://www.trustpilot.com/review/esimaccess.com
- https://repo.hex.pm/preview/esim_access/0.1.0/README.md (third-party client)

**Redtea Mobile**
- https://redteamobile.com/news/post21/ (Jan 2026)
- https://www.mwcbarcelona.com/exhibitors/redtea-mobile-pte-ltd
- https://www.gsma.com/membership/m/redtea-mobile/

**eSIM Go**
- https://esimgo.com/
- https://esimgo.com/pricing
- https://docs.esim-go.com/api/v2_5/
- https://help.esim-go.com/hc/en-gb/articles/29646503718545-Understanding-Your-Bundle-Groups
- https://esim-go.com/product/solutions/breeze
- https://www.trustpilot.com/review/esim-go.com

**CELITECH**
- https://www.celitech.com/
- https://docs.celitech.com/
- https://docs.celitech.com/webhooks/event-catalog
- https://docs.celitech.com/account-management/sandbox
- https://docs.celitech.com/account-management/billing
- https://docs.celitech.com/getting-started/main-concepts

**Airalo Partners**
- https://developers.partners.airalo.com/faq-752238m0
- https://developers.partners.airalo.com/webhook-definition-1380483m0
- https://developers.partners.airalo.com/webhooks-guide-930005m0
- https://blog.partners.airalo.com/blog/airalo-partners-guide-to-esim-partnership-solution (Jun 2025, mod. Oct 2025)
- https://airalopartners.zendesk.com/hc/en-us/articles/21068014955293-Can-I-build-a-complete-white-label-experience-using-the-Airalo-Partner-API

**zendit**
- https://zendit.io/
- https://zendit.io/best-esim-api/ (Aug 24, 2026, vendor)
- https://developers.zendit.io/
- https://www.idt.net/zendit-extends-global-prepaid-platform-with-esim-initiative/ (Feb 2024)
- https://www.fintechfutures.com/press-releases/idt-introduces-zendit-a-global-prepaid-as-a-service-platform (2023)
- https://apis.apievangelist.com/store/zendit-esim-api/

**Telna**
- https://www.telna.com/
- https://www.telna.com/connect-flex
- https://telna.com/blog/connect-flex (Mar 2025)

**1GLOBAL / Truphone**
- https://www.1global.com/partnerships
- https://www.1global.com/about-us
- https://en.wikipedia.org/wiki/Truphone
- https://www.truphone.com/business/

**BICS**
- https://www.bics.com/iot/travel-esim-voice-sms/
- https://www.bics.com/iot/esim/travel-esim-for-mno/
- https://www.lightreading.com/5g/bics-gets-set-for-big-push-on-travel-esims

**Maya Mobile**
- https://api.maya.net/business/esim-api
- https://esim.holafly.com/business/maya-business-esim-review/ (competitor)
- https://saily.com/en/blog/maya-esim-review/ (competitor)
- https://apis.apievangelist.com/store/maya-mobile-esims-api/

**Keepgo / esimba**
- https://partnerfaq.keepgo.com/kb/general-faqs
- https://partnerfaq.keepgo.com/kb/general-faqs/what-are-the-major-partnership-models-available
- https://www.esimba.ai/
- https://en.wikipedia.org/wiki/Keepgo

**eSIMCard**
- https://esimcard.com/partners/
- https://esimcard.com/partners/reseller-api/
- https://www.trustpilot.com/review/esimcard.com

**Citrus Mobile**
- https://citrusmobile.com/

**Breathe / Mobilise**
- https://www.mobiliseglobal.com/white-label-esim-app/

**Bitcall**
- https://www.capterra.com/p/10036617/Bitcall/
- https://www.saasworthy.com/product/bitcall
- https://www.g2.com/products/bitcall
- https://www.producthunt.com/p/bitcall

**Others**
- MobiMatter: https://mobimatter.com/reseller-partner
- Telnyx: https://telnyx.com/products/esim
- TripoSIM: https://triposim.com/blog/best-esim-api-for-resellers-guide (Apr 2026)
- TripoSIM: https://triposim.com/blog/esim-reseller-prices-by-country-2026 (Sep 2026)
- Omax / Bappy: https://bappy.dog/blog/white-label-esim-pricing-compared/ (Mar 29, 2026, vendor)
- Omax / Bappy: https://www.omaxtelecom.com/blog/white-label-esim-pricing-compared/ (Feb 2026 data)
- Roamic: https://reseller.roamic.com/
- AirHub: https://airhubapp.com/partner-with-us/wholesale-program
- TechBullion roundup: https://techbullion.com/6-best-esim-apis-for-travel-agencies-and-otas-in-2026/ (Sep 15, 2026, sponsorship undisclosed)
