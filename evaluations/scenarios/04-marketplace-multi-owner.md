# Evaluation Scenario 04: Marketplace with Multiple Plan Owners and Customer Types

## 1. Developer Prompt

> "We're building a fitness coaching marketplace where independent coaches create their own monthly subscription tiers (e.g., Bronze, Silver, Gold with custom prices and perks), and athletes can subscribe to one or more coaches at the same time. The platform takes a 10% commission on every subscription payment. How should we architect this?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: Ruby on Rails 7.2 (or Node/NestJS equivalent), PostgreSQL, Sidekiq + Redis for background jobs, RSpec.
- **Models**:
  - `User` (`id`, `email`, `name`).
  - `CoachProfile` (`id`, `user_id`, `bio`, `payout_account_id`, `commission_rate_bps` default `1000` for 10%).
  - `WorkoutProgram` (`id`, `coach_profile_id`, `title`, `minimum_tier_level`).
- **Payment Integration**: Stripe SDK + Stripe Connect (`payout_account_id` stores connected account IDs `acct_...`), or a gateway supporting marketplace split/token charges.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **Plan Ownership**: "Coaches create, price, and edit their own tiers whenever they want from their Coach Dashboard. We expect thousands of coaches, each with 1 to 3 tiers."
- **Subscriber Behavior**: "An athlete (`User`) can subscribe to Coach A's Gold plan AND Coach B's Bronze plan simultaneously. Coaches can also be athletes subscribing to other coaches."
- **Price Changes by Coach**: "If a coach raises their Gold tier price from $30/mo to $40/mo, existing subscribers stay at $30/mo (grandfathered) unless the coach explicitly migrates them and the athlete consents."
- **Platform Commission & Coach Offboarding Policy**: "Platform keeps 10% application fee on each charge via Stripe Connect. If a coach voluntarily leaves the platform, existing paid athlete subscriptions should continue until their current paid period expires (`cancel_at_period_end = true`); if a coach is banned for fraud/abuse, subscriptions must be canceled immediately with a prorated refund."
