# Subscription Implementation Plan: [Project Name]

> **Approved Architecture**: [Approach A / B / C / D]  
> **Operating Mode**: Implementation Mode  
> **Target Stack**: [Language / Framework / Database / Payment Provider]

Use this staged plan after the developer approves the Architecture Proposal. Complete and verify each stage before moving to the next. Do not run destructive migrations or live production operations without explicit permission.

---

## Stage 1: Database Schema, Migrations & Configuration

### Objectives
- Add required environment variable definitions (`.env.example` / config schema validation).
- Create non-destructive database migrations and ORM models for subscription entities.

### Planned File Changes
- `[path/to/schema-or-migration]` — Add `subscriptions`, `payment_webhook_events` (and `plans` / `subscription_payments` if applicable).
- `[path/to/env-config]` — Register payment provider keys and webhook secret variables.

### Verification Steps
- [ ] Validate schema/migration syntax using the project's ORM or migration check tool.
- [ ] Confirm monetary fields use integer minor units or exact decimals.
- [ ] Confirm unique constraints exist on `(provider, provider_event_id)` and active subscription ownership.

---

## Stage 2: Core Subscription Domain & Entitlement Guard

### Objectives
- Implement canonical subscription state evaluation (`hasActiveEntitlement`) and lifecycle transition helpers.
- Implement server-side authorization and entitlement middleware/guards.

### Planned File Changes
- `[path/to/subscription-domain-service]` — State transition validation, trial eligibility check, and entitlement evaluation logic.
- `[path/to/auth-entitlement-middleware]` — Server-side guard verifying subscription ownership (`user_id` / `organization_id`) and active entitlement status.

### Verification Steps
- [ ] Run unit tests covering all subscription statuses (`incomplete`, `trialing`, `active`, `past_due`, `canceled`, `expired`) and timestamp boundaries (`current_period_end`, `trial_ends_at`, `grace_period_ends_at`, `cancel_at_period_end`).
- [ ] Verify over-quota / downgrade behavior preserves existing user data safely.

---

## Stage 3: Payment Provider & Idempotent Webhook Integration

### Objectives
- Implement server-authoritative checkout initiation binding internal `owner_id` and `plan_id`.
- Implement raw-body cryptographic webhook signature verification and transactional idempotent event processing.
- Implement return-page server-to-server verification (or reconciliation helper) to handle redirect-before-webhook timing.

### Planned File Changes
- `[path/to/payment-gateway-service]` — Checkout creation, plan change, cancellation, and outbound idempotency key generation.
- `[path/to/webhook-controller-or-route]` — Raw body signature verification, `payment_webhook_events` deduplication, row-level locking, and state transition execution.

### Verification Steps
- [ ] Run integration tests for valid webhook signatures, invalid/tampered signatures (`400`/`401`), duplicate event replays (`200 OK` with single state update), and out-of-order events.
- [ ] Verify client requests cannot spoof prices, currencies, or subscription activation via redirect URLs.

---

## Stage 4: API Endpoints & Frontend / UI Integration

### Objectives
- Expose authenticated API routes for viewing subscription status, starting checkout, changing plans, and canceling/resuming.
- Integrate billing status, upgrade prompts, and cancellation controls into the application UI following existing design conventions.

### Planned File Changes
- `[path/to/subscription-routes]` — Authenticated endpoints with strict ownership/role checks.
- `[path/to/frontend-billing-components]` — Pricing/plan selector, current subscription badge, and management actions.

### Verification Steps
- [ ] Run API authorization tests confirming User A / Org A cannot view, upgrade, or cancel User B / Org B's subscription (IDOR protection).
- [ ] Run the full existing project test suite and typechecker/linter to confirm zero regressions in unrelated features.

---

## Stage 5: Documentation & Production Readiness Review

### Objectives
- Document the implemented subscription architecture, database schema, lifecycle states, and webhook setup for the project team.
- Complete the Production Readiness Report ([templates/production-readiness-report.md](./production-readiness-report.md)).

### Deliverables
- `[path/to/project-subscription-docs.md]` — Project-specific technical and business documentation.
- Completed Production Readiness Report detailing all executed tests and any remaining manual staging/provider setup steps.
