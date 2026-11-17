# Expected Decisions — Scenario 05: Application Using a Payment Provider Without Recurring Billing Support

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Approach A — Application-Owned Subscription Implementation** (with invoice-driven renewal lifecycle and scheduled background jobs).
- **Why**:
  - The local payment gateway explicitly supports **only one-time redirect checkouts** (`docs/gateway-spec.md`) and cannot be replaced due to regional banking requirements.
  - **Approach B (Provider-Managed)** is impossible here because the gateway has no native subscription engine or saved-card tokenization API.
  - The application database must own `plans`, `subscriptions`, `subscription_invoices`, and `payment_webhook_events`.
- **Renewal & Period Extension Mechanics**:
  1. **Scheduled Pre-Expiry Invoice Job**: At $T - 7$ days before `current_period_end`, generate a `pending` renewal invoice (`subscription_invoices`) for the next cycle and send the student/school a payment notification link (`T-7`, `T-3`, `T-1`).
  2. **Expiry & Grace Period Transition**: When `now >= current_period_end` and the renewal invoice is still unpaid, transition `status` from `active` to `past_due` and set `grace_period_ends_at = current_period_end + 5 days`.
  3. **Hard Cutoff**: When `now >= grace_period_ends_at` without payment, transition `status` to `expired` and revoke access.
  4. **Deterministic Period Extension Math on Webhook**:
     - If paid *before* `current_period_end` or *during* the 5-day grace period (`status IN ('active', 'past_due')`): `new_period_end = old_period_end + interval` (continuous service).
     - If paid *after* expiration (`status == 'expired'`): `new_period_end = paid_at + interval` (fresh cycle from payment date).

## 2. Important Interview Questions
- Confirm how users pay for renewals without saved cards (invoice link via email/SMS/in-app banner).
- Confirm grace period length (5 days) and whether paying during the grace period extends from the old expiry date vs. payment date.
- Confirm whether subscriptions belong to the individual student (`user_id`), the `school_id`, or both.

## 3. Mandatory Security Considerations
- **Amount & Currency Verification on Form-Post Gateways**: Because redirect/form-post gateways send checkout parameters through the user's browser, an attacker can tamper with the form's `amount` or `currency` before submitting if only `order_id` is checked. The webhook handler (`notify_url`) **must** verify:
  1. `VerifyNotificationSignature(payload, secret)` is valid.
  2. `payload.status == 'SUCCESS'`.
  3. `payload.amount == invoice.amount_minor` AND `payload.currency == invoice.currency`.
- **Idempotent Period Extension**: If the gateway sends the webhook notification twice for the same `order_id` / `invoice_id`, the database transaction must ensure `invoice.status` transitions from `pending -> paid` only once so `current_period_end` is never extended by 2 months for 1 payment!

## 4. Unacceptable Recommendations (Automatic Fail)
- Recommending Approach B or inventing imaginary recurring APIs (`localgateway.CreateSubscription()`) that the gateway does not have.
- Telling the developer they must switch to Stripe when they stated the local gateway is mandatory.
- Failing to validate `paid_amount == expected_amount` and `paid_currency == expected_currency` in the webhook handler.
- Double-extending `current_period_end` on duplicate webhook delivery.
