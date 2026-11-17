# Expected Decisions — Scenario 06: Application That Already Uses a Centralized Subscription Service

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Approach C (Existing Shared Subscription Service) combined with Approach D (Hybrid Local Quota Ledger for 4K Render Minutes)**.
- **Why**:
  - The company already runs `billing-core.internal` (`src/clients/corp-billing-client.ts`), and all payment processing, invoicing, and plan upgrades happen on `billing.company.com`.
  - **Do NOT build a second payment gateway integration or checkout flow inside AI Video Studio.**
  - However, because `billing-core` only knows the organization's tier (`Creative Cloud Pro` vs. `Enterprise`) and billing cycle dates—not per-second 4K video render consumption—AI Video Studio should maintain:
    1. A local **Cached Organization Entitlement Projection** (`organization_entitlements` table storing `organization_id`, `tier`, `status`, `current_period_start`, `current_period_end`, `synced_at`) updated via `billing-core` signed webhook/event bus messages AND refreshed lazily if stale or when the user's JWT `entitlements_version` increments.
    2. A local **Render Quota Ledger** (`render_usage_ledgers` or sum of `render_jobs` scoped to `(organization_id, period_start)`) enforcing the 120-minute (Pro) or 1,000-minute (Enterprise) monthly cap.

## 2. Important Interview Questions
- Confirm how `billing-core` notifies downstream apps of plan changes (signed webhooks/event bus vs. polling) and what fallback behavior is expected during a brief `billing-core` outage.
- Confirm whether the 120/1,000 render minutes are pooled across the entire `organization_id` and when a render credit is deducted (reserved on job start, refunded if render fails due to system error).

## 3. Mandatory Security Considerations
- **Verify `billing-core` Event Authenticity & Idempotency**: Authenticate incoming `entitlement.updated` events from `billing-core` (HMAC/mTLS/SNS signature) and deduplicate via insert-first atomic event claiming by `event_id` / `version`.
- **Atomic Quota Reservation with Lease Expiry & Crash Recovery**: When a user starts a 4K render job, lock the organization's quota row (`SELECT ... FOR UPDATE`), verify `used_seconds + requested_seconds <= tier_limit_seconds`, and create a `reserved` entry with a `reserved_until` lease timestamp. Include abandoned-job recovery (releasing expired reservations if a render worker crashes), idempotent commit/release transitions (`WHERE status = 'reserved'`), and protection preventing a delayed worker from consuming an already-released reservation.
- **Resilience to Shared Service Blips**: Reads check the local `organization_entitlements` table first (falling back to cached state with a short TTL if `billing-core` is briefly unreachable) so a 30-second `billing-core` outage does not take down active video editing.

## 4. Unacceptable Recommendations (Automatic Fail)
- Integrating Stripe/PayPal directly into AI Video Studio or building a custom credit card checkout when `billing-core.internal` already handles company-wide billing.
- Making a synchronous blocking HTTP call to `billing-core` on every single API request without local caching/projection, violating the 30-second outage resilience requirement.
- Failing to reserve/refund render minutes safely when render jobs run concurrently or fail mid-render.
