---
name: autonomous-pipeline
description: Chains the full development lifecycle automatically — from spec through ship — without waiting for human approval between steps. Uses subagent-driven development with adversarial review and git worktree isolation. Use when you want to go from idea or requirement to working code in a single uninterrupted run.
---

# Autonomous Pipeline

> **CRITICAL RULE — READ THIS FIRST**
>
> In the Build phase (Phase 4), you MUST call `Agent()` to dispatch subagents for EVERY task. You NEVER write implementation code yourself. No matter how small the task — even a one-line change — you dispatch it via `Agent()`. If you find yourself using Edit/Write/Bash to create or modify project code, STOP — you are violating this rule.

## Overview

Runs the full development lifecycle as a single autonomous flow. The human provides the initial requirement; the agent delivers working, tested, reviewed code. Implementation uses subagent-driven development (implementer + independent reviewers) inside an isolated git worktree. The human only needs to intervene at two points: spec approval and final merge decision.

## When to Use

- You want to go from idea/requirement to working code without manually triggering each step
- The task is well-scoped enough that an agent can make reasonable decisions autonomously
- You want independent verification (not just self-review) of the implementation

**NOT for:**
- Exploratory work where requirements will change mid-stream
- Tasks requiring external approvals (legal, design sign-off, stakeholder review)
- Production deployments (the pipeline stops before deploy — deploying always needs human confirmation)

## The Pipeline

```
Input (idea/requirement/bug)
    │
    ▼
┌─────────┐    ┌─────────┐  HUMAN   ┌─────────┐    ┌───────────┐    ┌─────────┐    ┌─────────┐  HUMAN
│ ASSESS  │───▶│  SPEC   │─APPROVE─▶│  PLAN   │───▶│ WORKTREE  │───▶│  BUILD  │───▶│ REPORT  │─MERGE─▶ Done
│         │    │         │          │         │    │  SETUP    │    │  LOOP   │    │         │  DECIDE
└─────────┘    └─────────┘          └─────────┘    └───────────┘    │(subagent│    └─────────┘
  Phase 0        Phase 1              Phase 2        Phase 3        │ driven) │      Phase 5
                                                                    │+visual  │
                                                                    │ review) │
                                                                    └─────────┘
                                                                      Phase 4
```

**Human touchpoints: only 2** — spec approval and merge decision. Everything else is autonomous.

## Model Strategy

The **controller** (the agent running this pipeline) is **opus**. Phases 0, 1, 2, 5 run inline (controller handles directly). **Phase 3 and Phase 4 MUST dispatch subagents via Agent() — no exceptions, no matter how small the task.**

| Phase | Model | Execution | Why |
|-------|-------|-----------|-----|
| Phase 0 | opus | Inline (controller) | Assess input — controller handles directly |
| Phase 1 | opus | Inline (controller) | Spec generation — controller handles directly |
| Phase 2 | opus | Inline (controller) | Plan generation — controller handles directly |
| Phase 3 | **haiku** | **Subagent** `Agent({model: "haiku"})` | Mechanical git ops — different model |
| Phase 4 | mixed | **Subagents** per role | **ALL code work MUST be subagents** — see Phase 4 |
| Phase 5 | opus | Inline (controller) | Report generation — controller handles directly |

**Rule:** The controller NEVER writes implementation code. Every task in Phase 4 MUST be dispatched as a subagent via `Agent()`, regardless of size or complexity. If a subagent needs a different model than opus, pass `model`. If it uses opus, omit `model` (inherits).

## Process

### Phase 0: Assess Input

Classify the input and determine the starting point:

```
Input type?
    │
    ├── Vague idea ──────────→ Start at Phase 1 (Spec)
    ├── Clear requirement ───→ Start at Phase 1 (Spec)
    ├── Existing spec ───────→ Start at Phase 2 (Plan)
    ├── Existing plan ───────→ Start at Phase 3 (Worktree)
    └── Bug report ──────────→ Skip to Bug Flow (below)
```

State assumptions before proceeding:

```
AUTONOMOUS PIPELINE — STARTING
Input type: [vague idea / clear requirement / existing spec / bug]
Starting at: Phase [N]
Consulting mode: [none / single-consultant / team-consulting]
Visual review mode: [ON (Figma URLs detected) / OFF]
Reference sources: [list or "none"]
Figma sources: [list of Figma URLs or "none"]
Assumptions:
1. [assumption]
2. [assumption]
Proceeding automatically unless blocked.
```

**Figma detection:** If the input includes Figma URLs or references Figma designs, activate the visual review mode:

```
Input includes Figma URL or design references?
    │
    ├── YES → Visual review mode: ON
    │   Read design with get_design_context — ⚠️ annotations are critical, always read them
    │   Phase 1 Spec: include Figma URL → nodeId mapping per page/component
    │                 extract and list all annotations (design intent, constraints, interaction specs)
    │   Phase 2 Plan: tag each UI task with its Figma nodeId + relevant annotations
    │   Phase 4 Build: add VISUAL REVIEWER to the subagent loop
    │                  require mock data in every UI task
    │                  pass annotations to implementer as requirements
    │
    └── NO  → Visual review mode: OFF (standard flow)
```

**Refactoring/rewriting detection:** If the input involves refactoring, rewriting, or migrating an existing system, inventory all reference sources and choose the consulting mode:

```
Reference sources inventory:
    │
    ├── 0 sources (greenfield) ──────→ Consulting mode: none
    │
    ├── 1-3 sources ─────────────────→ Consulting mode: single-consultant
    │   Use legacy-consultant-prompt.md with named sources table
    │   One ephemeral subagent per question, lower token cost
    │
    └── 4+ sources OR cross-domain ──→ Consulting mode: team-consulting
        Invoke team-consulting skill
        Persistent specialist agents per source domain
        Cross-reference via messaging, ~4x token cost
```

Reference sources can include:
- **Codebase:** old source code, different branch, separate project
- **Documentation:** API specs, Swagger/OpenAPI, business requirements, PRDs
- **Schema:** database schema, migrations, ERD
- **Config:** environment configs, feature flags, infra setup
- **External:** integration docs, third-party API references

### Phase 1: Spec (inline — opus)

Follow `spec-driven-development`. Generate a structured specification covering:
1. Objective and acceptance criteria
2. Technical approach
3. Boundaries (always do / never do)
4. Testing strategy

Save as `SPEC.md`.

**Gate 1 — Spec self-check:**
- [ ] Objective is specific and measurable
- [ ] Acceptance criteria are testable
- [ ] Technical approach is stated
- [ ] Boundaries are defined

→ Self-check passes: **present to human for approval.** This is the first and primary human touchpoint. Wait for explicit approval before proceeding.

→ Human approves: proceed to Phase 2.
→ Human requests changes: update spec, re-present.

### Phase 2: Plan (inline — opus)

Follow `planning-and-task-breakdown`. From the approved spec:
1. Identify dependency graph
2. Slice into vertical tasks (each independently testable)
3. Write acceptance criteria per task
4. Order by dependencies

Save plan to `tasks/plan.md` and task list to `tasks/todo.md`.

**Gate 2 — Plan quality check (autonomous):**
- [ ] Every task has acceptance criteria
- [ ] Every task is independently verifiable
- [ ] No circular dependencies
- [ ] Reasonable scope (< 20 tasks)

→ All pass: proceed to Phase 3 automatically.
→ Scope too large: split into milestones, build the first one.

### Phase 3: Worktree Setup (subagent — haiku)

**Different model required.** Dispatch a subagent with `model: "haiku"`:

```
Agent({
  description: "Setup worktree",
  model: "haiku",
  prompt: "Create worktree for <feature-name>. Follow using-git-worktrees: ..."
})
```

The subagent follows `using-git-worktrees`:
1. Creates worktree in `.worktrees/<feature-name>`
2. Creates feature branch
3. Runs project setup (npm ci, cargo build, etc.)
4. Verifies test baseline passes

**Gate 3 — Worktree ready check (autonomous):**
- [ ] Worktree created and git-ignored
- [ ] Setup completed without errors
- [ ] All tests pass (baseline verified)

→ All pass: proceed to Phase 4 automatically.
→ Baseline tests fail: report to human (pre-existing issue, not ours).

### Phase 4: Build Loop (subagent-driven — mixed models)

**You MUST NOT implement tasks yourself. You MUST use the Agent() tool to dispatch subagents for every task — no matter how small.** Even a one-line config change gets dispatched. The controller orchestrates — it NEVER writes implementation code. If you catch yourself writing code instead of an Agent() call, STOP and dispatch a subagent instead.

For each task in dependency order, make these Agent() calls in sequence:

#### 4a. IMPLEMENTER — writes code + tests + commits

```
Agent({
  model: "sonnet",           // straightforward task (CRUD, UI, config, tests)
  // omit model for complex task (architecture, security, refactoring) — inherits opus
  description: "Implement T[N]: <task name>",
  prompt: "You are an implementer. Your job is to write code and tests.

TASK:
<paste full task text + acceptance criteria here>

CONTEXT:
<where this fits in the project, relevant file paths>

FIGMA ANNOTATIONS (if UI task):
<paste all annotations from get_design_context — these are critical requirements>

RULES:
- Follow TDD: write a failing test FIRST, then implement to make it pass
- Commit after each meaningful change
- [If visual review ON] Include mock data for all UI elements
- [If Figma annotations exist] Treat annotations as hard requirements — they contain design intent, interaction specs, and constraints

TEST COMMAND: <npm test / cargo test / etc.>

Report status: DONE / DONE_WITH_CONCERNS / BLOCKED"
})
```

Wait for result. If BLOCKED about legacy system → dispatch a consultant (inherits opus). If DONE → proceed to 4b.

#### 4b. SPEC REVIEWER — verifies acceptance criteria independently

```
Agent({
  model: "sonnet",
  description: "Spec review T[N]: <task name>",
  prompt: "You are a spec reviewer. You DISTRUST the implementer's self-report.

TASK + ACCEPTANCE CRITERIA:
<paste original task text>

RULES:
- Read the ACTUAL CODE changes — do not trust summaries
- Verify EVERY acceptance criterion is met
- Check for scope creep (features beyond spec)
- Check for missing requirements

Report: APPROVED / NEEDS_FIXES (with specific list) / MAJOR_ISSUES"
})
```

If NEEDS_FIXES → re-dispatch implementer with fix list → re-run spec review. Max 3 cycles.

#### 4c. VISUAL REVIEWER (optional — only for UI tasks with Figma nodeId)

```
Agent({
  model: "haiku",
  description: "Visual review T[N]: <task name>",
  prompt: "You are a visual reviewer. Compare Figma design vs built UI.
<Figma nodeId, page URL, comparison instructions>"
})
```

#### 4d. BROWSER TESTER (all UI tasks)

```
Agent({
  model: "haiku",
  description: "Browser test T[N]: <task name>",
  prompt: "You are a browser tester. Run smoke tests on the built page.
<page URL, test checklist: console errors, RWD breakpoints, network, interaction, a11y>"
})
```

#### 4e. CODE QUALITY REVIEWER — only after all above pass

```
Agent({
  model: "sonnet",
  description: "Code quality review T[N]: <task name>",
  prompt: "You are a code quality reviewer.

TASK CONTEXT:
<task text>

FILES CHANGED:
<list of files>

Check: readability, naming, test quality, architecture alignment, no dead code.
Report: APPROVED / SUGGESTIONS (non-blocking) / NEEDS_FIXES"
})
```

If NEEDS_FIXES → re-dispatch implementer → re-run quality review. Max 2 cycles.

#### 4f. Mark task complete → next task

After all tasks complete:
- Run full test suite
- Run build
- Dispatch final cross-task reviewer (inherits opus) across ALL changes

**Gate 4 — Build completion check (autonomous):**
- [ ] All tasks complete
- [ ] All tests pass
- [ ] Build is clean
- [ ] Final review has no Critical/Important findings
- [ ] If visual review ON: all UI tasks passed visual comparison
- [ ] All UI tasks passed chrome smoke test (console, RWD, network, interaction, a11y)

→ All pass: proceed to Phase 5 automatically.
→ Stuck after 3 fix cycles on same issue: STOP and ask human.

### Phase 5: Report + Merge Decision

The controller generates the report directly (no subagent needed — this is a template fill).

Summarize what was done and present the merge decision to the human:

```
AUTONOMOUS PIPELINE — COMPLETE

Spec: [link to SPEC.md]
Plan: [N] tasks completed
Branch: [feature-branch-name]
Worktree: [.worktrees/feature-name]
Changes: [summary of what was built]
Commits: [N] commits
Tests: [N] passing
Build: clean

Subagent execution:
- Implementer dispatches: [N]
- Legacy consultant queries: [N] (if refactoring)
- Spec review cycles: [N]
- Visual review cycles: [N] (if Figma)
- Browser test cycles: [N]
- Quality review cycles: [N]
- Issues auto-fixed: [N]

Non-blocking suggestions:
- [list from quality reviewers]

Ready for your decision:
1. Merge to [base-branch]
2. Create Pull Request
3. Keep branch for further work
4. Discard
```

**This is the second human touchpoint.** Wait for the human to decide.

## Bug Flow

For bug reports, use a compressed pipeline without spec or plan:

```
1. Create worktree (isolate the fix)
2. Write a failing test that reproduces the bug
3. Localize the root cause
4. Fix the minimum code to pass the test
5. Run full test suite for regressions
6. Commit
7. Report + merge decision
```

Invokes `using-git-worktrees` + `debugging-and-error-recovery` + `test-driven-development`. No subagents needed for single-fix bugs.

## When to STOP and Ask

The pipeline halts and asks the human only when:

1. **Spec approval** — Always. This is the intentional human checkpoint.
2. **Merge decision** — Always. This is the intentional human checkpoint.
3. **Ambiguity that can't be reasonably inferred** — Two valid interpretations with significantly different implementations
4. **Missing information** — External API keys, credentials, third-party service details
5. **Architectural fork** — A decision that affects the entire system and can't be easily reversed
6. **Repeated failure** — Same issue after 3 fix attempts across subagent cycles
7. **Security concern** — Discovered vulnerability in existing production code
8. **Scope explosion** — Task breakdown reveals scope 3x larger than expected
9. **Pre-existing test failures** — Baseline tests fail in fresh worktree

**Before stopping for NEEDS_CONTEXT, try the legacy consultant first:**

```
Implementer reports NEEDS_CONTEXT
    │
    ├── Question about old/legacy system? ──→ Dispatch legacy consultant
    │   │                                     (reads old code, answers question)
    │   ├── Consultant answers ─────────────→ Feed answer to implementer, continue
    │   └── Consultant can't answer ────────→ STOP and ask human
    │
    └── Not about legacy system? ───────────→ Controller resolves or STOP and ask human
```

Everything else: make the reasonable choice, document it, keep going.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I should ask the user about this minor detail" | If both choices are reasonable and reversible, pick one, document it, and move on. Stopping for trivia defeats the purpose of autonomous execution. |
| "I'll skip the worktree, it's extra overhead" | 30 seconds of setup buys you a clean rollback. If the implementation goes sideways, discard the worktree instead of untangling main. |
| "I'll implement it myself instead of dispatching subagents" | NO. Every task in Phase 4 MUST be dispatched via Agent(), no matter how small — even a one-line change. Subagents get fresh context; your context is accumulating the entire pipeline. Dispatch. |
| "This task is too small for a subagent" | There is no such thing. The rule is absolute: all implementation work goes through Agent(). A 30-second dispatch is cheaper than a controller that drifts into writing code. |
| "The spec is obvious, I'll skip to planning" | The spec is the human's checkpoint. Skipping it means the human never approved what you're building. |
| "I'll skip the spec review, the implementer's tests pass" | Tests verify behavior. Spec review verifies completeness. An implementer can pass all tests while missing a requirement. |
| "I'll do one big commit at the end" | Per-task commits make it possible to revert individual changes. One commit is an all-or-nothing gamble. |
| "This is too complex to do autonomously" | Break it smaller. If a single task is too complex, it needs decomposition, not human hand-holding. |

## Red Flags

- **Controller writing ANY implementation code in Phase 4** — the controller NEVER writes code; every task MUST be dispatched via `Agent()`, no matter how small
- **Dispatching a sonnet/haiku subagent without specifying `model`** — if the role needs a different model, the Agent() call must include it
- Stopping to ask about trivial/reversible decisions
- Skipping worktree setup
- Skipping spec or quality reviews
- Not running full test suite between tasks
- Proceeding past spec phase without human approval
- Auto-merging without human decision
- One giant commit instead of per-task commits

## Verification

After the pipeline completes, confirm:
- [ ] Spec exists, was approved by human, and matches what was built
- [ ] All planned tasks are marked complete
- [ ] Work was done in an isolated worktree/branch
- [ ] Every task was implemented by a subagent (not the controller)
- [ ] Every task passed independent spec review
- [ ] Every task passed code quality review
- [ ] All tests pass
- [ ] Build is clean
- [ ] Each task has its own commit(s)
- [ ] Final report delivered with merge options
- [ ] Human made the merge/discard decision
