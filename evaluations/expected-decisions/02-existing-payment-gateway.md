# Expected Decisions — Scenario 02: Application with an Existing Payment Gateway (No Subscriptions)

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Approach B (Provider-Managed Billing) or Approach D (Hybrid: Provider-Managed Billing + Local Monthly Download Quota Ledger)**.
- **Why**:
  - Reuse the existing `stripe` SDK, existing `users.stripe_customer_id`, and existing `POST /api/v1/webhooks/stripe` endpoint rather than creating a conflicting payment pipeline.
  - Preserve the existing `orders` table and one-time purchase logic intact.
  - Update the download authorization check (`can_download_template(user, template)`) to allow download if **either**:
    1. `has_purchased_template(user.id, template.id)` is true (existing one-time purchase), **OR**
    2. User has an active/grace-period subscription AND (has already downloaded `template_id` in this cycle OR `monthly_quota_used < 50`).
  - Note the important business nuance: Annual subscribers still get **50 downloads per month** (not 600 upfront). Do **not** assume `current_period_start..current_period_end` equals the quota reset window on annual plans. The agent must ask the developer to choose the quota reset model (**calendar-month**, **subscription-anniversary** derived from `current_period_start + N months`, or **rolling 30-day window**) and track `quota_period_start` / `quota_period_end` explicitly.

## 2. Important Interview Questions
- Ask which **quota reset model** applies to the 50-downloads/month limit (especially on the `$249/yr` annual plan): calendar month, monthly subscription anniversary, or rolling 30-day window—and whether re-downloading an already-downloaded template in the same quota period counts against the 50 limit.
- Confirm whether one-time purchases remain available alongside the subscription.
- Confirm grace period duration (`past_due`) on failed renewals.

## 3. Mandatory Security Considerations
- **Race-safe webhook idempotency**: Add a `payment_webhook_events` table with a `UNIQUE (provider, provider_event_id)` constraint and use **insert-first atomic event claiming** (`INSERT ... ON CONFLICT DO NOTHING`) so two concurrent duplicate deliveries of `checkout.session.completed` or `invoice.paid` cannot both pass a `SELECT` check and extend `current_period_end` twice.
- Use row-level locking (`SELECT ... FOR UPDATE` on the `subscription` row) when checking and incrementing the 50-download quota so concurrent download requests cannot bypass the 50-template cap.

## 4. Unacceptable Recommendations (Automatic Fail)
- Replacing or breaking the existing `orders` table and one-time template purchase flow.
- Creating a second, separate Stripe client or duplicate webhook endpoint when `app/services/stripe_service.py` and `/api/v1/webhooks/stripe` already exist.
- Ignoring the race condition on the 50-downloads/month quota counter.
