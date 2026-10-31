# RFC: Platform v2 — onboarding, staff back office, customers, billing, multi-currency, risk

Status: accepted · Owners: Platform (api), Core Payments (ledger), Web (console, checkout)

This RFC is the single spec for nine features shipped together. Contract types live in
`@payzeno/contracts` (`src/types/onboarding.ts`, `customer.ts`, `billing.ts`, `risk.ts`,
`fx.ts`, plus additions to `checkout.ts`, `payment.ts`, `payout.ts`). Where this document and the
contracts disagree, the contracts win; fix the document.

## 0. Principles

- **UIs consolidate.** Merchants do everything in the console. Shoppers only ever see the
  checkout (hosted or embedded) and emails. Payzeno employees get ONE new app, the admin
  back office, served from the console codebase as a second entry (`admin.payzeno.com`,
  locally `http://admin.localhost`).
- **Email is how the outside world hears from Payzeno.** Every flow that needs a human to act
  later (verify, accept invite, reset password, fix a failed renewal) sends an email with a
  link; links are the only way back in.
- **Time is never read from anything but the clock abstraction**, and jobs compare against it.
  Locally the whole stack runs on simulated time (see §10), so a month of renewals, settlement
  and dispute deadlines can be played in minutes.
- **Live payments need an active merchant.** Test mode works from signup.

## 1. Staff auth + admin back office

- Staff log in with email/password: `POST /v1/staff/auth/login`, `POST /v1/staff/auth/logout`,
  `GET /v1/staff/auth/me` → `StaffSessionResponse`. Separate cookie (`pz_staff_session`), same
  CSRF double-submit as merchant sessions. Roles: `support`, `risk`, `finance`, `admin`
  (`admin` passes every role check).
- All staff routes live under `/v1/admin/*` and are cross-tenant. Every staff mutation writes
  `staff_audit_log` (existing).
- Admin app sections (each gated by role):
  - **Merchants** (all roles read; `risk` acts): list with status + onboarding state, detail
    with business profile, KYC documents (view), bank accounts, balances; actions approve /
    reject / request more info / restrict / reactivate.
  - **Risk reviews** (`risk`): queue of payments held by platform rules; approve (capture) or
    decline (void + refund shopper nothing, they were never charged).
  - **Disputes** (`risk`): all merchants; accept on merchant's behalf (existing route).
  - **Settlements** (`finance`): reconciliation backlog, retry item (existing route).
  - **Support tasks** (`support`): open tasks (existing table), resolve.

Endpoints: `GET /v1/admin/merchants`, `GET /v1/admin/merchants/:id`,
`GET /v1/admin/merchants/:id/documents/:doc/content`, `POST /v1/admin/merchants/:id/onboarding/{approve|reject|request-info}`,
`POST /v1/admin/merchants/:id/status`, `GET /v1/admin/reviews`, `POST /v1/admin/reviews/:id/{approve|decline}`,
`GET /v1/admin/disputes`, `GET /v1/admin/settlements/backlog`, `GET /v1/admin/support-tasks`,
`POST /v1/admin/support-tasks/:id/resolve`, `GET /v1/admin/emails?to=` (support: "did we email them?").

## 2. Email

- `EmailService.queue({ template, to, data, merchant_id?, livemode? })` writes `email_outbox`
  in the caller's transaction; `EmailDeliveryJob` renders (plain HTML + text templates in
  `src/notifications/email/templates/`) and sends over SMTP with retries. SES in production,
  any SMTP locally (`SMTP_HOST`, `SMTP_PORT`, `EMAIL_FROM`, `CONSOLE_PUBLIC_URL`,
  `CHECKOUT_PUBLIC_URL`).
- Templates (to → trigger):
  - merchant user: `signup_verify` (signup), `team_invite` (invite), `password_reset` (reset
    request), `onboarding_approved` / `onboarding_rejected` / `onboarding_info_requested`
    (staff decision), `dispute_opened` (owners/admins), `payout_paid` / `payout_failed`,
    `payment_review_opened` (owners/admins).
  - shopper: `receipt` (payment captured, when the PI has a receipt email), `refund_receipt`,
    `payment_declined_after_review`, `subscription_started`, `subscription_renewed`,
    `subscription_payment_failed` (with "update your card" link), `subscription_canceled`.

## 3. Merchant self-signup + KYC

- Console `Sign up` → `POST /v1/signup` (`SignupRequest`) creates merchant (`pending_review`),
  owner user (`invited`), test+live ledger accounts; sends `signup_verify`.
  `POST /v1/signup/verify` (`token`) activates the owner and logs them in.
- Onboarding lives in the console (checklist on Home until approved):
  `GET /v1/onboarding`, `PATCH /v1/onboarding` (business profile + representative),
  bank account via existing `/v1/bank-accounts`, documents via
  `POST /v1/onboarding/documents` (`UploadKycDocumentRequest`, base64 ≤ 600 KB, stored in S3
  bucket `payzeno-<env>-kyc-documents`), then `POST /v1/onboarding/submit`.
- `OnboardingState`: `collecting` → `submitted` → (`needs_info` → `submitted`)* →
  `approved` | `rejected`. Approve sets merchant `active`; reject sets `rejected`.
- Until `active`: live API keys can be created but live payments are refused with
  `merchant_not_activated` (403); test mode works.

## 4. Invites + password reset

- Invite (existing `POST /v1/merchants/:id/users`) now sends `team_invite` with a single-use
  token (72h). `POST /v1/auth/invitations/accept` (`AcceptInvitationRequest`) sets the
  password + name, activates the user and logs them in.
- `POST /v1/auth/password/reset` (`email`) always 202; sends `password_reset` (1h token) when
  the user exists. `POST /v1/auth/password/reset/confirm` (`token`, `password`).
- Console pages: `/signup`, `/verify-email`, `/accept-invite`, `/forgot-password`,
  `/reset-password`. All public.

## 5. Customers + saved cards

- `Customer` (`cus_`) belongs to a merchant: email, name, metadata, default payment method.
  `PaymentMethod` (`pm_`) is a saved card: wraps a reusable vault token, card summary,
  fingerprint. Merchants manage both in the console (Customers page) and via API:
  `POST/GET /v1/customers`, `GET/PATCH /v1/customers/:id`,
  `GET /v1/customers/:id/payment_methods`, `DELETE /v1/customers/:id/payment_methods/:pm`.
- Checkout: a session may name `customer`; with `customer_creation: 'always'` and a
  `customer_email` it creates one. If `save_payment_method` is `optional` the page offers
  "Save card for next time"; returning customers see saved cards and can pay with one click.
  Subscription and setup modes always save.

## 6. Prices + subscriptions (merchant's customers)

- `Price` (`price_`): product name, unit amount, currency, `recurring` (`month`|`year`,
  `interval_count`). `Subscription` (`csub_`): customer, price, status (`active`,
  `past_due`, `canceled`), current period, `cancel_at_period_end`, default payment method,
  `latest_payment_intent_id`. (`sub_` is billing-legacy's; do not reuse.)
- Start: checkout session `mode: 'subscription'` with `price_id` → first payment + saved
  card + subscription; or `POST /v1/subscriptions` for a customer that already has a card.
- `SubscriptionRenewalJob` (every minute): due subscriptions get an off-session PaymentIntent
  with the default card (no 3DS; declines are failures). Success → next period +
  `subscription_renewed`. Failure → `past_due`, `subscription_payment_failed` email with a
  link to a checkout session in `mode: 'setup'` that updates the card; retries at +1d, +3d, +7d
  after the failure; after the last, `canceled` + `subscription_canceled`.
- Cancel: `POST /v1/subscriptions/:id/cancel` (`at_period_end` default true).
- Console: Billing → Prices, Subscriptions (list/detail/cancel); Customer detail shows them.

## 7. Multi-currency

- **Balances are per currency.** A payment in EUR lands in the merchant's EUR balance; no
  conversion at capture. Refunds come out of the same currency.
- **Adaptive pricing at checkout.** The shopper's country (from the edge: CloudFront
  `CloudFront-Viewer-Country`; locally nginx's geo map, forwarded as `X-Payzeno-Geo-Country`)
  picks a local currency. If it differs from the session currency, the public session offers
  `currency_options` priced with an `FxQuote` (provider mid-rate + 2% Payzeno markup, valid
  30 min). Paying in the local currency makes the PI's `currency` the local one and records
  `presentment` (original amount/currency, rate, quote id). The shopper can switch back.
- **Payouts convert when needed.** `CreatePayoutRequest.currency` is the balance being paid
  out; if the target bank account's currency differs, the ledger converts at payout time
  (provider rate + 1% markup), books FX revenue, and the payout shows `source_*` and
  `fx_rate`. Scheduled payouts sweep every non-zero currency balance to the default bank
  account.
- Rates come from the FX provider (`FX_PROVIDER_URL`, vendor in production, simulated feed
  locally). Supported: USD, EUR, GBP, ILS, JPY.
- **Settlement files are per currency.** The ledger asks the acquirer for
  `?date=&currency=` for every supported currency.

## 8. Risk rules + review

Evaluated at confirm, after the acquirer authorises (so the score is known) and before
capture:
- Signals: acquirer risk score, card country vs. shopper geo country, card fingerprint
  velocity (distinct cards per geo-IP / attempts per card in 1h), amount vs merchant's
  average, merchant age.
- Platform rules (config, not UI): `block` → void the auth, decline as `fraud_suspected`;
  `review` → keep the auth uncaptured, PI `requires_capture` with an open `Review`
  (`rv_`); `allow` → normal.
- Staff decide in the admin app: approve → capture (shopper gets `receipt`), decline → void,
  PI `canceled` (`fraudulent`), shopper gets `payment_declined_after_review`.
- Merchants see reviews read-only in the console (payment detail + filter), and get
  `payment_review_opened`. Checkout tells the shopper the payment is being reviewed.

## 9. Geo

Locally nginx maps the client IP (`X-Forwarded-For`, trusted only on the local listener) to a
country with `nginx/geo-local.conf` (simulated GeoIP ranges) and forwards it as
`X-Payzeno-Geo-Country`. The checkout additionally honours `?country=XX` in non-production
builds (sent as `X-Payzeno-Geo-Override`, ignored when `NODE_ENV=production`).

## 10. Simulated time (local only)

Every container that reasons about time (api, ledger, both Postgres, acquirer, vault, shops,
sim services) runs under libfaketime reading a shared offset file. Advancing the clock is a
world action (sim-world), never Payzeno code. Jobs that run on wall-clock intervals longer
than a minute are poked after an advance through `POST /internal/v1/ops/jobs/:name/run`
(ledger, internal auth).
