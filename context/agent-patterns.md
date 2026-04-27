# Agent Orchestration Patterns

Proven patterns for orchestrating AI agents in DevOps workflows.

## The Core Hierarchy

```
Orchestrator Agent (zen-architect, pr-orchestrator)
├─ Analyzes requirements
├─ Diagnoses issues
├─ Designs workflows
├─ Delegates implementation
└─ Reviews quality
    │
    ├── Explorer Agents (foundation:explorer, python-dev:code-intel)
    │   ├─ Spec discovery
    │   ├─ Repository analysis
    │   └─ Code understanding
    │
    ├── Specialist Agents (recipe-author, test-coverage)
    │   ├─ Recipe validation
    │   ├─ Schema repair
    │   └─ Test generation
    │
    └── Executor Agents (foundation:modular-builder)
        ├─ Code implementation
        ├─ TDD discipline
        └─ Test execution
```

## Pattern 1: Spec-First Validation

**When to use:** Starting any new feature or phase of work.

**The pattern:**
```python
# ALWAYS read spec before implementing
delegate(agent="foundation:explorer", 
         instruction="Read the spec file and validate requirements")

# Validate inputs in parallel
delegate(agent="recipes:recipe-author",
         instruction="Validate the recipe schema")

# Identify discrepancies
# THEN proceed to implementation
```

**Why it works:**
- Prevents rework from misunderstood requirements
- Creates clear success criteria
- Transparent progress tracking

**Evidence:** 100% of successful phases used this pattern.

---

## Pattern 2: Parallel Validation Streams

**When to use:** Multiple independent verification tasks.

**The pattern:**
```python
# Delegate ALL independent tasks in parallel
delegate(agent="foundation:explorer", instruction="Survey codebase")
delegate(agent="python-dev:code-intel", instruction="Trace call hierarchy")  
delegate(agent="foundation:zen-architect", instruction="Review architecture")

# Collect results
# Batch fix any failures
# Proceed when all pass
```

**Why it works:**
- ~40% faster than sequential execution
- Discovers multiple issues at once
- Different perspectives reveal different problems

**Evidence:** Phase 2 completed in 48 turns vs 117 for Phase 1 (sequential).

---

## Pattern 3: Issue Diagnosis Before Fix

**When to use:** Something is broken or failing.

**The pattern:**
```python
# 1. Identify the SPECIFIC issue
delegate(agent="foundation:bug-hunter",
         instruction="Diagnose this error with full context")

# 2. Research ROOT CAUSE
# Agent returns detailed analysis

# 3. Communicate findings
# Document what failed and WHY

# 4. THEN decide on fix approach
delegate(agent="foundation:modular-builder",
         instruction="Fix issue X based on root cause Y")
```

**Why it works:**
- Prevents thrashing on symptoms
- Informs stakeholders early
- Builds confidence in the fix

**Evidence:** All phases that hit issues diagnosed before fixing. Zero issues blocked completion.

---

## Pattern 4: Honest Fallback Strategies

**When to use:** A tool or recipe fails.

**The pattern:**
```python
# Acknowledge failure clearly
"The recipe timed out on planning step (10 min limit)"

# Analyze WHY it failed
"Complexity requires detailed analysis"

# Identify manual alternative
"Execute step-by-step: delegate both Rust backend and Amplifier bundle in parallel"

# Execute fallback workflow
delegate(agent="foundation:modular-builder", ...)
delegate(agent="foundation:modular-builder", ...)
```

**Why it works:**
- Completes despite tool failures
- Maintains momentum
- Documents the workaround for future reference

**Evidence:** Phase 3 switched from recipe to manual delegation and still completed successfully.

---

## Pattern 5: Context Sink Delegation

**When to use:** Heavy documentation or complex methodology questions.

**The pattern:**
```python
# Root session has THIN awareness pointer only
# Heavy docs live in specialist agent

delegate(agent="methodology-expert",
         instruction="Explain STAGE THREE in detail",
         context_depth="none")  # Expert carries all docs via @mentions

# Expert returns concise answer
# Root session avoids 3-5k token documentation cost
```

**Why it works:**
- Root session stays lean (~400 tokens for awareness)
- Documentation cost absorbed by sub-session
- Expert has full context without cluttering root

**Evidence:** Foundation bundle architecture uses this extensively.

---

## Pattern 6: Test-Driven Verification

**When to use:** After any code implementation.

**The pattern:**
```python
# After implementation
delegate(agent="foundation:modular-builder",
         instruction="Implement feature X with TDD")

# Verify immediately
bash(command="pytest tests/ -v")

# Fix failures IMMEDIATELY (not later)
if failures:
    delegate(agent="foundation:bug-hunter",
             instruction="Diagnose test failures")
    delegate(agent="foundation:modular-builder",
             instruction="Fix based on diagnosis")
    bash(command="pytest tests/ -v")  # Re-verify

# Only proceed when tests pass
```

**Why it works:**
- Binary truth (pass/fail), no ambiguity
- Catches breaking changes early
- Confidence in deliverables

**Evidence:** All 3 phases ended with 100% test pass rate.

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails | Correct Pattern |
|--------------|--------------|-----------------|
| Implement before reading spec | Rework required | Spec-first validation |
| Fix before diagnosing | Thrashing on symptoms | Issue diagnosis before fix |
| Sequential validation | Slow | Parallel validation streams |
| "It should work" claims | No evidence | Test-driven verification |
| Fighting broken tools | Wastes time | Honest fallback strategies |
| Loading heavy docs in root | Token bloat | Context sink delegation |

---

## Decision Framework: When to Delegate

| Situation | Delegate? | To Which Agent? |
|-----------|-----------|-----------------|
| Understanding a spec | Yes | foundation:explorer |
| Validating a recipe | Yes | recipes:recipe-author |
| Writing code | Yes | foundation:modular-builder |
| Debugging errors | Yes | foundation:bug-hunter |
| Architecture questions | Yes | foundation:zen-architect |
| Methodology questions | Yes | methodology-expert (this bundle) |
| Quick file read (<5 files) | No | Use read_file directly |
| Simple bash command | No | Use bash directly |

**Rule of thumb:** Delegate when the task would consume >1000 tokens in the root session
or requires specialized domain knowledge.

---

## Agent Failure Handling

When an agent fails or returns incomplete results:

1. **Check the instruction** — Was it clear and specific?
2. **Check the context** — Did the agent have what it needed?
3. **Try a different agent** — Maybe a specialist would work better
4. **Fall back to manual** — Sometimes direct tool use is faster
5. **Document the workaround** — Help others avoid the same issue

**Never fight a failing agent.** Adapt and move forward.

---

## Measuring Effectiveness

Track these metrics to evaluate agent orchestration:

- **Turn efficiency** — Turns per hour (target: 50-70)
- **Parallel execution** — Tasks run simultaneously (target: 2-3)
- **Issue resolution** — Issues found vs. issues blocking (target: 0 blockers)
- **Test pass rate** — Final test status (target: 100%)
- **Token efficiency** — Root session size after major work (target: <50k tokens)

If metrics decline, review delegation patterns and agent selection.
