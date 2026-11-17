# Evaluation Scenario 09: Concurrent Webhook Deliveries and Worker Crash Recovery

## 1. Developer Prompt

> "Our SaaS app processes Stripe subscription renewals and gives Pro users 20 AI document-translation credits per month (even on our annual plan). In production last week, two simultaneous `invoice.paid` webhook deliveries with the exact same `evt_...` ID hit two of our app instances at the same millisecond and extended a customer's period twice! Also, when a translation worker container OOM-crashed mid-job, the user's reserved translation credit stayed locked forever. How should we fix our webhook idempotency, annual-plan monthly quota resets, and credit reservation recovery?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: TypeScript, Node.js (Express), PostgreSQL (or SQLite in local tests), Vitest/Node test runner.
- **Current Webhook Handler (`src/webhooks/stripe.ts`)**:
  - Verifies `Stripe-Signature` against raw body.
  - Runs `SELECT id FROM payment_webhook_events WHERE provider_event_id = $1`. If not found, runs business logic (`current_period_end += 30 days`) and then `INSERT INTO payment_webhook_events ...` at the end.
  - **Bug 1 (TOCTOU Race Condition)**: Two concurrent requests with the same `evt_123` both pass the `SELECT` before either reaches the `INSERT`, causing double execution.
- **Current Quota & Reservation Logic (`src/services/translation-quota.ts`)**:
  - Scopes the 20-credit limit to `subscriptions.current_period_start .. current_period_end` (which gives annual subscribers 20 credits for the entire year instead of 20 per month!).
  - Creates a `translation_reservations` row with `status = 'reserved'`, but has no `reserved_until` lease expiry, no abandoned-job cleanup, and its completion callback blindly sets `status = 'consumed'` without checking whether the reservation was already released.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **Quota Reset Policy**: "The 20 translation credits should reset monthly on the **subscription billing anniversary** (e.g., if an annual plan started on Jan 15, credits reset on the 15th of each month)."
- **Worker Timeout**: "A translation job never takes longer than 5 minutes. If a worker hasn't finished after 5 minutes, the reservation should expire so the user gets their credit back, and if the dead worker somehow reports back 10 minutes later, it must not consume the credit or overwrite state."
- **Webhook Retry Recovery**: "If our database or server fails transiently midway through a webhook, Stripe should be able to retry and recover cleanly."
