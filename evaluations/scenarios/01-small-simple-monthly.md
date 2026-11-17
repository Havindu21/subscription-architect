# Evaluation Scenario 01: Small Application with Simple Monthly Subscriptions

## 1. Developer Prompt

> "I want to add a $12/month Pro subscription to my habit-tracking web app so Pro users can create unlimited habits (Free users are limited to 5 habits). Can you design and build the subscription system for me?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: TypeScript, Next.js (App Router), PostgreSQL, Prisma ORM.
- **Authentication**: NextAuth / Auth.js with a `User` model (`id`, `email`, `name`, `createdAt`). There are **no** `Organization` or `Team` models—this is a single-user B2C app.
- **Domain Model**: A `Habit` model (`id`, `userId`, `title`, `streakCount`, `createdAt`). Currently, `POST /api/habits` creates habits with no count limit.
- **Payment Integration**: None installed yet (`package.json` has no payment SDK).
- **Existing Subscriptions**: None.
- **Deployment**: Single Next.js web service deployed on a standard PaaS with a managed PostgreSQL database. No background worker cluster (no Redis/BullMQ/Temporal).

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **Payment Gateway**: "We'll use Stripe."
- **Plans & Pricing**: "Just Free ($0, up to 5 active habits) and Pro ($12/month, unlimited habits). Maybe a $100/year option later, fixed prices only."
- **Trials**: "No free trial needed right now since there's already a Free tier."
- **Cancellations & Downgrades**: "If they cancel, let them keep Pro until the end of the month they paid for. If they go back to Free and already have 12 habits, don't delete their habits—just block them from creating new ones until they have fewer than 5."
