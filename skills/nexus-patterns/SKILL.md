---
skill: dev-patterns
version: 1.0.0
context: current
description: |
  Reusable development patterns library extracted from 368 real sessions.
  Reference this when implementing code to follow proven patterns.
tags: [patterns, best-practices, development]
---

# Development Patterns Library

Proven patterns from 368 real AI-assisted development sessions.

## Pattern: Specification-First Development

**When to use:** Starting any new feature or phase of work.

**The pattern:**
```python
# ALWAYS read the spec before implementing
1. Read specification/requirements document
2. Validate inputs (recipes, configs, plans)
3. Identify discrepancies
4. Fix discrepancies
5. Re-validate
6. ONLY THEN implement
```

**Why it works:**
- Zero rework from misunderstood requirements
- Clear success criteria
- Transparent progress

**Evidence:** 100% of successful phases used this.

---

## Pattern: Parallel Validation

**When to use:** Multiple independent verification tasks.

**The pattern:**
```python
# Delegate ALL independent tasks in parallel
delegate(agent="A", instruction="Task A")
delegate(agent="B", instruction="Task B")
delegate(agent="C", instruction="Task C")

# Collect results
# Batch fix failures
# Proceed when all pass
```

**Why it works:**
- ~40% faster than sequential
- Discovers multiple issues at once
- Different agents = different perspectives

**Example:**
```python
delegate(agent="foundation:explorer", instruction="Survey codebase")
delegate(agent="python-dev:code-intel", instruction="Trace call hierarchy")
delegate(agent="foundation:zen-architect", instruction="Review architecture")
```

---

## Pattern: Test-Driven Verification

**When to use:** After any code implementation.

**The pattern:**
```python
# Implement
delegate(agent="foundation:modular-builder",
         instruction="Implement feature X with TDD")

# Verify IMMEDIATELY
run_tests()

# Fix failures IMMEDIATELY (not later)
if failures:
    diagnose()
    fix()
    run_tests()  # Re-verify

# Only proceed when tests pass
```

**Why it works:**
- Binary truth (pass/fail)
- Catches breaking changes early
- Confidence in deliverables

**Requirement:** 100% pass rate, not "mostly passing".

---

## Pattern: Issue Diagnosis Before Fix

**When to use:** Something is broken or failing.

**The pattern:**
```python
# 1. Identify SPECIFIC issue
"Error: KeyError at line 42"  # Not "it's broken"

# 2. Research ROOT CAUSE
delegate(agent="foundation:bug-hunter",
         instruction="Diagnose this KeyError with full context")

# 3. Communicate findings
# Document what failed and WHY

# 4. THEN fix
delegate(agent="foundation:modular-builder",
         instruction="Fix based on root cause")
```

**Why it works:**
- Prevents thrashing on symptoms
- Informs stakeholders early
- Confidence in the fix

---

## Pattern: Honest Fallback Strategies

**When to use:** A tool or workflow fails.

**The pattern:**
```python
# Acknowledge failure clearly
"The recipe timed out (10 min limit)"

# Analyze WHY
"Complexity requires detailed analysis beyond recipe scope"

# Identify alternative
"Execute step-by-step with manual delegation"

# Execute fallback
delegate(agent="foundation:modular-builder", ...)
```

**Why it works:**
- Completes despite tool failures
- Maintains momentum
- Documents workaround for future

**Anti-pattern:** Fighting a failing tool for hours.

---

## Pattern: Context Sink Delegation

**When to use:** Heavy documentation or complex questions.

**The pattern:**
```python
# Root session: thin awareness only
# Heavy docs: loaded by specialist agent

delegate(agent="methodology-expert",
         instruction="Explain STAGE THREE in detail",
         context_depth="none")

# Expert carries docs via @mentions
# Root session stays lean
```

**Why it works:**
- Root session: ~400 tokens for awareness
- Heavy docs: absorbed by sub-session
- Expert has full context without bloating root

---

## Pattern: Git Workflow Safety

**When to use:** Creating commits and PRs.

**The pattern:**
```python
# NEVER use git directly
# ALWAYS delegate to git-ops

delegate(agent="foundation:git-ops",
         instruction="Commit these changes with semantic message:
         [describe what was accomplished]",
         context_depth="recent")

# For PRs
delegate(agent="foundation:git-ops",
         instruction="Create PR:
         - Summary: [what changed]
         - Tests: [count]
         - Findings: [review summary]",
         context_depth="all",
         context_scope="agents")
```

**Why git-ops:**
- Conventional commit messages
- Co-author attribution
- PR description quality
- Safety guardrails

---

## Pattern: Multi-Platform Selection

**When to use:** Choosing which AI platform for a task.

**Decision tree:**
```
Multi-phase work? → Amplifier
Need specialized agents? → Amplifier
Quick debug/fix? → Claude CLI
UI/frontend work? → GitHub Copilot CLI
GitHub-specific? → GitHub Copilot CLI
Repeatable automation? → Amplifier (recipe)
Exploratory? → Claude CLI
```

**Evidence:**
- Amplifier: 48 sessions (systematic workflows)
- Claude CLI: 292 sessions (rapid iteration)
- GitHub Copilot: 28 sessions (UI + CI/CD)

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails | Correct Pattern |
|--------------|--------------|-----------------|
| Implement before reading spec | Rework required | Specification-first |
| Fix before diagnosing | Thrashing | Issue diagnosis before fix |
| Sequential validation | Slow | Parallel validation |
| "It should work" | No evidence | Test-driven verification |
| Fighting broken tools | Wastes time | Honest fallback |
| Loading docs in root | Token bloat | Context sink delegation |
| Git commands directly | Inconsistent messages | git-ops delegation |

---

## Quick Reference: Agent Delegation

| Need | Delegate To |
|------|-------------|
| Code understanding | foundation:explorer |
| Architecture review | foundation:zen-architect |
| Bug diagnosis | foundation:bug-hunter |
| Code implementation | foundation:modular-builder |
| Test generation | foundation:test-coverage |
| Git operations | foundation:git-ops |
| Methodology questions | methodology-expert (this bundle) |
| PR workflow | pr-orchestrator (this bundle) |

---

## Measuring Success

Track these metrics:

- **Test pass rate:** Target 100%
- **Issues found before PR:** Target 5+
- **Issues blocking completion:** Target 0
- **Time to PR:** Target <2 hours
- **Rework after PR:** Target <10%

If metrics decline, review which patterns you're skipping.
