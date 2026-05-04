# DevOps Methodology

This session includes the DevOps methodology bundle. Four specialist agents are available:

- **pr-orchestrator** — Use when executing the dev-to-PR workflow. Drives STAGE ONE
  from test generation through PR creation. Run the `dev-to-pr` recipe for automation.

- **methodology-expert** — Use for questions about the 7-stage methodology, agent
  orchestration patterns, or platform selection (Amplifier vs Claude CLI vs GitHub Copilot).
  Delegate ALL methodology questions to this agent — it carries the full documentation.

- **workflow-coach** — Use when a developer wants interactive, guided STAGE ONE
  walkthrough instead of the automated recipe.

- **cname-orchestrator** — Use when setting up custom domains (CNAMEs) for Azure Container Apps.
  Guides through Microsoft internal DNS requests, Azure configuration, SSL certificate binding,
  and troubleshooting. Run the `azure-cname-setup` recipe for guided automation.

**Recipes available:**
- `dev-to-pr` — Automated STAGE ONE workflow (test generation → review → fixes → docs → PR)
- `azure-cname-setup` — Complete custom domain setup with approval gates for manual DNS steps

**Skills available** (use `load_skill` tool):
- `dev-workflow` — Guided STAGE ONE walkthrough
- `dev-patterns` — Reusable development patterns library
- `azure-deployment` — Azure Container Apps deployment patterns
- `azure-cname-setup` — Quick reference for custom domain setup

**CRITICAL:** Do not attempt to answer methodology questions from memory.
Delegate to methodology-expert for authoritative answers.
