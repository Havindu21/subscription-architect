# Evaluation Scenario 02: Application with an Existing Payment Gateway (No Subscriptions)

## 1. Developer Prompt

> "We already sell individual digital design templates in our app, and now we want to launch an 'All-Access Club' subscription where users pay monthly or yearly to download up to 50 templates per month without buying them individually. How should we architect this?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: Python 3.12, FastAPI, SQLAlchemy 2.0 (async), Alembic migrations, PostgreSQL, Pytest.
- **Authentication**: JWT + session cookies backed by a `users` table (`id`, `email`, `stripe_customer_id`, `role`).
- **Existing Payment Integration**:
  - `stripe` Python SDK is already installed and configured in `app/services/stripe_service.py`.
  - Users already have `stripe_customer_id` populated when they buy a single template!
  - Existing `orders` table (`id`, `user_id`, `template_id`, `stripe_payment_intent_id`, `amount_cents`, `status`, `created_at`).
  - Existing webhook endpoint at `POST /api/v1/webhooks/stripe` (`app/api/webhooks.py`) that verifies `Stripe-Signature` and handles `payment_intent.succeeded` and `charge.refunded` for one-time template orders.
  - **Gap in existing webhook code**: It updates `orders.status = 'paid'` directly by `payment_intent_id` without a dedicated webhook event idempotency table.
- **Download Authorization**: `GET /api/v1/templates/{id}/download` currently checks `has_purchased_template(user_id, template_id)` by querying the `orders` table.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **One-Time Purchases vs. Subscriptions**: "Keep one-time template purchases working exactly as they do today! Users should be able to either buy a single template forever OR subscribe to the All-Access Club ($29/mo or $249/yr)."
- **Download Quota Reset**: "The 50-downloads/month quota resets at each billing cycle renewal (even on the annual plan, it's 50 downloads per month). Downloading the same template twice by the same user shouldn't burn a second credit."
- **Failed Renewals**: "Give a 3-day grace period on `past_due` renewals before blocking club downloads."
