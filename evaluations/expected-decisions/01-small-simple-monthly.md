# Expected Decisions — Scenario 01: Small Application with Simple Monthly Subscriptions

## 1. Expected Architectural Reasoning
- **Recommended Approach**: **Approach B — Payment-Provider-Managed Subscriptions** (Stripe Checkout + Stripe Billing + local `subscriptions` and `payment_webhook_events` tables in Prisma/PostgreSQL).
- **Why**:
  - The app has a simple 2-tier model (`Free` and `Pro` at `$12/mo`) owned by an individual `User`.
  - Stripe natively handles recurring monthly charges, payment method updates, retries, and cancellation at period end.
  - Using **Minimal Schema Tier** (`subscriptions` + `payment_webhook_events` with static plan definitions in TypeScript config) keeps the codebase clean and avoids unnecessary database joins or admin UI overhead.
- **Over-Quota Downgrade Handling**:
  - In `POST /api/habits`, check `hasActiveEntitlement(user)` OR `currentHabitCount < 5`. When a user cancels/expires with 12 habits, existing habits remain untouched, and only creating habit #13 is blocked with `403 Payment Required / Plan Limit Reached`.

## 2. Important Interview Questions
- Because the repo shows `User` and `Habit` models with no payment SDK, the agent should ask **only** about:
  1. Preferred payment gateway (e.g., Stripe, Paddle).
  2. Whether a free trial or annual billing interval is needed alongside `$12/mo`.
  3. Cancellation behavior (`cancel_at_period_end` vs. immediate) and what happens to existing habits above 5 when reverting to Free.
- **Must NOT ask**: What framework, ORM, or database the project uses (already visible in `package.json` and `schema.prisma`).

## 3. Mandatory Security Considerations
- Bind `userId` in Stripe Checkout `metadata` / `client_reference_id`.
- Verify `Stripe-Signature` against the raw request body in the Next.js Route Handler (`await req.text()`).
- Record `event.id` in `payment_webhook_events` to guarantee idempotency.
- Never unlock Pro in the `/billing/success` frontend page without server-side verification.

## 4. Unacceptable Recommendations (Automatic Fail)
- Recommending a separate billing microservice, Redis/BullMQ queue cluster, or custom recurring cron engine.
- Designing a 10-table enterprise billing schema (`plans`, `plan_features`, `invoices`, `line_items`, `credit_notes`, `tax_rates`) for a single `$12/mo` tier.
- Deleting a user's existing habits (>5) when their subscription expires.
