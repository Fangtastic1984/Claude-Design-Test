# Skill update sheet — answers from the 2026-06-11 spec session

Paste-ready resolutions for `[NEEDS CLARIFICATION]` / `[TO FILL]` markers in the
`toekomsrus-hub` skill that were answered while producing `SPEC.md`. Each entry names the
skill section and the replacement content. Markers not listed here remain genuinely open.

## §1 Platform Overview

- **Service providers charged?** → Yes, indirectly: *anyone who creates a listing* (need
  or offering) must hold an active R30/month subscription (or be in the 30-day trial).
  Browsing and responding are free for registered users. No commission on closed jobs.
- **Subscription tiers (free vs paid)** → Free: browse active listings, see contact
  buttons (registration required), order food, rate orders. Paid (R30/month): everything
  free users get + create listings + submit banners + analytics page (own listings'
  views/responses, own banners' impressions/clicks).
- **Payment provider** → **Ozow** (Lereo already holds API access). Used for both food
  checkout and subscription billing.
- **Billing cycle** → Monthly. Because Ozow is instant-EFT (not card-on-file), renewal is
  a monthly payment-request link sent at T-5 and T-1 days; 7-day `past_due` grace
  (posting disabled), then `expired`. 30-day free trial is automatic.
- **Refund policy (food orders)** → No refunds; disputes are between cook and customer.
  Post-delivery rating prompts are the quality mechanism. (Subscription refund policy:
  still open.)

## §2–3 Architecture & stack (forward state)

- **Hub v2 rebuild** (see `SPEC.md` in the Claude-Design-Test repo): single Next.js
  (TypeScript) app — public site, PWA, admin dashboard, driver screen, API routes —
  with PostgreSQL via Prisma in its **own container** (separate from `n8n-postgres-1`),
  in-process background jobs, Dockerised on the existing Afrihost VPS.
- **Deployment** → `hub.olifantcollab.co.za` alongside live WordPress; root-domain
  cutover + WordPress retirement is a separate reversible step after phase 7. Nothing
  from WordPress or the standalone taskboard/dashboard is migrated (fresh start).
- **`api.olifantcollab.co.za`** → not needed; the Hub app exposes its API on its own
  domain, including a scoped service-token API for n8n.
- **Email for the app** → existing self-hosted Postfix via `noreply@olifantcollab.co.za`
  (OTP + transactional). DMARC stays at `p=quarantine` per the standing decision.

## §3 Automation / integrations (forward state)

- **n8n role** → WhatsApp/messaging workflows **only**. All other automation (listing
  expiry, response-cap close, rating prompts, subscription reminders, analytics rollups)
  is absorbed into the Hub app's job scheduler. The app emits signed webhooks
  (`listing.created`, `order.status_changed`, `rating.prompt_due`, …) for n8n to consume,
  and n8n writes back (e.g. rating replies) via the service API.
- **WhatsApp integration method** → `wa.me` links only until Meta Business Verification
  clears (submitted 2026-02-06); notifications fall back to web push + in-app + email.
  OTP delivery channel is pluggable: email now, WhatsApp when WABA lands.

## §4 Data model (forward state)

- Full entity definitions now live in `SPEC.md` §4: users, otp_codes, subscriptions,
  subscription_payments, categories, subcategories (L3 taxonomy), listings,
  listing_responses, banners, vendors, menu_items, orders, order_items, order_events,
  order_payments, driver_profiles, order_ratings, settings, notifications,
  push_subscriptions, webhook_endpoints, audit_logs.
- **Versioning strategy** → Prisma migrations, committed to git, backward-compatible per
  the pre-deployment checklist.
- **Authentication** → email + 6-digit OTP on first login → user sets password → email +
  password thereafter; OTP machinery reused for password reset.

## §5 Features (new functions decided 2026-06-11)

- **Banners** — subscriber-submitted, admin-approved promotional posters; home-page
  carousel (first element users see) + category-page placement; date-range + priority
  scheduling; impression/click metrics. (Spec §6.)
- **Food & Delivery** — admin-managed vendor records (local shops submit menus to admin);
  flat menu items, `instant` (cart/delivery) vs `enquiry` (catering, cakes — WhatsApp
  button) kinds; one vendor per order; hybrid cash/Ozow payment; flat delivery fee
  (value TBC with partner); broadcast first-to-accept dispatch to drivers of one onboarded
  partner business; driver-mediated fulfilment with status-stage tracking (no GPS in v1);
  post-delivery rating prompts; no refunds. (Spec §7.)
- **Mobile** — PWA (installable, web push, offline shell, low-data budget), not native.
- **Response-cap auto-close** → confirmed in scope (was 🟡).
- **Pothole pilot** → explicitly out of scope for the rebuild codebase.
- **Language** → English only; strings centralised for later i18n.

## Suggested decision-log entries

The 2026-06-11 entries are already written in `SPEC.md` §14 in the skill's own
`Date · Decision · Why · What it replaces` format — copy them into skill §9 verbatim.
