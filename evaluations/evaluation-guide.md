# Subscription Architect — Evaluation Guide & Benchmark Rubric

This guide explains how to evaluate an AI coding agent equipped with **Subscription Architect** across ten realistic architectural scenarios.

---

## 1. Evaluation Philosophy

A high-quality subscription architecture skill is defined not just by what code it can generate, but by **what mistakes it refuses to make**:
- Does it inspect the repository before asking questions or choosing a tech stack?
- Does it ask focused, non-redundant business questions rather than dumping a generic 20-question checklist?
- Does it right-size the architecture (avoiding microservices and 10-table schemas for simple apps, while properly handling marketplaces, non-recurring gateways, and zero-downtime migrations when required)?
- Does it question the premise when subscriptions are unnecessary (e.g., one-time purchases or prepaid credit packs)?
- Does it treat payments and webhooks as security-critical financial operations?

---

## 2. Scenario Matrix

| # | Scenario File | Expected Decisions File | Core Architectural Test | Expected Primary Pattern |
| :--- | :--- | :--- | :--- | :--- |
| **01** | [01-small-simple-monthly.md](./scenarios/01-small-simple-monthly.md) | [01-small-simple-monthly.md](./expected-decisions/01-small-simple-monthly.md) | Small B2C habit tracker adding a `$12/mo` Pro tier. | **Approach B** (Provider-Managed, Minimal 2-table schema, non-destructive over-quota limit) |
| **02** | [02-existing-payment-gateway.md](./scenarios/02-existing-payment-gateway.md) | [02-existing-payment-gateway.md](./expected-decisions/02-existing-payment-gateway.md) | App already selling one-time templates via Stripe adding a 50-downloads/mo club. | **Approach B / D** (Reuse existing Stripe client & webhook route; preserve `orders`; add local monthly download quota lock) |
| **03** | [03-existing-subscription-impl.md](./scenarios/03-existing-subscription-impl.md) | [03-existing-subscription-impl.md](./expected-decisions/03-existing-subscription-impl.md) | App with 400 live subscribers and 3 critical bugs (redirect spoofing, immediate cancel cutoff, no webhook idempotency). | **Incremental Hardening of Approach B** (Spot all 3 bugs in repo inspection; non-destructive schema evolution) |
| **04** | [04-marketplace-multi-owner.md](./scenarios/04-marketplace-multi-owner.md) | [04-marketplace-multi-owner.md](./expected-decisions/04-marketplace-multi-owner.md) | Two-sided fitness coaching marketplace where coaches define tiers and athletes subscribe to multiple coaches. | **Approach D / A** (Scope uniqueness and entitlements to `(athlete_id, coach_id)`; grandfather price edits; 10% platform fee split) |
| **05** | [05-no-recurring-billing-provider.md](./scenarios/05-no-recurring-billing-provider.md) | [05-no-recurring-billing-provider.md](./expected-decisions/05-no-recurring-billing-provider.md) | Regional payment gateway supporting only one-time redirect checkouts without recurring APIs or card vaulting. | **Approach A** (Application-owned billing cycle, scheduled pre-expiry invoices, 5-day grace period, strict webhook amount/currency verification) |
| **06** | [06-centralized-subscription-service.md](./scenarios/06-centralized-subscription-service.md) | [06-centralized-subscription-service.md](./expected-decisions/06-centralized-subscription-service.md) | New app inside an enterprise ecosystem that already runs `billing-core.internal`, plus a local 120-min 4K render quota. | **Approach C + D** (Integrate with `billing-core`; local cached entitlement projection for 30s outage resilience; atomic local render quota ledger) |
| **07** | [07-existing-production-data.md](./scenarios/07-existing-production-data.md) | [07-existing-production-data.md](./expected-decisions/07-existing-production-data.md) | Migrating 3,500 live `users.is_pro_subscriber` rows to multi-user `teams` and multi-tier plans without downtime. | **Approach B + 4-Phase Expand-Contract Migration** (Zero-downtime backfill, dual-write, `pro_legacy` grandfathering, no premature column drops) |
| **08** | [08-subscriptions-unnecessary.md](./scenarios/08-subscriptions-unnecessary.md) | [08-subscriptions-unnecessary.md](./expected-decisions/08-subscriptions-unnecessary.md) | Developer asks for "subscriptions" for a `$49` lifetime exam pass and `$9` pack of 3 non-expiring essay credits. | **Non-Subscription Pattern** (Push back on recurring machinery; recommend One-Time Entitlement + Atomic Prepaid Credit Ledger) |
| **09** | [09-concurrent-webhooks-and-crash-recovery.md](./scenarios/09-concurrent-webhooks-and-crash-recovery.md) | [09-concurrent-webhooks-and-crash-recovery.md](./expected-decisions/09-concurrent-webhooks-and-crash-recovery.md) | Concurrent duplicate `invoice.paid` webhooks, annual billing with monthly quota reset, and worker crash during quota reservation. | **Approach D Hardening** (Insert-first atomic event claiming, subscription-anniversary monthly sub-periods, reservation lease expiry & delayed-worker protection) |
| **10** | [10-marketplace-provider-billing-local-plans.md](./scenarios/10-marketplace-provider-billing-local-plans.md) | [10-marketplace-provider-billing-local-plans.md](./expected-decisions/10-marketplace-provider-billing-local-plans.md) | Creator marketplace with local seller-defined plans, Stripe Connect recurring billing, and configurable analyst offboarding policy. | **Approach D** (Separate billing scheduling from local plan/entitlement ownership; configurable end-of-period vs. immediate offboarding) |

---

## 3. Seven-Dimension Scoring Rubric

Score each scenario trial from **0 to 2** across each of the seven evaluation dimensions (Maximum Score: **14 / 14** per scenario):

| Dimension | 0 (Fail) | 1 (Partial) | 2 (Pass) |
| :--- | :--- | :--- | :--- |
| **1. Correctness of Architecture Decision** | Recommends an incompatible or broken architecture (e.g., Approach B for a non-recurring gateway, Approach A when provider Connect billing fits, or recurring billing for Scenario 08). | Selects a workable approach but misses a key domain nuance (e.g., misses annual-plan monthly quota reset model in Scenario 02/09 or offboarding policy in Scenario 04/10). | Separates recurring billing, plan management, ownership, entitlements, and quotas independently; selects the optimal approach and justifies why alternatives were rejected. |
| **2. Quality of Repository Inspection** | Proposes a stack, ORM, or payment client without reading the repository, or misses existing payment/subscription bugs. | Identifies the tech stack and models but overlooks subtle existing flaws (e.g., misses one of the 3 bugs in Scenario 03). | Thoroughly identifies stack, auth, tenancy, existing payment/webhook code, and existing bugs before proposing changes. |
| **3. Relevance of Interview Questions** | Asks >10 questions at once, or asks questions already answered by the codebase (`package.json`, schema files). | Asks reasonable business questions but misses 1 key edge case (e.g., grandfathering or grace periods). | States discovered repo facts first and asks 3–5 high-signal business questions tailored to the exact scenario. |
| **4. Security Awareness** | Trusts frontend redirect params, accepts client-supplied prices, skips webhook signature/idempotency, or allows IDOR. | Includes webhook signature verification and auth checks, but misses race-condition locking or amount/currency verification. | Enforces server-authoritative pricing, raw-body signature verification, `payment_webhook_events` idempotency, row locking, and strict tenant/owner checks. |
| **5. Avoidance of Overengineering** | Proposes unnecessary microservices, multi-gateway factory abstractions, or 10+ tables for a simple app. | Keeps architecture within the monolith but adds unneeded tables or background queues where simpler patterns suffice. | Right-sizes schema and infrastructure strictly to the application's actual requirements. |
| **6. Preservation of Existing Implementations** | Proposes dropping active production tables (`DROP TABLE`), breaks existing one-time orders, or ignores live subscribers. | Preserves tables but proposes a migration that causes brief deployment downtime. | Uses non-destructive Expand-Contract migrations, preserves existing flows (`orders`, `billing-core`), and grandfathers legacy subscribers cleanly. |
| **7. Quality of Implementation & Verification** | Modifies code without approval in Architecture Mode, or claims tests passed without executing them. | Waits for approval and writes code, but test coverage misses time boundaries, duplicate webhook replays, or IDOR. | Respects operating mode gates, implements incrementally, and writes/executes unit, webhook replay, and authorization tests. |

> **Automatic Failure Rule**: If an agent recommendation triggers any item listed under **"4. Unacceptable Recommendations (Automatic Fail)"** in a scenario's expected-decisions file, the scenario is marked **FAIL** regardless of the numeric score.

---

## 4. Phase 4 Self-Review & Gap Analysis

During Phase 4 development, `SKILL.md`, `references/`, and `templates/` were reviewed against all 8 scenarios above. The following nuances were identified and verified/strengthened across the skill:

1. **Detecting Security & Lifecycle Bugs in Existing Subscription Code (Scenario 03)**:
   - *Gap Checked*: An agent inspecting a repository that *already* has a `subscriptions` table might assume the existing code is correct and only add new features.
   - *Refinement Applied*: Updated `SKILL.md` Step 1 (Repository Inspection) to explicitly require auditing any existing payment/subscription/webhook code for security flaws (such as frontend redirect activation, missing webhook idempotency, or immediate-cancel access loss) and surfacing those findings in the proposal.
2. **Scoped Multi-Subscription Uniqueness & Quota Locking (Scenarios 02, 04, 06, 08)**:
   - *Gap Checked*: Developers often add a naive `UNIQUE (user_id)` index on `subscriptions` (which breaks marketplace multi-seller subscriptions in Scenario 04) or increment usage counters without row-level locking (which allows concurrent race conditions on the 50-download quota in Scenario 02, 120-min render quota in Scenario 06, or essay credits in Scenario 08).
   - *Refinement Applied*: Ensured `SKILL.md` and `references/database-design.md` / `references/security-checklist.md` explicitly highlight marketplace `(buyer_id, seller_id)` scoping and atomic row-locked quota/credit reservation (`SELECT ... FOR UPDATE`).
3. **Zero-Downtime Expand-Contract Migrations for Live Subscribers (Scenario 07)**:
   - *Gap Checked*: When migrating legacy `users.is_pro_subscriber` columns to a `subscriptions` table, dropping or renaming the legacy column in the same deployment breaks running containers.
   - *Refinement Applied*: Highlighted the 4-step Expand $\rightarrow$ Backfill/Dual-Write $\rightarrow$ Switch Reads $\rightarrow$ Contract requirement directly in `SKILL.md` Step 6 and `references/database-design.md`.
