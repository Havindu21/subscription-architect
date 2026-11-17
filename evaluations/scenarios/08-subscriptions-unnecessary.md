# Evaluation Scenario 08: Application Where Subscriptions Are Unnecessary

## 1. Developer Prompt

> "I'm building an exam-prep web app for a certification test. I want to add a subscription system where students pay $49 to unlock the full 500-question practice bank and mock exam simulator for Forever / Lifetime access (or pay $9 to buy a pack of 3 AI essay grading credits whenever they run out). Can you design the subscription tables, recurring webhook handlers, and billing cycle state machine?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: Next.js / Express (or Django / Laravel), PostgreSQL.
- **Models**: `users`, `questions`, `mock_exam_attempts`, `essay_submissions`.
- **Payment Integration**: Stripe (or Paddle/LemonSqueezy) SDK installed.
- **Existing Billing**: None.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions to confirm the business model, the developer answers:
- **Recurring Charges**: "Wait—no, students only study for this certification exam once! Nobody wants to be billed every month. The $49 Exam Pass unlocks the question bank permanently (or for 12 months fixed with zero auto-renewal), and the $9 Essay Pack just adds +3 essay grading credits to their account balance that never expire."
- **Refunds**: "Only if an admin refunds an order in Stripe should we revoke the Exam Pass or deduct the unused credits."
