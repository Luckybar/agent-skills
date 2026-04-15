---
name: subagent-driven-development
description: Dispatches subagents with adversarial review to implement tasks autonomously. Use when executing a plan with multiple tasks, when you want isolated implementation with independent verification, or when building within the autonomous pipeline.
---

# Subagent-Driven Development

## Overview

Uses a controller + subagent architecture to implement tasks autonomously with built-in quality gates. Each task is executed by an implementer subagent, then independently verified by a spec reviewer and a code quality reviewer. The reviewers distrust the implementer's self-report — they verify everything from source. This adversarial trust model catches mistakes that self-review misses.

## When to Use

- Executing a task plan with 2+ tasks
- Inside the `autonomous-pipeline` during the Build phase
- Any multi-task implementation where you want independent verification
- When context window pressure makes single-agent implementation unreliable

**NOT for:**
- Single, small changes (use `incremental-implementation` directly)
- Exploratory work where the plan is still forming
- Tasks that require continuous human feedback during implementation

## Architecture

```
┌──────────────┐
│  CONTROLLER  │  Reads plan, dispatches subagents, tracks progress
└──────┬───────┘
       │
       │  For each task:
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ IMPLEMENTER  │────▶│ SPEC REVIEWER│────▶│   VISUAL     │────▶│  BROWSER     │────▶│CODE QUALITY  │
│  (subagent)  │     │  (subagent)  │     │  REVIEWER    │     │  TESTER      │     │  REVIEWER    │
│              │     │              │     │  (subagent)  │     │  (subagent)  │     │  (subagent)  │
│ Implement +  │     │ Verify spec  │     │ Figma vs     │     │ Console,RWD  │     │ Architecture │
│ TDD + commit │     │ compliance   │     │ browser      │     │ Network,a11y │     │ + clean code │
│ + mock data  │     │              │     │ comparison   │     │ interaction  │     │              │
└──────┬───────┘     └──────────────┘     └──────┬───────┘     └──────┬───────┘     └──────────────┘
       │                    │                    │                    │                    │
       │              Independent          Only for UI          All UI tasks        Only runs
       │  NEEDS_       verification        tasks with           (SendMessage         AFTER all
       │  CONTEXT?    (reads code,         Figma nodeId         loop with            prior reviews
       │    │          not report)         (optional)           implementer)         pass
       ▼    ▼
┌──────────────────┐
│ LEGACY CONSULTANT│  (optional — for refactoring/rewriting tasks)
│   (subagent)     │
│                  │
│ Reads old code,  │
│ explains business│
│ logic, maps deps │
└──────────────────┘
```

### Subagent Roles

| Role | Job | Writes Code? | When Dispatched |
|------|-----|-------------|-----------------|
| **Implementer** | Write code + tests + commit | Yes | Every task |
| **Spec Reviewer** | Verify acceptance criteria independently | No | After implementer completes |
| **Code Quality Reviewer** | Check readability, architecture, tests | No | After spec review passes |
| **Visual Reviewer** | Compare Figma design vs browser rendering, report visual diffs | No | After spec review, on UI tasks with Figma nodeId (optional) |
| **Browser Tester** | Chrome smoke test: console, RWD, network, interaction, a11y | No | After visual review (or spec review if no Figma), on all UI tasks |
| **Legacy Consultant** | Read & explain old code/docs/schemas, cross-reference sources | No | When NEEDS_CONTEXT on legacy system (optional) |

### Model Assignment

The controller is **opus** (default). Subagents that need a different model **MUST** specify `model` in the `Agent()` call. Subagents that use opus inherit it automatically — do not specify `model` for them.

| Role | Model | Agent() `model` param | Rationale |
|------|-------|-----------------------|-----------|
| **Controller** | opus | — (this is you) | Orchestration requires judgment calls |
| **Implementer** (straightforward) | **sonnet** | `model: "sonnet"` | Clear spec + TDD — follows instructions efficiently |
| **Implementer** (complex) | opus | *(omit — inherits)* | Deep reasoning for architecture and edge cases |
| **Spec Reviewer** | **sonnet** | `model: "sonnet"` | Reads code and verifies against acceptance criteria |
| **Visual Reviewer** | **haiku** | `model: "haiku"` | Screenshot comparison following a structured checklist |
| **Browser Tester** | **haiku** | `model: "haiku"` | Follows a rigid test script — low judgment needed |
| **Code Quality Reviewer** | **sonnet** | `model: "sonnet"` | Evaluates readability, architecture, test quality |
| **Legacy Consultant** | opus | *(omit — inherits)* | Deep analysis of business logic, cross-referencing sources |
| **Final Cross-Task Reviewer** | opus | *(omit — inherits)* | Holistic review across all changes |

**Rule:** If the table says `model: "sonnet"` or `model: "haiku"`, you MUST pass that parameter. If it says *(omit — inherits)*, do NOT pass `model` — the subagent inherits opus from the controller.

**How to judge "straightforward" vs "complex" for implementer tasks:**
- **Straightforward (→ sonnet):** CRUD operations, UI components with clear design, adding a field/endpoint, writing tests for existing logic, config changes
- **Complex (→ opus):** New architectural patterns, cross-cutting concerns, performance-critical paths, security-sensitive code, significant refactoring, complex state management

When in doubt, use opus — the cost of a wrong implementation exceeds the savings of a cheaper model.

## Process

### Step 1: Controller Setup

The controller (you) prepares the execution:

1. Read the full plan once — extract ALL tasks with complete text
2. Create a task tracking list with status for each task
3. Verify a git worktree is ready (invoke `using-git-worktrees` if not)
4. Confirm test baseline passes in the worktree
5. **If refactoring/rewriting:** identify ALL reference sources for the legacy consultant and build the source table:

```
Reference Sources:
| Name        | Type          | Path                    | What's In It                  |
|-------------|---------------|-------------------------|-------------------------------|
| old-code    | codebase      | src/old-api/            | Legacy implementation         |
| api-spec    | documentation | docs/api-spec.yaml      | API endpoint contracts        |
| db-schema   | schema        | db/migrations/          | Database tables & relations   |
| ...         | ...           | ...                     | ...                           |
```

This table will be passed to every legacy consultant dispatch. For 4+ complex sources, consider using `team-consulting` instead (persistent specialist agents per source).

### Step 2: Per-Task Execution Loop

For each task in dependency order:

#### 2a. Dispatch Implementer

**Straightforward task →** `Agent({model: "sonnet", ...})`. **Complex task →** `Agent({...})` (omit model — inherits opus).

Spawn a subagent using the `implementer-prompt.md` template. Provide:
- Full task text (paste it, never reference a file)
- Scene-setting context: where this fits in the overall project
- Relevant file paths and patterns from the codebase
- The test command to use
- **If Figma task:** all annotations from the design (annotations contain design intent, interaction specs, constraints — they are critical requirements, not optional notes)

The implementer must:
- Follow `test-driven-development` — write failing test first, then implement
- Commit after each meaningful change
- Report back with status: `DONE`, `DONE_WITH_CONCERNS`, `NEEDS_CONTEXT`, or `BLOCKED`

If `BLOCKED` or `NEEDS_CONTEXT`:

```
NEEDS_CONTEXT or BLOCKED
    │
    ├── About legacy/old system? ────→ Dispatch LEGACY CONSULTANT (inherits opus)
    │   Use legacy-consultant-prompt.md
    │   Provide: legacy source paths + the specific question
    │   Feed consultant's answer back to implementer and re-dispatch
    │
    ├── Controller can resolve? ─────→ Provide context, re-dispatch implementer
    │
    └── Requires human knowledge? ───→ STOP and ask human
```

The legacy consultant is a read-only subagent — it analyzes old code and answers questions but never writes new code. See `legacy-consultant-prompt.md` for the template.

#### 2b. Dispatch Spec Reviewer

**→** `Agent({model: "sonnet", ...})`

After implementer reports `DONE` or `DONE_WITH_CONCERNS`, spawn a spec reviewer using `spec-reviewer-prompt.md`. Provide:
- The original task text and acceptance criteria
- The implementer's reported status (but instruct reviewer NOT to trust it)

The spec reviewer must:
- Read the actual code changes (NOT the implementer's summary)
- Verify every acceptance criterion is met
- Check for scope creep (features added beyond spec)
- Check for missing requirements
- Report: `APPROVED`, `NEEDS_FIXES` (with specific list), or `MAJOR_ISSUES`

If `NEEDS_FIXES`: re-dispatch implementer with the fix list. Then re-run spec review. Max 3 cycles.

#### 2c. Dispatch Visual Reviewer (Optional — UI tasks with Figma)

**→** `Agent({model: "haiku", ...})`

Only for tasks that have a Figma nodeId mapping. Runs after spec review passes. Follows the `figma-visual-review` skill process.

The visual reviewer uses **team mode** (SendMessage) to communicate with the implementer in real-time:

```
Visual Reviewer activated
    │
    ├── 1. Get Figma screenshot (get_screenshot + get_design_context)
    ├── 2. Capture browser screenshot of built page
    ├── 3. Page-level comparison (layout, spacing, colors)
    ├── 4. Component-level comparison (typography, colors, borders, shadows)
    │
    ├── All match? → APPROVED → proceed to code quality review
    │
    └── Differences found? → SendMessage to implementer:
        │   - List each difference with expected vs actual values
        │   - Severity: high / medium / low
        │
        ├── Implementer fixes → SendMessage back: FIXES_APPLIED
        ├── Visual reviewer re-captures + re-compares
        └── Repeat (max 3 rounds per task)
            │
            └── Still failing after 3 rounds → log as non-blocking,
                proceed to code quality review
```

**Mock data requirement:** The implementer MUST include mock data for all UI tasks when visual review is active. Empty pages cannot be visually compared. Mock data rules:
- Text fields: realistic-length content matching the language/context
- Lists/tables: 3-5 sample entries
- Images: placeholder with correct aspect ratio
- Numbers: reasonable range values
- Mock data in dedicated fixture/seed files for easy replacement

#### 2d. Dispatch Browser Tester (All UI Tasks)

**→** `Agent({model: "haiku", ...})`

Runs on every task that produces UI output (pages, components with routes). Unlike visual review (Figma-dependent), browser testing always runs for UI tasks. Follows the `chrome-smoke-test` skill process.

The browser tester uses **team mode** (SendMessage) to communicate with the implementer:

```
Browser Tester activated
    │
    ├── 1. Navigate to page URL in Chrome
    ├── 2. Console test: read_console_messages → zero errors
    ├── 3. RWD test: resize to 320/375/768/1440 → screenshot each
    ├── 4. Network test: read_network_requests → all 2xx
    ├── 5. Interaction test: click links/buttons → no errors
    ├── 6. Accessibility test: alt, headings, focus, aria-label
    │
    ├── All pass? → APPROVED → proceed to code quality review
    │
    └── Failures found? → SendMessage to implementer:
        │   - List each failure with category + details
        │   - Attach RWD screenshots showing issues
        │
        ├── Implementer fixes → SendMessage back: FIXES_APPLIED
        ├── Browser tester re-runs failed tests
        └── Repeat (max 2 rounds per task)
            │
            └── ⚠️ warnings after 2 rounds → log as non-blocking
                ❌ errors after 2 rounds → escalate to human
```

**What gets tested:**
- **Console**: zero errors, zero warnings (warnings logged but non-blocking)
- **RWD**: 4 breakpoints (320/375/768/1440), no overflow or broken layout
- **Network**: all API requests return 2xx
- **Interaction**: internal links navigate, buttons respond, forms validate
- **Accessibility**: img alt, heading hierarchy, focusable elements, aria-labels

#### 2e. Dispatch Code Quality Reviewer

**→** `Agent({model: "sonnet", ...})`

Only after spec review, visual review, and browser test (all applicable ones) pass. Spawn using `code-quality-reviewer-prompt.md`. Provide:
- The task text for context
- Files changed by the implementer

The quality reviewer checks:
- Code readability and naming
- Test quality (not just existence — are they meaningful?)
- Architecture alignment with existing patterns
- Error handling
- No dead code or debug artifacts left behind

Report: `APPROVED`, `SUGGESTIONS` (non-blocking), or `NEEDS_FIXES`

If `NEEDS_FIXES`: re-dispatch implementer. Re-run quality review. Max 2 cycles.

#### 2f. Mark Task Complete

After all applicable reviews pass:
1. Update task status to complete
2. Log: task name, commits made, review cycles needed
3. Move to next task

### Step 3: Final Review

After all tasks are complete:
1. Run full test suite
2. Run build
3. Dispatch a final code quality reviewer across ALL changes (not per-task) — inherits opus from controller
4. Fix any cross-task issues found

### Step 4: Report

```
SUBAGENT EXECUTION — COMPLETE

Tasks completed: [N/N]
Total commits: [N]
Review cycles: [breakdown per task]
Final test suite: [pass/fail]
Final build: [pass/fail]

Issues for human review:
- [any DONE_WITH_CONCERNS notes]
- [any non-blocking SUGGESTIONS from reviewers]
```

## Controller Rules

1. **Read the plan once, extract everything upfront.** Don't re-read the plan file for each task — it wastes context.
2. **Paste task text into subagent prompts.** Never tell a subagent "read task 3 from the plan file." The subagent has fresh context — give it everything it needs.
3. **Don't do the implementer's job.** The controller dispatches and coordinates. If you catch yourself writing implementation code, stop — spawn a subagent instead.
4. **Trust the reviewers, not the implementer.** The implementer is optimistic by nature. The reviewers verify independently.
5. **Cap review cycles.** Spec review: max 3 cycles. Quality review: max 2 cycles. If still failing, escalate to human.
6. **Pass `model` when the subagent needs a different model than the controller (opus).** Check the Model Assignment table — if it says `model: "sonnet"` or `model: "haiku"`, you MUST include it in the Agent() call. If it says *(omit — inherits)*, do not pass `model`. Forgetting to pass `model` on a sonnet/haiku role wastes cost; passing it unnecessarily on an opus role is harmless but noisy.
7. **Route legacy questions to the consultant, not the human.** When an implementer reports NEEDS_CONTEXT about the old system, dispatch the legacy consultant first. Only escalate to the human if the consultant can't answer (e.g., requires production config or business decisions).

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll implement this myself, it's faster than dispatching a subagent" | Fresh context per subagent prevents drift. The 30-second dispatch cost saves 10 minutes of confused debugging in a polluted context. |
| "The implementer said it's done, I'll skip the spec review" | Implementers over-report completion. The spec reviewer exists precisely because self-assessment is unreliable. |
| "The spec review passed, quality review is overkill" | Spec compliance and code quality are orthogonal. Code can meet the spec while being unmaintainable. |
| "I'll batch all tasks into one subagent for efficiency" | One subagent doing 5 tasks will lose context by task 3. One task per subagent is the rule. |
| "The review found minor issues, I'll fix them myself instead of re-dispatching" | The controller doesn't write code. Re-dispatch the implementer — it maintains the separation of concerns. |
| "I'll just read the old code myself instead of dispatching the consultant" | The consultant gets fresh context dedicated to the legacy code. Your context is full of the plan, task tracking, and coordination state. Dispatch. |
| "The implementer is BLOCKED, I should ask the human immediately" | Try the legacy consultant first if the question is about the old system. Most NEEDS_CONTEXT issues can be resolved by reading the legacy code carefully. |

## Red Flags

- **Dispatching a sonnet/haiku subagent without `model` parameter** — check the Model Assignment table; if it says `model: "sonnet"` or `model: "haiku"`, the Agent() call MUST include it
- Controller writing implementation code instead of dispatching
- Skipping spec review because implementer "seems confident"
- Skipping quality review to save time
- Batching multiple tasks into a single subagent
- Not providing full task text to subagents (referencing files instead)
- Review cycle count exceeding caps without escalation
- Implementer not writing tests before implementation

## Verification

After subagent-driven execution completes, confirm:
- [ ] Every task was implemented by a subagent (not the controller)
- [ ] Every task passed spec review (independent verification)
- [ ] UI tasks with Figma nodeId passed visual review (if visual review active)
- [ ] UI tasks include mock data (if visual review or browser test active)
- [ ] UI tasks passed chrome smoke test (console, RWD, network, interaction, a11y)
- [ ] Every task passed code quality review
- [ ] Each task has its own commit(s)
- [ ] Full test suite passes
- [ ] Build is clean
- [ ] Final cross-task review completed
- [ ] All review findings addressed or logged for human
