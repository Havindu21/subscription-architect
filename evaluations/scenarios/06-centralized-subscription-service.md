# Evaluation Scenario 06: Application That Already Uses a Centralized Subscription Service

## 1. Developer Prompt

> "We're building a new 'AI Video Studio' web app inside our company's suite of creative tools. We need to gate 4K exports and cloud rendering behind our company's 'Creative Cloud Pro' and 'Enterprise' subscription tiers. How should we design the subscription system for this app?"

---

## 2. Simulated Repository State

When the agent inspects the target repository, it discovers:
- **Stack**: TypeScript, NestJS backend, React frontend, PostgreSQL (for video projects and render jobs).
- **Existing Ecosystem Integration**:
  - `src/clients/corp-billing-client.ts` already exists in the repo!
  - `docs/internal-billing-service.md` documents the company's existing internal **Shared Subscription & Entitlement Service (`billing-core.internal`)**:
    - All checkout, invoicing, Stripe/Adyen processing, and plan management happen on the central `billing.company.com` portal.
    - `billing-core` publishes signed EventBridge/SNS/webhook events (`entitlement.updated`, `subscription.state_changed`) per `organization_id` and exposes `GET /v1/organizations/{orgId}/entitlements`.
    - The authentication JWT issued by the company SSO (`auth-core`) includes `org_id` and `org_role`, plus a short-lived `entitlements_version` claim.
- **Current App Database**: Contains `video_projects` and `render_jobs` tables scoped by `organization_id`, with no local billing tables yet.

---

## 3. Simulated Developer Interview Responses

If the agent asks clarifying questions, the developer answers:
- **Checkout & Plan Management**: "Users never enter credit cards inside the AI Video Studio app—clicking 'Upgrade' should redirect them to the central `billing.company.com/upgrade?app=video-studio` portal managed by our platform billing team."
- **App-Specific Need**: "However, 'Creative Cloud Pro' includes 120 minutes of 4K cloud rendering per month specifically inside AI Video Studio (Enterprise gets 1,000 minutes). `billing-core` tells us which tier the organization has and their current billing period dates, but `billing-core` does NOT track per-second video render consumption—our app has to enforce and track that monthly render quota locally."
- **Reliability Requirement**: "Checking whether a user can open a project or start a render must be fast and shouldn't fail if `billing-core` has a brief 30-second network blip."
