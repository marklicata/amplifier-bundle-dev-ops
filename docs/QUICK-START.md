# Quick Start Guide

Get up and running with the DevOps methodology bundle in 5 minutes.

## Installation

### Option 1: Add to Existing Workspace

```bash
cd your-project/
amplifier bundle add git+https://github.com/your-org/amplifier-bundle-dev-ops@main
```

### Option 2: Create Workspace Config

```bash
cd your-project/
mkdir -p .amplifier

cat > .amplifier/bundle.yaml <<EOF
bundle:
  includes:
    - bundle: git+https://github.com/your-org/amplifier-bundle-dev-ops@main
EOF
```

### Option 3: Global Installation

```bash
# Add to your global Amplifier config
amplifier config bundle add git+https://github.com/your-org/amplifier-bundle-dev-ops@main
```

## Verify Installation

```bash
# List available agents
amplifier agents list

# Should see:
# - pr-orchestrator
# - methodology-expert
# - workflow-coach

# List available recipes
amplifier recipes list

# Should see:
# - dev-to-pr
```

## First Use: The Automated Way

After making code changes:

```bash
# Run the complete STAGE ONE workflow
amplifier run -r dev-to-pr

# The recipe will:
# 1. Assess your changes
# 2. Generate tests
# 3. Run initial tests
# 4. Code review (2 agents in parallel)
# 5. Apply high-priority fixes
# 6. Update tests
# 7. Update documentation
# 8. Create PR with full context
```

## First Use: The Guided Way

If you prefer interactive guidance:

```bash
# Start a session
amplifier chat

# Then
> I'm ready to create a PR, guide me through it
```

The system will delegate to `workflow-coach` automatically and walk you through each step.

## First Use: Questions and Learning

To understand the methodology:

```bash
amplifier chat

# Ask questions like:
> What is the 7-stage methodology?
> When should I use Amplifier vs Claude CLI?
> Walk me through STAGE THREE
```

The system will delegate to `methodology-expert` which has all the documentation loaded.

## Understanding STAGE ONE

The bundle automates the pre-commit workflow:

```
Your finished code
  ↓ Tests generated (unit, integration, E2E)
  ↓ Code review (architecture + bug hunting)
  ↓ High-priority issues fixed
  ↓ Documentation updated
  ↓ PR created with full context
  ↓
STAGE TWO (your CI/CD) takes over
```

## Skills Available

After installation, load skills for specific guidance:

```bash
# In any Amplifier session
load_skill(skill_name="dev-workflow")   # Guided STAGE ONE walkthrough
load_skill(skill_name="dev-patterns")   # Reusable pattern library
```

## Common Workflows

### Creating a PR for a Feature

```bash
# After implementing your feature
git add .
amplifier run -r dev-to-pr
# Wait for completion, PR is created automatically
```

### Learning the Methodology

```bash
amplifier chat
> I want to understand the 7-stage methodology
# System delegates to methodology-expert
```

### Getting Guided Help

```bash
amplifier chat
> Walk me through creating a PR step by step
# System delegates to workflow-coach
```

### Quick Pattern Reference

```bash
amplifier chat
> load_skill(skill_name="dev-patterns")
> What's the parallel validation pattern?
```

## Customization

### Override Recipe Behavior

Create `.amplifier/recipes/dev-to-pr.yaml` in your project to customize the workflow:

```yaml
name: dev-to-pr
description: Custom STAGE ONE workflow

# Inherit from bundle recipe
extends: dev-ops:recipes/dev-to-pr

# Override specific steps
steps:
  - name: generate-tests
    # Your custom test generation logic
```

### Add Project-Specific Patterns

Create `.amplifier/skills/project-patterns/SKILL.md`:

```markdown
---
skill: project-patterns
version: 1.0.0
description: Project-specific development patterns
---

# Project Patterns

[Your patterns here]
```

## Troubleshooting

### Recipe fails to run

```bash
# Validate the recipe
amplifier recipe validate dev-to-pr

# Check logs
amplifier logs
```

### Agents not available

```bash
# Verify bundle is loaded
amplifier bundle list

# Should see: dev-ops (1.0.0)

# If not, re-add the bundle
amplifier bundle add git+https://github.com/your-org/amplifier-bundle-dev-ops@main
```

### Skills not loading

```bash
# Skills from bundles are auto-discovered
# Just use load_skill directly:
load_skill(skill_name="dev-workflow")

# If that fails, check bundle installation
amplifier bundle list
```

## Next Steps

1. **Read the playbook:** `docs/DEV-WORKFLOW-PLAYBOOK.md` for complete methodology
2. **Try the recipe:** Run `amplifier run -r dev-to-pr` on a feature branch
3. **Ask questions:** Use the `methodology-expert` agent for any methodology questions
4. **Customize:** Adapt the recipe and skills to your team's needs

## Team Onboarding

For your 15-developer team:

1. **Create a team doc:** Add `.amplifier/TEAM-SETUP.md` to your repo
2. **Standard installation:** Document your preferred installation method
3. **First PR demo:** Have each dev run the recipe on a small change
4. **Q&A session:** Use `methodology-expert` in a group session
5. **Iterate:** Collect feedback, customize the recipe

## Support

- **Methodology questions:** Ask `methodology-expert` agent
- **Workflow guidance:** Use `workflow-coach` agent
- **Pattern reference:** Load `dev-patterns` skill
- **Full documentation:** See `docs/DEV-WORKFLOW-PLAYBOOK.md`

---

## Azure Deployment

After your PR workflow is set up, add Azure deployment:

### Quick Setup

```bash
amplifier chat
> load_skill(skill_name="azure-deployment")
# Follow the guided walkthrough
```

### Pattern Reference

See `docs/AZURE-INTEGRATION.md` for the 7 proven patterns:
1. Managed Identity for ACR Pull
2. Secrets vs Environment Variables
3. OIDC Authentication
4. Application Insights Integration
5. Blue-Green Deployment
6. Comprehensive Smoke Tests
7. GitHub Secrets Management

### Intelligent Deployment

```bash
amplifier chat
> Deploy my application to Azure

# Azure ZAP will analyze and orchestrate the deployment
```

### Example GitHub Workflow

Use a production workflow as a reference:
- `.github/workflows/api-build-deploy.yml` — Complete production example

Or reference:
- `@dev-ops:context/azure-deployment-patterns.md` — Pattern 5 for workflow template
