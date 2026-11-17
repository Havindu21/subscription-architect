# Expected Decisions — Scenario 07: Application with Existing Production Subscription Data

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Approach B (Provider-Managed Subscriptions) with a Zero-Downtime 4-Phase Expand-Contract Migration**.
- **Why**:
  - There are 3,500 live paying customers in production, 18 controllers reading `req.user.is_pro_subscriber`, and a deployment pipeline that runs migrations before rolling out new containers.
  - **Zero-Downtime Expand-Contract Plan**:
    - **Phase 1 (Expand — Safe Schema Addition)**: Create the new `subscriptions` table (`id`, `team_id`, `plan_code` supporting `'free'`, `'starter'`, `'pro_legacy'`, `'business'`, `status`, `stripe_customer_id`, `stripe_subscription_id`, `current_period_end`, `cancel_at_period_end`) and `payment_webhook_events` table. **Do NOT drop or rename `users.is_pro_subscriber`, `users.stripe_sub_id`, or `users.paid_until` yet!**
    - **Phase 2 (Idempotent Backfill & Dual-Write)**:
      - Run an idempotent backfill script/migration that inserts a `subscriptions` row for each of the 3,500 users where `is_pro_subscriber = true` (or `stripe_sub_id IS NOT NULL`), linking `team_id = user.personal_team_id`, `plan_code = 'pro_legacy'`, `status = 'active'`, and preserving their `stripe_subscription_id` and `paid_until`.
      - Update the Stripe webhook handler to **dual-write** to both the new `subscriptions` table and legacy `users.is_pro_subscriber` column during the transition window.
      - Verify parity (`SELECT COUNT(*)` check and spot audit).
    - **Phase 3 (Switch Application Reads)**: Introduce a centralized `getTeamEntitlements(teamId)` service that grants `'business'` feature entitlements to both `plan_code == 'business'` and `plan_code == 'pro_legacy'`. Refactor the 18 controllers to use the centralized entitlement middleware instead of `req.user.is_pro_subscriber`.
    - **Phase 4 (Contract — Future Release Only)**: Only after Phase 3 is deployed and verified in production, remove the dual-write and drop the legacy `users` billing columns in a subsequent release with explicit approval.

## 2. Important Interview Questions
- Confirm how existing `is_pro_subscriber` users map to `teams` (`user.personal_team_id`) and that their `$25/mo` Stripe subscription remains untouched (`pro_legacy` plan code mapped to Business features).
- Confirm deployment migration ordering constraints (migrations run before container rollout).

## 3. Mandatory Security Considerations
- Ensure `pro_legacy` cannot be selected by new checkouts in `POST /api/subscriptions/checkout` (it is a grandfathered plan code only for existing subscribers).
- Enforce team-level authorization (`team_members.role IN ('owner', 'admin')`) for managing a team's subscription, while all team members inherit the team's feature entitlements.

## 4. Unacceptable Recommendations (Automatic Fail)
- Dropping or renaming `users.is_pro_subscriber` in the same migration that creates `subscriptions` (which immediately crashes the 18 controllers on running production containers during rollout).
- Forcing the 3,500 existing `$25/mo` subscribers to cancel and re-subscribe at `$49/mo`.
- Leaving the 18 scattered `if (!req.user.is_pro_subscriber)` checks un-centralized.
