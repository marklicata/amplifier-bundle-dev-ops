# Multi-Platform Development Guide

Guidance on when to use Amplifier, Claude CLI, or GitHub Copilot CLI for development work.

## The Three Platforms

From analysis of 368 sessions across all three platforms:

| Platform | Sessions | Primary Strengths |
|----------|----------|-------------------|
| **Amplifier** | 48 | Phase-based orchestration, agent delegation, systematic workflows |
| **Claude CLI** | 292 | Rapid iteration, debugging, 2-3 minute cycles |
| **GitHub Copilot CLI** | 28 | UI/UX features, CI/CD debugging, long conversations |

## Decision Framework

### Use Amplifier When:

**✓ Multi-phase feature work**
- Building features that span multiple logical phases
- Clear milestones with review checkpoints
- Systematic orchestration needed

**Example:** Phase 1 (Foundation) → Phase 2 (Selection) → Phase 3 (Integration)

**✓ Agent specialization required**
- Need zen-architect for design decisions
- Need bug-hunter for systematic debugging
- Need test-coverage for comprehensive test generation

**Evidence:** 32 Amplifier sessions on April 21 completed Phases 1-4 with clear agent delegation.

**✓ Recipe-driven automation**
- Repeatable workflows you want to codify
- Multi-step processes with approval gates
- Team standardization

**Example:** The dev-to-PR recipe in this bundle.

**✓ Long-running complex work**
- Work that would exceed Claude CLI's context window
- Multiple specialist agents needed
- State management across many steps

---

### Use Claude CLI When:

**✓ Rapid iteration debugging**
- Docker errors that need quick fixes
- API endpoint debugging
- Shell script compatibility issues

**Example:** 292 Claude sessions averaged 2-3 minutes per iteration for debugging.

**✓ Quick fixes and tweaks**
- Single-file changes
- Configuration adjustments
- Simple bug fixes

**Evidence:** April 22-24 saw 256 Claude sessions with fast turnaround cycles.

**✓ Exploratory work**
- Trying different approaches
- Quick validation of ideas
- Prototype iteration

**✓ File-heavy work**
- Large codebase survey needed
- Multiple file changes rapidly
- Quick grep/search iterations

**Note:** Claude CLI excels at speed but lacks the orchestration capabilities of Amplifier.

---

### Use GitHub Copilot CLI (Agency) When:

**✓ UI/UX feature development**
- Frontend components
- User interaction flows
- Visual design work

**Example:** Point+click feature session — 172 messages iterating on UI behavior.

**✓ CI/CD debugging**
- GitHub workflow failures
- GitHub Actions debugging
- Deploy pipeline issues

**Example:** GitHub workflow debugging session — 597 messages, longest single session.

**✓ Long conversational work**
- Need extended context retention
- Interactive design sessions
- Complex problem-solving requiring back-and-forth

**Evidence:** Agency sessions averaged 150-400 messages vs Amplifier's 10-30 turns.

**✓ GitHub-specific work**
- Repository management
- Issue/PR workflows
- GitHub API interactions

---

## Platform Comparison

| Aspect | Amplifier | Claude CLI | GitHub Copilot CLI |
|--------|-----------|------------|---------------------|
| **Orchestration** | ★★★★★ Agent delegation | ★★☆☆☆ Manual | ★★☆☆☆ Manual |
| **Speed** | ★★★☆☆ Deliberate | ★★★★★ Fast | ★★★☆☆ Interactive |
| **Context Window** | ★★★★★ Unlimited via delegation | ★★★☆☆ Limited | ★★★★☆ Extended |
| **Repeatability** | ★★★★★ Recipes | ★☆☆☆☆ Manual | ★☆☆☆☆ Manual |
| **UI/Visual Work** | ★★☆☆☆ Text-focused | ★★☆☆☆ Text-focused | ★★★★☆ Better for UI |
| **GitHub Integration** | ★★★☆☆ Via git-ops | ★★☆☆☆ Manual | ★★★★★ Native |

## Real-World Session Examples

### Amplifier: Phase-Based Feature Work
**Session:** Phase 2 Point+Click Selection (771b50fce62d44ed)
- **Duration:** 48 turns (~45 minutes)
- **Pattern:** zen-architect → explorer → modular-builder → test-coverage
- **Outcome:** 101 tests passing, feature complete
- **Why Amplifier:** Systematic phase execution with multiple specialized agents

### Claude CLI: Rapid Debugging
**Sessions:** Docker + API debugging (April 20)
- **Duration:** 13 sessions, ~2-3 minutes each
- **Pattern:** Observe error → quick fix → test → iterate
- **Outcome:** Auth issues resolved, Docker entrypoint fixed
- **Why Claude CLI:** Fast iteration on well-defined problems

### Agency: UI Feature Development
**Session:** Point+click panel context (session_20260423_093720_27668)
- **Duration:** 172 messages
- **Pattern:** Design → implement → refine → polish
- **Outcome:** Visual element highlighting + context injection working
- **Why Agency:** Long interactive session for UI behavior refinement

## Hybrid Workflows

**Don't force one platform for everything.** Use them in combination:

### Pattern: Design → Build → Debug
```
1. Amplifier (zen-architect) — Design the architecture
2. Claude CLI — Rapid implementation iteration  
3. Amplifier (bug-hunter + test-coverage) — Systematic verification
4. Agency — GitHub workflow debugging if needed
```

### Pattern: Feature → Review → Ship
```
1. Claude CLI — Quick implementation
2. Amplifier (dev-to-PR recipe) — Systematic review + PR creation
3. Agency — GitHub workflow monitoring
```

### Pattern: Bug → Root Cause → Fix
```
1. Claude CLI — Initial debugging, narrow scope
2. Amplifier (bug-hunter) — Root cause analysis if complex
3. Claude CLI — Rapid fix iteration
4. Amplifier (test-coverage) — Regression test generation
```

## When NOT to Use Each Platform

### Don't use Amplifier for:
- Single-line fixes
- Quick config tweaks
- Exploratory "what if" questions
- Simple grep/search operations

→ Use Claude CLI instead

### Don't use Claude CLI for:
- Multi-agent orchestration
- Systematic code review
- Recipe-driven workflows
- Long-running complex features

→ Use Amplifier instead

### Don't use Agency for:
- Backend API logic
- Non-GitHub CI/CD
- Rapid iteration (too slow)
- Systematic orchestration

→ Use Amplifier or Claude CLI instead

## Platform Selection Checklist

Before starting work, ask:

1. **Is this multi-phase work?** → Amplifier
2. **Do I need specialized agents?** → Amplifier
3. **Is this a quick fix/debug?** → Claude CLI
4. **Is this UI/frontend work?** → Agency
5. **Is this GitHub-specific?** → Agency
6. **Do I need repeatable automation?** → Amplifier (recipe)
7. **Is this exploratory?** → Claude CLI

## Success Metrics by Platform

Track these to validate platform choice:

**Amplifier:**
- Phases completed successfully: Target 100%
- Agent delegation depth: Target 2-3 levels
- Recipe success rate: Target 80%+

**Claude CLI:**
- Average iteration time: Target <5 minutes
- Sessions per hour: Target 15-20
- Issue resolution rate: Target 90%+

**Agency:**
- Average message count: 100-500 (complex work)
- UI feature completion: Target 90%+
- GitHub workflow resolution: Target 85%+

If metrics don't meet targets, reconsider platform choice.

## Cross-Platform Learning

Patterns learned on one platform often transfer:

- **Claude CLI debugging patterns** → Amplifier bug-hunter instructions
- **Amplifier orchestration patterns** → Manual Agency workflows
- **Agency UI iteration** → Amplifier component-designer patterns

Document successful patterns for reuse across platforms.
