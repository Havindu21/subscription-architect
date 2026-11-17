# Expected Decisions — Scenario 10: Marketplace Using Provider-Managed Recurring Billing with Local Plans and Entitlements

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Approach D (Hybrid Implementation: Provider-Managed Recurring Billing via Stripe Connect + Application-Owned Plans & Entitlements)**.
- **Why**:
  - Explicitly refute the misconception that dynamic seller plans or custom entitlements require building an application-owned recurring billing scheduler (Approach A).
  - Separate the **five independent subscription concerns**:
    1. **Recurring Payment Scheduling & Collection**: Handled by Stripe Billing + Stripe Connect (automatic renewals, card retries, and `application_fee_percent` / destination charges).
    2. **Plan & Price Management**: Owned locally in the application's `analyst_plans` table (passing server-verified price snapshots/inline `price_data` or mapped Connect prices at checkout, with grandfathering for existing subscribers).
    3. **Subscription Ownership**: Scoped to `(subscriber_user_id, analyst_id)` in the local `subscriptions` table.
    4. **Application-Specific Entitlements**: Enforced locally against `analyst_plans.perks_json` and the local `subscriptions` table.
    5. **Marketplace Commission & Offboarding Policy**: Server-calculated 15% fee (`commission_bps = 1500`) and configurable offboarding workflow:
       - **Voluntary retirement/deactivation**: Sets `cancel_at_period_end = true` on active Stripe subscriptions so subscribers retain access until `current_period_end` without further charges.
       - **Compliance termination**: Immediately cancels active subscriptions (`ended_at = now`) and initiates prorated refunds.

## 2. Important Interview Questions
- Confirm whether Stripe Connect recurring billing covers all required countries and currencies.
- Confirm the exact **analyst offboarding policy** (end-of-period expiry vs. immediate cancellation/refund) so existing buyer entitlements are protected according to business policy.
- Confirm price grandfathering behavior when an analyst edits `analyst_plans.price_cents`.

## 3. Mandatory Security Considerations
- **Server-Authoritative Plan Price & Commission**: Never accept price or commission rate from the client; resolve `price_cents` from `analyst_plans` and `commission_bps` from `analyst_profiles` on the server.
- **Scoped Entitlements & Uniqueness**: Partial unique index on `subscriptions (subscriber_user_id, analyst_id) WHERE status IN ('trialing', 'active', 'past_due')`.
- **Race-Safe Webhook Idempotency**: Insert-first claiming on `payment_webhook_events (provider, provider_event_id)`.

## 4. Unacceptable Recommendations (Automatic Fail)
- Recommending Approach A (building a custom cron recurring billing and retry scheduler from scratch) solely because analysts define plans locally in the database.
- Assuming analyst deactivation always cancels buyer subscriptions immediately without honoring the end-of-period continuation policy for paid subscribers.
