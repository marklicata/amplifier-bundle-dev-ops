# DevOps Workflow Playbook

Complete team reference for the DevOps methodology.

## Table of Contents

1. [Overview](#overview)
2. [When to Use Which Platform](#when-to-use-which-platform)
3. [The STAGE ONE Workflow](#the-stage-one-workflow)
4. [The 7-Stage Methodology](#the-7-stage-methodology)
5. [Agent Usage Patterns](#agent-usage-patterns)
6. [Success Metrics](#success-metrics)
7. [Real Examples](#real-examples)
8. [Common Pitfalls](#common-pitfalls)
9. [Team Practices](#team-practices)

---

## Overview

This methodology was extracted from **368 real AI-assisted development sessions** over 2 weeks.
It represents a proven, systematic approach to development from code to production.

### The Two Stages

**STAGE ONE** (Pre-Commit) — Automated by this bundle:
```
Finish work → Generate tests → Code review → Fix issues → 
Update docs → Create PR
```

**STAGE TWO** (CI/CD) — Your existing GitHub workflows:
```
PR opened → Build → Test → Deploy staging → Smoke test → 
Promote production
```

This playbook focuses on STAGE ONE.

---

## When to Use Which Platform

From 368 sessions across three AI platforms:

| Platform | Sessions | Best For |
|----------|----------|----------|
| **Amplifier** | 48 | Phase-based work, agent orchestration, systematic workflows |
| **Claude CLI** | 292 | Quick debugging, rapid iteration, 2-3 min cycles |
| **GitHub Copilot CLI** | 28 | UI/UX work, CI/CD debugging, long conversations |

### Decision Tree

```
What are you doing?

Multi-phase feature work? → Amplifier
  ├─ Example: Phase 1 (Foundation) → Phase 2 (Selection) → Phase 3 (Integration)
  └─ Why: Systematic orchestration with zen-architect → explorer → modular-builder

Need specialized agents? → Amplifier
  ├─ Architecture review (zen-architect)
  ├─ Bug diagnosis (bug-hunter)
  └─ Test generation (test-coverage)

Quick debugging? → Claude CLI
  ├─ Example: Docker errors, API endpoint issues
  └─ Why: Fast iteration, 2-3 minute cycles

UI/UX feature? → GitHub Copilot CLI
  ├─ Example: Point+click selection, visual components
  └─ Why: Long conversations, visual work expertise

GitHub-specific work? → GitHub Copilot CLI
  ├─ Example: Workflow debugging, GitHub Actions
  └─ Why: Native GitHub integration

Repeatable automation? → Amplifier (recipe)
  └─ Example: The dev-to-PR recipe in this bundle

Exploratory work? → Claude CLI
  └─ Example: "What if we tried..." scenarios
```

---

## The STAGE ONE Workflow

### Quick Reference

```bash
# Automated (recommended)
amplifier run -r dev-to-pr

# Interactive
amplifier chat
> delegate to workflow-coach

# Q&A
amplifier chat
> Ask methodology-expert about [topic]
```

### What Happens in STAGE ONE

1. **Assess Changes** (1-2 min)
   - What files changed?
   - What features/fixes included?
   - What testing needed?

2. **Generate Tests** (3-5 min)
   - Unit tests for functions/classes
   - Integration tests for APIs
   - E2E tests for workflows

3. **Run Initial Tests** (2-5 min)
   - Establish baseline
   - Catch obvious issues
   - Document failures

4. **Code Review** (5-10 min)
   - zen-architect: Architecture, design, simplicity
   - bug-hunter: Bugs, edge cases, error handling
   - Parallel execution

5. **Apply Fixes** (10-30 min)
   - Fix CRITICAL/HIGH priority issues
   - Document MEDIUM/LOW issues
   - Verify with tests

6. **Update Tests** (3-5 min)
   - Fix broken tests
   - Add new tests if needed
   - 100% pass rate required

7. **Update Documentation** (3-10 min)
   - API specs
   - README
   - CHANGELOG
   - Config docs

8. **Create PR** (1-2 min)
   - Semantic commit message
   - Comprehensive PR description
   - Link to documented issues

**Total time:** 30-90 minutes (depending on complexity)

---

## The 7-Stage Methodology

### Stage 1: Request Validation

**Goal:** Understand what needs to be done.

**Actions:**
- Read specification/requirements
- Validate inputs (recipes, plans, configs)
- Identify discrepancies
- Document findings

**Success criteria:**
- ✓ Clear understanding of what to build
- ✓ All inputs validated
- ✓ Discrepancies documented

**Anti-pattern:** Starting without reading the spec.

### Stage 2: Issue Diagnosis

**Goal:** Find and understand problems.

**Actions:**
- Identify SPECIFIC issues (not "it's broken")
- Research ROOT CAUSE
- Communicate findings with CONTEXT
- Document all issues

**Success criteria:**
- ✓ Each issue clearly described
- ✓ Root causes identified
- ✓ Issues prioritized

**Anti-pattern:** Jumping to fixes without diagnosis.

### Stage 3: Remediation

**Goal:** Fix issues before proceeding.

**Actions:**
- Fix in isolation (not in-place)
- Re-validate after each fix
- Verify no new issues introduced

**Success criteria:**
- ✓ All issues fixed
- ✓ Validation passes
- ✓ No regressions

**Anti-pattern:** Fixing without re-validation.

### Stage 4: Execution Strategy

**Goal:** Decide how to proceed.

**Actions:**
- Assess feasibility
- Choose approach (automated vs manual)
- Communicate strategy

**Success criteria:**
- ✓ Clear plan chosen
- ✓ Stakeholders informed
- ✓ Fallback identified

**Anti-pattern:** Forcing a broken tool to work.

### Stage 5: Implementation

**Goal:** Build the feature/fix.

**Actions:**
- Delegate to specialists
- Run independent tasks in parallel
- Document what was built

**Success criteria:**
- ✓ Code written
- ✓ Matches spec
- ✓ No syntax errors

**Anti-pattern:** Doing it all yourself (burns context).

### Stage 6: Quality Assurance

**Goal:** Verify it works.

**Actions:**
- Run type checks
- Run full test suite
- Fix failures IMMEDIATELY
- Re-run until 100% pass

**Success criteria:**
- ✓ Type checks pass
- ✓ All tests pass (100%)
- ✓ Test count documented

**Anti-pattern:** "Most tests pass" (must be 100%).

### Stage 7: Reporting

**Goal:** Document completion with evidence.

**Actions:**
- Summarize deliverables
- Document test status
- List concerns/caveats
- Provide "done" signal with metrics

**Success criteria:**
- ✓ Clear summary
- ✓ Evidence provided
- ✓ Next steps identified

**Anti-pattern:** "It's done" without evidence.

---

## Agent Usage Patterns

### Pattern: Spec-First Development

1. Read specification first
2. Validate inputs
3. Fix discrepancies
4. Then implement

```bash
amplifier chat
> delegate(agent="foundation:explorer", 
           instruction="Read spec and validate requirements")
```

### Pattern: Parallel Validation

1. Identify independent tasks
2. Delegate ALL in parallel
3. Collect results
4. Batch fix

```bash
delegate(agent="foundation:explorer", instruction="Survey codebase")
delegate(agent="python-dev:code-intel", instruction="Trace calls")
delegate(agent="foundation:zen-architect", instruction="Review architecture")
```

### Pattern: Honest Fallback

1. Acknowledge failure
2. Analyze WHY
3. Execute alternative
4. Document workaround

**Example:** Recipe times out → Switch to manual step-by-step delegation.

---

## Success Metrics

From 368 real sessions:

| Metric | Target | Evidence |
|--------|--------|----------|
| Test pass rate | 100% | All phases achieved |
| Issues found before PR | 5+ | 6-8 per phase |
| Issues blocking | 0 | Zero blockers |
| Time to PR | <2 hours | 45-120 min |
| Rework after PR | <10% | Minimal |

Track these for your team to measure effectiveness.

---

## Real Examples

### Example 1: Phase 2 Point+Click (Amplifier)
- **Session:** 771b50fce62d44ed
- **Duration:** 48 turns (~45 minutes)
- **Pattern:** zen-architect → explorer → modular-builder
- **Outcome:** 101 tests passing, complete
- **Lesson:** Systematic phase execution with specialized agents

### Example 2: Docker Debugging (Claude CLI)
- **Date:** April 20
- **Sessions:** 13 sessions
- **Duration:** 2-3 minutes each
- **Pattern:** Error → fix → test → iterate
- **Outcome:** Auth + Docker issues resolved
- **Lesson:** Fast iteration for well-defined problems

### Example 3: UI Feature (GitHub Copilot CLI)
- **Session:** session_20260423_093720_27668
- **Duration:** 172 messages
- **Pattern:** Design → implement → refine
- **Outcome:** Visual highlighting working
- **Lesson:** Long interactive sessions for UI work

---

## Common Pitfalls

| Pitfall | Why It Fails | Solution |
|---------|--------------|----------|
| Skip spec reading | Rework required | Always read spec first |
| Fix before diagnosing | Thrashing | Diagnose root cause first |
| Sequential execution | Slow | Parallel where possible |
| "Should work" claims | No evidence | Test-driven verification |
| Fight broken tools | Wastes time | Honest fallback |
| Heavy docs in root | Token bloat | Context sink pattern |

---

## Team Practices

### For Individual Developers

**Daily workflow:**
1. Implement feature
2. Run `amplifier run -r dev-to-PR`
3. Review PR description, make adjustments
4. Submit PR
5. Watch CI/CD (STAGE TWO)

### For Code Reviewers

**PR review checklist:**
- ✓ Tests added (count documented)
- ✓ Code review findings addressed
- ✓ Documentation updated
- ✓ 100% test pass rate
- ✓ PR description comprehensive

### For Team Leads

**Adoption metrics:**
- Recipe usage frequency
- PR quality (time in review, rework needed)
- Test coverage trends
- Team velocity

### For Onboarding

**New developer checklist:**
1. Install bundle (see QUICK-START.md)
2. Run recipe on sample change
3. Ask questions to methodology-expert
4. Use workflow-coach for first real PR
5. Review playbook (this doc)

---

## Quick Command Reference

```bash
# Run automated workflow
amplifier run -r dev-to-pr

# Get guided help
amplifier chat
> delegate to workflow-coach

# Ask methodology questions
amplifier chat
> Ask methodology-expert about [topic]

# Load pattern library
amplifier chat
> load_skill(skill_name="dev-patterns")

# Validate recipe
amplifier recipe validate dev-to-pr
```

---

**Last updated:** 2026-04-27  
**Version:** 1.0.0  
**Based on:** 368 sessions, April 13-27, 2026
