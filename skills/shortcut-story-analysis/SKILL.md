---
name: shortcut-story-analysis
description: Fetches Shortcut stories and runs iterative requirement analysis. Posts clarifying questions as comments, re-analyzes when answers arrive, and hands off to /auto when requirements are clear. Use when the user mentions a Shortcut story ID (sc-1234 or plain number), asks to analyze a ticket, or provides a Shortcut URL. Triggers on "shortcut", "story", "ticket", "sc-\d+", or app.shortcut.com URLs.
---

# Shortcut Story Analysis

## Overview

從 Shortcut 取得 story 內容，進行迭代式需求分析。每輪分析後若有不明確之處，在 story 留言提問；待回覆後由使用者重新觸發下一輪分析。反覆直到需求完全明確，最終銜接 `/auto` 進入自動開發流程。

## When to Use

- 使用者提到 Shortcut story ID（`sc-1234` 或純數字 `1234`）
- 使用者要求分析或查看某張 ticket
- 使用者提供 `https://app.shortcut.com/` 開頭的 URL
- 使用者說「繼續分析」或「有人回覆了」（重新觸發下一輪）

**NOT for:**
- 建立新 story
- Epic/Iteration 管理
- 需求已經明確，直接要開工的情境（直接用 `/auto`）

## Prerequisites

### Shortcut MCP Server

需在 Claude Code 中配置 Shortcut MCP server：

```json
{
  "mcpServers": {
    "shortcut": {
      "url": "https://mcp.shortcut.com/mcp"
    }
  }
}
```

### Fallback

MCP 不可用時，使用 `SHORTCUT_API_TOKEN` 環境變數 + curl 呼叫 Shortcut API v3。

## The Flow

```
使用者提供 story ID
        │
        ▼
┌──────────────┐
│  ROUND 1     │
│  取得 Story   │──→ 取得描述、圖片、留言、上下文
│  分析需求     │──→ 評估明確度、找出缺口
│              │
│  有問題？     │
│  ├─ YES ─────│──→ 留言提問到 story → 告知使用者等待回覆
│  │           │                          │
│  │           │    ┌─────────────────────┘
│  │           │    │ 使用者重新觸發（有人回覆了）
│  │           │    ▼
│  │           │  ROUND N
│  │           │  重新取得 story（含新留言）
│  │           │  比對已提問 vs 已回覆
│  │           │  分析新資訊
│  │           │  還有問題？ ──→ YES → 留言追問 → 等待
│  │           │       │
│  └───────────│───────┘
│              │   NO
│  需求明確 ✅  │──→ 輸出完整需求摘要
└──────────────┘    提示使用者執行 /auto
```

**人工觸發點**：每輪留言提問後，等使用者確認「有回覆了」再進入下一輪。不自動輪詢。

## Process

### Step 1：取得 Story 資料

從 story ID 擷取純數字部分（`sc-1234` → `1234`）。

**使用 MCP 工具（優先）：**
- `stories-get-by-id` — 取得 story 詳情（含描述、留言、狀態、owner）

**並行取得上下文**（若 story 有關聯資料）：
- `epics-get-by-id` — epic 上下文
- `workflows-get-by-id` — workflow 及 state 名稱
- `stories-get-history` — 變更歷史

**Fallback（MCP 不可用時）：**
```bash
curl -s -H "Content-Type: application/json" \
  -H "Shortcut-Token: $SHORTCUT_API_TOKEN" \
  "https://api.app.shortcut.com/api/v3/stories/{story-id}"

curl -s -H "Content-Type: application/json" \
  -H "Shortcut-Token: $SHORTCUT_API_TOKEN" \
  "https://api.app.shortcut.com/api/v3/stories/{story-id}/comments"
```

**處理圖片**：若描述中包含 `![...](https://media.app.shortcut.com/...)`，用 curl 下載到 `/tmp/sc-{id}-img-{n}.png`，再用 Read tool 讀取顯示。

### Step 2：結構化呈現

```
## Story sc-{id}：{name}

**類型**：{story_type}　**狀態**：{workflow_state_name}
**Epic**：{epic_name}（若有）
**Owner**：{owner_names}
**Labels**：{label_names}

### 描述
{description}

### 圖片
[用 Read tool 逐一顯示]

### 留言（共 N 則）
- [{author} @ {created_at}] {text}
```

### Step 3：需求分析

針對 story 內容進行分析，產出：

```
### 需求分析

**需求摘要**：[1-2 句話總結核心問題/目標]

**明確度評估**：
- 驗收條件：✅ 明確 / ⚠️ 模糊 / ❌ 缺失
- 技術方案：✅ 已指定 / ⚠️ 需推斷 / ❌ 未說明
- 設計稿：✅ 已附 / ❌ 未附
- 邊界情況：✅ 已考慮 / ⚠️ 部分 / ❌ 未提及

**複雜度判斷**：低 / 中 / 高
[判斷依據]
```

### Step 4：決策——有問題還是可以開工

```
分析結果
    │
    ├── 有不明確的需求 ──→ Step 5（留言提問）
    └── 需求完全明確 ────→ Step 6（準備開工）
```

判斷標準——以下任一項成立即視為「有問題」：
- 驗收條件缺失或模糊，無法從描述推斷
- 存在多種合理的技術方案，無法從上下文判斷偏好
- 影響範圍不明（例如：改 A 會不會連帶影響 B？）
- 邊界情況未定義且影響實作方向

**不要過度提問**。如果能從 codebase 或既有慣例合理推斷，就寫下假設並繼續，不需要每個細節都問。

### Step 5：留言提問

將待釐清的問題組織成留言，發佈到 story。

**使用 MCP 工具（優先）：**
- `stories-create-comment` — 在 story 新增留言

**Fallback：**
```bash
curl -s -X POST -H "Content-Type: application/json" \
  -H "Shortcut-Token: $SHORTCUT_API_TOKEN" \
  -d '{"text": "留言內容"}' \
  "https://api.app.shortcut.com/api/v3/stories/{story-id}/comments"
```

**留言格式**：

```markdown
## 需求釐清 — 第 {N} 輪分析

分析 sc-{id} 後，有以下問題需要確認：

**問題 1：[標題]**
[具體疑問 + 為何無法從現有資訊推斷]

**問題 2：[標題]**
[具體疑問 + 為何無法從現有資訊推斷]

---
_由 shortcut-story-analysis 產生。回覆後請通知重新分析。_
```

留言完成後，告知使用者：

```
已在 sc-{id} 留言提問 {N} 個問題。
等 story 上有人回覆後，告訴我「繼續分析 sc-{id}」即可進入下一輪。
```

**流程在此暫停，等待使用者重新觸發。**

### Step 5b：重新觸發（Round 2+）

使用者說「繼續分析」、「有人回覆了」、或再次提供同一個 story ID 時：

1. **重新執行 Step 1**：取得最新的 story 內容（含新留言）
2. **比對留言**：找出上一輪提問後新增的回覆
3. **重新執行 Step 3**：基於新回覆重新分析，更新明確度評估
4. **重新執行 Step 4**：判斷是否還有未解決的問題
   - 還有 → 回到 Step 5，留言追問
   - 全部解決 → 進入 Step 6

呈現時標明這是第幾輪分析，並顯示哪些問題已解決、哪些是新發現的：

```
### 第 {N} 輪分析

**已解決**：
- ✅ 問題 1：[回覆摘要]
- ✅ 問題 2：[回覆摘要]

**仍待釐清**：
- ❓ 問題 3：[新發現的問題]

**新增資訊**：
- [從回覆中獲得的重要資訊]
```

### Step 6：需求確認——準備開工

所有問題解決後，輸出完整的需求摘要，作為 `/auto` 的輸入：

```
### ✅ 需求分析完成 — sc-{id}

**需求摘要**：
[完整的需求描述，整合原始 story + 所有留言討論的結論]

**驗收條件**：
1. [具體、可驗證的條件]
2. [具體、可驗證的條件]
3. ...

**技術約束**：
- [從討論中確認的技術方向或限制]

**影響範圍**：
- [預期會動到的模組/檔案]

**邊界情況**：
- [已確認的邊界情況處理方式]

---
需求已明確，可執行 `/auto` 開始自動開發流程。
```

提示使用者：
```
sc-{id} 的需求分析已完成，共經過 {N} 輪分析。
輸入 /auto 即可開始自動開發（spec → plan → build → review）。
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「Story 內容很短，不需要分析」 | 短 story 更容易隱藏模糊需求。分析能揭露缺失的驗收條件。 |
| 「這些問題我自己假設就好」 | 錯誤的假設比多問一輪的成本高得多。影響實作方向的問題必須確認。 |
| 「問題太多會煩到對方」 | 把問題整理成結構化留言，一次問清楚，比反覆做錯重來好。 |
| 「回覆還沒來，我先開始做」 | 在需求不明確時開工是最常見的返工原因。等待確認是值得的。 |
| 「差不多可以了，直接 /auto」 | 如果明確度評估中有任何 ⚠️ 或 ❌，就還不夠明確。 |

## Red Flags

- 未做分析就直接建議執行 `/auto`
- 提問留言中包含可以從 codebase 推斷的問題（過度提問）
- 分析中標記了 ⚠️ 或 ❌ 卻沒有在 Step 5 提問
- 重新觸發時沒有重新取得最新留言
- 需求摘要遺漏了留言討論中確認的重要細節
- 跳過圖片處理，但 story 描述中明確有圖片

## Verification

每一輪分析完成後，確認：

- [ ] Story 內容已完整取得（描述、圖片、留言）
- [ ] 結構化呈現包含所有關鍵欄位（類型、狀態、owner、labels）
- [ ] 需求分析包含：摘要、明確度評估、複雜度判斷
- [ ] 若有圖片，已下載並顯示
- [ ] 明確度有 ⚠️ 或 ❌ 時，已在 story 留言提問
- [ ] 留言格式結構化，問題具體且附帶無法推斷的理由

最終輪（準備開工）額外確認：

- [ ] 所有先前提出的問題都已在留言中得到回覆
- [ ] 需求摘要整合了所有輪次的討論結論
- [ ] 驗收條件具體且可驗證
- [ ] 已提示使用者可執行 `/auto`
