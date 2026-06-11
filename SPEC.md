# Toekomsrus Hub v2 — Technical Specification

> Status: **Draft for sign-off** · Date: 2026-06-11 · Author: drafted with Lereo's answers
> (Q&A session 2026-06-11) and the `toekomsrus-hub` platform skill as source material.
>
> This document specifies the full rebuild of the Toekomsrus Hub as a single modern codebase,
> plus two new functions (Banners, Food & Delivery) and PWA mobile delivery. Nothing from the
> current WordPress/standalone-HTML system is migrated; this is a clean start.

---

## 1. Scope

### In scope
1. **Taskboard** — rebuild of the existing needs/offerings marketplace (12-category taxonomy,
   L3 posting, response caps, 90-day expiry).
2. **Banners** — promotional poster section: subscriber-submitted, admin-approved, shown on
   the homepage (first thing users see) and on category pages.
3. **Food & Delivery** — structured food ordering from registered local shops with delivery
   by a partner three-wheeler business; hybrid cash/Ozow payment; status-stage tracking.
4. **PWA** — the whole platform installable as a mobile app (web push, offline shell,
   low-data optimised).
5. **Accounts & subscriptions** — email OTP first login, email+password thereafter; free vs
   R30/month tiers; 30-day free trial.
6. **Admin dashboard** — users, listings, banners, vendors, menus, orders, drivers, settings.
7. **Background jobs absorbed from n8n** — expiry, response-cap close, rating prompts,
   subscription reminders.

### Out of scope (explicit)
- Pothole pilot (Lereo, Q6).
- WordPress content migration — nothing survives (Q7).
- Refunds/dispute resolution flows — disputes are between cook and customer (Q21).
- Vendor self-service portal — admin manages all vendor data in v1 (Q13).
- Live GPS tracking — status stages only; data model leaves room for GPS later.
- WhatsApp Business API — blocked; `wa.me` links + in-app/push/email fallback (Q4).
- Languages other than English (Q22). Strings are centralised so i18n can be added later.
- Goods marketplace, Provider Directory, referral content type (skill §9 open questions —
  unchanged, still open).

### Principled exclusions (inherited, enforced)
The Hub will not facilitate: informal lending (mashonisa), informal medicine selling,
private guarding. Exclusion ethic applies to any future category: high-risk service +
no Hub vetting capacity = stay out.

---

## 2. Roles & permissions

| Capability | Visitor (no account) | Free user | Subscriber (R30/m) | Driver | Admin |
|---|---|---|---|---|---|
| Browse active listings & categories | ✅ | ✅ | ✅ | ✅ | ✅ |
| See listing contact / WhatsApp button | ❌ (prompt to register) | ✅ | ✅ | ✅ | ✅ |
| Create/edit own listings | ❌ | ❌ | ✅ | ❌ | ✅ |
| Analytics page (own listings + banners) | ❌ | ❌ | ✅ | ❌ | ✅ (all) |
| Submit banners | ❌ | ❌ | ✅ | ❌ | ✅ |
| Order food (cash or Ozow) | ❌ | ✅ | ✅ | ✅ | ✅ |
| Rate completed orders | ❌ | ✅ | ✅ | — | — |
| See/accept delivery jobs, advance order status | ❌ | ❌ | ❌ | ✅ | ✅ |
| Manage vendors, menus, banners, users, settings | ❌ | ❌ | ❌ | ❌ | ✅ |

Notes:
- **Free users browse listings but cannot create them** (Q3). Food ordering is open to any
  registered user — it drives vendor/driver revenue, so it is not paywalled.
- **Vendors are records, not logins** in v1: local shops register with admin, who captures
  their profile and menu (Q13). A vendor portal is a v2 candidate.
- **Drivers** belong to one onboarded partner delivery business (Q19). Admin creates driver
  accounts; drivers toggle on-duty/off-duty themselves.

---

## 3. Architecture

```
┌─────────────────────────────── existing self-hosted server (Docker) ───────────────────────────────┐
│                                                                                                     │
│  ┌──────────────────────────────┐   ┌────────────────┐   ┌─────────────────────────────────────┐   │
│  │ hub-app (Next.js, TS)        │   │ hub-postgres   │   │ n8n (existing containers, kept)     │   │
│  │  • public site + PWA         │──▶│ (PostgreSQL 16)│   │  • WhatsApp/messaging workflows ONLY │   │
│  │  • API route handlers        │   └────────────────┘   │  • consumes hub-app webhooks + API   │   │
│  │  • admin dashboard           │                        └─────────────────────────────────────┘   │
│  │  • driver app (route)        │   ┌────────────────┐                                             │
│  │  • in-process job scheduler  │──▶│ uploads volume │                                             │
│  └──────────────────────────────┘   └────────────────┘                                             │
│                          ▲ reverse proxy (nginx/caddy, TLS) ▲                                       │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
External: Ozow (payments) · Web Push (VAPID, no third party) · Claude API (used by n8n)
Local:    Postfix/Dovecot (existing self-hosted email stack — OTP + transactional via
          noreply@olifantcollab.co.za; SPF/DKIM/DMARC already configured)
```

### Hosting reality (from platform skill, 2026-06 revision)
- **Server**: Afrihost VPS `165.73.0.106`, Ubuntu 24.04 LTS, R400/month all-in. Existing
  services that must keep running untouched: nginx (TLS via Let's Encrypt, auto-renew),
  WordPress (live site with published Privacy Policy & ToS), Postfix/Dovecot mail stack,
  UFW + Fail2Ban, n8n containers. RAM/CPU headroom **[TBC — check before phase 7]**.
- **Coexistence, then cutover**: the live WordPress site at `olifantcollab.co.za` stays up
  during the build. Hub v2 deploys to **`hub.olifantcollab.co.za`** (new nginx server block
  + Let's Encrypt cert) alongside it. Root-domain cutover (and WordPress retirement) is a
  separate, reversible step at the end of phase 7 — Privacy Policy and ToS pages are
  recreated in the app before cutover so the published URLs never go dark.
- **Email**: OTP and transactional mail go through the local Postfix relay — no external
  provider needed. Respect the standing DMARC decision: stays at `p=quarantine`; the app
  adds no new sending domains or `From` identities beyond `noreply@olifantcollab.co.za`.
- **Database**: PostgreSQL 15 already runs for n8n (`n8n-postgres-1`, localhost-only).
  Hub gets its **own postgres container** so Hub load/upgrades never touch n8n (now firm,
  not a deploy-time call). MySQL/WordPress is untouched until retirement.

- **One repo, one deployable app.** Next.js (App Router, TypeScript) serves public pages,
  the PWA, the admin dashboard, the driver screen, and all API routes. Prisma owns the
  PostgreSQL schema and migrations.
- **n8n stays for messaging only** (Q5). The app emits webhooks (order events, rating
  prompts, new listings) that n8n workflows can subscribe to for WhatsApp engagement once
  WABA is resolved. All other former n8n duties (expiry, caps, reminders) move into the
  app's job scheduler.
- **Jobs**: in-process scheduler (cron-style) inside the app container — no extra
  infrastructure on a single server. Every job is idempotent and logged.
- **Database**: a new `hub` PostgreSQL database in its own container, separate from
  `n8n-postgres-1` (see Hosting reality below).
- **Files** (banner images, menu photos): stored on a Docker volume, served via the app
  with size/type validation; images resized server-side to web-friendly variants.
- **Domains**: `hub.olifantcollab.co.za` for the app during build and initial launch;
  root-domain cutover decided at end of phase 7 (see Hosting reality).

---

## 4. Data model

Conventions: all tables have `id` (uuid), `created_at`, `updated_at`. Monetary values in
**ZAR cents** (integer). Soft state via status enums, not deletes, wherever history matters.

### 4.1 Identity & subscription

**users**
| field | type | notes |
|---|---|---|
| email | text, unique, required | login identifier |
| password_hash | text, nullable | null until first OTP login completes and password is set |
| name | text, required | display name |
| phone | text, required | E.164; used for wa.me links and driver/vendor contact |
| role | enum: `user`, `driver`, `admin` | subscriber-ness lives on `subscriptions`, not here |
| status | enum: `active`, `suspended` | |
| area_note | text, nullable | free-text locality; geographic gating is soft (open question in skill §1 stays open) |

**otp_codes** — `user_id`, `code_hash`, `purpose` (`signup`, `password_reset`), `expires_at`
(10 min), `consumed_at`. Delivered by email in v1; delivery channel is abstracted so
WhatsApp OTP can replace email when WABA lands (Q2).

**subscriptions**
| field | type | notes |
|---|---|---|
| user_id | fk users | one active subscription per user |
| status | enum: `trialing`, `active`, `past_due`, `cancelled`, `expired` | |
| trial_ends_at | timestamptz | signup + 30 days, automatic on first subscription intent |
| current_period_end | timestamptz | |
| price_cents | int | 3000 |

**subscription_payments** — `subscription_id`, `amount_cents`, `provider` (`ozow`),
`provider_ref`, `status` (`pending`, `paid`, `failed`), `paid_at`.
> Billing model: Ozow is instant-EFT, not card-on-file, so v1 renewal is a **monthly payment
> request**: 5 and 1 days before `current_period_end` the user gets a notification with an
> Ozow payment link; on webhook confirmation the period extends 30 days; unpaid at period end
> → `past_due` (7-day grace, posting disabled) → `expired`. If Ozow recurring debit becomes
> available on Lereo's account, swap in behind the same interface. **[TBC with Ozow account]**

### 4.2 Taxonomy & taskboard

**categories** — the 12 top-level categories, seeded from `Toekomsrus_Hub_Taxonomy_v0_12.md`:
`slug`, `name`, `lucide_icon`, `sort_order`, `active`.
1 Clothing & Laundry · 2 Building & Construction · 3 Electrical · 4 Plumbing ·
5 Food & Cooking · 6 Transport & Vehicles · 7 Digital & Internet · 8 Health & Care ·
9 Money & Finance · 10 Childcare & Schooling · 11 Safety & Security · 12 Leisure & Social

**subcategories** — L3 pick items: `category_id`, `slug`, `name`, `form_hints` (text[],
the L4–L5 enrichment), `search_synonyms` (text[]), `sort_order`, `active`. Posters pick at
L3; L4–L5 never appear as pick items (depth rule). Uneven depth across branches is correct.

**listings**
| field | type | notes |
|---|---|---|
| user_id | fk users | poster; must hold `trialing`/`active` subscription at creation |
| type | enum: `need`, `offering` | |
| title | text, required, ≤120 chars | |
| description | text, required, ≤2000 chars | |
| category_id / subcategory_id | fk | poster picks subcategory (L3); category derived |
| location | text, required | free text within Toekomsrus |
| contact_phone | text, required | drives the wa.me button |
| response_cap | int, required, 1–20 | |
| response_count | int, default 0 | incremented atomically (`UPDATE … WHERE response_count < response_cap`) — race-safe per checklist |
| duration_days | int, 1–90 | poster choice, hard max 90 |
| expires_at | timestamptz | created_at + duration_days |
| status | enum: `active`, `closed_cap`, `closed_expired`, `closed_manual`, `removed` | |
| view_count | int | for the subscriber analytics page |

**listing_responses** — `listing_id`, `responder_user_id`, `created_at`. Created when a
registered user taps the WhatsApp/contact button (this is what increments `response_count`
and powers analytics). One response per user per listing.

Rules carried over from the decision log:
- **Category display rule**: browse view shows only categories with ≥1 active listing.
- **Response-cap auto-close**: when `response_count == response_cap`, status →
  `closed_cap` immediately (Q8). 90-day expiry enforced by nightly job (§11).

### 4.3 Banners

**banners**
| field | type | notes |
|---|---|---|
| owner_user_id | fk users | submitting subscriber, or admin |
| image_path | text, required | poster image; validated, resized to slot variants |
| headline | text, nullable, ≤80 | optional overlay/caption |
| link_type | enum: `food_page`, `whatsapp`, `none` | per Q11 |
| link_value | text, nullable | wa.me number for `whatsapp`; vendor slug optional for `food_page` |
| placement | enum: `home`, `category` | `category` requires `category_id` |
| category_id | fk, nullable | which category page |
| status | enum: `pending`, `approved`, `rejected`, `archived` | subscriber submits → admin approves (Q10) |
| starts_at / ends_at | timestamptz, nullable | simple date-range scheduling (Q12) |
| priority | int, default 0 | higher first; ties rotate |
| impressions / clicks | int counters | feeds the analytics page |

Display: the **home banner carousel is the first element on the homepage** (Q9) — a
full-width swipeable carousel of approved, in-window banners ordered by priority. Category
pages show their banners above the listing grid. No audience targeting in v1 (Q12).

### 4.4 Food & delivery

**vendors** — admin-managed records (Q13): `name`, `description`, `phone` (shop contact),
`location`, `image_path`, `open_hours` (per-day open/close), `is_open_override` (admin
on/off switch), `status` (`active`, `paused`), `rating_avg`, `rating_count`.

**menu_items** — flat items (Q15): `vendor_id`, `name`, `description`, `price_cents`,
`image_path`, `kind` enum: **`instant`** (orderable now, goes in cart) or **`enquiry`**
(catering, wedding cakes, pre-packed bulk meals — shows a WhatsApp-enquiry button instead
of add-to-cart; resolves Q14's "other food functions" without a second data model),
`sold_out` (bool, admin toggle), `active`.

**orders**
| field | type | notes |
|---|---|---|
| customer_user_id | fk users | any registered user |
| vendor_id | fk vendors | **one vendor per order** (Q16) |
| status | enum — see state machine §7.3 | |
| payment_method | enum: `cash`, `ozow` | hybrid (decision) |
| payment_status | enum: `n/a` (cash), `pending`, `paid`, `failed`, `abandoned` | |
| items_total_cents / delivery_fee_cents / total_cents | int | fee frozen at order time from settings |
| delivery_pin | point (lat/lng), required | pin-drop (Q20) |
| delivery_landmark | text, required | "blue gate opposite the spaza", etc. |
| contact_phone | text, required | prefilled from profile, editable |
| driver_id | fk users, nullable | set on first-accept |
| placed_at / accepted_at / collected_at / delivered_at / cancelled_at | timestamptz | stage timestamps; doubles as audit trail and leaves room for GPS later |
| cancel_reason | text | |

**order_items** — `order_id`, `menu_item_id`, `name_snapshot`, `price_cents_snapshot`,
`quantity`. Snapshots so menu edits never rewrite history.

**order_events** — append-only: `order_id`, `actor` (`customer`/`driver`/`admin`/`system`),
`from_status`, `to_status`, `note`, `created_at`. Immutable audit log per checklist.

**order_payments** — `order_id`, `provider` (`ozow`), `provider_ref`, `amount_cents`,
`status`, `raw_webhook` (jsonb), `paid_at`. Ozow webhook is verified by hash check and
treated as the only source of payment truth.

**driver_profiles** — `user_id` (role=driver), `partner_business` (text — the onboarded
three-wheeler business, Q19), `vehicle_note`, `on_duty` (bool, self-toggled),
`last_seen_at`.

**order_ratings** — `order_id` (unique), `customer_user_id`, `stars` (1–5), `comment`
(nullable), `prompted_at`, `submitted_at`. Updates vendor `rating_avg/count` on submit.
Per Q21: ratings are the quality mechanism; **no refunds, no platform dispute flow** —
the dispute is between cook and customer; admin can suspend repeat-offender vendors.

### 4.5 Platform plumbing

- **settings** (key/value, admin-editable): `delivery_fee_cents` (flat fee, Toekomsrus —
  **value TBC with delivery partner**, Q17), `delivery_area_note`, `order_accept_timeout_min`,
  `subscription_price_cents`, banner slot limits.
- **notifications** — in-app inbox: `user_id`, `kind`, `title`, `body`, `link`, `read_at`.
- **push_subscriptions** — web-push endpoints per user/device (VAPID).
- **webhook_endpoints / webhook_deliveries** — outbound events for n8n (§10), with
  signature, retries, and delivery log.
- **audit_logs** — admin/destructive actions: actor, action, entity, before/after (jsonb).

---

## 5. Taskboard specification

### Browse (public)
- Homepage: banner carousel → category cards (Lucide icons; **only categories with ≥1
  active listing**) → latest listings strip.
- Category page: category banners → subcategory filter chips → listing cards
  (title, type badge Need/Offering, subcategory, location, response counter `2/3`,
  time-left). Search box matches title/description **and subcategory `search_synonyms`**.
- Listing detail: full description, poster name, location, counter, **WhatsApp button**
  (`wa.me/<contact_phone>?text=<prefill referencing the listing>`). Visitors see the
  listing but the contact button prompts registration; registered users tapping it create
  a `listing_response` (atomic increment; at cap → `closed_cap` and the button disables
  with "This listing has reached its response limit").

### Post (subscribers only)
Form: type → category card → subcategory (L3 pick list, with that branch's `form_hints`
shown as helper text) → title, description, location, contact phone (prefilled),
response cap (1–20), duration (≤90 days). Server-side validation mirrors every client
rule; all input sanitised, all output escaped (checklist §10 of the skill is the gate).
Listings publish immediately (no pre-moderation; admin can remove).

### Manage & analytics (subscribers)
"My listings": status, counters, close-early, edit (edits don't reset expiry).
**Analytics page** (the paid perk, Q3): per-listing views and responses over time,
plus the user's banner impressions/clicks. Simple counters + daily rollup table — no
external analytics dependency. GA4 can be layered on the public site separately.

---

## 6. Banners specification

- **Submission** (subscribers): upload image (jpg/png/webp ≤2 MB; server resizes to slot
  sizes), optional headline, link type (Food page / WhatsApp / none), placement (home or a
  category), requested date range. Status `pending`.
- **Moderation** (admin): approve/reject with reason; set/override priority and dates;
  archive at any time. Rejection notifies the submitter with the reason.
- **Display**: approved + within date window. Home carousel = first element on the homepage,
  auto-advance, swipeable, lazy-loaded images. Category pages show that category's banners
  above listings. Multiple active banners in one slot: order by priority, rotate within ties.
- **Metrics**: impression counted once per banner per page-view, click on tap; visible to
  the owner on the analytics page and to admin in aggregate.
- v2 candidates (not built): paid banner slots, time-of-day scheduling, targeting.

---

## 7. Food & delivery specification

### 7.1 Customer flow
1. **Food page**: banner-linkable landing; grid of `active` vendors (photo, name, rating
   stars, open/closed from `open_hours` + override). Closed vendors visible but not orderable.
2. **Vendor page**: menu split into "Order now" (`instant` items) and "Enquiries"
   (`enquiry` items → WhatsApp button with prefilled item reference; no cart).
3. **Cart**: single vendor; adding from another vendor prompts to clear. Quantities,
   items total + flat delivery fee = total.
4. **Checkout** (registered users; visitors are sent through signup and return to cart):
   delivery **pin-drop on map + required landmark note + contact phone** (Q20);
   payment choice **Cash on delivery** or **Pay now with Ozow**.
   - Cash → order goes straight to `broadcasting`.
   - Ozow → order `awaiting_payment`; redirect to Ozow; webhook `paid` → `broadcasting`;
     failed/abandoned (30 min) → order closed, nothing broadcast, customer notified.
5. **Tracking page**: live status stepper (placed → driver assigned → at the shop →
   on the way → delivered), driver name + wa.me button once assigned, order summary.
   Updates via lightweight polling (PWA-friendly, low-data); push notification per stage.

### 7.2 Driver flow (mobile-first route in the same app)
- On-duty toggle. **Job board: all `broadcasting` orders visible to all on-duty drivers;
  first to tap Accept wins** (atomic claim — `UPDATE … SET driver_id WHERE driver_id IS
  NULL`; losers get "already taken"). Job card shows vendor, item summary, delivery pin
  distance, payment method (cash amount to collect vs prepaid), fee.
- Active job screen: vendor address/phone, customer landmark + pin (opens device maps),
  customer wa.me; buttons advance the stages; **Release** returns the order to
  `broadcasting` with a reason (the escape hatch for "shop can't fulfil", since vendors
  have no portal).
- Cash orders: driver collects `total_cents` from the customer. Cash settlement between
  driver, shop, and partner business happens **off-platform** in v1; the order record is
  the reconciliation source. **[Settlement mechanics TBC with the delivery partner — same
  conversation as the fee, Q17/Q19]**
- If no driver accepts within `order_accept_timeout_min` (default 20): admin alerted;
  at 2× timeout the order auto-cancels and the customer is notified (Ozow-paid orders
  flagged to admin for manual resolution — accepted consequence of "no refund flow";
  expected to be rare and admin-visible).

### 7.3 Order state machine
```
                      [ozow]                       [cash]
 placed ──► awaiting_payment ──paid──► broadcasting ◄── placed
                    │                    │      ▲
              failed/abandoned       accept │      │ release (driver, with reason)
                    ▼                    ▼      │
              payment_failed        driver_assigned ──► at_vendor ──► out_for_delivery ──► delivered
                                         │                                                  │
 cancelled ◄── (admin, any pre-delivery state; system on broadcast timeout)        rating prompt (job, +30 min)
```
Every transition writes an `order_event`. `delivered` is terminal; `cancelled` and
`payment_failed` are terminal.

### 7.4 Ratings (Q21)
30 minutes after `delivered`, a job sends the customer a rating prompt (push + in-app;
**webhook also emitted so the n8n WhatsApp workflow can deliver the same prompt and write
the reply back via API once WABA exists** — this matches Lereo's "automated message,
they reply, rating saved" intent). One rating per order, 1–5 stars + optional comment;
vendor aggregate shown on the Food page. Admin sees all ratings/comments.

---

## 8. Accounts, auth & subscription flows

- **Sign-up / first login (Q2)**: email + name + phone → 6-digit OTP emailed (10-min
  expiry, 5 attempts, resend throttle) → verified → **set password** → session. Thereafter
  email + password; "forgot password" reuses the OTP machinery. OTP delivery is a pluggable
  interface (`email` now, `whatsapp` later when WABA resolves).
- Sessions: secure HTTP-only cookies; standard rate limiting on auth endpoints.
- **Subscription (Q3 + skill)**: any user can start the 30-day trial (full subscriber
  capabilities). 5 and 1 days before period end: Ozow payment-request link by notification +
  email. Paid → +30 days. Unpaid → `past_due` (7-day grace: existing listings stay live,
  posting and banner submission disabled) → `expired` (analytics hidden too). Listings of
  lapsed users run to their natural expiry; they are not pulled down.

---

## 9. PWA & mobile experience

- **Installable PWA** (decision): manifest + service worker; install prompts on Android;
  iOS via Safari add-to-home-screen.
- **Offline**: app shell + last-viewed listings/categories cached (stale-while-revalidate).
  Posting, ordering, and payment require connectivity and say so clearly.
- **Low-data discipline** (low-end Android on mobile data is the design target): server
  rendering, responsive `webp` variants, lazy images, no heavy client libraries, polling
  over websockets, aggressive HTTP caching. Page-weight budget: <300 KB first load on the
  homepage.
- **Web push** (VAPID, free, no third party): order stages for customers, new-job alerts
  for on-duty drivers, rating prompts, subscription reminders, banner moderation results.
  Email is the fallback for users who decline push.
- **Priority of polish (Q23)**: taskboard browsing/posting is the flagship mobile
  experience and gets first attention in every build phase; food ordering second; the
  driver screen is mobile-only by design.

---

## 10. n8n & external integration surface

The app emits **signed outbound webhooks** (HMAC header, retries with backoff, delivery
log) that n8n subscribes to for the WhatsApp/messaging layer (Q5):
`listing.created`, `listing.closed`, `order.placed`, `order.status_changed`,
`order.delivered`, `rating.prompt_due`, `subscription.renewal_due`, `user.registered`.

A scoped **service API token** lets n8n write back: submit a rating reply
(`POST /api/integrations/ratings`), look up a user by phone, fetch order status. The
Claude API stays where it is today — inside n8n workflows; the app itself has no direct
Claude dependency in v1 (AI listing extraction remains parked, per the decision log).

Inbound webhooks: **Ozow** payment notifications (order + subscription), hash-verified,
idempotent by `provider_ref`.

---

## 11. Background jobs (absorbed from n8n)

| Job | Schedule | Action |
|---|---|---|
| Listing expiry | hourly | `active` + past `expires_at` → `closed_expired`; notify poster |
| Expiry warning | daily | poster notified 7 days before expiry (renew = post anew, per current model) |
| Broadcast timeout | every 5 min | unaccepted orders: alert admin at T, auto-cancel at 2T |
| Payment abandonment | every 10 min | `awaiting_payment` > 30 min → `payment_failed` |
| Rating prompts | every 15 min | delivered + 30 min, not yet prompted → prompt + webhook |
| Subscription reminders | daily | T-5/T-1 payment links; period-end → `past_due`; grace-end → `expired` |
| Analytics rollup | nightly | daily counters for the analytics page |
| Backup | nightly | `pg_dump` + uploads volume → off-server target; restore verified before launch (checklist) |

All jobs idempotent, logged, and surfaced on an admin "job health" panel (silent failure
is a checklist violation).

---

## 12. Page map

```
/                       homepage: banner carousel · category cards · latest listings
/c/[category]           category page: banners · subcategory chips · listings
/l/[listing]            listing detail
/post                   create listing (subscriber)
/me                     my listings · my orders · subscription · settings
/me/analytics           subscriber analytics
/me/banners             my banners + submit
/food                   vendor grid
/food/[vendor]          menu (order-now + enquiries)
/cart  /checkout        cart · pin-drop checkout
/orders/[id]            order tracking
/driver                 job board · active job   (role: driver)
/admin/*                dashboard: listings · users · subscriptions · banners · vendors ·
                        menus · orders · drivers · ratings · settings · job health · audit log
/login  /register       auth (OTP first login, then password)
```

---

## 13. Build phases

1. **Foundation** — repo scaffold (Next.js + TS + Prisma + Docker Compose), schema +
   migrations + seeds (12 categories, L3 subcategories from the taxonomy file), auth
   (OTP → password), roles, settings, notification/push plumbing, PWA shell.
2. **Taskboard** — browse/search/detail, posting, response-cap close, expiry jobs,
   my-listings. *The existing platform is fully recreated at the end of this phase.*
3. **Subscriptions & analytics** — trial, Ozow payment requests, gating, analytics page.
4. **Banners** — submission, moderation, carousel + category placement, metrics.
5. **Food & delivery** — vendors/menus (admin), food pages, cart/checkout, Ozow order
   payments, driver job board + state machine, tracking, ratings.
6. **Admin & ops hardening** — remaining admin panels, audit log, job health, backups,
   webhook layer for n8n, pre-deployment checklist pass (skill §10 — run as a gate).
7. **Deploy** — Docker Compose to the Afrihost VPS beside n8n and the mail stack; new
   nginx server block + cert for `hub.olifantcollab.co.za`; monitoring/alerting; verified
   backup restore; recreate Privacy Policy & ToS in-app; then (as a separate, reversible
   step) root-domain cutover and WordPress retirement.

Each phase ends in a deployable state; phases 2 and 5 each conclude with a review
checkpoint with Lereo before the next begins.

---

## 14. Decision log (this spec)

> Format: Date · Decision · Why · What it replaces

- **2026-06-11** · Clean rebuild, no data migration; WordPress/Elementor + standalone HTML
  fully replaced by one Next.js codebase · Pre-launch platform, nothing must survive (Q1, Q7)
  · Replaces: WP + HTML + scattered React.
- **2026-06-11** · Stack: Next.js (TS) + PostgreSQL/Prisma + in-process jobs + Docker on the
  existing server · One deployable, fits single-server reality, no objections (Q24–25)
  · Replaces: multi-system stack.
- **2026-06-11** · PWA, not native · One codebase, no store friction, fits low-end Android
  target · Replaces: undecided.
- **2026-06-11** · Auth: email OTP first login → set password → email+password thereafter;
  OTP channel pluggable for future WhatsApp · WABA blocked (Q2, Q4) · Replaces: WP auth.
- **2026-06-11** · Free = browse only; R30/month = post + analytics; food ordering open to
  all registered users · Lereo (Q3); food paywall would strangle the new revenue loop
  · Replaces: undefined gating.
- **2026-06-11** · Ozow for food checkout AND subscription (monthly payment-request links,
  not card-on-file) · Existing Ozow API (Q18); resolves the skill's open "payment provider"
  item · Replaces: provider TBD.
- **2026-06-11** · n8n retained for WhatsApp/messaging only, fed by signed app webhooks;
  all other automation absorbed as app jobs · Lereo (Q5) · Replaces: n8n as general
  automation layer.
- **2026-06-11** · Banners: subscriber-submitted, admin-approved; home carousel is the first
  homepage element + category placements; date-range + priority only; impression/click
  metrics · Lereo (Q9–12) · Replaces: prototype BannerCarousel.
- **2026-06-11** · Food: admin-managed vendors, flat menu items with `instant` vs `enquiry`
  kinds, one vendor per order · Lereo (Q13–16); `enquiry` covers catering/cakes without a
  second data model · Replaces: nothing (new function).
- **2026-06-11** · Delivery: broadcast first-to-accept dispatch; **driver-mediated
  fulfilment** (driver advances all stages, may release back to broadcast) · No vendor
  portal exists to confirm orders; smallest workable ops model · Replaces: nothing.
- **2026-06-11** · Status-stage tracking + notifications; no GPS in v1 (timestamps/model
  leave room) · Data/battery/complexity vs value at this scale · Replaces: nothing.
- **2026-06-11** · Flat delivery fee (admin setting), cash settled off-platform, **no
  refunds/dispute flow**; post-delivery rating prompt is the quality mechanism · Lereo
  (Q17, Q21); fee value and settlement TBC with delivery partner · Replaces: nothing.
- **2026-06-11** · English only; strings centralised for later i18n · Lereo (Q22)
  · Replaces: undecided.
- **2026-06-11** · Pothole pilot out of scope for this codebase · Lereo (Q6) · Replaces:
  its standing as first live use case *within this rebuild* (the pilot itself is unaffected).
- **2026-06-11** · OTP + transactional email via the existing self-hosted Postfix stack
  (`noreply@olifantcollab.co.za`), honouring the standing DMARC `p=quarantine` decision ·
  Stack is operational with SPF/DKIM/DMARC (skill 2026-06 revision); no external provider
  cost · Replaces: "SMTP provider TBC" open item.
- **2026-06-11** · Hub gets its own PostgreSQL container, separate from `n8n-postgres-1` ·
  Hub load/upgrades must never touch n8n · Replaces: deploy-time call.
- **2026-06-11** · Deploy to `hub.olifantcollab.co.za` alongside the live WordPress site;
  root-domain cutover is a separate reversible step after phase 7 · Live site (legal pages,
  email, analytics dashboard) must not go dark mid-build · Replaces: implied immediate
  replacement of the root domain.

## 15. Open items (TBC before or at the relevant phase)

1. **Delivery fee value + cash settlement mechanics** — confirm with the three-wheeler
   partner business (blocks phase 5 launch, not its build).
2. **Ozow recurring** availability on Lereo's account (payment-request fallback already
   specced either way).
3. **Server headroom check** — Afrihost VPS RAM/CPU specs unknown; verify the box can run
   hub-app + hub-postgres alongside WordPress/MySQL, n8n, and the mail stack (phase 7;
   if tight, WordPress retirement frees resources at cutover).
4. **WABA resolution** — Meta Business Verification submitted 2026-02-06, still pending.
   When approved: WhatsApp OTP, order notifications, and the n8n rating reply loop
   activate against interfaces already in place.
5. **Root-domain cutover timing** — when does `olifantcollab.co.za` switch from WordPress
   to the Hub app (and WordPress retire)? Lereo's call after phase 7 ships on the subdomain.
6. **Launch-timing alignment** — the skill targets a June 2026 launch (~100–150 cohort)
   on the *existing* React taskboard, while this rebuild lands phase-by-phase. Needs an
   explicit call: launch on the old taskboard and migrate later, or hold launch for
   phase 2 of the rebuild (which recreates the platform). Fresh-start decision (Q1) makes
   "launch old, rebuild quietly, swap" messy if real listings accumulate in the old system.
7. Inherited and still open from the skill: goods marketplace, provider directory,
   referral content, multi-category bundled postings, taxonomy validation with other
   residents, out-of-area signup policy, staging environment.
