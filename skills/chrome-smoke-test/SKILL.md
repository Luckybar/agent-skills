---
name: chrome-smoke-test
description: 使用 Chrome 瀏覽器對建置完成的頁面執行全面冒煙測試 — Console 零錯誤、RWD 多斷點截圖、API 回應驗證、互動流程測試、無障礙檢查。可手動觸發 (/chrome-test) 或嵌入 autonomous-pipeline 的 build 階段自動執行。Use when UI implementation is complete and needs runtime verification before marking done.
---

# Chrome Smoke Test

## Overview

在 UI 建置完成後，透過 Chrome 瀏覽器自動化工具對每個頁面執行冒煙測試。確認頁面在真實瀏覽器中能正常運作：沒有 Console 錯誤、RWD 響應正確、API 回應正常、互動功能可用、無障礙基本合規。這是 code review 看不出來但使用者一定會碰到的問題。

## When to Use

- UI 實作完成後，需要 runtime 驗證
- 搭配 `autonomous-pipeline` 或 `subagent-driven-development` 在 build 階段自動執行
- 手動觸發 `/chrome-test` 對已建置的頁面做冒煙測試
- 修完 bug 後確認修復有效且沒有 regression

**NOT for:**
- 純後端/API 開發（無瀏覽器畫面）
- 需要登入狀態的深度功能測試（用 `/playwright` 更適合）
- 效能調校（用 `performance-optimization` skill）

## Prerequisites

### Chrome MCP Tools
需要 `mcp__claude-in-chrome__*` 工具可用。核心工具：
- `tabs_context_mcp` — 取得分頁資訊
- `tabs_create_mcp` — 建立新分頁
- `navigate` — 導航到 URL
- `computer` — 截圖
- `resize_window` — 調整視窗大小（RWD 測試）
- `read_console_messages` — 讀取 console 訊息
- `read_network_requests` — 讀取網路請求
- `javascript_tool` — 執行唯讀 JS 檢查
- `find` — 搜尋頁面元素

### Dev Server
專案的 dev server 必須跑起來。

## Architecture

### Standalone Mode（`/chrome-test`）

```
使用者提供頁面 URL（或 figma-mapping.json 中的路由清單）
    │
    ▼
對每個頁面依序執行 5 項測試
    │
    ▼
輸出完整測試報告
```

### Pipeline Mode（嵌入 build loop）

```
┌──────────────────────────────────────────────────────────┐
│  Per-task execution loop (UI task):                      │
│                                                          │
│  1. IMPLEMENTER → 實作 + mock data + commit              │
│  2. SPEC REVIEWER → 驗收條件                              │
│  3. VISUAL REVIEWER → Figma 截圖比對（若有 Figma）         │
│  4. BROWSER TESTER → 冒煙測試 ← THIS                     │
│  5. CODE QUALITY REVIEWER → 程式碼品質                    │
│  6. 標記完成                                              │
└──────────────────────────────────────────────────────────┘
```

在 team mode 中，BROWSER TESTER 透過 SendMessage 與 IMPLEMENTER 溝通：
- 發現問題 → 回報具體錯誤 → implementer 修正 → 重新測試
- 所有測試通過 → APPROVED

## Context Management

RWD 測試會產生 4 張截圖（4 個斷點），多頁面模式下截圖會快速累積（4 × N 頁）。必須控制 context 用量。

### 核心原則：測一個斷點、記錄文字、丟掉截圖

```
RWD 測試流程：
1. resize → screenshot → 檢查問題 → 記錄文字結果（✅/❌ + 問題描述）
2. ⚠️ 截圖只用於當下判斷，不累積
3. resize 到下一個斷點 → 重複
4. 最後只保留文字報告
```

### 多頁面模式策略

- **逐頁測試**：測完一頁、輸出該頁文字報告，再測下一頁
- **不要同時持有多頁的截圖**
- **超過 5 頁**：用 subagent 分批（每個 subagent 負責 2-3 頁）
- **最終匯總表**是文字表格，不附截圖

### 何時需要保留截圖

只有以下情況需要保留截圖直到使用者看過：
- 使用者明確要求「給我看截圖」
- 發現 ❌ FAIL 的斷點，截圖作為問題證據展示給使用者（只保留 FAIL 的，不保留 PASS 的）

## Process

### Step 0：初始化

1. 用 ToolSearch 載入所需的 Chrome MCP 工具
2. 呼叫 `tabs_context_mcp` 確認瀏覽器連線
3. 確定測試目標：
   - Standalone：使用者提供 URL，或讀取 `figma-mapping.json` 的路由清單
   - Pipeline：從 task 描述取得頁面 URL

### Step 1：Console 零錯誤測試

**目的**：確保頁面沒有 JavaScript 錯誤或警告。

```
1. 導航到目標頁面
2. 等待頁面完全載入
3. read_console_messages(onlyErrors: true)
4. read_console_messages() — 取得所有訊息，檢查 warnings
```

**判定標準**：
- ✅ PASS：零 error、零 warning
- ⚠️ WARN：有 warning 但無 error（記錄但不阻擋）
- ❌ FAIL：有 error

**報告格式**：
```
### Console 測試
- 狀態：✅ PASS / ⚠️ WARN / ❌ FAIL
- Errors：0
- Warnings：0
- 詳細：[若有錯誤，列出每個錯誤訊息 + 可能原因]
```

### Step 2：RWD 響應式測試

**目的**：確認頁面在不同裝置尺寸下正確響應。

```
對每個斷點（逐一處理，不累積截圖）：
1. resize_window(width, height)
2. 等待重排
3. computer(action: screenshot) — 截圖
4. 檢查是否有水平溢出、元素重疊、文字截斷
5. 記錄文字結果（✅/❌ + 問題描述）
6. ⚠️ 截圖已完成使命 — 只保留 ❌ FAIL 的截圖作為證據
7. 進入下一個斷點
```

**測試斷點**：

| 裝置 | 寬度 | 高度 | 代表 |
|------|------|------|------|
| Mobile S | 320 | 568 | iPhone SE |
| Mobile | 375 | 667 | iPhone 8 |
| Tablet | 768 | 1024 | iPad |
| Desktop | 1440 | 900 | 標準桌面 |

**判定標準**：
- ✅ PASS：所有斷點下佈局正確、無溢出
- ❌ FAIL：有破版、水平滾動條、元素重疊、文字被截斷

**報告格式**：
```
### RWD 測試
- 狀態：✅ PASS / ❌ FAIL

| 斷點 | 結果 | 問題 |
|------|------|------|
| Mobile S (320px) | ✅ | — |
| Mobile (375px) | ✅ | — |
| Tablet (768px) | ⚠️ | Nav 選單未正確收合 |
| Desktop (1440px) | ✅ | — |

[附上每個斷點的截圖]
```

### Step 3：API / Network 驗證

**目的**：確認頁面的 API 請求正常、無失敗請求。

```
1. 導航到頁面（觸發初始 API 請求）
2. read_network_requests()
3. 過濾出 XHR/Fetch 請求（排除靜態資源）
4. 檢查每個請求的 status code
```

**判定標準**：
- ✅ PASS：所有 API 請求返回 2xx
- ⚠️ WARN：有 3xx（重導向，記錄但不阻擋）
- ❌ FAIL：有 4xx 或 5xx

**報告格式**：
```
### Network 測試
- 狀態：✅ PASS / ❌ FAIL
- API 請求數：N

| Endpoint | Method | Status | 結果 |
|----------|--------|--------|------|
| /api/posts | GET | 200 | ✅ |
| /api/user | GET | 401 | ❌ 未授權 |
```

### Step 4：互動功能測試

**目的**：確認頁面的互動元素（連結、按鈕、表單）可正常運作。

```
1. 用 find 工具找到頁面上的所有可互動元素
2. 測試導航連結：
   - 找到所有 <a> 標籤
   - 點擊每個內部連結，確認導航成功
   - 返回原頁面
3. 測試按鈕：
   - 找到所有 <button> 標籤
   - 點擊非破壞性按鈕（避免刪除、送出等）
   - 確認有回應（無 JS error、有視覺反饋）
4. 測試表單（若有）：
   - 用 form_input 填入 mock 資料
   - 確認 validation 正常
   - 不實際提交（避免副作用）
```

**安全規則**：
- 不點擊「刪除」、「送出」、「確認付款」等有副作用的按鈕
- 不提交表單到後端
- 只做讀取性質的互動測試

**判定標準**：
- ✅ PASS：所有連結可導航、按鈕有回應、表單 validation 正常
- ❌ FAIL：死連結、按鈕無反應、表單 validation 異常

**報告格式**：
```
### 互動測試
- 狀態：✅ PASS / ❌ FAIL

**連結**：N 個測試，M 個通過
- ❌ /about → 404
- ✅ /contact → 正常

**按鈕**：N 個測試，M 個通過
- ✅ "Learn More" → 正常展開內容

**表單**：N 個測試
- ✅ Email validation → 阻擋無效格式
```

### Step 5：無障礙基本檢查

**目的**：確認基本無障礙合規。

```
1. 用 javascript_tool 執行唯讀檢查：
   - 所有 <img> 有 alt 屬性
   - heading 層級正確（h1 → h2 → h3，不跳級）
   - 所有互動元素可被 focus
   - 色彩對比度（若可取得 computed styles）
2. 用 find 搜尋缺少 aria-label 的互動元素
```

**判定標準**：
- ✅ PASS：所有基本檢查通過
- ⚠️ WARN：有小問題（如部分圖片缺 alt）
- ❌ FAIL：嚴重問題（如完全無法用鍵盤操作）

**報告格式**：
```
### 無障礙測試
- 狀態：✅ PASS / ⚠️ WARN / ❌ FAIL

- 圖片 alt 屬性：8/8 ✅
- Heading 層級：正確 ✅
- 可 focus 元素：12/12 ✅
- 缺少 aria-label：2 個 ⚠️
  - <button> (line 45) — 無文字也無 aria-label
  - <input> (line 72) — 無對應 <label>
```

### Step 6：輸出最終報告

```
## Chrome 冒煙測試報告

**測試頁面**：[URL]
**測試時間**：[timestamp]

### 總結
| 項目 | 結果 |
|------|------|
| Console | ✅ / ❌ |
| RWD | ✅ / ❌ |
| Network | ✅ / ❌ |
| 互動 | ✅ / ❌ |
| 無障礙 | ✅ / ⚠️ / ❌ |

**總體判定**：✅ PASS / ❌ FAIL

### 需修正項目（按嚴重度排序）
1. [❌ 高] Console error: Uncaught TypeError...
2. [❌ 中] Mobile 320px 水平溢出
3. [⚠️ 低] 2 個元素缺少 aria-label

### RWD 截圖
[附上各斷點截圖]
```

### Step 7：修正循環（Pipeline Mode）

在 pipeline 中，發現問題後透過 SendMessage 通知 implementer：

**訊息格式（Tester → Builder）：**

```
FROM: browser-tester
STATUS: NEEDS_FIXES
FIXES_COUNT: N

FIX 1 [高]: Console Error
- 錯誤：Uncaught TypeError: Cannot read property 'map' of undefined
- 頁面：/home
- 觸發條件：頁面載入時

FIX 2 [中]: RWD 破版
- 斷點：320px
- 問題：Hero section 水平溢出
- 截圖：[附上]
```

**修正循環上限：每個頁面最多 2 輪。** Console errors 和 network failures 必須在 2 輪內解決，RWD 和無障礙的 ⚠️ 級問題可標記為 non-blocking。

## Integration with autonomous-pipeline

### 啟用條件

所有包含 UI 實作的 task 都應啟用 chrome-smoke-test。不需要額外條件（不像 visual review 需要 Figma URL）。

```
Task 包含 UI 實作？
    │
    ├── YES → 啟用 chrome-smoke-test
    │         在 build loop 中 visual review 之後、code quality review 之前
    │
    └── NO  → 不啟用（純後端 task）
```

### Build Loop 整合

```
1. Dispatch IMPLEMENTER（含 mock data）
2. Spec review 通過
3. Visual review 通過（若有 Figma）
4. Chrome smoke test → THIS
   → 5 項測試全部執行
   → 有 ❌ → SendMessage 回 implementer → 修正 → 重測（max 2 rounds）
   → 全部 ✅ 或只剩 ⚠️ → 繼續
5. Code quality review
6. 標記完成
```

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「Code review 有看過了，不需要跑瀏覽器」 | Code review 看不出 CSS cascade 造成的破版、runtime JS error、CORS 問題。瀏覽器是唯一的真相來源。 |
| 「RWD 之後再測就好」 | 事後修 RWD 常常會改壞 desktop 版。同時測才能確保不互相影響。 |
| 「Console 的 warning 不重要」 | 今天的 warning 是明天的 error。React deprecation warning 忽略一年後就是強制升級。 |
| 「無障礙是 nice-to-have」 | 無障礙是法律要求（ADA, WCAG）。缺少 alt 和 aria-label 是最容易修的問題，沒理由跳過。 |
| 「Mock data 看起來正常就好」 | Mock data 必須測試邊界情況：超長文字、空值、特殊字元。只用 "Hello World" 測不出截斷問題。 |
| 「互動測試太慢了」 | 點幾個連結和按鈕只要幾秒。漏掉一個死連結的使用者體驗代價遠高於測試成本。 |

## Red Flags

- 跳過 Console 檢查就說頁面正常
- 只測一個斷點就說 RWD 通過
- 有 4xx/5xx network 請求但標記為通過
- 互動測試中點擊了有副作用的按鈕（刪除、送出）
- 無障礙測試完全跳過
- 修正循環超過 2 輪仍有 ❌ 但沒有升級給人工
- 測試報告沒有附 FAIL 項目的截圖證據
- 同時持有 4+ 張 RWD 截圖在 context 中（應逐一測試、只保留文字結果）
- 多頁面模式沒有逐頁處理，所有頁面截圖堆在同一個 context

## Verification

測試完成後確認：
- [ ] Console 零 error（warning 已記錄）
- [ ] RWD 四個斷點都已截圖並檢查
- [ ] 所有 API 請求返回 2xx
- [ ] 內部連結都可正常導航
- [ ] 按鈕有正確回應
- [ ] 圖片都有 alt 屬性
- [ ] Heading 層級正確
- [ ] 所有 ❌ 項目已修正或升級給人工
- [ ] 最終報告已輸出（含截圖）
