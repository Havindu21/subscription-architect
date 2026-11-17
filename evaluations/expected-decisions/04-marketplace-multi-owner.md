# Expected Decisions — Scenario 04: Marketplace with Multiple Plan Owners and Customer Types

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Approach D (Hybrid Implementation: Provider-Managed Recurring Billing via Stripe Connect + Application-Owned Coach Plans & Entitlements)** (or Approach A only if the payment gateway lacks recurring Connect/split support).
- **Why**:
  - Separate the **plan ownership & entitlement concern** from the **recurring payment scheduling concern**:
    - Even though thousands of coaches dynamically create and edit their own plans (`coach_plans` owned by `coach_profile_id`), the marketplace does **not** need to build a custom cron billing scheduler (Approach A) when Stripe Connect natively supports recurring subscriptions with `application_fee_percent` / destination charges and inline `price_data` (or connected prices).
    - Local `coach_plans` and `(athlete_user_id, coach_profile_id)` entitlements live in the application database, while Stripe Connect handles recurring charge execution, retries, and 10% commission splits (**Approach D**).
  - **Crucial Schema Decision**: The unique active subscription constraint **must** be scoped to `(athlete_user_id, coach_profile_id)`—**NOT** `(athlete_user_id)` alone! If scoped only to `user_id`, an athlete could only subscribe to one coach on the entire platform.
  - **Grandfathering / Price Versioning**: When a coach changes a plan's price from `$30/mo` to `$40/mo`, existing `subscriptions` must retain their snapshot `unit_amount_cents = 3000` (or immutable `plan_price_version_id`) so existing athletes are not silently overcharged.
  - **Configurable Coach Offboarding Policy**: Do **not** assume seller deactivation always requires immediate cancellation. Ask the developer to define the offboarding policy (e.g., voluntary coach departure sets `cancel_at_period_end = true` so athletes keep access until `current_period_end`, whereas a fraud ban triggers immediate cancellation + refund) and protect buyer entitlements according to that approved policy.

## 2. Important Interview Questions
- Confirm whether an athlete can subscribe to multiple coaches simultaneously (and only one tier per coach).
- Confirm price change grandfathering rules when a coach edits a tier's price.
- Ask what **offboarding policy** applies when a coach deactivates their account vs. is banned (end immediately, continue until `current_period_end`, or enter a transition/refund process), and who bears refund/dispute liability.

## 3. Mandatory Security Considerations
- **Plan Ownership Authorization**: Verify that Coach A can only create/edit/archive plans where `plan.coach_profile_id == current_user.coach_profile.id` (preventing Coach A from modifying Coach B's pricing or payout destination).
- **Content Entitlement Scoping**: Accessing a `WorkoutProgram` must verify `hasActiveCoachEntitlement(athlete_user_id, program.coach_profile_id, program.minimum_tier_level)`—having a Gold subscription to Coach A must never unlock Coach B's Gold programs!
- **Server-Calculated Application Fee**: The 10% platform commission must be calculated exclusively on the backend from `coach_profile.commission_rate_bps`, never passed from client requests.

## 4. Unacceptable Recommendations (Automatic Fail)
- Putting a global `UNIQUE (user_id)` active constraint on `subscriptions`, blocking athletes from subscribing to more than one coach.
- Checking only `user.has_active_subscription == true` on workout programs without checking *which coach* the subscription is for.
- Allowing a coach's price edit on a `plans` row to automatically alter the billing amount of existing subscribers without grandfathering/versioning.
