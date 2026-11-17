# Expected Decisions — Scenario 03: Application with an Existing Subscription Implementation

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Incremental Refactoring & Hardening of Existing Approach B Implementation** (Expand-Contract schema evolution, not a destructive rewrite).
- **Why**:
  - There are ~400 active paying subscribers in production using the existing `subscriptions` table.
  - The agent must identify the **three critical bugs/vulnerabilities** in the existing implementation during Step 1 (Repository Inspection) and highlight them clearly in the proposal:
    1. **Security Vulnerability**: `GET /api/billing/success?plan=pro` trusts query parameters to activate subscriptions without server-side Stripe verification.
    2. **Lifecycle Bug**: `POST /api/billing/cancel` sets `status = 'canceled'` immediately even though Stripe is set to `cancel_at_period_end: true`, prematurely revoking paid access.
    3. **Reliability Bug**: Webhook handler lacks `payment_webhook_events` deduplication, row locking, and out-of-order timestamp/state protection.
  - Extend `subscriptions` non-destructively (adding `workspace_id`, `cancel_at_period_end`, `scheduled_plan_code`, `grace_period_ends_at`, and `updated_at`) while backfilling/supporting existing `user_id` rows.

## 2. Important Interview Questions
- Clarify how existing solo `user_id` subscriptions map to the new `workspaces` model (e.g., belong to the user's default workspace vs. dual owner support).
- Confirm upgrade proration behavior (`create_prorations` / immediate charge) vs. end-of-period downgrade behavior.
- Confirm production migration constraints (~400 live subscribers).

## 3. Mandatory Security Considerations
- **Immediately remediate the `/api/billing/success` privilege escalation vulnerability**: Replace query-param mutation with a server-to-server `stripe.checkout.sessions.retrieve(sessionId)` check verifying `payment_status === 'paid'` and reading `metadata.plan_code` and `metadata.owner_id` set by the server.
- Separate `cancel_at_period_end: boolean` from `status: 'active'` so canceling mid-cycle keeps access until `current_period_end`.
- Add `payment_webhook_events` and transactional locking to prevent out-of-order webhooks from corrupting subscription state during upgrades.

## 4. Unacceptable Recommendations (Automatic Fail)
- Failing to spot the insecure `GET /api/billing/success?plan=pro` activation bug during repository inspection.
- Proposing `DROP TABLE subscriptions` or a breaking rewrite that orphans the 400 live production subscribers.
- Leaving the immediate cancellation access-loss bug unaddressed.
