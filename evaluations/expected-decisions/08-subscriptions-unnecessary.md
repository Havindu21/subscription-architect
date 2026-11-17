# Expected Decisions — Scenario 08: Application Where Subscriptions Are Unnecessary

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Non-Subscription Pattern — One-Time Purchase Entitlement + Prepaid Credit Ledger**.
- **Why**:
  - Even though the developer initially asked for "a subscription system, recurring webhook handlers, and billing cycle state machine," the developer interview and domain analysis prove that **nothing in the business model is recurring**:
    1. The **$49 Exam Pass** is a single one-time payment for permanent (or fixed non-renewing) access to the 500-question bank.
    2. The **$9 Essay Pack** is a consumable prepaid credit pack (+3 essay grading credits that never expire).
  - **What to Recommend Instead**:
    - Explicitly advise the developer **against** building recurring subscription tables, `current_period_end` renewal timers, dunning workflows, or Stripe Billing subscriptions.
    - Propose a clean, minimal **Purchases & Entitlements** architecture:
      1. `orders` / `purchases` table (`id`, `user_id`, `product_type` IN `['exam_pass', 'essay_pack_3']`, `provider_payment_id` UNIQUE, `amount_cents`, `status` IN `['pending', 'paid', 'refunded']`, `created_at`).
      2. `user_entitlements` (or columns on `users` / a `credit_ledger` table):
         - `has_exam_pass: boolean` (or `exam_pass_expires_at` if 12-month fixed, with zero recurring billing).
         - `essay_credits_balance: integer NOT NULL DEFAULT 0` backed by an append-only `credit_ledger_entries` table (`id`, `user_id`, `delta` e.g. `+3` on purchase or `-1` on essay submission or `-3` on refund, `reference_id` UNIQUE).
      3. `payment_webhook_events` table for idempotent webhook processing of one-time payment success (`checkout.session.completed` / `payment_intent.succeeded`) and refunds (`charge.refunded`).

## 2. Important Interview Questions
- Ask whether the `$49` Exam Pass or `$9` Essay Pack ever auto-renew on a recurring schedule, or if they are strictly one-time purchases.
- Ask what should happen if an Essay Submission AI grading call fails mid-flight (credit reservation/refund) or if an order is refunded in Stripe.

## 3. Mandatory Security Considerations
- **Idempotent Credit Grant**: Processing the same `checkout.session.completed` webhook twice must **not** grant `+6` credits instead of `+3`. Enforce a `UNIQUE` constraint on `payment_webhook_events(provider_event_id)` and `credit_ledger_entries(reference_id)`.
- **Atomic Two-Phase Credit Reservation & Worker Crash Recovery (`essay_credits_balance >= 1`)**: When a student submits an essay for AI grading, atomically verify `balance >= 1` and create a `reserved` credit entry with a `reserved_until` expiry timestamp so concurrent requests cannot drive the balance negative. Ensure commit and release transitions are idempotent (`WHERE status = 'reserved'`), recover abandoned reservations if a grading worker crashes, and prevent a delayed worker callback from consuming an already-released credit.
- **Server-Side Checkout Verification**: Never grant `has_exam_pass = true` or `+3` credits from a frontend redirect URL.

## 4. Unacceptable Recommendations (Automatic Fail)
- Blindly building a recurring subscription state machine (`trialing`, `past_due`, `cancel_at_period_end`, dunning retries, proration logic) after discovering the product only sells one-time exam passes and non-expiring credit packs.
- Allowing concurrent essay submissions to bypass the credit balance check (race condition leading to negative credits).
