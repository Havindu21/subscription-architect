# Subscription Production Readiness Report: [Project Name]

> **Date**: [YYYY-MM-DD]  
> **Architecture Implemented**: [Approach A / B / C / D]  
> **Payment Provider**: [Provider Name]  
> **Verification Status**: [Ready for Staging / Ready for Production / Blocked — See Open Items]

---

## 1. Implementation Summary

*Briefly summarize the implemented subscription architecture, owning entity (`User` vs. `Organization`), plan structure, and files created or updated.*

- **Architecture Pattern**: [e.g., Approach B — Payment-Provider-Managed Subscriptions with local entitlement projection]
- **Database Tables Added/Updated**: [e.g., `subscriptions`, `payment_webhook_events`]
- **Key Backend Modules**: [List paths]
- **Key API Endpoints**: [List routes]

---

## 2. Security & Financial Reliability Audit

Mark each control based on actual code inspection and executed tests (`PASS`, `FAIL`, or `N/A`):

| Category | Control | Status | Evidence / File Location |
| :--- | :--- | :--- | :--- |
| **Checkout Security** | Client sends only `plan_id`; server resolves authoritative price & currency | `[PASS/FAIL]` | `[file:line]` |
| **Checkout Security** | Internal `owner_id` & `plan_id` bound in provider-signed session metadata | `[PASS/FAIL]` | `[file:line]` |
| **Activation Safety** | Frontend redirects never grant access without server-side payment verification | `[PASS/FAIL]` | `[file:line]` |
| **Webhook Security** | Cryptographic signature verified against raw request body using env secret | `[PASS/FAIL]` | `[file:line]` |
| **Webhook Security** | Timestamp / replay window validated | `[PASS/FAIL]` | `[file:line]` |
| **Idempotency & Crash Recovery** | Consistent webhook processing model implemented: either Durable Leased Inbox (`payload_json`, `lease_owner`, `lease_expires_at`, fenced atomic completion, and self-healing internal recovery without relying on provider retries) OR Single-Transaction Synchronous processing (atomic claim + state update with full rollback on error) | `[PASS/FAIL]` | `[file:line]` |
| **Ordering & Concurrency** | Simultaneous duplicate deliveries, out-of-order events, expired worker lease fencing, and quota reservation leases handled safely | `[PASS/FAIL]` | `[file:line]` |
| **Outbound Safety** | Outbound provider charge/mutation calls include deterministic idempotency keys | `[PASS/FAIL]` | `[file:line]` |
| **Access Control** | All subscription read/write endpoints enforce strict ownership / role checks (no IDOR) | `[PASS/FAIL]` | `[file:line]` |
| **Lifecycle Accuracy** | Mid-cycle `cancel_at_period_end` preserves paid access until `current_period_end` | `[PASS/FAIL]` | `[file:line]` |
| **Data Safety** | Downgrades and expirations restrict over-quota actions without deleting user data | `[PASS/FAIL]` | `[file:line]` |
| **Secrets & PCI** | All API/webhook secrets in env vars; zero raw card details logged or stored | `[PASS/FAIL]` | `[file:line]` |

---

## 3. Test Execution Results

> **Truthfulness Note**: Only list tests that were actually executed during this session. Clearly separate automated local test results from manual staging verification steps.

### 3.1 Automated Tests Executed
- **Test Command Run**: `[e.g., npm test / pytest / go test ./...]`
- **Result Summary**: `[e.g., 28 passed, 0 failed]`
- **Scenarios Verified by Automated Tests**:
  - [ ] Entitlement evaluation across `trialing`, `active`, `past_due` (grace period), `canceled` (`cancel_at_period_end`), and `expired`.
  - [ ] Webhook signature verification (valid signature accepted, invalid/tampered signature rejected with `400`/`401`).
  - [ ] Webhook idempotency (duplicate `provider_event_id` replayed multiple times results in single state transition).
  - [ ] Out-of-order webhook delivery resilience.
  - [ ] Cross-user / cross-organization IDOR rejection (`403`/`404`) on checkout, upgrade, and cancel endpoints.
  - [ ] Checkout price tampering rejection.

### 3.2 Unverified / Live Environment Items (Requires Developer Action in Staging/Production)
*List any items that could not be verified inside the local test environment and must be checked against the live payment provider sandbox/production account:*
1. [e.g., Configure live/staging webhook endpoint URL `https://<domain>/api/webhooks/stripe` in the provider dashboard and set `STRIPE_WEBHOOK_SECRET`.]
2. [e.g., Verify live provider Price IDs in production environment variables match the configured plan tiers.]
3. [e.g., Run one end-to-end sandbox checkout using the provider's test card to confirm live webhook network reachability through your firewall/CDN.]

---

## 4. Operational Runbook & Environment Checklist

### Required Environment Variables
| Variable Name | Description | Example / Format |
| :--- | :--- | :--- |
| `[PAYMENT_SECRET_KEY]` | Server-side API secret key for payment provider | `sk_live_...` / `sk_test_...` |
| `[PAYMENT_WEBHOOK_SECRET]` | Cryptographic signing secret for incoming webhooks | `whsec_...` |
| `[APP_BASE_URL]` | Canonical application URL for checkout return redirects | `https://app.example.com` |

### Common Operational Procedures
- **How to manually inspect a customer's subscription state**: [Query or admin command]
- **How to force-resync a subscription if a webhook was missed during downtime**: [Reconciliation command or endpoint]
- **How to handle a customer refund or dispute**: [Expected webhook behavior and admin steps]
