---
name: using-agent-skills
description: Discovers and invokes agent skills. Use when starting a session or when you need to discover which skill applies to the current task. This is the meta-skill that governs how all other skills are discovered and invoked.
---

# Using Agent Skills

## Overview

Agent Skills is a collection of engineering workflow skills organized by development phase. Each skill encodes a specific process that senior engineers follow. This meta-skill helps you discover and apply the right skill for your current task.

## Skill Discovery

When a task arrives, identify the development phase and apply the corresponding skill:

```
Task arrives
    │
    ├── Want full auto (spec→ship)? ──→ autonomous-pipeline
    ├── Vague idea/need refinement? ──→ idea-refine
    ├── New project/feature/change? ──→ spec-driven-development
    ├── Have a spec, need tasks? ──────→ planning-and-task-breakdown
    ├── Refactoring/rewriting? ────────→ autonomous-pipeline (with consulting)
    │   ├── 1-3 reference sources? ──→ single legacy consultant
    │   └── 4+ sources/cross-domain? → team-consulting
    ├── Implementing code? ────────────→ incremental-implementation
    │   ├── Multi-task plan? ─────────→ subagent-driven-development
    │   ├── Need isolation? ──────────→ using-git-worktrees
    │   ├── UI work? ─────────────────→ frontend-ui-engineering
    │   │   └── Has Figma design? ──→ + figma-visual-review
    │   ├── API work? ────────────────→ api-and-interface-design
    │   ├── Need better context? ─────→ context-engineering
    │   └── Need doc-verified code? ───→ source-driven-development
    ├── Writing/running tests? ────────→ test-driven-development
    │   ├── Browser-based? ───────────→ browser-testing-with-devtools
    │   └── UI smoke test? ──────────→ chrome-smoke-test
    ├── Something broke? ──────────────→ debugging-and-error-recovery
    ├── Reviewing code? ───────────────→ code-review-and-quality
    │   ├── Security concerns? ───────→ security-and-hardening
    │   └── Performance concerns? ────→ performance-optimization
    ├── Committing/branching? ─────────→ git-workflow-and-versioning
    ├── CI/CD pipeline work? ──────────→ ci-cd-and-automation
    ├── Writing docs/ADRs? ───────────→ documentation-and-adrs
    └── Deploying/launching? ─────────→ shipping-and-launch
```

## Core Operating Behaviors

These behaviors apply at all times, across all skills. They are non-negotiable.

### 1. Surface Assumptions

Before implementing anything non-trivial, explicitly state your assumptions:

```
ASSUMPTIONS I'M MAKING:
1. [assumption about requirements]
2. [assumption about architecture]
3. [assumption about scope]
→ Correct me now or I'll proceed with these.
```

Don't silently fill in ambiguous requirements. The most common failure mode is making wrong assumptions and running with them unchecked. Surface uncertainty early — it's cheaper than rework.

### 2. Manage Confusion Actively

When you encounter inconsistencies, conflicting requirements, or unclear specifications:

1. **STOP.** Do not proceed with a guess.
2. Name the specific confusion.
3. Present the tradeoff or ask the clarifying question.
4. Wait for resolution before continuing.

**Bad:** Silently picking one interpretation and hoping it's right.
**Good:** "I see X in the spec but Y in the existing code. Which takes precedence?"

### 3. Push Back When Warranted

You are not a yes-machine. When an approach has clear problems:

- Point out the issue directly
- Explain the concrete downside (quantify when possible — "this adds ~200ms latency" not "this might be slower")
- Propose an alternative
- Accept the human's decision if they override with full information

Sycophancy is a failure mode. "Of course!" followed by implementing a bad idea helps no one. Honest technical disagreement is more valuable than false agreement.

### 4. Enforce Simplicity

Your natural tendency is to overcomplicate. Actively resist it.

Before finishing any implementation, ask:
- Can this be done in fewer lines?
- Are these abstractions earning their complexity?
- Would a staff engineer look at this and say "why didn't you just..."?

If you build 1000 lines and 100 would suffice, you have failed. Prefer the boring, obvious solution. Cleverness is expensive.

### 5. Maintain Scope Discipline

Touch only what you're asked to touch.

Do NOT:
- Remove comments you don't understand
- "Clean up" code orthogonal to the task
- Refactor adjacent systems as a side effect
- Delete code that seems unused without explicit approval
- Add features not in the spec because they "seem useful"

Your job is surgical precision, not unsolicited renovation.

### 6. Prefer MCP Tools Over CLI When Available

When MCP servers are connected, prefer MCP tools over equivalent CLI commands. This applies especially to GitHub:

```
GitHub URL or operation needed?
    │
    ├── GitHub MCP available (mcp__plugin_github_github__*)? 
    │   └── YES → Use MCP tools first
    │       Examples:
    │       - Read issue → mcp__plugin_github_github__issue_read
    │       - Read PR → mcp__plugin_github_github__pull_request_read
    │       - List PRs → mcp__plugin_github_github__list_pull_requests
    │       - Create PR → mcp__plugin_github_github__create_pull_request
    │       - Search code → mcp__plugin_github_github__search_code
    │
    └── NO → Fall back to gh CLI via Bash tool
```

**Why MCP first:** MCP tools return structured data, handle auth automatically, and avoid shell-parsing issues. Only fall back to `gh` CLI when the MCP tool doesn't support the specific operation.

This principle extends to all MCP servers — if a Figma MCP, Chrome MCP, or any other MCP is connected, prefer its tools over CLI/script equivalents.

### 7. Verify, Don't Assume

Every skill includes a verification step. A task is not complete until verification passes. "Seems right" is never sufficient — there must be evidence (passing tests, build output, runtime data).

## Failure Modes to Avoid

These are the subtle errors that look like productivity but create problems:

1. Making wrong assumptions without checking
2. Not managing your own confusion — plowing ahead when lost
3. Not surfacing inconsistencies you notice
4. Not presenting tradeoffs on non-obvious decisions
5. Being sycophantic ("Of course!") to approaches with clear problems
6. Overcomplicating code and APIs
7. Modifying code or comments orthogonal to the task
8. Removing things you don't fully understand
9. Building without a spec because "it's obvious"
10. Skipping verification because "it looks right"

## Skill Rules

1. **Check for an applicable skill before starting work.** Skills encode processes that prevent common mistakes.

2. **Skills are workflows, not suggestions.** Follow the steps in order. Don't skip verification steps.

3. **Multiple skills can apply.** A feature implementation might involve `idea-refine` → `spec-driven-development` → `planning-and-task-breakdown` → `incremental-implementation` → `test-driven-development` → `code-review-and-quality` → `shipping-and-launch` in sequence.

4. **When in doubt, start with a spec.** If the task is non-trivial and there's no spec, begin with `spec-driven-development`.

5. **For full autonomy, use `/auto`.** The `autonomous-pipeline` skill chains spec → plan → worktree → subagent build → review → report with only two human touchpoints (spec approval and merge decision).

## Lifecycle Sequence

For a complete feature, the typical skill sequence is:

```
1. idea-refine                     → Refine vague ideas
2. spec-driven-development         → Define what we're building
3. planning-and-task-breakdown     → Break into verifiable chunks
4. context-engineering             → Load the right context
5. source-driven-development       → Verify against official docs
6. using-git-worktrees             → Isolate work in a clean branch
7. subagent-driven-development     → Implement with adversarial review
   (or incremental-implementation  → Build slice by slice, single-agent)
8. test-driven-development         → Prove each slice works
9. code-review-and-quality         → Review before merge
10. git-workflow-and-versioning    → Clean commit history
11. documentation-and-adrs         → Document decisions
12. shipping-and-launch            → Deploy safely
```

Or use `autonomous-pipeline` (/auto) to chain steps 2–7 automatically.

Not every task needs every skill. A bug fix might only need: `debugging-and-error-recovery` → `test-driven-development` → `code-review-and-quality`.

## Quick Reference

| Phase | Skill | One-Line Summary |
|-------|-------|-----------------|
| Auto | autonomous-pipeline | Full lifecycle in one run — spec through reviewed code |
| Define | idea-refine | Refine ideas through structured divergent and convergent thinking |
| Define | spec-driven-development | Requirements and acceptance criteria before code |
| Plan | planning-and-task-breakdown | Decompose into small, verifiable tasks |
| Build | incremental-implementation | Thin vertical slices, test each before expanding |
| Build | subagent-driven-development | Dispatch implementer + adversarial reviewers per task |
| Build | using-git-worktrees | Isolated branch/directory for safe implementation |
| Build | team-consulting | Persistent specialist agents for complex refactoring with 4+ sources |
| Build | source-driven-development | Verify against official docs before implementing |
| Build | context-engineering | Right context at the right time |
| Build | frontend-ui-engineering | Production-quality UI with accessibility |
| Build | api-and-interface-design | Stable interfaces with clear contracts |
| Verify | test-driven-development | Failing test first, then make it pass |
| Verify | browser-testing-with-devtools | Chrome DevTools MCP for runtime verification |
| Verify | figma-visual-review | Figma vs browser screenshot comparison with team-mode feedback |
| Verify | chrome-smoke-test | Console, RWD, network, interaction, a11y testing via Chrome |
| Verify | debugging-and-error-recovery | Reproduce → localize → fix → guard |
| Review | code-review-and-quality | Five-axis review with quality gates |
| Review | security-and-hardening | OWASP prevention, input validation, least privilege |
| Review | performance-optimization | Measure first, optimize only what matters |
| Ship | git-workflow-and-versioning | Atomic commits, clean history |
| Ship | ci-cd-and-automation | Automated quality gates on every change |
| Ship | documentation-and-adrs | Document the why, not just the what |
| Ship | shipping-and-launch | Pre-launch checklist, monitoring, rollback plan |
