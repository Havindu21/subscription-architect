# Evaluation Scenario 03: Application with an Existing Subscription Implementation

## 1. Developer Prompt

> "We built a basic subscription system six months ago, and now we want to add an 'Enterprise Team' tier, support mid-cycle upgrades/downgrades, and fix a bug where users sometimes get charged or have their subscription status stuck when webhooks arrive late. Can you redesign or update our subscription architecture?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: Node.js (Express), TypeScript, PostgreSQL with Drizzle ORM, Jest.
- **Existing Subscription Implementation**:
  - `subscriptions` table already exists (`id`, `user_id`, `plan_code`, `status`, `stripe_subscription_id`, `current_period_end`, `created_at`).
  - **Critical Flaw 1 (Frontend Redirect Activation)**: `GET /api/billing/success?session_id=...&plan=pro` immediately sets `subscriptions.status = 'active'` and `plan_code = req.query.plan` based on the query string without verifying the session status or `plan` with Stripe!
  - **Critical Flaw 2 (Immediate Cancel Bug)**: `POST /api/billing/cancel` calls Stripe with `cancel_at_period_end: true`, but immediately updates the local DB to `status = 'canceled'`, and the auth middleware `requireActiveSub` checks `sub.status === 'active'`, stripping paid access immediately when a user cancels mid-month.
  - **Critical Flaw 3 (No Webhook Idempotency or Out-of-Order Handling)**: `POST /api/billing/webhook` processes `invoice.paid` and `customer.subscription.updated` without recording `event.id` and without row locking.
- **Existing Usage**: References in migrations and seed scripts show this schema is already deployed in production.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **Ownership for 'Enterprise Team'**: "Currently `subscriptions` is linked to `user_id`, and we just added a `workspaces` table where a user owns a workspace and invites members. Should the new Enterprise tier be on the workspace or the user?" (Developer agrees moving/binding workspace subscriptions cleanly while preserving existing solo user subscriptions is desired.)
- **Mid-Cycle Changes**: "Upgrades should happen right away with proration; downgrades should wait until the end of the billing cycle."
- **Production Impact**: "We have ~400 active paying subscribers in production right now—we cannot drop the `subscriptions` table or break their active sessions."
