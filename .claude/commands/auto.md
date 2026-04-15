---
description: Run the full development lifecycle autonomously — spec, plan, build, test, review — without stopping between steps
---

Invoke the agent-skills:autonomous-pipeline skill.

Run the complete development pipeline from the user's input to working, tested, reviewed code — automatically, with only two human touchpoints.

## MANDATORY RULE — BUILD PHASE

**In Phase 4 (Build), you MUST call the `Agent()` tool to dispatch subagents for EVERY task. No exceptions. No matter how small.**

You are the controller. You NEVER write implementation code yourself. For each task, you call in sequence:
```
Agent({model: "sonnet", description: "Implement T[N]: ...", prompt: "..."})
// straightforward task → model: "sonnet" (CRUD, UI, config, tests)
// complex task → omit model (inherits opus: architecture, security, refactoring)
```
Then spec review:
```
Agent({model: "sonnet", description: "Spec review T[N]: ...", prompt: "..."})
```
Then visual review (only for UI tasks with Figma nodeId):
```
Agent({model: "haiku", description: "Visual review T[N]: ...", prompt: "..."})
```
Then browser test (all UI tasks):
```
Agent({model: "haiku", description: "Browser test T[N]: ...", prompt: "..."})
```
Then code quality review (only after all above pass):
```
Agent({model: "sonnet", description: "Code quality review T[N]: ...", prompt: "..."})
```

If you find yourself using Edit, Write, or Bash to create/modify project code — STOP. You are violating this rule. Dispatch a subagent instead.

---

## Pipeline Phases

0. **Assess** the input — classify type, inventory reference sources, choose consulting mode
1. **Spec** — generate a structured specification, then **present to human for approval** (touchpoint 1)
2. **Plan** — break into vertical tasks with acceptance criteria
3. **Worktree** — create isolated git worktree + verify test baseline. Dispatch subagent: `Agent({model: "haiku", ...})`
4. **Build** — for EACH task, dispatch subagents via `Agent()` tool:
   - `Agent()` → IMPLEMENTER (TDD: failing test → implement → pass → commit)
   - `Agent()` → SPEC REVIEWER (adversarial — reads code, distrusts implementer)
   - `Agent()` → VISUAL REVIEWER (optional — UI tasks with Figma nodeId, compare design vs built)
   - `Agent()` → BROWSER TESTER (all UI tasks — console, RWD, network, interaction, a11y)
   - `Agent()` → CODE QUALITY REVIEWER (readability, architecture, tests — only after all above pass)
   - Legacy questions → `Agent()` → CONSULTANT (no human interruption)
   - All autonomous — no human approval between tasks
5. **Report** — summarize what was built, present **merge decision to human** (touchpoint 2)

Human intervenes only at: spec approval (Phase 1) and merge decision (Phase 5).

Consulting modes (auto-detected based on reference sources):
- **none** — greenfield project, no legacy to reference
- **single-consultant** — 1-3 reference sources, ephemeral subagent per question
- **team-consulting** — 4+ sources or cross-domain, persistent specialist agents (invoke `team-consulting`)

Gate rules:
- Each phase has a quality gate. Pass → proceed automatically. Fail → attempt self-fix.
- STOP only for: spec approval, merge decision, genuine ambiguity, missing external info, repeated failures (3x same issue), architectural forks, or security concerns.
- Everything else: make the reasonable choice, document it, keep going.

For bug reports: skip spec/plan, create worktree, go straight to reproduce → localize → fix → verify → commit → report.
