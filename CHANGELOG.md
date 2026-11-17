# Changelog

All notable changes to the **Subscription Architect** skill are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-11-16

### Fixed — Final Pre-Release Security & Architecture Review
- **Consistent Durable Leased Webhook Processing & Crash Recovery (`references/payment-integration.md`, `references/database-design.md`, `references/security-checklist.md`, `references/testing-strategy.md`)**: Selected **Durable Asynchronous Processing (Transactional Inbox + Leased Worker Fencing)** as the canonical model (`payload_json`, `status`, `attempt_count`, `lease_owner`, `lease_expires_at`, `last_error`) while documenting **Synchronous Single-Transaction Processing (Option A)** as the lightweight alternative and forbidding mixing incompatible parts of both. Added lease-fenced completion (`WHERE lease_owner = :token AND lease_expires_at > :now`), internal self-healing recovery without relying on provider retries, and executed automated tests covering simultaneous deliveries, crash after claim, crash after durable enqueue, provider-independent recovery, expired-worker lease fencing (`409 STALE_LEASE`), post-completion replay, and transient DB failure retry (`10/10` passing in `template-hub`).
- **Clarified Architecture Decision Boundaries (`SKILL.md`, `references/architecture-patterns.md`)**: Explicitly separated five independent subscription concerns (recurring payment scheduling, plan/price management, subscription ownership, application entitlements, and usage quotas / marketplace commissions) so custom entitlements or dynamic marketplace plans do not automatically force Approach A when provider-managed recurring billing (Approach B/D) fits.
- **Strengthened Provider Capability Verification (`references/payment-integration.md`, `references/security-checklist.md`)**: Added the 8-point Provider Capability Verification Checklist covering native recurring engines, tokenization, mandates, retries, countries/currencies, refund/dispute events, strong vs. weak/legacy webhook signatures, and outbound idempotency keys.
- **Quota Reservation Lease Recovery & Reset Models (`references/subscription-lifecycle.md`, `references/database-design.md`, `references/testing-strategy.md`)**: Added explicit quota reset models (calendar-month, subscription-anniversary, rolling 30-day window), reservation lease expiry (`reserved_until`), abandoned-job recovery, idempotent commit/release transitions, and delayed-worker protection (`409` when attempting to commit an already-released reservation), verified with automated tests.
- **Configurable Marketplace Offboarding Policy (`references/subscription-lifecycle.md`, `references/subscription-domain.md`, `references/security-checklist.md`)**: Required asking the developer to choose between immediate cancellation, end-of-period expiry (`cancel_at_period_end = true`), or transition/refund when a seller is deactivated, rather than assuming immediate cancellation.
- **Expanded Evaluation Suite & Release Cleanup (`evaluations/`, `README.md`, `.gitignore`, `.npmignore`)**: Added Scenarios `09` (concurrent webhooks + worker crash recovery) and `10` (marketplace with provider-managed billing + local plans), updated Scenarios `02`, `04`, `06`, and `08`, replaced placeholder URLs and machine-specific paths, and added `.gitignore` and `.npmignore`.

### Added — Antigravity Live Compatibility Test Suite & Report
- Executed multi-turn live compatibility tests in Google Antigravity against disposable sample applications (`template-hub` with an existing backend, `node:sqlite` database, HMAC authentication, and Stripe one-time order integration, and `exam-prep` for the unnecessary-subscription scenario) and recorded full execution results in `integrations/antigravity/compatibility-report.md`.
- Refined `SKILL.md` Section 5 (**Reference & Template Index**) with stage-guided progressive loading so agents only load `testing-strategy.md`, `implementation-plan.md`, and `production-readiness-report.md` after passing the Step 5 approval gate, reducing initial turn token usage.

### Fixed — Simulated 8-Scenario Benchmark Evaluation
- Added Section 3.E (**Metered Quotas, Credit Ledgers & Sub-Cycle Resets**) and Section 3.F (**Marketplace Seller Offboarding & Price Grandfathering**) to `references/subscription-lifecycle.md` covering monthly quota sub-cycles on annual plans (Scenario 02), deduplicating repeat asset downloads (Scenario 02), two-phase quota reservation/release for long-running jobs that fail mid-flight (Scenarios 06 & 08), and seller deactivation cancellation cascades (Scenario 04).
- Enhanced `references/architecture-patterns.md` (Approach C) with the **Local Projection & Outage Resilience Pattern** (`organization_entitlements` projection + JWT `entitlements_version` lazy refresh) so brief central billing service outages never block application reads (Scenario 06).
- Enhanced `references/database-design.md` Section 4 with explicit rules for grandfathering legacy tiers (`is_selectable_for_new_checkout = false`, Scenario 07) and handling hybrid user-to-workspace ownership migrations (Scenario 03).
- Added threat vectors and defenses for **Marketplace Plan / Commission Manipulation** (Scenario 04) and **Legacy / Internal Plan Checkout Spoofing** (Scenario 07) to `references/security-checklist.md`.

### Fixed — Independent Senior Review Audit
- Resolved Approach A vs. Approach B overlap in `SKILL.md` Section 2 so standard recurring tiers with a capable provider (e.g., Stripe Billing/Paddle) clearly favor Approach B while Approach A targets gateways without recurring engines, custom domain-driven billing cycles, or dynamic seller-owned plans.
- Closed the Implementation Mode approval loophole in `SKILL.md` Section 1 so even if a developer asks to "implement immediately," the agent must still inspect the repository, present a concise Architecture Proposal summary, and obtain confirmation before modifying application files.
- Fixed step-count and proposal-section-count discrepancies in `SKILL.md` Section 3 (aligned to 8 workflow steps and all 15 proposal sections matching `templates/architecture-proposal.md`).
- Standardized the end-of-period cancellation convention across `references/subscription-lifecycle.md` and `templates/architecture-proposal.md` (keeping `status = 'active'` with `cancel_at_period_end = true` until `current_period_end` as the canonical default for new implementations).
- Fixed a database lock-contention anti-pattern in `references/payment-integration.md` Step 4 to ensure external HTTP API reconciliation calls are never made while holding open `SELECT ... FOR UPDATE` database row locks.
- Updated `templates/architecture-proposal.md` Sections 6 and 7 so Non-Subscription recommendations (`orders` / `credit_ledger_entries`) and marketplace `(buyer_id, seller_id)` constraints fit cleanly without forcing a `subscriptions` table onto non-recurring designs.

### Added — Phase 6: Release Preparation
- Validated Agent Skills directory structure, `SKILL.md` YAML frontmatter, and all relative Markdown links across all 32 repository documentation, reference, template, evaluation, and integration files.
- Updated `README.md` with GitHub installation instructions (workspace, global, and manifest options) and an explicit **Verification & Compatibility Matrix** documenting what has been live-tested versus what remains unverified.

### Added — Phase 5: Antigravity Integration (`integrations/`)
- Created `integrations/antigravity/README.md` with step-by-step installation instructions for Google Antigravity (`.agents/skills/` workspace installation, `~/.gemini/config/skills/` global installation, and `.agents/skills.json` manifest configuration) plus operating mode usage examples.
- Created `integrations/codex/README.md` documenting `.agents/skills/` and `AGENTS.md` setup while truthfully marking Codex CLI compatibility as unverified in this environment.
- Verified **Architecture Mode** live in a simulated regional tutoring repository (`tutoring-regional` with a non-recurring gateway `LankaPay`), confirming the skill inspected `docs/gateway-spec.md` and `src/lankapay.mjs`, selected **Approach A (Application-Owned Implementation)**, produced a complete 15-section `ARCHITECTURE_PROPOSAL.md`, and left `src/` completely unmodified.
- Verified **Implementation Mode** live in a simulated SaaS repository (`habit-tracker`), confirming the skill implemented **Approach B**, enforced the 5-habit Free limit without deleting over-quota habits on expiration, verified HMAC-SHA256 webhook signatures, enforced `payment_webhook_events` idempotency and out-of-order timestamp protection, blocked price tampering and cross-user IDOR (`403`), passed all 6 automated test suites (`node --test`), and generated `PRODUCTION_READINESS_REPORT.md`.

### Added — Phase 4: Evaluation (`evaluations/`)
- Created 8 realistic architectural evaluation scenarios in `evaluations/scenarios/`:
  1. `01-small-simple-monthly.md`: Small B2C habit tracker adding a `$12/mo` Pro tier.
  2. `02-existing-payment-gateway.md`: App with existing Stripe one-time template orders adding a 50-downloads/mo club.
  3. `03-existing-subscription-impl.md`: App with 400 live subscribers and 3 critical bugs (redirect spoofing, immediate cancel access loss, missing webhook idempotency).
  4. `04-marketplace-multi-owner.md`: Two-sided coaching marketplace with seller-owned tiers, multi-coach athlete subscriptions, grandfathering, and 10% platform fee splits.
  5. `05-no-recurring-billing-provider.md`: Regional payment gateway supporting only one-time redirect checkouts without recurring APIs or card tokenization.
  6. `06-centralized-subscription-service.md`: Enterprise ecosystem app integrating with an existing `billing-core.internal` service plus a local 120-min 4K render quota.
  7. `07-existing-production-data.md`: Zero-downtime migration of 3,500 live `users.is_pro_subscriber` rows to multi-user `teams` and multi-tier plans.
  8. `08-subscriptions-unnecessary.md`: Certification exam-prep app where the developer asks for "subscriptions" for a `$49` lifetime pass and `$9` non-expiring essay credit pack.
- Created matching expected decision rubrics in `evaluations/expected-decisions/01..08` defining expected architectural reasoning, interview questions, mandatory security controls, and automatic-fail anti-patterns.
- Created `evaluations/evaluation-guide.md` with the 7-dimension scoring rubric and Phase 4 gap analysis.
- Refined `SKILL.md` (Step 1 and Step 6) based on the Phase 4 evaluation audit to explicitly require auditing existing billing code for security/lifecycle flaws, scoping marketplace ownership to `(buyer_id, seller_id)`, locking quota/credit rows atomically, and following the 4-step Expand-Contract pattern for live production data migrations.

### Added — Phase 3: Templates (`templates/`)
- Created `templates/architecture-proposal.md`: 15-section structured proposal template covering executive summary, repository facts, interview findings, assumptions, recommended architecture (Approaches A/B/C/D/Non-Subscription), rejected alternatives, database schema, lifecycle state machine, payment/webhook verification flow, API changes, auth/security controls, failure recovery, migration strategy, testing plan, risks, implementation phases, and explicit developer approval gate.
- Created `templates/implementation-plan.md`: 5-stage incremental implementation and verification blueprint (Schema & Config, Core Domain & Entitlements, Payment & Webhooks, API & UI, Documentation & Readiness).
- Created `templates/production-readiness-report.md`: Pre-launch audit template with a 12-point financial security verification matrix, truthful test execution reporting (separating automated local tests from live staging steps), and operational runbook checklist.

### Added — Phase 2: Domain Knowledge (`references/`)
- Created `references/subscription-domain.md`: Business model archetypes (B2C, B2B multi-tenant, marketplaces, metered hybrid, ecosystem), non-subscription alternatives, pricing dimensions, and the 2-round progressive developer interview protocol.
- Created `references/architecture-patterns.md`: Decision matrix and deep comparison of Approach A (Application-Owned), Approach B (Provider-Managed), Approach C (Existing Shared Service), Approach D (Hybrid), and anti-overengineering guardrails.
- Created `references/payment-integration.md`: Gateway capability classification (native recurring vs. tokenized mandate vs. one-time redirect), checkout verification sequence, raw-body webhook signature validation, idempotent event processing, and reconciliation strategies.
- Created `references/subscription-lifecycle.md`: Canonical state machine (`incomplete`, `trialing`, `active`, `past_due`, `paused`, `canceled`, `expired`), entitlement mapping table, upgrade/downgrade proration rules, grace periods, and over-quota handling.
- Created `references/database-design.md`: Right-sized schema tiers (Minimal, Standard, Multi-Tenant/Marketplace), ER diagram, minor-unit currency rules, partial unique indexes for preventing duplicate active subscriptions, and expand-contract migration steps.
- Created `references/security-checklist.md`: Financial threat model (redirect spoofing, price tampering, webhook forgery/replay, cross-tenant IDOR, race conditions, currency mismatch) and pre-launch security checklist.
- Created `references/testing-strategy.md`: 3-layer subscription testing pyramid, time-boundary entitlement test matrix, webhook replay/out-of-order test cases, and staging verification guide.

### Added — Phase 1: Foundation
- Created core `SKILL.md` following the open Agent Skills specification with YAML frontmatter (`name`, `description`).
- Defined the 10-step end-to-end workflow, operating modes (**Architecture Mode** vs. **Implementation Mode**), architectural comparison framework, and non-negotiable financial security rules.
- Added initial `README.md`, `LICENSE` (MIT), and `CHANGELOG.md`.
