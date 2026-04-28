---
description: Run the full development lifecycle autonomously — spec, plan, build, test, review — without stopping between steps
---

Invoke the agent-skills:autonomous-pipeline skill.

Run the complete development pipeline from the user's input to working, tested, reviewed code — automatically, with only two human touchpoints.

## Flow Selection

根據 input 自動選擇流程，也可由用戶明確指定：

| 觸發條件 | 流程 |
|----------|------|
| 用戶提供 Figma URL 且**只有** UI 實作需求（無後端） | **Figma-Lite Flow** |
| 用戶明確說「figma-lite」或「直接從 Figma 做」 | **Figma-Lite Flow** |
| 用戶提供 Figma URL + 後端/API 等混合需求 | Standard Flow + Figma-Direct mode |
| Bug report | Bug Flow |
| 其他 | Standard Flow |

**如果不確定走哪條路，問用戶。**

## MANDATORY RULE — BUILD PHASE

**In Phase 4 (Build), you MUST call the `Agent()` tool to dispatch subagents for EVERY task. No exceptions. No matter how small.**

You are the controller. You NEVER write implementation code yourself.

**Model rule: Implementer 一律 opus，reviewer/tester 一律 sonnet。**

For each task, you call in sequence:
```
// IMPLEMENTER — always opus (omit model)
Agent({description: "Implement T[N]: ...", prompt: "..."})
// Figma UI task: 加上 figma:figma-implement-design skill + nodeId

// SPEC REVIEWER — sonnet
Agent({model: "sonnet", description: "Spec review T[N]: ...", prompt: "..."})

// VISUAL REVIEWER (UI tasks with Figma) — sonnet
Agent({model: "sonnet", description: "Visual review T[N]: ...", prompt: "..."})

// BROWSER TESTER (UI tasks) — sonnet
Agent({model: "sonnet", description: "Browser test T[N]: ...", prompt: "..."})

// CODE QUALITY REVIEWER — sonnet
Agent({model: "sonnet", description: "Code quality review T[N]: ...", prompt: "..."})
```

If you find yourself using Edit, Write, or Bash to create/modify project code — STOP. You are violating this rule. Dispatch a subagent instead.

---

## Pipeline Phases

0. **Assess** the input — classify type, detect Figma URLs, inventory reference sources
   - **Refined Figma design** (有清晰區塊劃分 + annotations) → **Figma-Lite Flow**（跳過 Spec/Plan，見下方）
   - Figma + 額外需求（後端、API 等混合） → 標準流程 with Figma-Direct mode
   - No Figma → standard flow
1. **Spec** — generate specification, then **present to human for approval** (touchpoint 1)
   - **Standard:** full text spec (objective, approach, boundaries, testing)
   - **Figma-Direct:** Figma-Mapped Spec — nodeId 對照表 + 行為規格 + 技術決策 + annotations 清單。**不寫視覺描述**，視覺留在 Figma 讓 implementer 自己讀
2. **Plan** — break into vertical tasks with acceptance criteria
   - **Figma-Direct:** 每個 UI task 帶 nodeId + fileKey，描述只寫行為/互動，不寫視覺細節
3. **Worktree** — create isolated git worktree + verify test baseline. Dispatch subagent: `Agent({model: "haiku", ...})`
4. **Build** — for EACH task, dispatch subagents via `Agent()` tool:
   - `Agent()` → IMPLEMENTER
     - Non-UI / no Figma: sonnet, TDD — failing test → implement → pass → commit
     - **Figma UI task: opus + `figma:figma-implement-design` skill** — 專門的 skill 處理 Figma 讀取和設計解讀
   - `Agent()` → SPEC REVIEWER (adversarial — reads code, distrusts implementer)
   - `Agent()` → VISUAL REVIEWER (optional — UI tasks with Figma nodeId)
     - **Figma-Direct: reviewer 自己呼叫 `get_design_context(nodeId)` + `get_screenshot(nodeId)` 比對**
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

Figma-Direct 核心規則：
- **UI task 用 opus + `figma:figma-implement-design` skill** — 專門的 skill 處理 Figma 讀取和視覺還原
- **非 UI task 用 sonnet 標準流程** — 後端、config、infra 等不需要 Figma skill
- 不要在 controller 層轉述視覺細節 — skill 會自己讀 Figma
- MCP 是 session 層級共用，subagent 呼叫 Figma MCP 不會重新開連線

For refined Figma designs: use **Figma-Lite Flow** — skip spec/plan, Figma IS the spec:
1. **Scan** — `get_metadata` + `get_design_context` → 產出 block 清單（nodeId + name + annotations）→ 丟棄原始回傳保持 context 乾淨
2. **Confirm blocks** — 向用戶確認 block 劃分 + 技術決策 (touchpoint 1)
3. **Worktree** — 建立隔離環境
4. **Build per block** — 每個 block:
   - `Agent()` → IMPLEMENTER (opus + `figma:figma-implement-design` skill) — 包含跑測試
   - `Agent()` → VISUAL REVIEWER — 比對 Figma vs 建置結果（get_design_context + Chrome 截圖）
   - `Agent()` → BROWSER TESTER — smoke test（console, RWD, network, a11y）
   - 完成後只保留一行摘要，丟棄 subagent 完整報告
5. **Report** — merge 決定 (touchpoint 2)

For bug reports: skip spec/plan, create worktree, go straight to reproduce → localize → fix → verify → commit → report.
