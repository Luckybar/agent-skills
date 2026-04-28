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

**Mode flags (偵測於 Phase 0，影響 Phase 1–5 的細部行為)：**
- **Figma-Direct mode** — 輸入含 Figma URL / 設計稿引用時開啟，UI task 交給 figma skill 自己讀設計。
- **Backend mode** — 輸入針對 API / schema / migration / service / integration 時開啟，使用 API Contract / Data Layer / Security / Integration Test reviewer 套組取代 Visual / Browser tester，並在 Phase 4 結束後加跑 **Phase 4.5 Codex Red Team Review**（不同模型家族做對抗式審查）。
- 兩個 mode 互不衝突；全後端 task 就 Backend ON / Figma OFF，純前端就相反，full-stack 可兩個都 ON。

## Model Strategy

The **controller** (the agent running this pipeline) is **opus**. Phases 0, 1, 2, 5 run inline (controller handles directly). **Phase 3 and Phase 4 MUST dispatch subagents via Agent() — no exceptions, no matter how small the task.**

| Phase | Model | Execution | Why |
|-------|-------|-----------|-----|
| Phase 0 | opus | Inline (controller) | Assess input — controller handles directly |
| Phase 1 | opus | Inline (controller) | Spec generation — controller handles directly |
| Phase 2 | opus | Inline (controller) | Plan generation — controller handles directly |
| Phase 3 | sonnet | **Subagent** `Agent()` | Git worktree setup — 機械性操作用 sonnet |
| Phase 4 — IMPLEMENTER | opus | **Subagent** `Agent()` | **寫 code 用 opus** — 最需要判斷力的環節 |
| Phase 4 — REVIEWERS | sonnet | **Subagents** per role | 檢查清單式的驗證用 sonnet |
| Phase 4.5 (Backend only) | sonnet orchestrator + **Codex CLI** | **Subagent** shells out | 不同模型家族做對抗式審查，抓同家族共通盲點 |
| Phase 5 | opus | Inline (controller) | Report generation — controller handles directly |

**Model rule:** Implementer 一律用 opus（omit model），所有 reviewer/tester 一律用 sonnet（pass `model: "sonnet"`）。寫 code 是最需要能力的環節，review 是結構化檢查。

**Figma-Direct 額外規則：** UI task 的 implementer 用 opus + `figma:figma-implement-design` skill。

## Process

### Phase 0: Assess Input

Classify the input and determine the starting point:

```
Input type?
    │
    ├── Refined Figma design ─→ Skip to Figma-Lite Flow (below)
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
Figma-Direct mode: [ON (refined Figma provided) / OFF]
Reference sources: [list or "none"]
Figma sources: [list of Figma URLs or "none"]
Assumptions:
1. [assumption]
2. [assumption]
Proceeding automatically unless blocked.
```

**Figma detection:** If the input includes Figma URLs or references Figma designs, activate Figma-Direct mode:

```
Input includes Figma URL or design references?
    │
    ├── YES → Figma-Direct mode: ON
    │
    │   核心原則：不替 implementer 摘要視覺細節，讓每個 subagent 自己讀 Figma。
    │   Controller 只負責結構掃描和行為規格，視覺細節留在 Figma 裡。
    │
    │   Phase 0: 結構掃描 + annotations 提取（見下方協議）
    │            產出 nodeId ↔ 區塊 ↔ annotations 對照表
    │   Phase 1: Figma-Mapped Spec（簡化版，不寫視覺描述）
    │            annotation 含連結時（如 Figma Make prototype），跟進連結讀取內容
    │            在 Spec 中列出 Annotations 區塊，交由人在 spec approval 時決定 scope
    │   Phase 2: 每個 task 帶 nodeId，描述只寫行為/互動，不寫視覺
    │   Phase 4: Implementer 自己呼叫 get_design_context(nodeId) 讀設計
    │            Visual reviewer 用同一個 nodeId 比對
    │            mock data required for every UI task
    │
    └── NO  → Figma-Direct mode: OFF (standard flow)
```

**Backend detection:** If the input targets backend work (API, schema, migration, service, integration, backend bug), activate Backend mode:

```
Input targets backend?
    │
    ├── YES (keywords: API / endpoint / schema / migration / service /
    │        controller / repository / queue / webhook / integration,
    │        OR target repo is backend stack with no UI surface)
    │   │
    │   └── Backend mode: ON
    │
    │       核心原則：後端的正確性是 contract + data + security + integration
    │       四軸共同保證。不是看得見的東西，要靠自動化檢查和真 DB 測試。
    │
    │       Phase 0: 後端結構掃描（見下方協議）
    │       Phase 1: Backend Spec（加 API contract / schema / authZ / SLA / 相容性）
    │       Phase 2: Plan 中 migration 先於 code、contract 先於實作
    │       Phase 3: Worktree 起 DB container + 跑 migration + load seed
    │       Phase 4: Reviewer 套組改為 API Contract / Data Layer /
    │                Security / Integration Test（取代 Visual / Browser）
    │       Phase 5: Report 加 migration 部署計畫 + rollback plan
    │
    └── NO  → Backend mode: OFF (standard flow)
```

**Backend 輸入盤點（先做，優先於結構掃描）：**

Backend mode 開始時，預期會收到兩類補充輸入 — 沒提供就明確記「無」，不要自己腦補：

```
1. 流程 Figma（若有）— 通常是 UI flow / 序列圖 / API 呼叫順序圖
   用途：
   - 理解前端會呼叫哪些 endpoint、順序、條件分支
   - 從 UI 欄位反推 API response 應該回傳什麼
   - 理解錯誤情境（UI 要顯示什麼 → 後端回什麼 error code）
   處理方式：
   - 先 get_metadata + get_design_context 掃過，**只取「流程/欄位/狀態」資訊**
   - 不取視覺細節（色、間距、字型）— Backend mode 不在乎這些
   - 產出「UI ↔ API」對照表：哪個畫面 / 操作 → 打哪個 endpoint → 預期什麼回應

2. 舊版 API 補充資料（若有）— OpenAPI / Swagger / 舊 doc / 舊 code
   用途：
   - 判斷 backward compatibility：新 API 能不能取代、要保留多久
   - 判斷 deprecation path：舊 endpoint 何時下線
   - 找出舊 contract 的潛規則（undocumented behavior）
   處理方式：
   - 用 legacy-consultant pattern 讀（而不是把全部塞進 controller context）
   - 產出「舊 API ↔ 新 API」對照表：每個舊 endpoint 的去留決策
     （保留 / 取代 / deprecate with timeline / break）

3. 兩份輸入彙整成 Backend Input Table：
   ┌─────────────┬──────────────────────────────────────┐
   │ 輸入類別      │ 來源 / 內容 / 對照表                   │
   ├─────────────┼──────────────────────────────────────┤
   │ Figma flow   │ <URL> / <UI↔API 對照表 或「無」>       │
   │ Legacy API   │ <source> / <舊↔新 對照表 或「無」>     │
   └─────────────┴──────────────────────────────────────┘
```

**後端結構掃描協議：**

```
1. 讀實際 DB schema（\d / information_schema / schema 檔）— 不讀 docs
   列出相關表、欄位、現有索引
2. 盤點現有 API surface — routes / controllers / OpenAPI / GraphQL SDL
3. 盤點外部整合 — MQ、第三方 API、auth provider、webhook
4. 判斷變更類型：breaking / additive / deprecation
5. 彙整成 backend context table：
   ┌─────────┬────────────────────────────────┐
   │ 類別     │ 現況                            │
   ├─────────┼────────────────────────────────┤
   │ Schema   │ 相關表、欄位、索引               │
   │ API      │ 現有 endpoint、contract 形式     │
   │ 整合     │ 依賴的外部服務                   │
   │ 變更類型  │ breaking / additive / deprecation │
   └─────────┴────────────────────────────────┘
   此表 + Backend Input Table 為 Phase 1 Spec 的基礎。
```

**Figma 結構掃描協議：**

```
1. get_metadata(頂層) → 記錄完整 node tree，列出所有主要區塊的 nodeId + 名稱
2. get_design_context(頂層) → 掌握全貌 + 搜尋 data-development-annotations 屬性
3. 若 code 被截斷 → 對主要子區塊逐一呼叫 get_design_context（excludeScreenshot: true）
4. 彙整成對照表：
   ┌──────────┬──────────────┬─────────────────┐
   │ nodeId   │ 區塊名稱      │ annotations/備註  │
   ├──────────┼──────────────┼─────────────────┤
   │ 120:163  │ Header       │ 固定置頂, blur bg  │
   │ 120:201  │ Hero Section │ 動畫進場           │
   │ ...      │ ...          │ ...             │
   └──────────┴──────────────┴─────────────────┘
   此對照表為 Phase 1 Spec 和 Phase 2 Plan 的基礎。
```

（annotations 深度提取協議見 `figma-visual-review` Step 1a）

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

**Standard flow (Figma-Direct OFF):** Follow `spec-driven-development`. Generate a structured specification covering:
1. Objective and acceptance criteria
2. Technical approach
3. Boundaries (always do / never do)
4. Testing strategy

**Figma-Direct flow (Figma-Direct ON):** 產出 Figma-Mapped Spec — 不寫視覺描述，視覺細節留在 Figma 裡讓 implementer 自己讀。

Figma-Mapped Spec 包含：

1. **Figma 來源** — fileKey, 頂層 nodeId, Figma URL
2. **區塊對照表** — 從 Phase 0 掃描結果帶入
   ```
   ┌──────────┬──────────────┬─────────────────┬──────────────┐
   │ nodeId   │ 區塊名稱      │ annotations     │ 行為/互動規格  │
   ├──────────┼──────────────┼─────────────────┼──────────────┤
   │ 120:163  │ Header       │ 固定置頂, blur   │ scroll 時縮小 │
   │ 120:201  │ Hero Section │ 動畫進場         │ fade-in 0.3s │
   │ ...      │ ...          │ ...             │ ...          │
   └──────────┴──────────────┴─────────────────┴──────────────┘
   ```
3. **技術決策** — stack, responsive strategy, state management
4. **全局規格** — RWD 斷點、共用 token、動畫策略
5. **Annotations 清單** — 所有從 Figma 提取的 annotations（含連結跟進結果）
6. **Boundaries** — always do / never do
7. **測試策略** — 驗證方式

**Backend mode flow (Backend mode ON):** 產出 Backend Spec — 標準 spec 加上後端特有欄位。不要把後端 spec 省成「呼叫 X 打 Y」這種一句話，後端的 bug 大多來自 contract 模糊和相容性沒想清楚。

Backend Spec 除標準 4 項外加入：

0. **Input references**（直接引用 Phase 0 的 Backend Input Table）
   - Figma flow：UI↔API 對照表放這，response shape 要對得上 UI 欄位
   - Legacy API：舊↔新對照表放這，每個舊 endpoint 的去留決策明寫
1. **API contract** — 逐 endpoint 描述
   - path, method, request schema（型別、nullability、validation 規則）
   - response schema per status code（**欄位來源要標註：是從 Figma UI 反推、還是新增、還是延續舊 API**）
   - 錯誤格式（專案既有 convention：例如 `{error: {code, message, details?}}`）
2. **Data model changes**
   - 新表 / 新欄位 / 新索引
   - Migration 策略：expand → backfill → contract 分階段
   - 是否 destructive、是否可能鎖表、預估時間
3. **Error taxonomy** — 錯誤類別 × HTTP code × client 應有行為
   （含 Figma 流程中 UI 會顯示的所有錯誤情境）
4. **Authorization matrix** — role × resource × action 矩陣，含租戶隔離 / IDOR 點位
5. **SLA** — p95 latency、throughput、timeout（若無明確要求寫「無 SLA」）
6. **Backward compatibility** — breaking / additive / deprecation timeline
   （對應到 legacy API 對照表，每個舊 endpoint 有明確退場計畫）
7. **Observability** — log 欄位、metric、trace span、告警條件

Save as `SPEC.md`.

**Gate 1 — Spec self-check:**
- [ ] Objective is specific and measurable
- [ ] Acceptance criteria are testable
- [ ] Technical approach is stated
- [ ] Boundaries are defined
- [ ] (Figma-Direct) 區塊對照表完整，每個區塊有 nodeId
- [ ] (Figma-Direct) Annotations 已全部列出，含連結跟進結果
- [ ] (Backend) API contract 逐 endpoint 寫出 request/response schema + 錯誤格式
- [ ] (Backend) Migration 策略標明 destructive 與否、是否分階段
- [ ] (Backend) Authorization matrix 覆蓋所有新 endpoint
- [ ] (Backend) Backward compatibility 已標示（break/additive/deprecation）
- [ ] (Backend) Backend Input Table 已填：Figma flow 與 Legacy API 狀態（有就列出對照表，沒有明寫「無」）
- [ ] (Backend) 若有 Figma：response shape 欄位可對應 UI 欄位
- [ ] (Backend) 若有 Legacy API：每個舊 endpoint 有去留決策（保留/取代/deprecate/break）

→ Self-check passes: **present to human for approval.** This is the first and primary human touchpoint. Wait for explicit approval before proceeding.

→ Human approves: proceed to Phase 2.
→ Human requests changes: update spec, re-present.

### Phase 2: Plan (inline — opus)

Follow `planning-and-task-breakdown`. From the approved spec:
1. Identify dependency graph
2. Slice into vertical tasks (each independently testable)
3. Write acceptance criteria per task
4. Order by dependencies

**Figma-Direct 額外規則：** 每個 UI task 必須帶 Figma nodeId。task 描述只寫行為/互動規格，**不寫視覺細節**（色值、spacing、字型等）— implementer 會自己從 Figma 讀取。

```
## Task format (Figma-Direct mode)

T[N]: <task name>
  nodeId: 120:245
  fileKey: B61Y1Z4zeaook33d8phYrw
  行為規格: [互動、狀態變化、RWD 行為]
  annotations: [relevant annotations from spec]
  acceptance criteria:
    - [ ] ...
  depends on: T[N-1]
```

Save plan to `tasks/plan.md` and task list to `tasks/todo.md`.

**Backend mode 額外規則：**
- Migration task **獨立** 且排在對應 code task **之前**（expand 階段先上）
- Contract task（定義 API shape / OpenAPI / DTO 型別）先於 handler 實作 task
- 共用 domain type / validation schema 先建立
- Seed / fixture 更新跟 migration task 綁在一起，不獨立
- Breaking change 拆成 additive → migrate callers → remove old 三階段

**納入輸入參考物的 plan 規則（Backend mode ON）：**
- **Figma flow → task 排序**：按 UI flow 順序排 endpoint task，前端需要先用到的 endpoint 優先
  - 每個 endpoint task 在 `touches` 欄位註明對應的 Figma node / 畫面名稱
  - response shape 的 acceptance criteria 要對照 Figma UI 欄位（「回傳欄位足以渲染這個畫面」）
- **Legacy API → 拆成明確的 migration tasks**：不要把「相容舊 API」藏在實作 task 裡
  - 每個舊 endpoint 的去留產生獨立 task：
    - 保留 → 無 task（只在 spec 記錄）
    - 取代 → T: 實作新 endpoint + T: 切流量 + T: 下線舊 endpoint（三個 task）
    - Deprecate → T: 新 endpoint 上線 + T: 加 Deprecation header + T: 文件更新（退場時間）
    - Break → 改走 Spec 的 additive→migrate→remove 三階段
  - 若舊 API 有 undocumented behavior（從 legacy consultant 挖出來的），在對應 task 明寫「需保留此行為：...」

```
## Task format (Backend mode)

T[N]: <task name>
  kind: migration | contract | handler | integration | test
  touches: [table names / endpoint paths / module paths]
  acceptance criteria:
    - [ ] ...
  depends on: T[N-1]
```

**Gate 2 — Plan quality check (autonomous):**
- [ ] Every task has acceptance criteria
- [ ] Every task is independently verifiable
- [ ] No circular dependencies
- [ ] Reasonable scope (< 20 tasks)
- [ ] (Figma-Direct) Every UI task has a Figma nodeId
- [ ] (Figma-Direct) Task descriptions contain no visual details (colors, spacing, fonts)
- [ ] (Backend) Migration task 排在對應 code task 之前
- [ ] (Backend) Contract task 排在 handler 實作之前
- [ ] (Backend) Breaking change 已拆成 additive / migrate / remove 三階段
- [ ] (Backend) 若有 Figma：task 順序呼應 UI flow，每個 endpoint task 標註對應畫面
- [ ] (Backend) 若有 Legacy API：每個「取代/deprecate/break」的舊 endpoint 有獨立 migration task

→ All pass: proceed to Phase 3 automatically.
→ Scope too large: split into milestones, build the first one.

### Phase 3: Worktree Setup (subagent — haiku)

Dispatch a subagent:

```
Agent({
  description: "Setup worktree",
  prompt: "Create worktree for <feature-name>. Follow using-git-worktrees: ..."
})
```

The subagent follows `using-git-worktrees`:
1. Creates worktree in `.worktrees/<feature-name>`
2. Creates feature branch
3. Runs project setup (npm ci, cargo build, etc.)
4. Verifies test baseline passes

**Backend mode 額外步驟：**
5. 起 DB container（docker compose / testcontainers / 專案約定的方式）
6. 跑 migration 到 HEAD
7. Load seed data / fixtures
8. 跑一次完整 integration test 確認 baseline 綠（打真 DB，不 mock）

**Gate 3 — Worktree ready check (autonomous):**
- [ ] Worktree created and git-ignored
- [ ] Setup completed without errors
- [ ] All tests pass (baseline verified)
- [ ] (Backend) DB container 啟動成功
- [ ] (Backend) Migration 跑到 HEAD 無錯誤
- [ ] (Backend) Integration test baseline 綠

→ All pass: proceed to Phase 4 automatically.
→ Baseline tests fail: report to human (pre-existing issue, not ours).

### Phase 4: Build Loop (subagent-driven — mixed models)

**You MUST NOT implement tasks yourself. You MUST use the Agent() tool to dispatch subagents for every task — no matter how small.** Even a one-line config change gets dispatched. The controller orchestrates — it NEVER writes implementation code. If you catch yourself writing code instead of an Agent() call, STOP and dispatch a subagent instead.

**Context management:** After each task's full review cycle completes, retain only a one-line summary (e.g., "T1: DONE, 1 spec-review cycle, 0 visual-review rounds"). Discard full subagent reports — they served their purpose during the review cycle. This prevents controller context from ballooning across tasks.

For each task in dependency order, make these Agent() calls in sequence:

#### 4a. IMPLEMENTER — writes code + tests + commits

**Standard flow (Figma-Direct OFF):**

```
Agent({
  // omit model — inherits opus. 所有 implementer 一律用 opus。
  description: "Implement T[N]: <task name>",
  prompt: "You are an implementer. Your job is to write code and tests.

TASK:
<paste full task text + acceptance criteria here>

CONTEXT:
<where this fits in the project, relevant file paths>

RULES:
- Follow TDD: write a failing test FIRST, then implement to make it pass
- Commit after each meaningful change
- Self-review before reporting DONE: check naming, dead code, error handling, test quality

TEST COMMAND: <npm test / cargo test / etc.>

Report status: DONE / DONE_WITH_CONCERNS / BLOCKED"
})
```

**Backend mode flow (Backend mode ON):**

Implementer 規則：
- Migration 先寫先 commit，code 實作後一併確認可以 up/down
- Test 必須包含 **integration test（打真 DB，不 mock）**
- 若改動涉及 authZ，test 必須涵蓋未授權使用者被擋的 case
- OpenAPI / SDL 跟 handler 同步更新，不允許 drift

```
Agent({
  // omit model — inherits opus
  description: "Implement T[N]: <task name>",
  prompt: "You are a backend implementer.

TASK:
<paste task + acceptance criteria + API contract + migration strategy from spec>

CONTEXT:
<project stack, repo layout, relevant existing files, DB connection>

RULES:
- Follow TDD: write a failing integration test FIRST（打真 DB，不 mock）
- Migration 先寫，跑 up 驗證 schema 對
- Implement handler / service / repository 讓 test 過
- 跑完整 integration test 確認 regression 無
- OpenAPI / SDL 跟實作同步更新
- Commit after each meaningful change（migration、contract、implementation 分開 commit）
- Self-review: naming, dead code, error handling, authZ 覆蓋完整

TEST COMMAND: <integration test command>

Report: DONE / DONE_WITH_CONCERNS / BLOCKED"
})
```

**Figma-Direct flow (Figma-Direct ON — UI task with nodeId):**

UI task 使用 `figma:figma-implement-design` skill，由該 skill 處理 Figma 讀取、設計解讀、code 產出。
非 UI task（後端、config、infra 等）即使在 Figma-Direct mode 下仍使用 Standard flow。

```
Agent({
  // omit model — inherits opus, UI 實作需要較強的設計解讀能力
  description: "Implement T[N]: <task name>",
  prompt: "You are an implementer. Your job is to implement UI from Figma design.

TASK:
<paste task text + acceptance criteria — 行為/互動規格>

FIGMA SOURCE:
  URL: https://www.figma.com/design/<fileKey>/...?node-id=<nodeId>
  fileKey: <fileKey>
  nodeId: <nodeId>

使用 figma:figma-implement-design skill 來實作這個 Figma 設計。
該 skill 會處理：讀取設計 context、解讀視覺規格、產出符合專案 stack 的 code。

CONTEXT:
<where this fits in the project, relevant file paths, project stack>
<annotations from spec relevant to this task>

RULES:
- 先 invoke figma:figma-implement-design skill，再根據其產出調整
- Commit after each meaningful change
- Include mock data for all UI elements（visual review 需要非空頁面）
- Self-review before reporting DONE: check naming, dead code, error handling, test quality

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

If NEEDS_FIXES → re-dispatch implementer with fix list → re-run spec review. Max 2 cycles.

#### 4c. VISUAL REVIEWER (optional — only for UI tasks with Figma nodeId)

```
Agent({
  model: "sonnet",
  description: "Visual review T[N]: <task name>",
  prompt: "You are a visual reviewer. Compare Figma design vs built UI.

FIGMA SOURCE — 你必須自己讀設計來比對：
  fileKey: <fileKey>
  nodeId: <nodeId>
  呼叫 get_design_context({ fileKey, nodeId }) 取得設計截圖和 code。
  呼叫 get_screenshot({ fileKey, nodeId }) 取得 Figma 渲染圖。

BUILT UI:
  URL: <page URL>
  用 Chrome MCP 截圖取得實際畫面。

COMPARISON:
  逐項比對：layout, spacing, colors, typography, border-radius, shadows, icons, alignment
  每個差異寫成文字描述（截圖觀察完立即丟棄，只保留文字報告）

Report: APPROVED / NEEDS_FIXES (with specific diff list) / MAJOR_ISSUES"
})
```

#### 4d. BROWSER TESTER (all UI tasks)

```
Agent({
  model: "sonnet",
  description: "Browser test T[N]: <task name>",
  prompt: "You are a browser tester. Run smoke tests on the built page.
<page URL, test checklist: console errors, RWD breakpoints, network, interaction, a11y>"
})
```

#### 4c-backend. API CONTRACT REVIEWER (Backend mode ON — replaces Visual reviewer)

```
Agent({
  model: "sonnet",
  description: "API contract review T[N]: <task name>",
  prompt: "You are an API contract reviewer. You DISTRUST the implementer's self-report.

TASK + SPEC CONTRACT:
<paste API contract section from spec>

FILES CHANGED:
<controller / handler / OpenAPI / SDL / DTO files>

Check (read ACTUAL code, not summaries):
- Response shape matches spec per status code (field names, types, nullability)
- HTTP status codes correct (200 vs 201 vs 204, 404 vs 403, 422 vs 400, 409 for conflict)
- Error format consistent with project convention
- OpenAPI / SDL updated and in sync with handler
- Backward compatibility: no field removed / renamed / type-changed without deprecation
- Request validation matches spec (required vs optional, format constraints)

Report: APPROVED / NEEDS_FIXES (specific list) / MAJOR_ISSUES"
})
```

#### 4d-backend. DATA LAYER REVIEWER (Backend mode ON — replaces Browser tester)

```
Agent({
  model: "sonnet",
  description: "Data layer review T[N]: <task name>",
  prompt: "You are a data layer reviewer.

TASK + MIGRATION STRATEGY:
<paste migration strategy from spec>

FILES CHANGED:
<migration files, repository / query files>

Check (read ACTUAL code):
- Migration safety:
  - Destructive op (DROP / rename / type change) 有保護 / 分階段 / backfill 計劃
  - 加 NOT NULL 欄位：有預設值 或 分階段（add nullable → backfill → 加 NOT NULL）
  - 大表 DDL 不會長時間鎖表（若鎖表，是否在 spec 標註維護窗口）
  - Migration 能 up 能 down（除非 forward-only 已明示）
- Query quality:
  - 新 query 查得到 index（對齊現有 index 或 migration 有加新 index）
  - N+1：for loop 內打 DB / ORM lazy load 連打
  - Transaction 邊界：該包的有包（多表寫入）、不該包的沒包太大（外部 API call 不應在 tx 內）
- 無 unguarded destructive operation

Report: APPROVED / NEEDS_FIXES / MAJOR_ISSUES"
})
```

#### 4e-backend. SECURITY REVIEWER (Backend mode ON)

```
Agent({
  model: "sonnet",
  description: "Security review T[N]: <task name>",
  prompt: "You are a security reviewer. Reference: security-and-hardening skill.

TASK + AUTHZ MATRIX:
<paste authorization matrix from spec>

FILES CHANGED:
<controller / service / middleware / auth files>

Check:
- Input validation on all untrusted fields (type, length, format)
- AuthZ checks present on every endpoint (IDOR: user only accesses own data)
- 租戶隔離：query 有帶 tenant_id filter，不依賴 app 層過濾
- SQL injection：所有 query 使用 parameterized / prepared statement
- Secrets：無 hard-coded key、無 secret 出現在 log
- Rate limiting：auth / expensive / write endpoint 有限流
- PII：log 無 PII、sensitive 欄位 encrypted at rest（若 spec 要求）

Report: APPROVED / NEEDS_FIXES / MAJOR_ISSUES"
})
```

#### 4f-backend. INTEGRATION TEST RUNNER (Backend mode ON)

```
Agent({
  model: "sonnet",
  description: "Integration test T[N]: <task name>",
  prompt: "You are an integration test runner.

TEST COMMAND: <e.g., npm run test:integration / ./gradlew integrationTest>

Run the integration test suite against the real DB container from Phase 3.
**Do NOT mock the DB**（per team convention — mock DB 已踩過雷）.

Verify:
- 新增 test 覆蓋本 task 所有 acceptance criteria
- 現有 endpoint 的 regression test 全綠（執行全量，不只跑新 test）
- Migration 可 up / 可 down（若非 forward-only）
- DB fixture 未被污染（每個 test 隔離）

Report: PASS / FAIL (with failing test output + root cause guess)"
})
```

**Backend mode review order：** 4a implementer → 4b spec review → 4c-backend contract → 4d-backend data layer → 4e-backend security → 4f-backend integration test → 4g-backend code quality。任一環節 NEEDS_FIXES → 回 implementer，同一環節 max 2 輪。

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

If NEEDS_FIXES → re-dispatch implementer → re-run quality review. Max 1 cycle.

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
- [ ] (Backend) 所有 endpoint 通過 API contract review
- [ ] (Backend) 所有 migration 通過 data layer review
- [ ] (Backend) 所有 endpoint 通過 security review
- [ ] (Backend) Full integration test suite 綠（真 DB）

**Backend mode 下 Gate 4 通過後 → 進 Phase 4.5（Codex Red Team），通過後才進 Phase 5。**

→ All pass: proceed to Phase 5 automatically.
→ Stuck after 2 fix cycles on same issue: STOP and ask human.

### Phase 4.5: Codex Red Team Review (Backend mode only)

Backend mode Gate 4 通過後、進 Phase 5 前，先由 **Codex（不同模型家族）做一次對抗式審查**。同家族 reviewer (opus/sonnet) 有共同 training bias，換個模型能抓到本家族看不見的盲點 — 尤其後端「看不見的正確性」（race condition / migration safety / authZ 死角）。

**前提：** 本機已裝 `codex` CLI（`/usr/local/bin/codex`），使用 `codex exec` 非互動模式執行一次完整 audit。模型走 Codex `config.toml` 預設。

**Dispatch（orchestrator 用 sonnet，它只負責組 context + shell out + 解析回應）：**

```
Agent({
  model: "sonnet",
  description: "Codex red team review",
  prompt: "You are orchestrating a Codex red team review for a backend PR.

STEP 1 — Assemble context:
  - Read SPEC.md (backend spec)
  - Run `git diff <base>..HEAD` to get full branch diff
  - List all new/modified files, tag each as: migration / handler / service / repository / test / infra
  - Extract API contract section, migration strategy, authZ matrix from SPEC.md

STEP 2 — Invoke Codex (non-interactive):
  Run: codex exec '<ASSEMBLED_PROMPT_BELOW>'
  The assembled prompt must include:
    (a) SPEC sections above
    (b) full diff
    (c) the fixed question set below verbatim
    (d) the required output format

STEP 3 — Parse Codex output into severity buckets (Critical / High / Medium / Low).

STEP 4 — Report back to controller.

=== FIXED QUESTION SET (送給 Codex 的內容) ===

You are a senior backend engineer doing a hostile code review. For each question,
if the issue exists point to file:line and explain impact; if it doesn't exist say 'None'.
Do not be polite — your job is to find problems.

[Concurrency]
Q1. 這個 PR 中，哪兩個 endpoint 並發執行最有可能觸發 race condition？列出觸發條件和後果。
Q2. 資料存取有沒有遺漏 locking（optimistic/pessimistic）？哪裡會出現 lost update？

[Migration safety]
Q3. 這批 migration 在 production（假設 10M+ row 的大表）執行會遇到什麼問題？鎖表時間？
Q4. Migration 能否安全回滾？若不能，是什麼阻擋？
Q5. 有沒有 destructive operation（DROP / rename / type change）沒分階段保護？

[AuthZ / 多租戶]
Q6. 列出 3 個最可能的 IDOR 點（user X 能讀/寫 user Y 的資料）。
Q7. 租戶隔離有沒有漏 — 每個 query 都有 tenant_id filter 嗎？依賴 app 層過濾的點在哪？

[Error handling & failure modes]
Q8. 外部依賴（DB / MQ / 第三方 API）掛掉或 timeout 時，哪個 endpoint 會出現最糟狀況
    （資料不一致 / 無限等待 / 靜默失敗 / 資源洩漏）？
Q9. 哪個錯誤情境在 UI 上沒被正確呈現（對照 spec 的 error taxonomy）？

[Observability]
Q10. 若這段 code 在 production 出問題，目前的 log / metric / trace 足夠定位嗎？缺什麼？

[Backward compatibility]
Q11. 若有舊 client 正在用舊 API，這批改動會不會讓舊 client 壞掉？哪些欄位或行為是關鍵？
Q12. 有沒有 undocumented behavior（從舊 code 觀察到的潛規則）沒被保留？

=== OUTPUT FORMAT (Codex must follow) ===

對每題輸出一個區塊：

Qn [Severity: Critical|High|Medium|Low|None]
  Location: <file:line 或 N/A>
  Issue: <一句描述問題>
  Impact: <後果>
  Fix: <具體修法建議，一句>

最後附：
SUMMARY OF BLOCKERS (Critical + High only):
  - <bullet list>

=== SEVERITY 定義（給 Codex 對齊） ===

- Critical: production 會翻車 / 資料遺失 / 安全破口
- High: 特定條件下會壞、修起來不難
- Medium: 品質問題，不會壞但值得修
- Low: 風格 / 偏好，可略

STEP 4 output to controller:
{
  critical_high: [ {q, location, issue, impact, fix}, ... ],
  medium_low: [ ... ],
  codex_exit_code: <0|非0>,
  codex_raw_output_path: <存檔路徑，供除錯>
}"
})
```

**處理結果：**

- **Critical / High findings** → 自動 dispatch implementer (opus) 修 → 再跑一輪 Codex red team。**Max 2 輪**。第 2 輪仍有 Critical/High → STOP 交給 human 仲裁。
- **Medium / Low findings** → 彙整到 Phase 5 report，由 human 在 merge decision 時判斷。
- **None** → 不處理。

**Gate 4.5 — Red team pass check (autonomous):**
- [ ] Codex exec 成功回傳（exit code 0）
- [ ] 所有 Critical findings 已修（或 human 仲裁通過）
- [ ] 所有 High findings 已修（或 human 仲裁通過）
- [ ] Medium / Low findings 已整理進 report 草稿

→ 全通過：進 Phase 5。
→ 2 輪後仍有 Critical/High：STOP，把 findings 呈給 human。
→ `codex exec` 失敗（exit 非 0）：報告失敗原因，STOP 問 human（不要靜默跳過這個 gate）。

**Common Rationalizations（Phase 4.5 專屬）：**

| Rationalization | Reality |
|---|---|
| "我本家族 reviewer 都過了，Codex 應該沒問題，跳過" | 跳過就失去這個階段的意義。同家族 training bias 是共通盲點，一定要跑。 |
| "Codex 挑的都是風格問題，直接合" | 若真的都是 Low/Medium，會在 report 呈現，不擋合。但 Critical/High 不可略。 |
| "第 2 輪還有 Critical/High，我幫 Codex 判斷那其實不嚴重" | NO。Severity 定義給 Codex 對齊過，若 Codex 判 Critical 你不同意，那是 human 仲裁的事，不是 controller 私下覆寫。 |

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
- Codex red team rounds: [N] (Backend mode only)
- Issues auto-fixed: [N]

Non-blocking suggestions:
- [list from quality reviewers]

Ready for your decision:
1. Merge to [base-branch]
2. Create Pull Request
3. Keep branch for further work
4. Discard
```

**Backend mode report 額外區塊（Backend mode ON 時必附）：**

```
=== Codex Red Team Review (Phase 4.5) ===

Rounds executed: <N>
Critical findings: <N> (auto-fixed: <N>, human-arbitrated: <N>)
High findings:     <N> (auto-fixed: <N>, human-arbitrated: <N>)
Medium findings:   <N> （未修，供 human 決定）
Low findings:      <N> （未修，供 human 決定）

Medium / Low findings（human 決定要不要修後再合）:
  - [Q3/Medium] <file:line> <issue> → 建議: <fix>
  - [Q8/Low]   <file:line> <issue> → 建議: <fix>
  - ...

Codex raw output: <存檔路徑>

=== Backend deployment context ===

Migration deployment plan:
  Step 1 (expand): <additive migration — 加欄位 nullable / 加表 / 加索引>
  Step 2 (backfill): <資料補值，可分批，不鎖表>
  Step 3 (contract): <加 NOT NULL / 刪舊欄位，需舊 code 下線後>
  預估影響：<鎖表時間 / 執行時長>

API changes summary:
  Breaking:   <列 endpoint + 影響範圍，若無寫「無」>
  Additive:   <新 endpoint / 新欄位>
  Deprecated: <路徑 + 退場時間>

Rollback plan:
  Code:      <git revert / 部署前一個 build>
  Migration: <能 down：執行 down；forward-only：補償 script 或接受風險>

Observability:
  新增 log 欄位：<list>
  新增 metric：  <list>
  建議告警：    <list，或已配置>

需 human 在 merge 前確認：
  - [ ] Breaking change 已通知下游 / API consumer
  - [ ] Migration 計畫已跟 DBA / ops 對過
  - [ ] Rollback 路徑清楚
```

**This is the second human touchpoint.** Wait for the human to decide.

## Figma-Lite Flow

For refined Figma designs (design already organized, annotated, ready for implementation), use a compressed pipeline that **skips Spec and Plan** — the Figma design IS the spec.

**判斷標準：** 用戶提供了 Figma URL，且設計已經精煉過（有清晰的區塊劃分、annotations、命名）。如果 Figma 只是草稿或 wireframe，走標準流程。

```
Figma URL (refined design)
    │
    ▼
┌───────────┐    ┌───────────┐  HUMAN   ┌───────────┐    ┌──────────────┐    ┌─────────┐  HUMAN
│  SCAN &   │───▶│  CONFIRM  │─APPROVE─▶│ WORKTREE  │───▶│  BUILD LOOP  │───▶│ REPORT  │─MERGE─▶ Done
│  PLAN     │    │  BLOCKS   │          │  SETUP    │    │ (per block)  │    │         │  DECIDE
└───────────┘    └───────────┘          └───────────┘    └──────────────┘    └─────────┘
  Step 1           Step 2                 Step 3            Step 4             Step 5
```

**Human touchpoints: 2** — block 確認 (Step 2) + merge 決定 (Step 5)

### Step 1: Scan & Plan（controller — opus）

直接從 Figma 掃描結構，不寫 Spec。

```
1. get_metadata(頂層 nodeId) → 記錄完整 node tree
2. get_design_context(頂層) → 掌握全貌 + 提取 annotations
3. 若 code 被截斷 → 對主要子區塊逐一呼叫（excludeScreenshot: true）
4. 彙整成 block 清單：
   ┌──────┬──────────┬──────────────┬─────────────────┬──────────┐
   │ 順序  │ nodeId   │ 區塊名稱      │ annotations     │ 依賴     │
   ├──────┼──────────┼──────────────┼─────────────────┼──────────┤
   │ 1    │ 120:163  │ Header       │ 固定置頂, blur   │ 無       │
   │ 2    │ 120:201  │ Hero Section │ 動畫進場         │ 無       │
   │ 3    │ 120:245  │ Card Grid    │ RWD 3→2→1      │ 無       │
   │ ...  │ ...      │ ...          │ ...             │ ...      │
   └──────┴──────────┴──────────────┴─────────────────┴──────────┘
5. 記錄全局技術決策：stack, RWD 斷點, 共用 token
```

**重要：掃描完畢後，丟棄 get_metadata/get_design_context 的原始回傳，只保留上面的 block 清單和技術決策。** 為 Step 4 保留乾淨的 context。

### Step 2: Confirm Blocks（人工確認）

向用戶展示 block 清單 + 技術決策，確認：
- block 劃分是否合理
- 有沒有遺漏的區塊
- 技術決策是否正確
- 哪些 block 優先

**這是第一個 human touchpoint。** 等待明確確認後繼續。

### Step 3: Worktree Setup

同標準流程 Phase 3。建立 worktree + 驗證 test baseline。

### Step 4: Build Loop（per block — 關鍵步驟）

**每個 block 執行以下流程。Controller 不寫 code — 全部 dispatch subagent。**

#### 4a. IMPLEMENTER（opus + figma:figma-implement-design）

```
Agent({
  // omit model — inherits opus
  description: "Implement block: <block name>",
  prompt: "You are an implementer. Implement this UI block from Figma design.

BLOCK:
  名稱: <block name>
  Figma URL: https://www.figma.com/design/<fileKey>/...?node-id=<nodeId>
  fileKey: <fileKey>
  nodeId: <nodeId>
  annotations: <annotations from scan>

全局技術決策:
  <stack, RWD breakpoints, shared tokens>

使用 figma:figma-implement-design skill 來實作。

CONTEXT:
  <project file paths, existing components, conventions>
  <previously implemented blocks and their file locations>

RULES:
- Invoke figma:figma-implement-design skill 讀取設計並實作
- Include mock data（visual review 需要非空頁面）
- 跑測試確認不破壞現有功能：<test command>
- Commit after implementation
- Self-review: naming, dead code, responsive behavior

Report: DONE / DONE_WITH_CONCERNS / BLOCKED"
})
```

#### 4b. VISUAL REVIEWER（比對 Figma vs 建置結果）

```
Agent({
  model: "sonnet",
  description: "Visual review block: <block name>",
  prompt: "You are a visual reviewer.

FIGMA SOURCE:
  fileKey: <fileKey>
  nodeId: <nodeId>
  呼叫 get_design_context + get_screenshot 取得設計。

BUILT UI:
  URL: <dev server URL>
  用 Chrome MCP 截圖。

比對：layout, spacing, colors, typography, border-radius, shadows, icons, alignment
截圖觀察完寫成文字差異清單，立即丟棄截圖。

Report: APPROVED / NEEDS_FIXES (specific diff list)"
})
```

如果 NEEDS_FIXES → 重新 dispatch implementer 修正 → 重新 visual review。最多 2 輪。

#### 4c. BROWSER TESTER（smoke test）

```
Agent({
  model: "sonnet",
  description: "Browser test block: <block name>",
  prompt: "Run smoke tests on <dev URL>.
  Check: console errors, RWD (desktop/tablet/mobile), network errors, interaction, a11y.
  Report: PASS / FAIL (with details)"
})
```

#### 4d. Mark block complete → next block

每個 block 完成後，controller 只保留一行摘要：
`Block 1 (Header): DONE, 1 visual review round, browser test PASS`

全部 block 完成後：
- 跑 full test suite
- 跑 build
- 確認所有 block 整合正常

### Step 5: Report + Merge Decision

```
FIGMA-LITE PIPELINE — COMPLETE

Figma source: <Figma URL>
Branch: <feature-branch>
Blocks completed: [N] / [N]
Commits: [N]
Tests: [N] passing
Build: clean

Block summary:
- Header: DONE, 0 visual review rounds
- Hero Section: DONE, 1 visual review round (spacing fix)
- Card Grid: DONE, 0 visual review rounds
- ...

Ready for your decision:
1. Merge to [base-branch]
2. Create Pull Request
3. Keep branch for further work
4. Discard
```

**這是第二個 human touchpoint。**

---

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
| "I'll quietly skip this annotation" | Annotations must always be surfaced in the Spec's Annotations section — never silently omitted. The human decides at spec approval which ones are in scope. If an annotation has a link, follow it so the spec has full context. |
| "I'll summarize the Figma visual details in the task description for the implementer" | NO. In Figma-Direct mode, UI task 交給 `figma:figma-implement-design` skill 處理。不要在 task 描述中轉述視覺細節，那個 skill 會自己讀 Figma。 |
| "I'll use sonnet for this implementation task, it's simple enough" | NO. 所有 implementer 一律用 opus（omit model）。寫 code 是最需要判斷力的環節。Sonnet 用在 reviewer/tester — 結構化檢查不需要 opus。 |
| "I'll skip the spec review, the implementer's tests pass" | Tests verify behavior. Spec review verifies completeness. An implementer can pass all tests while missing a requirement. |
| "I'll do one big commit at the end" | Per-task commits make it possible to revert individual changes. One commit is an all-or-nothing gamble. |
| "This is too complex to do autonomously" | Break it smaller. If a single task is too complex, it needs decomposition, not human hand-holding. |
| "Backend task 沒 UI 我就 mock DB 跑 unit test 就好" | NO。Backend mode integration test 一律打真 DB — mock DB 過得了 unit test，prod migration 翻車你就知道。Phase 3 起 DB container 就是為了這個。 |
| "這個 migration 就只是加一個欄位，不用分階段" | 加欄位 + NOT NULL + 無預設值 = 會擋住舊 code 寫入。預設先 nullable，或分成三階段。Data layer reviewer 會擋下來。 |
| "我 handler 改完就好，OpenAPI / SDL 之後再更新" | Contract drift 的 bug 最貴 — 前端以為回傳格式是 A，實際是 B。API contract reviewer 會擋。同一次 commit 更新。 |
| "AuthZ 在 middleware 有做過了，handler 不用再檢查" | Middleware 檢查 authN（身份），handler 要檢查 authZ（這個人能不能動這筆資料 / IDOR）。兩件事。 |
| "Breaking change 上線前我再通知下游" | Breaking 要拆成 additive → migrate callers → remove old 三階段，不是 big bang。一次上完就算通知下游也來不及改。 |

## Red Flags

- **Controller writing ANY implementation code in Phase 4** — the controller NEVER writes code; every task MUST be dispatched via `Agent()`, no matter how small
- **Using sonnet for implementer** — implementer 一律用 opus（omit model）。Reviewer/tester 用 sonnet（pass `model: "sonnet"`）
- Stopping to ask about trivial/reversible decisions
- Skipping worktree setup
- Skipping spec or quality reviews
- Not running full test suite between tasks
- Proceeding past spec phase without human approval
- Auto-merging without human decision
- One giant commit instead of per-task commits
- **(Backend) Mock DB 跑 integration test** — 一律打真 DB container
- **(Backend) Migration 跟 code 同一個 commit** — 應分開（expand commit 先、code commit 後）
- **(Backend) Handler 改了但 OpenAPI/SDL 沒同步** — contract drift 是最貴的 bug
- **(Backend) Destructive migration 無保護直接上** — DROP / rename / type change 要分階段

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
