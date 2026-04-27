# The 7-Stage DevOps Methodology

This methodology was extracted from 368 real AI-assisted development sessions over 2 weeks.
It represents a proven, replicable process for systematic development.

## Overview

The workflow consists of two stages:

**STAGE ONE** (Pre-Commit) — Automated by this bundle
```
Finish work
  ↓ Auto-generate tests (unit, integration, E2E)
  ↓ Code review (zen-architect + bug-hunter)
  ↓ Fix high-priority issues
  ↓ Document unfixed issues
  ↓ Update tests as needed
  ↓ Finalize documentation
  ↓ Commit to branch + open PR
```

**STAGE TWO** (CI/CD) — Your existing GitHub workflows
```
PR opened
  ↓ Lint + Type Check + Test
  ↓ Docker build
  ↓ Deploy to staging (0% traffic)
  ↓ Smoke tests (comprehensive suites)
  ↓ Promote to production (100% traffic)
```

## The 7 Stages (STAGE ONE Detail)

### 1. REQUEST VALIDATION (3-5 turns)

**Purpose:** Understand what needs to be done before starting.

**Actions:**
- Read the specification or requirements
- Validate input artifacts (recipes, plans, configs)
- Identify discrepancies between intent and inputs
- Document findings

**Success Criteria:**
- Clear understanding of what to build
- All input artifacts validated
- Discrepancies documented

**Anti-Pattern:** Starting implementation without reading the spec.

---

### 2. ISSUE DIAGNOSIS (2-4 turns)

**Purpose:** Find and understand problems before attempting fixes.

**Actions:**
- Identify SPECIFIC issues (not "it's broken")
- Research ROOT CAUSE thoroughly
- Communicate findings with CONTEXT
- Document all issues found

**Success Criteria:**
- Each issue has a clear description
- Root causes identified
- Issues categorized by priority

**Anti-Pattern:** Jumping to fixes without understanding why something failed.

---

### 3. REMEDIATION (3-8 turns)

**Purpose:** Fix issues in isolation before proceeding.

**Actions:**
- Fix issues in isolated copies (not production artifacts)
- Re-validate after each fix
- Verify fixes don't introduce new issues

**Success Criteria:**
- All issues fixed
- Validation passes
- No new issues introduced

**Anti-Pattern:** Fixing issues in-place without re-validation.

---

### 4. EXECUTION STRATEGY (1-2 turns)

**Purpose:** Decide how to proceed given the current state.

**Actions:**
- Assess feasibility of original plan
- Decide on approach (automated recipe vs manual workflow)
- Communicate strategy decision

**Success Criteria:**
- Clear execution plan chosen
- Stakeholders informed
- Fallback strategy identified if needed

**Anti-Pattern:** Forcing a broken recipe to work instead of adapting.

---

### 5. IMPLEMENTATION (10-30 turns)

**Purpose:** Build the feature/fix following best practices.

**Actions:**
- Delegate to specialized agents (foundation:modular-builder, etc.)
- Run independent tasks in parallel
- Collect results
- Document what was built

**Success Criteria:**
- All code written
- Implementation matches spec
- No syntax errors

**Anti-Pattern:** Implementing without delegation (burns context, slower).

---

### 6. QUALITY ASSURANCE (5-10 turns)

**Purpose:** Verify the implementation works correctly.

**Actions:**
- Run type checks (TypeScript, Python)
- Run full test suite
- Fix failures IMMEDIATELY
- Re-run until all tests pass
- Document test count and pass rate

**Success Criteria:**
- Type checks pass
- All tests pass (100% pass rate)
- Test count documented

**Anti-Pattern:** "Most tests pass" — must be 100%.

---

### 7. REPORTING (1-2 turns)

**Purpose:** Document what was completed with evidence.

**Actions:**
- Summarize what was delivered
- Document test status with evidence
- List any concerns or caveats
- Provide clear "done" signal with metrics

**Success Criteria:**
- Clear completion summary
- Evidence provided (test counts, etc.)
- Next steps identified

**Anti-Pattern:** "It's done" without evidence.

---

## Key Success Metrics

From 368 real sessions:

| Metric | Target | Evidence |
|--------|--------|----------|
| Test pass rate | 100% | All 3 analyzed phases achieved this |
| Issues found before PR | 5+ | 6-8 issues found per phase |
| Issues blocking completion | 0 | All phases completed despite issues |
| Time to PR | <2 hours | 45-120 min observed |
| Rework after PR | <10% | Minimal rework needed |

## Effectiveness Drivers

These patterns made the methodology work:

1. **Specification-first** — Prevents rework (100% of sessions)
2. **Issue diagnosis before fix** — Prevents thrashing (100% of sessions)
3. **Parallel validation** — ~40% faster (multiple agents simultaneously)
4. **Honest failure handling** — Zero blockers (switched strategies when needed)
5. **Test-driven verification** — 100% pass rate (binary truth, not claims)
6. **Documented reasoning** — Clear audit trail (every decision explained)

## When to Deviate

The methodology is a guide, not a straitjacket. Deviate when:

- **Emergency hotfix** — Skip stages 1-4, go straight to fix + test + PR
- **Trivial change** — Single typo or comment fix doesn't need full workflow
- **Exploratory work** — Research and prototyping don't fit this structure

But for production features, the full 7-stage workflow prevents costly mistakes.
