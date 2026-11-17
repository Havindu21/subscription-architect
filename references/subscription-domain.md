# Subscription Domain & Developer Interview Guide

Use this reference during **Step 1 (Repository Inspection)** and **Step 2 (Adaptive Developer Interview)** to classify the application's subscription model and ask only high-signal, non-redundant questions.

---

## 1. Do They Actually Need Subscriptions?

Before designing a recurring billing state machine, verify whether the developer's goal actually requires subscriptions. Developers frequently ask for "subscriptions" when a simpler monetization model fits better:

| Requirement Pattern | Better Alternative | Why Full Subscriptions Are Overkill |
| :--- | :--- | :--- |
| Customer pays once for permanent feature access (e.g., pro desktop license, course bundle, lifetime unlock). | **One-Time Purchase + Entitlement Flag** | No renewals, billing cycles, dunning, proration, or expiration timers needed. Only requires an order/payment record and an access grant. |
| Customer buys blocks of usage that do not expire on a schedule (e.g., 500 AI generations, 100 SMS credits). | **Prepaid Credit Ledger / Wallet** | Access depends on remaining balance (`credits > 0`), not billing period dates (`current_period_end`). Optional auto-top-up can be added later without plan tiers. |
| Internal tool or B2B app where sales/admins manually toggle access based on offline contracts. | **Admin-Managed Feature Flags / Account Tier** | No online checkout or automated payment webhook pipeline is needed; a simple `plan_tier` and optional `expires_at` managed via an admin UI suffices. |
| Pay-per-transaction marketplace fee (e.g., platform takes 5% per booking). | **Transaction Commission / Split Payment** | Revenue is tied to individual orders rather than recurring membership states. |

**Rule**: If repository inspection or interview answers reveal one of these patterns, explicitly recommend the simpler model in your architecture proposal and explain the engineering and operational savings.

---

## 2. Subscription Business Archetypes

Identify which archetype matches the application. This directly shapes database ownership, payment flows, and architecture selection:

### A. Single-Tier or Multi-Tier B2C SaaS
- **Structure**: Platform defines 1–4 fixed plans (e.g., Free, Pro, Ultra) billed monthly or annually.
- **Subscriber Entity**: Individual `User` (`user_id`).
- **Typical Complexity**: Low. Focus on clean checkout, webhook sync, and simple entitlement middleware.

### B. Multi-Tenant B2B / Team SaaS
- **Structure**: Plans belong to an `Organization`, `Workspace`, or `Team` (`organization_id`), not an individual user.
- **Billing Mechanics**: Often includes per-seat pricing (`quantity`), role-based billing management (only `billing_admin` or `owner` can upgrade/cancel), and invoice history.
- **Key Edge Cases**: What happens when a user belongs to multiple organizations? What happens when a seat is added mid-cycle? What happens if the user who set up billing leaves the organization?

### C. Two-Sided Marketplace / Creator Platform
- **Structure**: Multiple plan owners! Sellers, creators, instructors, or merchants define their own plans, prices, and perks; buyers subscribe to specific sellers (e.g., Patreon, Substack, coaching platforms, multi-vendor SaaS).
- **Subscriber Entity**: `User` (or `Buyer`) subscribing to a `Seller`/`Creator`'s `Plan`. A single buyer may hold multiple active subscriptions to different sellers simultaneously.
- **Billing Mechanics**: Requires platform fee splitting, seller payout tracking, or connected accounts, plus scoping entitlements per `(buyer_id, seller_id)`.

### D. Usage-Based / Metered Hybrid
- **Structure**: Combines a recurring base fee with variable consumption (e.g., API calls, storage GB, compute minutes, overage charges).
- **Billing Mechanics**: Requires an idempotent usage-ingestion log, aggregation per billing period, and either threshold billing or end-of-period arrears invoicing.

### E. Multi-Application / Ecosystem Membership
- **Structure**: One subscription grants entitlements across multiple distinct web/mobile applications or services owned by the same company.
- **Billing Mechanics**: Strong candidate for **Approach C (Existing Shared Subscription Service)** or **Approach D (Hybrid)** with signed claims/tokens or centralized entitlement APIs.

---

## 3. Pricing & Plan Dimensions

When inspecting the codebase or interviewing the developer, determine which dimensions apply:

1. **Billing Cadence**: Monthly, annual, weekly, custom days, or one-time + recurring setup fee.
2. **Tier Structure**:
   - **Hard-coded / Static tiers**: Plans rarely change (e.g., `free`, `pro`, `enterprise`).
   - **Admin-configurable plans**: Platform admins create/edit plans dynamically via dashboard.
   - **User/Seller-configurable plans**: Marketplace sellers create custom plans dynamically.
3. **Trial Models**:
   - **No-card trial**: Immediate access upon signup or button click; converts only if user completes checkout before/after `trial_ends_at`.
   - **Card-upfront trial**: Checkout collects payment method immediately (`$0` initial authorization) and auto-charges when `trial_ends_at` is reached unless canceled.
   - **Trial abuse policy**: One trial per user/organization lifetime, or allowed per plan?
4. **Discounts, Coupons & Add-ons**:
   - Percentage vs. fixed-amount discounts, duration (once, repeating for $N$ months, forever), and stackable add-on modules.

---

## 4. Progressive Developer Interview Protocol

### Core Interview Rules
1. **Inspect First, Ask Second**: Never ask what framework, database, ORM, or auth library the project uses if `package.json`, `pyproject.toml`, `go.mod`, `Gemfile`, `Cargo.toml`, or schema files are present. State what you found ("I see a Next.js + Prisma + PostgreSQL app with Clerk authentication and an existing `Organization` model...") and ask only about the missing business decisions.
2. **Batch by Priority (Max 3–5 Questions per Turn)**: Overwhelming a developer with 15 questions leads to vague or skipped answers. Ask the most foundational questions first (Round 1: Ownership, Plans, and Payment Provider), then follow up on lifecycle edge cases (Round 2: Trials, Upgrades/Downgrades, Failures, Cancellations) only as needed.
3. **Offer Sensible Defaults**: When asking about edge cases (like proration or cancellation timing), briefly state the industry-standard default so the developer can simply agree if they have no custom preference.

---

### Round 1: Foundational Business & Provider Questions
Ask only the questions from this list that are not already obvious from the repository:

1. **What is being subscribed to, and who owns the subscription?**
   - Does a subscription belong to an individual user, an organization/workspace, or a specific resource?
   - Who defines the subscription plans (are they fixed platform tiers, admin-configured, or created by marketplace sellers/tenants)?
2. **What are the plan tiers, pricing rules, and billing intervals?**
   - Are there free and paid tiers? Fixed monthly/annual prices, per-seat pricing, or usage-based charges?
   - Do you need free trials (with or without a payment method upfront), promotional discounts, or add-ons?
3. **Which payment gateway will process payments, and how does it support recurring billing?**
   - Are you using an already-integrated gateway (if detected in the repo) or introducing a specific provider (e.g., Stripe, PayHere, Paddle, Razorpay, PayPal)?
   - Does your chosen payment account/gateway support native automatic recurring charges, or only one-time checkout notifications requiring application-driven renewals?
4. **Are there existing subscribers or external services involved?**
   - Is this a brand-new subscription launch, or do we need to migrate existing paying users/data or integrate with an existing internal billing service?

---

### Round 2: Lifecycle, Access Control & Failure Edge Cases
Ask these once the foundational model is clear (or propose standard defaults and ask the developer to confirm):

1. **Upgrades, Downgrades, and Plan Changes**:
   - *Standard default*: Upgrades take effect immediately (with prorated charge or cycle reset), while downgrades take effect at the end of the current billing period. Does that match your business rules?
2. **Cancellation & Expiration Behavior**:
   - When a user cancels, should they keep access until the end of their paid billing period (`cancel_at_period_end`), or lose access immediately with a refund?
   - When a subscription expires completely, is the user's data preserved in a read-only/restricted state or archived?
3. **Failed Payments & Grace Periods**:
   - When a renewal payment fails, should the user keep access during a short retry/grace window (e.g., 3–7 days in `past_due` status while payment is retried), or lose access immediately?
4. **Refunds & Chargebacks**:
   - If a refund or dispute occurs, should the subscription be revoked immediately? Are refunds processed solely by admins in the payment dashboard, or do you need in-app self-serve refund flows?
5. **Quota Reset Period (When Plans Include Metered Usage Quotas)**:
   - Does the usage quota reset on the **calendar month** (e.g., 1st of each month UTC), the **subscription billing anniversary** (anchored to `current_period_start`), or a **rolling 30-day window**? (Especially critical when offering an annual billing plan with a monthly usage quota.)
6. **Marketplace Seller Offboarding Policy (When Sellers/Coaches Own Plans)**:
   - If a seller or creator deactivates their account or is suspended, should existing paid buyer subscriptions **end immediately** (with prorated refund), **continue until the current paid period expires** (`cancel_at_period_end = true`), or enter a **managed transition/refund workflow**?
