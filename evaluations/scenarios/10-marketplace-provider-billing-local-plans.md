# Evaluation Scenario 10: Marketplace Using Provider-Managed Recurring Billing with Local Plans and Entitlements

## 1. Developer Prompt

> "We run a B2B creator marketplace where independent industry analysts publish paid research feeds. Analysts define their own monthly/annual plans and custom perks in our database (`analyst_plans`). We already use Stripe Connect (`acct_...`) for analyst payouts. A colleague suggested we have to build our own cron-based recurring billing engine (Approach A) just because analysts create plans dynamically in our database and we have custom offboarding rules. Is that true, or how should we architect this?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: Python (FastAPI) or Node.js, PostgreSQL, Stripe SDK with Stripe Connect Express accounts (`analyst_profiles.stripe_account_id`).
- **Models**:
  - `users` (`id`, `email`).
  - `analyst_profiles` (`id`, `user_id`, `stripe_account_id`, `commission_bps` = `1500` for 15%).
  - `analyst_plans` (`id`, `analyst_id`, `name`, `price_cents`, `billing_interval`, `perks_json`, `is_active`).
- **Existing Payment Setup**: Stripe Connect is already configured and supported in all target countries/currencies.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **Recurring Billing vs. Local Plans**: "We want Stripe to handle automatic recurring card charges, dunning retries, and 15% platform commission splits via Stripe Connect, while keeping `analyst_plans` and subscriber perks in our own PostgreSQL database."
- **Analyst Offboarding Policy**: "If an analyst retires or deactivates their feed, existing paid subscribers must keep access until the end of their current paid billing period (`cancel_at_period_end = true`), and not be charged again. Only if an analyst is terminated for compliance violations should subscriptions be canceled immediately with prorated refunds."
