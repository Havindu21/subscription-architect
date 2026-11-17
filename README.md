# Subscription Architect

**Subscription Architect** is an open, portable AI coding skill that helps software developers and AI coding agents understand, design, implement, test, and document subscription functionality inside existing software applications.

> **What this repository is (and is not):**
> This repository is **not** a SaaS billing platform, payment gateway wrapper, SDK, or centralized subscription microservice. It is an instruction and reference package built on the open **Agent Skills** (`SKILL.md`) format that teaches AI coding agents how to make sound, evidence-based subscription architecture and engineering decisions inside your own codebase.

---

## Why Subscription Architect?

When asked to "add subscriptions" to an application, AI coding agents frequently fall into two traps:
1. **Overengineering**: Designing complex microservice billing engines, custom invoicing pipelines, or multi-layered abstraction frameworks for a small app that only needed two fixed monthly tiers.
2. **Under-securing or Blindly Coding**: Writing database tables and frontend checkout redirects immediately without inspecting the existing codebase, clarifying business edge cases (proration, failed payment grace periods, cancellation timing, user vs. organization ownership), or securing webhooks against replay and duplicate delivery.

**Subscription Architect** enforces a disciplined, architecture-first engineering workflow:
1. **Inspect the Repository First**: Understand the existing language, framework, database, authentication model, payment integrations, and conventions—and audit any existing billing code for security or lifecycle bugs—before proposing anything.
2. **Adaptive Developer Interview**: Ask 3–5 targeted, progressive questions at a time only about business rules that cannot be inferred from the codebase.
3. **Compare Architectural Approaches**: Evaluate four core patterns—and whether subscriptions are even needed—based on evidence:
   - **Approach A — Application-Owned Implementation**: Subscription tables, billing cycle dates, and lifecycle state machine live inside the application's existing backend and database, using the existing payment gateway.
   - **Approach B — Payment-Provider-Managed Subscriptions**: Native recurring billing, retries, and proration handled by the payment provider (e.g., Stripe Billing, Paddle) paired with application-owned authorization and entitlements.
   - **Approach C — Existing Shared Subscription Service**: Integration with an existing internal or external shared subscription service across a multi-app ecosystem.
   - **Approach D — Hybrid Implementation**: Combines provider-managed billing, local domain/quota ledgers, marketplace splits, or shared services where justified.
   - **Non-Subscription Alternatives**: Recommends one-time purchase entitlements or prepaid credit ledgers when recurring billing is unnecessary.
4. **Propose & Wait for Approval**: Present a comprehensive 15-section architecture proposal covering schema design, state lifecycle, payment verification, failure handling, and migration strategy—then wait for explicit developer approval.
5. **Implement Incrementally & Securely**: Build in verifiable stages with server-side payment verification, raw-body webhook signature validation, idempotency, transactional row locking, and automated tests.

---

## Quick Start & Installation

### 1. Google Antigravity (Verified)

**Option A — Install into a Project Workspace (`.agents/skills/`):**
```bash
mkdir -p .agents/skills
git clone https://github.com/Havindu21/subscription-architect.git .agents/skills/subscription-architect
```

**Option B — Install Globally for All Projects on Your Machine (`~/.gemini/config/skills/`):**
```bash
mkdir -p ~/.gemini/config/skills
git clone https://github.com/Havindu21/subscription-architect.git ~/.gemini/config/skills/subscription-architect
```

See [integrations/antigravity/README.md](./integrations/antigravity/README.md) for `.agents/skills.json` team manifest configuration and detailed usage.

### 2. Other Agent Skills / `AGENTS.md` Hosts (e.g., OpenAI Codex)

Clone into `.agents/skills/subscription-architect` and optionally reference `.agents/skills/subscription-architect/SKILL.md` from your root `AGENTS.md`. See [integrations/codex/README.md](./integrations/codex/README.md) for details.

---

## Operating Modes

You can invoke **Subscription Architect** in two explicit modes:

- **Architecture Mode (Default)**:
  > *"Use `subscription-architect` in Architecture Mode to inspect this repository and design a subscription architecture for our Pro and Team tiers."*
  - Inspects your repository, conducts a focused developer interview, compares Approaches A, B, C, D, and produces a detailed architecture proposal ([templates/architecture-proposal.md](./templates/architecture-proposal.md)). **No application code is modified.**

- **Implementation Mode**:
  > *"I approve the architecture proposal. Switch to Implementation Mode and implement the subscription system incrementally."*
  - Implements the approved design in 5 verifiable stages ([templates/implementation-plan.md](./templates/implementation-plan.md)), executes the test suite, and generates a [Production Readiness Report](./templates/production-readiness-report.md).

---

## Repository Structure

```text
subscription-architect/
├── SKILL.md                                  # Core portable Agent Skill instructions & workflow
├── README.md                                 # Project overview, installation, and verification status
├── LICENSE                                   # MIT License
├── CHANGELOG.md                              # Version history and release changelog
│
├── references/                               # Deep domain knowledge loaded progressively on demand
│   ├── subscription-domain.md                # Business models, pricing structures & interview guide
│   ├── architecture-patterns.md              # Comparison of Approaches A, B, C, D & non-subscription options
│   ├── payment-integration.md                # Recurring vs. non-recurring gateways, webhooks & idempotency
│   ├── subscription-lifecycle.md             # State machines, trials, upgrades, grace periods & dunning
│   ├── database-design.md                    # Schema patterns for plans, subscriptions & entitlements
│   ├── security-checklist.md                 # Financial security, webhook verification & access control
│   └── testing-strategy.md                   # Unit, integration, webhook replay & failure testing
│
├── templates/                                # Structured deliverable templates
│   ├── architecture-proposal.md              # 15-section architecture proposal template
│   ├── implementation-plan.md                # Phased implementation & verification blueprint
│   └── production-readiness-report.md        # Pre-launch security, reliability & testing audit
│
├── evaluations/                              # Realistic architectural evaluation scenarios & benchmarks
│   ├── scenarios/                            # 10 test scenarios (simple SaaS, marketplaces, legacy data, concurrency, etc.)
│   ├── expected-decisions/                   # Expected architectural reasoning & unacceptable anti-patterns
│   └── evaluation-guide.md                   # 7-dimension rubric for evaluating agent decisions
│
└── integrations/                             # Host-specific installation guides (kept separate from core skill)
    ├── antigravity/                          # Google Antigravity installation & testing guide
    └── codex/                                # OpenAI Codex CLI / AGENTS.md integration guide
```

---

## Portability Principles

The core skill ([SKILL.md](./SKILL.md), `references/`, and `templates/`) follows the open Agent Skills specification and is strictly agnostic to:
- **AI Agent Host**: Does not depend on Antigravity-specific or Codex-specific proprietary tools.
- **AI Model**: Structured for clear reasoning across modern coding models.
- **Programming Language & Framework**: Works with TypeScript/Node.js, Python, Go, Ruby, Java, C#, PHP, Rust, and others.
- **Database & ORM**: Adapts to PostgreSQL, MySQL, SQLite, MongoDB, Prisma, Drizzle, TypeORM, SQLAlchemy, ActiveRecord, etc.
- **Payment Provider**: Adapts to Stripe, PayHere, PayPal, Paddle, Adyen, Razorpay, Braintree, regional one-time redirect gateways, or existing internal billing services.

---

## Verification & Compatibility Matrix (What Has Been Tested vs. Unverified)

In accordance with our truthfulness and engineering rigor guidelines, below is the exact record of what has been executed and verified versus what remains unverified:

### Verified
- **Agent Skills Structural Validation**: Verified `SKILL.md` YAML frontmatter (`name`, `description`), progressive disclosure sizing, and validated all relative Markdown links across all 37 Markdown files in the repository (`0` broken links).
- **Google Antigravity — Architecture Mode (Live Subagent Execution)**:
  - Tested on disposable repositories (`template-hub`, `exam-prep`, and `tutoring-regional`).
  - Verified that the agent inspected the repository first, separated billing scheduling from local plan/entitlement/quota concerns, selected the appropriate architecture (`Approach D` for `template-hub`, `Non-Subscription` for `exam-prep`, `Approach A` for `tutoring-regional`), and left `src/` completely untouched prior to approval.
- **Google Antigravity — Implementation Mode (Live Subagent Execution)**:
  - Tested on `template-hub` (**Approach D: Hybrid Provider-Managed Billing + Local Quota & Reservation Ledger**, `10 / 10` tests passing) and `habit-tracker` (**Approach B: Provider-Managed Subscriptions**, `6 / 6` tests passing).
  - Verified durable leased webhook processing (simultaneous deliveries, crash after claim, crash after durable enqueue, internal recovery from `payload_json` without relying on provider retries, expired-worker lease fencing (`409 STALE_LEASE`), replay after completion, and transient DB failure retry), quota reservation lease expiry and abandoned-job recovery, delayed-worker rejection (`409`), duplicate completion callback idempotency, annual-plan monthly quota sub-periods, 3-day `past_due` grace periods, price-tampering rejection, and cross-user IDOR (`403`) protection.
- **Architectural Evaluation Benchmark (`evaluations/`)**:
  - Audited `SKILL.md`, `references/`, and `templates/` against all 10 scenarios in [evaluations/evaluation-guide.md](./evaluations/evaluation-guide.md) (clearly distinguished as simulated architectural benchmark evaluations vs. the live Antigravity subagent tests above).

### Unverified (Requires Host- or Project-Specific Testing)
- **OpenAI Codex CLI Live Execution**: While `SKILL.md` uses zero Antigravity-specific tools and follows the portable Agent Skills format, live end-to-end execution inside the OpenAI Codex CLI binary has not been run in this environment (documented in [integrations/codex/README.md](./integrations/codex/README.md)).
- **Live External Payment Gateway Network Calls**: Local verification tests use deterministic signed webhook fixtures and local state stores rather than live network calls to Stripe/PayHere/LankaPay production servers.

---

## Development Roadmap Status

- [x] **Phase 1 — Foundation**: Portable Agent Skills directory structure, core `SKILL.md` workflow and safety rules, `README.md`, `LICENSE`, and `CHANGELOG.md`.
- [x] **Phase 2 — Domain Knowledge**: Comprehensive reference guides in `references/`.
- [x] **Phase 3 — Templates**: Deliverable templates in `templates/`.
- [x] **Phase 4 — Evaluation**: 10 realistic evaluation scenarios and expected architectural decision rubrics in `evaluations/`.
- [x] **Phase 5 — Antigravity Integration**: Installation guide and live verification in Google Antigravity.
- [x] **Phase 6 — Release Preparation**: Final consistency review, structural validation, GitHub installation instructions, and verification matrix.

---

## License

Released under the [MIT License](./LICENSE).
