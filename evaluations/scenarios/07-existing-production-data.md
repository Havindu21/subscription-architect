# Evaluation Scenario 07: Application with Existing Production Subscription Data

## 1. Developer Prompt

> "Our B2B SaaS currently stores billing state directly as columns on the `users` table (`is_pro_subscriber: boolean`, `stripe_sub_id: varchar`, `paid_until: timestamp`). We now have 3,500 active paying users in production, and we're launching multi-user 'Teams' where subscriptions belong to a Team, plus adding a 'Starter' ($15/mo) and 'Business' ($49/mo) tier instead of just a boolean `is_pro_subscriber`. How should we architect and migrate this safely without downtime or breaking billing for our 3,500 paying customers?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: TypeScript, Express/Fastify API, PostgreSQL, Prisma/Knex migrations.
- **Current Schema (`users` table)**:
  - `id`, `email`, `is_pro_subscriber` (boolean, default `false`), `stripe_customer_id`, `stripe_sub_id`, `paid_until` (timestamp).
  - A new `teams` table (`id`, `name`, `owner_user_id`) and `team_members` table (`team_id`, `user_id`, `role`) were recently added in the last sprint, and every existing user already has a default personal `team` created for them (`personal_team_id`).
- **Current Codebase Usage**:
  - 18 different controller files check `if (!req.user.is_pro_subscriber)` directly!
  - The live webhook handler (`POST /webhooks/stripe`) updates `users.is_pro_subscriber` and `users.paid_until` by matching `stripe_sub_id`.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **Grandfathering Existing 3,500 Pro Users**: "Existing `is_pro_subscriber = true` users are paying $25/mo on a legacy 'Pro Legacy' price in Stripe. They must keep their $25/mo price and get all 'Business' tier features automatically mapped to their default Team without any interruption or re-checkout."
- **Zero-Downtime Requirement**: "Our deploy pipeline runs database migrations a few minutes before rolling out new container instances, so dropping or renaming `users.is_pro_subscriber` immediately will crash the running app containers during deployment."
