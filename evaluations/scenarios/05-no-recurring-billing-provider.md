# Evaluation Scenario 05: Application Using a Payment Provider Without Recurring Billing Support

## 1. Developer Prompt

> "We operate an online tutoring platform in a region where our local payment gateway (LankaPay / regional bank redirect gateway) only supports one-time checkout payments via form redirect and server webhook notifications—it has no recurring billing API or saved card tokenization. However, our schools and students need monthly and quarterly subscription plans. How should we architect and implement subscriptions?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: Go 1.23 (or Node.js/TypeScript), PostgreSQL, `sqlc` / `pgx`, standard cron/scheduler worker built into the backend binary.
- **Existing Payment Integration**:
  - `internal/payments/localgateway.go` implements a signed checkout form builder (`merchant_id`, `order_id`, `amount`, `currency`, `return_url`, `notify_url`, `hmac_signature`) and a webhook verifier (`VerifyNotificationSignature(payload, secret)`).
  - Gateway documentation in `docs/gateway-spec.md` confirms: **One-time checkout only**. No recurring subscriptions API, no card vaulting/tokenization API.
- **Models**: `users`, `schools`, `courses`.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **Gateway Constraint**: "We cannot switch to Stripe or PayPal—this local bank gateway is mandatory for local currency cards and QR payments."
- **Renewal UX**: "Since we can't auto-charge their card each month, we want the system to generate a renewal invoice 7 days before their current period ends, email/SMS them a direct payment link, and let them pay before expiry so they don't lose access."
- **Grace Period & Late Renewal**: "If they don't pay by the expiry date, give them a 5-day grace period (`past_due`) before cutting off access. If they pay during the 5-day grace period, extend their cycle by 1 month from the original expiry date. If they pay 2 weeks after being cut off (`expired`), start a fresh 1-month cycle from the new payment date."
