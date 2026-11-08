+++
id = "P-024"
title = "Stripe payment integration: platform subscription billing"
status = "open"
created = "2026-11-08"
priority = "medium"
labels = ["billing"]
+++

## Idea

Add payments via Stripe, starting with a **platform subscription** model:
clinics/doctors pay KTB Health for access tiers. Stripe handles checkout,
invoicing, customer portal, and dunning. Architecture keeps the payment
provider behind a thin adapter so alternatives (Asaas, Mercado Pago,
Paddle) can be swapped without touching billing logic.

## Likely scope

- Stripe integration in `apps/web` (or a small `packages/billing`
  package): Checkout Sessions / Payment Links for plan purchase,
  webhooks for `checkout.session.completed`, `invoice.paid`,
  `customer.subscription.*` events
- Plan ↔ feature gating: subscription status on the doctor/clinic
  entity (P-022 registry), consulted by feature flags or entitlements
- Customer portal link for self-serve plan changes, payment method,
  cancellation
- Webhook idempotency + signature verification; secrets via Cloudflare
  secrets/env
- Brazil: Pix via Stripe's supported payment methods where available;
  BRL pricing, CNPJ/CPF fields on checkout if required

## Why it matters

Nothing is chargeable today. A subscription is the simplest monetizable
surface — no marketplace payouts, no per-consult split, no Stripe
Connect needed initially. It also forces the provider-abstraction
decision early, before per-transaction flows make it expensive.

## Positioning cautions

- Do NOT build Stripe Connect / doctor payouts yet — out of scope until
  there's a marketplace reason.
- Keep provider calls behind an interface; the codebase should not
  import `stripe` outside the billing module.
- Prices/plans are placeholders until business plan pricing is decided —
  plan config should be data-driven (env or DB), not hardcoded.
- Subscription state must fail safe: billing webhooks down ≠ access
  down; decide grace-period policy before launch.

## Depends on

- P-022 Doctor registry: multi-practitioner accounts (billing attaches
  to an org/clinic entity, not a single user)
- Possibly P-008 agenda / consult flows for "which plan features gate
  what"

## Open questions

- Self-serve signup→paid flow vs sales-assisted onboarding?
- Tier structure: free trial? per-doctor seat pricing?
- Stripe Brazil account vs US entity — affects Pix and tax handling.
