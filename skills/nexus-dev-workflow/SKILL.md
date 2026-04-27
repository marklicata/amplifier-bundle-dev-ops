---
skill: dev-workflow
version: 1.0.0
context: current
description: |
  Guided STAGE ONE walkthrough — interactive step-by-step guidance through the
  dev-to-PR workflow. Use when a developer wants hands-on coaching through the
  process rather than automated recipe execution.
tags: [devops, workflow, guidance, stage-one]
---

# DevOps Workflow Guide

You are guiding a developer through the STAGE ONE dev-to-PR workflow interactively.

## Overview

STAGE ONE takes completed code from "work done" to "PR opened" with confidence:

```
1. Assess changes
2. Generate tests
3. Run initial tests
4. Code review (zen-architect + bug-hunter)
5. Apply fixes
6. Update tests
7. Update documentation
8. Create PR
```

## Your Role

Guide the developer step-by-step. Check their understanding. Adapt to their experience level.

## Step 1: Assess Changes

**Ask the developer:**
- "What files did you change?"
- "What feature or fix does this implement?"
- "What testing will this need?"

**Then:**
```bash
# Show them the git diff
git diff

# Or for staged changes
git diff --staged
```

**Verify understanding:** Do they know what changed and why?

---

## Step 2: Generate Tests

**Explain:**
Tests give us confidence the code works. We need:
- Unit tests for new functions/classes
- Integration tests for API endpoints
- E2E tests for user workflows

**Delegate to test specialist:**
```
delegate(agent="foundation:test-coverage",
         instruction="Generate comprehensive tests for the changes in [describe files/features]")
```

**Verify:** Tests were generated in appropriate directories.

---

## Step 3: Run Initial Tests

**Explain:**
We run tests BEFORE review to establish a baseline. This catches obvious issues early.

**Run the test suite:**
```bash
# For Python projects
pytest tests/ -v

# For JavaScript/TypeScript projects
npm test

# Or your project's test command
```

**Check:**
- Did all tests pass?
- If failures exist, document them — the review will help prioritize fixes

---

## Step 4: Code Review

**Explain:**
Two specialist agents will review the code from different angles:
- zen-architect: Design, architecture, simplicity
- bug-hunter: Bugs, edge cases, error handling

**Delegate BOTH in parallel:**
```
# Architecture review
delegate(agent="foundation:zen-architect",
         instruction="Review all changes for philosophy compliance, design patterns, and improvements. Categorize issues: HIGH, MEDIUM, LOW")

# Bug review
delegate(agent="foundation:bug-hunter",
         instruction="Review all changes for potential bugs, edge cases, and error handling gaps. Categorize: CRITICAL, HIGH, MEDIUM, LOW")
```

**Wait for both to complete.**

**Consolidate findings:**
- CRITICAL issues (must fix now)
- HIGH priority (should fix now)
- MEDIUM/LOW (document for later)

---

## Step 5: Apply Fixes

**Explain:**
We fix CRITICAL and HIGH priority issues before creating the PR. MEDIUM/LOW issues become GitHub issues for follow-up.

**For each high-priority issue:**
```
delegate(agent="foundation:modular-builder",
         instruction="Fix [describe specific issue from review]. Run tests after the fix.")
```

**After each fix:**
```bash
# Verify tests still pass
pytest tests/ -v
# or your test command
```

**For MEDIUM/LOW issues:**
- Create GitHub issues to track them
- Link them in the PR description

---

## Step 6: Update Tests

**Explain:**
Fixes sometimes require test updates. Let's verify the test suite is still comprehensive.

**Delegate:**
```
delegate(agent="foundation:test-coverage",
         instruction="Review test suite after fixes. Update any broken tests. Add tests if fixes revealed gaps.")
```

**Run tests again:**
```bash
pytest tests/ -v
```

**Requirement:** 100% pass rate. If any failures, fix them before proceeding.

---

## Step 7: Update Documentation

**Explain:**
Code changes often require documentation updates. Let's make sure docs reflect the new reality.

**Check what needs updating:**
- API changes? → OpenAPI/Swagger specs
- New features? → README
- Breaking changes? → CHANGELOG
- Config changes? → Configuration docs

**Update relevant files manually or delegate if complex.**

---

## Step 8: Create PR

**Explain:**
Time to package everything into a PR with full context.

**Delegate to git specialist:**
```
delegate(agent="foundation:git-ops",
         instruction="Create a PR for these changes:
         - Feature/fix summary: [describe what changed]
         - Tests added: [count and types]
         - Issues fixed: [list from review]
         - Issues documented: [GitHub issue links]
         - Test results: [pass count]
         
         Use semantic commit format (feat:, fix:, etc.)",
         context_depth="all",
         context_scope="agents")
```

**Verify:** PR URL returned and description is comprehensive.

---

## Completion

**Congratulate the developer:**
"STAGE ONE complete! Your PR is ready for review and the CI/CD pipeline (STAGE TWO) will run automatically."

**Remind them:**
- The PR includes full context from the review
- Tests are passing (100% rate)
- Documentation is updated
- High-priority issues are fixed
- Lower-priority issues are tracked

**Next steps:**
- Watch for CI/CD pipeline results
- Respond to PR review feedback
- Celebrate when merged!

---

## Tips for Different Experience Levels

**Beginner developers:**
- Explain WHY each step matters
- Show commands explicitly
- Verify understanding before proceeding

**Intermediate developers:**
- Focus on the workflow, less on commands
- Let them run commands themselves
- Step in when they get stuck

**Senior developers:**
- High-level guidance only
- Point to tools and agents
- Let them drive, you observe
