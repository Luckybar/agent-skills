---
name: figma-visual-review
description: Figma 視覺比對 agent — 將建置結果與 Figma 設計稿進行頁面級與元件級的截圖比對，透過 team 模式即時與 builder agent 溝通差異。可手動觸發 (/figma-review) 或嵌入 autonomous-pipeline 的 build 階段自動執行。Use when implementing UI from Figma designs and you need visual fidelity verification.
---

# Figma Visual Review

## Overview

一個專職視覺比對的 agent，在 build 階段與 builder agent 組成 team，即時比對建置結果與 Figma 設計稿。比對粒度涵蓋整頁截圖和單一元件層級。發現差異時，透過 SendMessage 即時回報 builder 修正，反覆直到畫面與 Figma 一致。

## When to Use

- 從 Figma 設計稿實作 UI 時（搭配 `autonomous-pipeline` 或 `subagent-driven-development`）
- 手動觸發 `/figma-review` 對已建置的畫面做視覺驗收
- 任何需要確保 UI 與 Figma 1:1 一致的場景

**NOT for:**
- 沒有 Figma 設計稿的專案
- 純後端/API 開發
- 設計稿還在草稿階段、尚未定稿

## Prerequisites

### Figma MCP Server
需要 `figma-remote-mcp` 可用，用於：
- `get_screenshot(fileKey, nodeId)` — 取得 Figma 設計截圖
- `get_design_context(fileKey, nodeId)` — 取得結構化設計資料（顏色、間距、字體）
- `get_metadata(fileKey, nodeId)` — 取得節點結構（用於定位子元件）

### Browser Tools
需要瀏覽器工具可用（以下擇一）：
- `mcp__claude-in-chrome__*` — Chrome 瀏覽器截圖
- `mcp__plugin_playwright_playwright__*` — Playwright 截圖

### Dev Server
建置的專案必須有 dev server 跑起來，reviewer 才能截取實際畫面。

## Architecture

### Team Mode（嵌入 pipeline 時）

```
┌─────────────────────────────────────────────────┐
│ TEAM: build-squad                               │
│                                                 │
│ BUILDER (implementer)                           │
│  ├─ 實作 UI 元件/頁面                             │
│  ├─ 完成後 SendMessage → VISUAL REVIEWER        │
│  └─ 收到差異報告後修正                              │
│                                                 │
│ VISUAL REVIEWER (this agent)                    │
│  ├─ 收到通知後取 Figma 截圖                        │
│  ├─ 截取瀏覽器實際畫面                              │
│  ├─ 比對差異                                     │
│  └─ SendMessage → BUILDER 回報結果               │
└─────────────────────────────────────────────────┘
```

### Standalone Mode（手動 `/figma-review`）

```
使用者提供 Figma URL + 實際頁面 URL
    │
    ▼
取得 Figma 截圖 ──→ 截取瀏覽器畫面 ──→ 比對 ──→ 輸出差異報告
```

## Context Management

截圖和設計資料是 context 殺手。一張 Figma 截圖 + 一張瀏覽器截圖 + `get_design_context` 的結構化資料可以吃掉大量 context window。必須嚴格控制。

### 核心原則：比對一個、記錄文字、丟掉圖片

```
對每個比對區塊：
1. 取得 Figma 截圖 + 瀏覽器截圖
2. 比對，產出文字差異報告（期望值 vs 實際值）
3. ⚠️ 不要保留截圖在 context 中 — 只保留文字報告
4. 進入下一個區塊
```

### Standalone 模式策略

- **單頁面比對**：先用 `get_metadata`（輕量）取得節點結構，規劃要比對哪些區塊。然後逐一比對，每個區塊獨立取截圖 → 比對 → 記錄文字 → 繼續下一個
- **多元件比對**：不要同時取所有元件的截圖。一個一個來
- **修正循環**：每次修正後重新截圖時，不需要保留上一輪的截圖。只保留「第 N 輪仍未通過的項目」文字清單
- **超過 5 個區塊**：考慮用 subagent 分批處理（每 3-5 個區塊一個 subagent）

### Pipeline 模式策略

Pipeline 模式下 visual reviewer 本身就是 subagent（fresh context），天然不會有 context 壓力。但仍應遵守：
- 每個區塊比對完只保留文字報告
- SendMessage 回報時發送文字差異清單，不要附截圖

## Process

### Step 1：收集比對資料

**1a. 取得 Figma 設計結構（輕量優先）**

從 Figma URL 解析 `fileKey` 和 `nodeId`：
- `figma.com/design/:fileKey/:fileName?node-id=:nodeId` → 將 nodeId 中的 `-` 轉為 `:`

```
1. get_metadata(fileKey, nodeId) → 節點結構（輕量，用於規劃比對區塊）
2. get_design_context(fileKey, nodeId) → 取得設計資料，⚠️ 特別注意 annotations
3. 規劃比對清單：哪些子節點需要逐一比對
4. ⚠️ 不要一次取所有截圖。等 Step 2-4 逐一比對時再取
```

**⚠️ Annotations 是高優先項目：** Figma 設計稿中的 annotation（設計師留下的標記/備註）都是重要資訊，包含設計意圖、互動規格、邊界條件等。在 `get_design_context` 回傳結果中看到 annotations 時，必須逐條閱讀並作為比對標準。有 annotation 的元件應優先比對。

**1b. 截取瀏覽器實際畫面**

確保 dev server 已啟動，然後：

```
1. 導航到目標頁面 URL
2. 等待頁面完全載入（含圖片、字體）
3. 整頁截圖
4. 針對關鍵元件個別截圖（使用 CSS selector 定位）
```

**如果是 team mode，builder 會在 SendMessage 中提供：**
- 實作完成的頁面 URL
- 哪些元件/區塊是新完成的
- 對應的 Figma node ID（如果有）

### Step 2：頁面級比對

將 Figma 整頁截圖與瀏覽器整頁截圖並排比對，檢查：

```
### 頁面級比對清單

- [ ] **整體佈局**：區塊排列順序、比例是否一致
- [ ] **間距**：區塊之間的 margin/padding 是否符合設計
- [ ] **背景色**：整頁背景、各區塊背景色是否正確
- [ ] **響應式**：在目標 viewport 下佈局是否正確
- [ ] **內容完整性**：所有設計稿中的區塊是否都已實作
```

### Step 3：元件級比對

針對每個關鍵元件，比對 Figma 設計與實際渲染：

```
### 元件級比對清單

- [ ] **Annotations**：設計師標記的備註、互動規格、邊界條件（有 annotation 必看，最高優先）
- [ ] **字體**：font-family、font-size、font-weight、line-height
- [ ] **顏色**：文字色、背景色、邊框色是否精確匹配
- [ ] **間距**：元件內部 padding、元素間 gap
- [ ] **圓角**：border-radius 是否一致
- [ ] **陰影**：box-shadow 是否一致
- [ ] **圖標/圖片**：尺寸、位置、是否正確顯示
- [ ] **互動狀態**：hover、active、disabled 狀態（若設計稿有定義）
- [ ] **對齊**：文字對齊、元素在容器中的對齊方式
```

使用 `get_design_context` 取得的結構化資料來驗證具體數值（如精確的色碼、px 值）。

### Step 4：截圖比對（逐一處理，不累積）

對每個比對的區塊/元件，**一個一個來**：

```
For 區塊 in 比對清單:
    1. get_screenshot(fileKey, 子節點 nodeId) → Figma 截圖
    2. get_design_context(fileKey, 子節點 nodeId) → 取得精確數值
    3. 截取瀏覽器中對應區域截圖
    4. 比對兩張截圖，記錄文字差異（期望值 vs 實際值）
    5. ⚠️ 截圖已完成使命 — 後續只參考文字報告
    6. 進入下一個區塊
```

**關鍵：不要同時持有多個區塊的截圖。** 每個區塊比對完，文字報告是唯一需要保留的輸出。

**比對結果格式：**

```
## 視覺比對報告

### ✅ 通過
- Header 區塊：佈局、顏色、字體一致
- Footer 區塊：完全匹配

### ⚠️ 需修正
- **Hero Section — 間距問題**
  Figma：padding-top: 80px, padding-bottom: 60px
  實際：padding-top: 64px, padding-bottom: 48px
  嚴重度：中

- **CTA Button — 顏色不符**
  Figma：background #2563EB, border-radius 12px
  實際：background #3B82F6, border-radius 8px
  嚴重度：高

### ❌ 缺失
- Testimonials 區塊：設計稿有但尚未實作
```

### Step 5：回報與修正循環（Team Mode）

發現差異後，透過 SendMessage 回報 builder：

**訊息格式（Reviewer → Builder）：**

```
FROM: visual-reviewer
STATUS: NEEDS_FIXES
FIXES_COUNT: 3

FIX 1 [高]: CTA Button 背景色
- 期望: #2563EB → 實際: #3B82F6
- 檔案提示: 可能在 Button 元件的 primary variant

FIX 2 [中]: Hero Section 間距
- 期望: padding 80px/60px → 實際: 64px/48px
- 檔案提示: Hero 元件的外層容器

FIX 3 [低]: Card 圓角
- 期望: 12px → 實際: 8px
```

**訊息格式（Builder → Reviewer，修正後）：**

```
FROM: builder
STATUS: FIXES_APPLIED
DETAILS: 已修正 FIX 1, FIX 2, FIX 3
URL: http://localhost:3000/page
READY_FOR_REVIEW: true
```

**修正循環上限：每個區塊最多 3 輪。** 超過 3 輪仍有差異，記錄為 non-blocking 問題交由人工確認。

### Step 5b：最終確認

所有區塊通過後：

```
FROM: visual-reviewer
STATUS: APPROVED
SUMMARY: 所有 N 個區塊/元件通過視覺比對
ROUNDS: 平均 M 輪修正
NOTES:
- [non-blocking 的微小差異，如 sub-pixel rendering]
```

### Step 6：輸出最終報告（Standalone Mode）

手動觸發時，輸出完整報告：

```
## Figma 視覺比對報告

**Figma 來源**：[figma URL]
**比對頁面**：[page URL]
**比對時間**：[timestamp]

### 摘要
- 比對區塊：N 個
- 通過：X 個
- 需修正：Y 個
- 缺失：Z 個

### 詳細差異
[Step 4 的完整報告]

### 建議修正優先順序
1. [高嚴重度項目]
2. [中嚴重度項目]
3. [低嚴重度項目]
```

## Integration with autonomous-pipeline

### 啟用條件

在 `autonomous-pipeline` Phase 0 偵測到以下條件時啟用：

```
Input 包含 Figma URL？
    │
    ├── YES → 啟用 figma-visual-review
    │         在 Phase 1 Spec 中要求包含 Figma URL mapping
    │         在 Phase 2 Plan 中每個 UI task 標記對應的 Figma nodeId
    │         在 Phase 4 Build Loop 中加入 Visual Reviewer
    │
    └── NO  → 不啟用（維持原有流程）
```

### Build Loop 整合

在 `subagent-driven-development` 的 per-task loop 中，當 task 有 Figma 對應時：

```
1. Dispatch IMPLEMENTER（含 mock data 要求）
2. Implementer 完成 → Dispatch SPEC REVIEWER
3. Spec review 通過 → Dispatch VISUAL REVIEWER
   → Visual reviewer 取 Figma 截圖 + 瀏覽器截圖
   → 比對差異
   → 有差異 → SendMessage 回 implementer 修正 → 重新比對（max 3 rounds）
   → 通過 → 繼續
4. Visual review 通過 → Dispatch CODE QUALITY REVIEWER
5. 全部通過 → 標記 task 完成
```

### Mock Data 要求

**為什麼需要 mock data：** 空白頁面無法做視覺比對。builder 必須填充合理的假資料，reviewer 才能準確比對佈局、文字溢出、圖片位置等。

**Implementer 的 mock data 規則：**
- 所有文字欄位必須填入接近真實長度的假文字（非 "Lorem ipsum"，要用符合語境的內容）
- 列表/表格需要 3-5 筆假資料
- 圖片使用 placeholder（帶正確尺寸比例）
- 數字欄位使用合理範圍的假數值
- Mock data 應放在獨立的 fixture/seed 檔案中，方便日後替換

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「差 2px 而已，不用這麼嚴格」 | 2px 的間距差異累積起來會讓整個頁面看起來「哪裡怪怪的」。設計師會注意到。 |
| 「顏色差不多，可能是螢幕問題」 | 用色碼比對，不要用肉眼。#2563EB 和 #3B82F6 是完全不同的顏色。 |
| 「先做完功能再調畫面」 | 畫面和功能同時做才能確保一致。事後調畫面常常會打破已完成的功能。 |
| 「Mock data 之後再加就好」 | 沒有 mock data 就無法比對。空白頁面看起來都一樣——都是空的。 |
| 「Figma 上這個元件沒有定義互動狀態」 | 沒有設計不代表不需要。至少 hover 狀態要有合理的預設行為。 |
| 「截圖比對太慢了，看 code 就好」 | Code 正確不代表渲染正確。CSS 的 cascading、繼承、覆蓋常常造成意外結果。 |

## Red Flags

- 跳過截圖比對，只看 code 就說「匹配」
- 忽略 Figma annotations — 有 annotation 的元件是設計師特別標記的，必須閱讀並遵循
- 沒有用 `get_design_context` 驗證具體數值（色碼、px），只靠視覺感覺
- Builder 沒有填入 mock data 就送審
- 修正循環超過 3 輪仍未收斂，但沒有升級給人工
- 只做整頁比對，沒有做元件級比對
- 比對報告沒有具體的期望值 vs 實際值
- Reviewer 自己去改 code（reviewer 是唯讀角色）
- 同時持有 3 張以上截圖在 context 中（應逐一比對、只保留文字報告）
- Standalone 模式下沒有分批處理超過 5 個區塊的比對

## Verification

比對完成後確認：
- [ ] Figma 截圖與瀏覽器截圖都已取得
- [ ] 頁面級比對已完成（佈局、間距、背景色、內容完整性）
- [ ] 元件級比對已完成（字體、顏色、間距、圓角、陰影、圖標）
- [ ] 所有差異都有具體的期望值 vs 實際值
- [ ] 高嚴重度差異已全部修正
- [ ] 修正循環未超過 3 輪上限
- [ ] Mock data 已填充，非空白頁面
- [ ] 最終截圖確認修正後的畫面與 Figma 一致
