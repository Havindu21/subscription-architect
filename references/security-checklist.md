# Subscription Security & Financial Reliability Checklist

Use this reference during **Step 4 (Architecture Proposal)**, **Step 6 (Implementation Mode)**, and **Step 8 (Production Readiness Report)**.

Subscription and payment systems are prime targets for privilege escalation, free-riding, replay attacks, and financial loss. Verify every applicable control below.

---

## 1. Threat Model & Attack Surface

| Threat Scenario | How It Happens | Mandatory Defense |
| :--- | :--- | :--- |
| **Frontend Redirect Spoofing** | Attacker navigates directly to `/checkout/success?plan=pro&status=paid` or intercepts client state to unlock features without paying. | Never mutate subscription status from client redirects or unverified frontend calls. Activate only via signed server webhooks or direct server-to-provider API verification. |
| **Price / Plan Tampering at Checkout** | Client sends `{ "plan": "pro", "price": 0.01 }` in the checkout POST body, and the backend passes the client-supplied price to the payment gateway. | Client sends **only** `plan_id` (and `quantity` if applicable). The backend looks up the authoritative price, currency, and provider price ID from the server database/config. |
| **Webhook Forgery** | Attacker sends a crafted `POST /api/webhooks/payment` payload pretending to be Stripe/PayHere/PayPal (`invoice.paid`). | Verify cryptographic signature (`HMAC-SHA256` / RSA) against raw request body using the provider's webhook secret before parsing or trusting any field. |
| **Webhook Replay Attack** | Attacker or faulty proxy replays a captured valid webhook payload weeks later to extend `current_period_end` for free. | Enforce timestamp tolerance (e.g., $\le 5$ minutes) where supported, AND use insert-first atomic event claiming on `(provider, provider_event_id)` so each payment event is applied at most once regardless of concurrent delivery. |
| **Cross-Tenant Subscription Hijacking (IDOR)** | User A passes User B's `subscription_id` or `organization_id` to `/api/subscriptions/:id/cancel`, `/upgrade`, or `/portal`. | Enforce strict server-side ownership checks (`subscription.owner_id == current_authenticated_owner.id`) and role checks (e.g., `org_role IN ('owner', 'billing_admin')`) on every endpoint. |
| **Webhook Identity Spoofing** | Attacker initiates checkout as User A, but edits the email field on the hosted checkout page to Victim B's email, or modifies unsigned query params. | Bind internal `user_id` / `organization_id` in provider-signed session metadata at checkout creation time. Never match subscriptions by mutable billing email alone. |
| **Concurrent Race Condition (Double Activation / Double Charge)** | User double-clicks "Upgrade" or two webhooks (`checkout.session.completed` and `invoice.paid`) execute simultaneously in parallel worker threads. | Use insert-first atomic event claiming (not SELECT-then-INSERT) so only one concurrent delivery succeeds; use partial unique indexes on active subscriptions; use deterministic outbound idempotency keys. |
| **Currency / Amount Mismatch Attack** | Attacker pays `100 LKR` instead of `100 USD` or pays for a Basic plan reference while claiming a Pro plan reference. | When verifying a payment webhook or callback, assert that both `paid_amount >= expected_amount` and `paid_currency == expected_currency` for the bound `plan_id` before activating. |
| **Marketplace Plan / Commission Manipulation** | In a marketplace, Seller A modifies Seller B's plan pricing, or client request overrides the platform commission percentage. | Verify `plan.seller_id == authenticated_seller.id` on all plan mutations, calculate platform fees (`commission_rate_bps`) strictly on the server, and scope buyer entitlements to `(buyer_id, seller_id)`. |
| **Legacy / Internal Plan Checkout Spoofing** | Client passes a grandfathered or internal plan ID (e.g., `planId: 'pro_legacy'` or `'enterprise_internal'`) to the self-serve checkout endpoint to get a lower legacy price. | Restrict self-serve checkout initiation to plans with `is_selectable_for_new_checkout == true` in server config/DB. |

---

## 2. Pre-Implementation & Pre-Launch Security Checklist

### A. Payment & Checkout Verification
- [ ] **Server-Authoritative Pricing**: Checkout initiation endpoints accept only plan identifiers (`plan_id`, `billing_interval`, `quantity`), never raw prices or discount amounts from the client.
- [ ] **Signed Metadata Binding**: `user_id` / `organization_id` and internal `plan_id` are embedded in the payment provider's server-created session metadata or signed order reference.
- [ ] **Amount & Currency Validation**: For gateways where the client submits a form (e.g., MD5/HMAC form post gateways), the webhook handler explicitly verifies that the paid amount and currency match the server's expected plan price.
- [ ] **No Client-Side Activation**: Frontend success pages only read status or trigger a server-verified lookup; they never directly set `status = 'active'`.
- [ ] **Outbound Idempotency Keys**: Every server-initiated charge, subscription creation, plan update, or refund API call includes an idempotency key to prevent duplicate charges on network retry.
- [ ] **Provider Capability Verified**: Before recommending an approach, the provider's recurring billing support, webhook signature mechanism, supported countries/currencies, and specific event names have been verified against official documentation (not assumed from generic SDK availability). See `payment-integration.md` §1.1.

### B. Webhook Endpoint Hardening
- [ ] **Raw Body Signature Verification**: Webhook signature verification uses the unparsed raw request body and constant-time comparison (`crypto.timingSafeEqual` or official SDK verifier).
- [ ] **Timestamp / Replay Protection**: Stale webhook timestamps outside the allowed tolerance window are rejected.
- [ ] **Consistent Idempotency & Crash-Recovery Model (No Half-Mixed State)**:
  - **Durable Asynchronous Processing (Primary Model)**: The HTTP handler atomically persists the verified event (`payload_json`, `status = 'pending'`) on `UNIQUE (provider, provider_event_id)` before acknowledging the provider (`202`/`200`). Workers claim events using a lease fencing token (`lease_owner`, `lease_expires_at`, `attempt_count`) and commit the subscription state transition + `status = 'processed'` in a single transaction **only if `lease_owner` still matches and the lease has not expired**.
  - **Synchronous Single-Transaction Alternative (Option A)**: The event insert (`status = 'processed'`) and subscription state update execute inside **one** atomic database transaction so any crash or failure rolls back the event claim completely, allowing the provider's HTTP retry to claim and process it cleanly.
- [ ] **Self-Healing Stale Event Recovery (Without Relying on Provider Retries)**: In the Durable Asynchronous model, an internal recovery job reclaims rows where `status IN ('pending', 'failed_transient')` or `status = 'processing' AND lease_expires_at <= now` using a new `lease_owner` fencing token and completes processing from the stored `payload_json` without needing another provider webhook delivery.
- [ ] **Out-of-Order Resilience**: Older events cannot overwrite newer terminal states (e.g., a delayed `updated` event does not overwrite a newer `canceled` state).
- [ ] **Fast 2xx Response**: Webhook endpoints do not run slow synchronous tasks (like SMTP email sending or external provisioning) before returning `2xx`.

### C. Authentication, Authorization & Multi-Tenancy
- [ ] **Centralized Entitlement Guard**: Feature access checks go through a single, well-tested server-side policy/middleware (`requireEntitlement(actor, feature)`) that validates both subscription `status` and `current_period_end` / `grace_period_ends_at`.
- [ ] **IDOR Protection on Billing Actions**: View, upgrade, downgrade, cancel, resume, and billing-portal endpoints verify that the authenticated user owns the subscription (or holds `billing_admin` / `owner` permission in the owning organization).
- [ ] **Trial Abuse Prevention**: The backend enforces trial eligibility per user/organization rather than relying on the frontend hiding the "Start Free Trial" button.
- [ ] **Downgrade & Quota Enforcement**: Downgrading or expiring a subscription properly restricts over-quota actions on the server without corrupting or deleting existing user data.
- [ ] **Marketplace Seller Offboarding Policy**: The developer has explicitly confirmed the buyer subscription policy on seller deactivation (Immediate cancellation / End-of-period expiry / Transition or escrow). The implementation reflects that confirmed policy, and existing buyer entitlements are protected according to it during the offboarding window.

### D. Quota & Credit Ledger Safety
- [ ] **Atomic Reservation**: Quota and credit reservations are made inside a database transaction; available balance is verified and the reservation row is inserted atomically before any job is started.
- [ ] **Reservation Lease Expiry**: Each reservation row has a `reserved_until` timestamp. A cleanup job releases reservations that have been in `status = 'reserved'` past their lease deadline, restoring the quota balance.
- [ ] **Idempotent Commit**: The commit step verifies `entry.status = 'reserved'` before marking `consumed`; a late-arriving worker whose reservation was already released by the cleanup job cannot re-consume the balance.
- [ ] **Idempotent Release**: The release (refund) step verifies `entry.status = 'reserved'` before restoring quota; double-release by two concurrent error handlers is prevented by the same status check inside a transaction.

### E. Secrets, Privacy & Auditability
- [ ] **Environment-Scoped Secrets**: API secret keys and webhook signing secrets are loaded strictly from server environment variables or secret managers—never hardcoded, committed to git, or exposed in frontend bundles (`NEXT_PUBLIC_`, `VITE_`, etc.).
- [ ] **PCI Scope Minimization**: Raw credit card numbers (PAN) and CVVs never touch the application backend or logs; card collection uses provider-hosted pages, iframes, or client-side tokenization SDKs.
- [ ] **PII-Safe Logging**: Webhook and payment logs redact sensitive customer PII, tokens, and secret headers while preserving `subscription_id`, `provider_event_id`, `previous_status`, `new_status`, and `actor_id` for financial auditing.
- [ ] **Immutable Audit Trail**: All state transitions (who or what triggered the change, old status, new status, provider event ID, timestamp) are recorded in an audit log or payment history table.
