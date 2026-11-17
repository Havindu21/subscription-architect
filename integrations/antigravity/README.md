# Installing and Using Subscription Architect in Google Antigravity

This guide explains how to install, configure, and invoke the **Subscription Architect** skill in **Google Antigravity** (IDE and `agy` CLI).

---

## 1. Compatibility & Verification Status

| Host Environment | Status | Tested Modes | Notes |
| :--- | :--- | :--- | :--- |
| **Google Antigravity** | **Verified** | Architecture Mode & Implementation Mode | Verified live across 4 disposable repositories (`template-hub`, `exam-prep`, `tutoring-regional`, and `habit-tracker`). See [compatibility-report.md](./compatibility-report.md) for full execution logs. |

---

## 2. Installation Methods

Antigravity discovers skills either per-workspace (`.agents/skills/`) or globally across all projects on your machine (`~/.gemini/config/skills/`), as well as via explicit `skills.json` manifests.

### Option A: Workspace Installation (Recommended for Teams)

Install the skill directly into a project's `.agents/skills/` directory so it is available to everyone working in that repository:

```bash
# From the root of your target application repository:
mkdir -p .agents/skills

# Clone or copy subscription-architect into .agents/skills/
git clone https://github.com/Havindu21/subscription-architect.git .agents/skills/subscription-architect
```

*(Alternatively, if you already cloned `subscription-architect` locally on your machine, you can symlink it during local development:)*

```bash
mkdir -p .agents/skills
ln -s "/path/to/subscription-architect" .agents/skills/subscription-architect
```

### Option B: Global Installation (Available in All Projects on Your Machine)

To make **Subscription Architect** available across every project you open in Antigravity without copying files into each repository:

```bash
mkdir -p ~/.gemini/config/skills
git clone https://github.com/Havindu21/subscription-architect.git ~/.gemini/config/skills/subscription-architect
```

### Option C: Registration via `.agents/skills.json`

If you store shared skills in a custom directory (for example, `tools/skills/subscription-architect` or an external shared folder), create `.agents/skills.json` in your project root:

```json
{
  "entries": [
    {
      "path": "/absolute/or/workspace-relative/path/to/skills-parent-folder",
      "include_only": ["subscription-architect"]
    }
  ]
}
```

> **Note**: In `skills.json`, `path` points to the **parent directory** containing `subscription-architect/`, because Antigravity scans each `entries` directory one level deep for `<skill-name>/SKILL.md`.

---

## 3. How Antigravity Loads the Skill

1. **Progressive Disclosure**: Antigravity reads the YAML frontmatter (`name: subscription-architect` and `description`) from `SKILL.md` at session startup. The full `SKILL.md` and the deep reference files in `references/` and `templates/` consume zero context window tokens until a subscription-related task is triggered.
2. **On-Demand Reference Loading**: When activated, the agent reads `SKILL.md` first and then loads specific files from `references/` (such as `architecture-patterns.md`, `payment-integration.md`, `security-checklist.md`) and `templates/` (`architecture-proposal.md`) as needed for the current step.

---

## 4. Invoking Operating Modes in Antigravity

### Invoking Architecture Mode (Default)
Ask the agent to evaluate or design subscriptions for your project. For example:

> "Use `subscription-architect` in **Architecture Mode** to inspect this repository and design a subscription architecture for adding a Pro tier."

In **Architecture Mode**, the agent will:
1. Inspect your codebase (framework, database/ORM, auth/tenancy, and any existing payment or billing code).
2. Ask 3–5 targeted clarifying questions about business requirements that cannot be inferred from the code.
3. Compare **Approaches A, B, C, D** (and non-subscription alternatives).
4. Produce a complete 15-section **Architecture Proposal** without modifying any application code.

### Invoking Implementation Mode
Once you review and approve the architecture proposal:

> "I approve the architecture proposal. Switch to **Implementation Mode** and implement the subscription system incrementally."

In **Implementation Mode**, the agent will:
1. Follow the 5-stage implementation plan (Schema/Migrations $\rightarrow$ Core Domain & Entitlement Guard $\rightarrow$ Payment & Idempotent Webhooks $\rightarrow$ API & UI $\rightarrow$ Documentation & Readiness Report).
2. Execute the project's test suite and report verified test results alongside the **Production Readiness Report**.
