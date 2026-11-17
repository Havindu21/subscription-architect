# Using Subscription Architect with OpenAI Codex / AGENTS.md Hosts

This guide explains how to use **Subscription Architect** in AI coding agents that support the open **Agent Skills** (`SKILL.md`) convention or `AGENTS.md` instruction files (such as OpenAI Codex CLI and compatible agents).

> **Verification Status**: Unverified on live Codex CLI in this repository's test environment (developed and verified first in Google Antigravity; core skill instructions in `SKILL.md`, `references/`, and `templates/` are 100% host-neutral Markdown and contain zero host-specific tool dependencies).

---

## 1. Installation Options

### Option A: Standard `.agents/skills/` Directory
For agents that discover `SKILL.md` packages under `.agents/skills/` or `.codex/skills/`:

```bash
mkdir -p .agents/skills
git clone https://github.com/Havindu21/subscription-architect.git .agents/skills/subscription-architect
```

### Option B: Referencing from Project `AGENTS.md`
If your coding agent loads `AGENTS.md` from the root of your repository, add a section pointing the agent to the installed skill directory when working on billing, plans, entitlements, or payment webhooks:

```markdown
## Subscription & Billing Architecture (`subscription-architect`)

Whenever asked to design, add, refactor, audit, or test subscriptions, recurring billing, plan tiers, or payment webhooks:
1. Read and follow `.agents/skills/subscription-architect/SKILL.md` before making architectural decisions or modifying code.
2. Start in **Architecture Mode** (inspect repository -> interview developer -> compare Approaches A/B/C/D -> present proposal using `templates/architecture-proposal.md`) and wait for explicit developer approval before entering **Implementation Mode**.
```
