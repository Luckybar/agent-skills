---
name: figma-content-restructure
description: 從 Figma A 掃描、分析並推理畫面結構與流程步驟，重新組織、命名並寫入 Figma B。全程使用 Figma MCP 完成跨檔案的內容遷移與重組。Use when reorganizing Figma files, migrating design content between files, cleaning up messy designs into structured flows, or extracting and documenting user flows from unorganized Figma files.
---

# Figma Content Restructure

## Overview

從使用者提供的 Figma URL 指定的節點（頁面或 frame）出發，只掃描該節點及其子項，分析內容結構、推理使用者流程與步驟順序，然後在目標位置建立重新命名與組織過的乾淨版本。不掃描檔案中的其他頁面。全程透過 Figma MCP 工具操作，不需要瀏覽器或本地開發環境。

## When to Use

- Figma 檔案內容雜亂，需要重新整理命名與組織結構
- 需要從一個 Figma 檔案提取畫面，推理流程後寫入另一個檔案
- 將草稿/原型檔整理成正式的設計文件
- 從設計稿中推理出使用者流程步驟並建立有序的畫面集
- 跨團隊交接時，需要將設計內容遷移到新檔案並重新命名

**NOT for:**
- Figma 內單一頁面的微調（直接用 `use_figma` 即可）
- 需要程式碼實作的 design-to-code 工作（用 `figma-implement-design` skill）
- 視覺比對已建置的 UI 與 Figma 設計（用 `figma-visual-review`）

## Prerequisites

### Figma MCP Server

需要 `plugin:figma:figma` MCP server 可用，用於：

| 工具 | 用途 | 階段 |
|---|---|---|
| `get_metadata(fileKey, nodeId)` | 取得節點結構（輕量，只有 name/type/id/position） | Phase 1 掃描 |
| `get_design_context(fileKey, nodeId)` | 取得結構化設計資料（code、顏色、字體、佈局） | Phase 1 分析、Phase 4 遷移 |
| `get_screenshot(fileKey, nodeId)` | 取得節點截圖 | Phase 1 參考、Phase 5 驗證 |
| `use_figma(fileKey, code, description)` | 執行 Plugin API 寫入目標檔 | Phase 4 寫入 |
| `search_design_system(query, fileKey)` | 搜尋可複用的設計系統元件 | Phase 4 元件引入 |

### 輸入

- **Figma A URL**（來源檔）— 必要。URL 中的 `node-id` 指定掃描範圍（頁面或特定節點），**只掃描該節點及其子項，不掃描其他頁面**
- **輸出模式**（流程啟動時詢問使用者）：
  - **獨立目標檔** — 使用者提供 Figma B URL，寫入另一個檔案
  - **同檔新頁面** — 在 Figma A 中建立新頁面，以 `[Restructured]` 前綴區隔

## Process

```
Figma A URL
    │
    ▼
Phase 0: 確認輸出模式 ⬅ 詢問使用者
    │
    ├── 獨立目標檔 → 使用者提供 Figma B URL
    └── 同檔新頁面 → 寫在 A 的新頁面（[Restructured] 前綴）
    │
    ▼
Phase 1: 掃描 ──→ 指定節點結構清單
    │
    ▼
Phase 2: 分析 ──→ 畫面分類 + 流程推理 + 命名分析
    │
    ▼
Phase 2.5: 標注 ──→ 在 Figma 中標注問題 ──→ ⬅ 使用者回覆
    │
    ▼
Phase 3: 規劃 ──→ 遷移映射表（含回饋調整）──→ ⬅ 人工確認
    │
    ▼
Phase 4: 寫入 ──→ 在目標位置建立重組後的內容
    │
    ▼
Phase 5: 驗證 ──→ 結構比對 + 抽樣視覺比對 ──→ 最終報告
```

---

### Phase 0：確認輸出模式（Output Mode）— HUMAN TOUCHPOINT

**流程啟動時，立即詢問使用者兩件事：**

1. **來源檔 URL** — 如果使用者還沒提供 Figma A URL，先要求提供
2. **輸出模式** — 詢問使用者選擇：

```
請問重組後的內容要輸出到哪裡？

A）獨立目標檔 — 請提供目標 Figma 檔案的 URL
B）同檔新頁面 — 我會在來源檔中建立新頁面，以 [Restructured] 前綴區隔原始內容
```

**根據選擇決定後續的 `targetFileKey` 和頁面命名策略：**

| 輸出模式 | targetFileKey | 頁面命名 | 注意事項 |
|---|---|---|---|
| **獨立目標檔** | 從 Figma B URL 解析 | `01-Authentication` | 先用 `get_metadata` 確認 B 檔現有內容，有衝突要回報 |
| **同檔新頁面** | 與 Figma A 相同 | `[Restructured] 01-Authentication` | 不修改、不刪除原有頁面 |

**Phase 0 完成條件：** 有 Figma A URL + 確認輸出模式（+ 如果選獨立目標檔，有 Figma B URL）。

---

### Phase 1：掃描指定節點（Scoped Scan）

**目標：** 只掃描使用者 URL 指定的節點及其子項，建立該範圍的畫面清單與結構地圖。

**⚠️ 掃描範圍限定：** 使用者提供的 URL 中包含 `node-id` 參數，這就是掃描的根節點。**不要掃描檔案中的其他頁面或不相關的節點**。一個 Figma 檔案可能包含多個不相關的流程/專案，只處理使用者指向的那個。

**1a. 解析 URL 並取得目標節點結構**

從 Figma A URL 解析 `fileKey` 和 `nodeId`（將 URL 中的 `-` 轉為 `:`）：

```
get_metadata(fileKey=A, nodeId=targetNodeId)
→ 取得目標節點及其所有子項
→ 記錄：name, id, type, position, size
→ 依位置排序（左→右, 上→下）推斷設計順序
```

如果目標節點是 **Page（canvas）**，其 children 就是要分析的 frame 清單。
如果目標節點是 **Frame**，則分析該 frame 本身及其子結構。

**1b. 用 Plugin API 補充大型頁面的完整子項清單**

`get_metadata` 可能不回傳所有層級。對大型頁面，用 `use_figma` 確保取得完整的頂層子項：

```javascript
// use_figma — 取得目標頁面的所有頂層子項
const page = figma.root.children.find(p => p.id === 'TARGET_PAGE_ID');
await figma.setCurrentPageAsync(page);
return page.children.map(c => ({
  name: c.name, id: c.id, type: c.type,
  x: Math.round(c.x), y: Math.round(c.y),
  w: Math.round(c.width), h: Math.round(c.height)
}));
```

**1c. 取得關鍵 frame 的設計資料**

對主要 frame 逐一取得詳細資料：

```
For frame in key_frames:
    get_design_context(fileKey=A, nodeId=frame.id, excludeScreenshot=true)
    → 分析：使用的元件、文字內容、顏色、佈局模式
    → 記錄文字摘要
    → ⚠️ 丟掉原始資料，只保留摘要（節省 context）
```

**1d. 取得代表性截圖（按需）**

只對需要視覺判斷的 frame 取截圖，**一次一張**：

```
get_screenshot(fileKey=A, nodeId=frame.id)
→ 觀察截圖
→ 記錄文字描述（例如：「登入頁，含 email/password 輸入框和登入按鈕」）
→ ⚠️ 丟掉截圖，只保留文字描述
```

**Phase 1 產出格式：**

```
## 掃描範圍

**目標節點：** {nodeName} (id: {nodeId})
**節點類型：** Page / Frame

### 子項清單
- Frame "yyy" (id: 1:2, 1440×900) — 登入頁面
- Frame "zzz" (id: 1:3, 1440×900) — 商品列表
- ...

### 統計
- 總 frame 數：M
- 識別元件：[component 名稱列表]
```

---

### Phase 2：分析與推理（Analysis & Inference）

**目標：** 理解內容的邏輯關係，推理出流程步驟與畫面分類。

**2a. 畫面分類**

將每個 frame 歸入以下分類：

| 分類 | 特徵 |
|---|---|
| **Screen** | 完整的頁面級設計，有明確的 UI 佈局 |
| **Component** | 可複用的 UI 元件（按鈕、卡片、表單等） |
| **Variant** | 同一畫面或元件的不同狀態（hover、error、loading） |
| **Flow** | 多個畫面組成的連續操作步驟 |
| **Reference** | 設計規格、色彩表、字體表等輔助內容 |
| **Draft** | 疑似未完成或已廢棄的內容（標記待人工確認） |

**2b. 推理流程步驟**

根據以下線索推斷畫面之間的流程關係（優先順序由高到低）：

```
1. Figma 中的 flow connections（連線/箭頭）
2. Frame 名稱中的序號或步驟標記（Step 1, 2, 3…）
3. 頁面內 frame 的空間位置（左→右 = 先→後）
4. 內容邏輯（登入→首頁→詳情→結帳…）
5. UI 狀態關係（預設→填寫中→完成→錯誤）
```

**每條推理必須寫明依據。** 例如：「Step 2 在 Step 1 右邊且內容為 OTP 輸入，推斷為登入後的驗證步驟」。

**2c. 命名分析**

檢查現有命名的問題：
- 命名不一致（混合中英文、駝峰/底線混用）
- 無意義名稱（Frame 1, Group 3, Rectangle 12）
- 缺少分類前綴或流程標記
- 重複或衝突的名稱

**Phase 2 產出格式：**

```
## 分析報告

### 畫面分類
| # | 來源名稱 | 分類 | 內容描述 | 推理依據 |
|---|---|---|---|---|
| 1 | "login screen" | Screen | 登入頁面 | 包含帳號密碼輸入框 |
| 2 | "Frame 37" | Screen | 商品列表 | 佈局為多欄商品卡片 |
| 3 | "btn-primary" | Component | 主按鈕 | 獨立小元件 |
| 4 | "old header v2" | Draft（待確認） | 舊版 header | 有新版存在 |

### 推理流程
Flow 1: 登入流程
  Step 1: login screen（依據：流程起點，無前置頁面）
  Step 2: OTP 驗證（依據：位於 login 右側，內容為驗證碼輸入）
  Step 3: 首頁（依據：登入成功後的目的地）

Flow 2: 購物流程
  Step 1: 商品列表 → Step 2: 商品詳情 → Step 3: 購物車 → Step 4: 結帳

### 命名問題
- "Frame 37" → 無意義名稱
- 混合中英文命名
- 缺少流程分組前綴
```

---

### Phase 2.5：標注問題並收集回饋（Annotate & Feedback Loop）

**目標：** 將分析中發現的不合理之處直接標注在 Figma 中，等使用者回覆後再調整計畫。

#### 什麼算「不合理」

在 Phase 2 分析過程中，以下情況應標注為問題：

| 類型 | 範例 |
|---|---|
| **結構不合理** | 同一個流程步驟的畫面分散在不相鄰的位置 |
| **缺失狀態** | 有「成功」畫面但沒有「錯誤」畫面 |
| **疑似重複** | 兩個 frame 視覺幾乎相同但命名不同 |
| **用途不明** | 無法從內容判斷 frame 屬於哪個流程步驟 |
| **設計不一致** | 同類元件在不同畫面中有不同的間距/顏色/尺寸 |
| **命名矛盾** | frame 名稱暗示的功能與實際內容不符 |

#### 標注方式：在 Figma 中建立 Annotation Frame

對每個問題，在**來源頁面**中建立一個視覺化的標注 frame：

```javascript
// use_figma — 建立 annotation frame
// ⚠️ 先載入 figma-use skill

const annotation = figma.createFrame();
annotation.name = '[Issue #1] 缺少錯誤狀態';
annotation.resize(320, 120);

// 視覺樣式：淺橘底 + 橘色邊框
annotation.fills = [{ type: 'SOLID', color: { r: 1, g: 0.95, b: 0.85 } }];
annotation.strokes = [{ type: 'SOLID', color: { r: 1, g: 0.5, b: 0 } }];
annotation.strokeWeight = 2;
annotation.cornerRadius = 8;
annotation.paddingLeft = 16;
annotation.paddingRight = 16;
annotation.paddingTop = 12;
annotation.paddingBottom = 12;
annotation.layoutMode = 'VERTICAL';
annotation.itemSpacing = 8;
annotation.primaryAxisSizingMode = 'AUTO';

// 問題描述
await figma.loadFontAsync({ family: 'Inter', style: 'Bold' });
await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });

const title = figma.createText();
title.fontName = { family: 'Inter', style: 'Bold' };
title.characters = '⚠ Issue #1';
title.fontSize = 14;
annotation.appendChild(title);

const desc = figma.createText();
desc.fontName = { family: 'Inter', style: 'Regular' };
desc.characters = '登入流程有「成功」畫面但缺少「錯誤」狀態。\n這是刻意的還是遺漏？';
desc.fontSize = 12;
annotation.appendChild(desc);

// 預留回覆區（空文字框，使用者可直接在 Figma 中填寫）
const reply = figma.createText();
reply.fontName = { family: 'Inter', style: 'Regular' };
reply.characters = '💬 回覆：';
reply.fontSize = 12;
reply.fills = [{ type: 'SOLID', color: { r: 0.5, g: 0.5, b: 0.5 } }];
annotation.appendChild(reply);

// 定位在問題 frame 右側
annotation.x = targetFrame.x + targetFrame.width + 40;
annotation.y = targetFrame.y;
```

**Annotation 命名規範：** `[Issue #N] 簡短問題描述`
- N 從 1 開始遞增
- 使用者可以直接在 Figma 中編輯「💬 回覆：」後面的文字

#### 標注完成後的輸出

在對話中向使用者彙報：

```
## 發現的問題

我在 Figma 中標注了 N 個問題，請在 Figma 中查看並回覆：

1. [Issue #1] 缺少錯誤狀態 — 位於「登入成功」畫面旁
2. [Issue #2] 疑似重複 — 位於「商品列表」畫面旁
3. [Issue #3] 用途不明 — 位於 "Frame 42" 旁

請在 Figma 中每個標注的「💬 回覆：」後面填寫你的回覆，
完成後告訴我，我會讀取回覆並調整計畫。
```

#### Feedback Loop：讀取回覆並調整

使用者回覆後，用 Plugin API 讀取所有 annotation 的回覆內容：

```javascript
// use_figma — 讀取使用者在 annotation 中的回覆
const page = figma.root.children.find(p => p.id === 'TARGET_PAGE_ID');
await figma.setCurrentPageAsync(page);

const annotations = page.findAll(n => n.name.startsWith('[Issue #'));
const results = [];

for (const ann of annotations) {
  const texts = ann.findAll(n => n.type === 'TEXT');
  const replyText = texts.find(t => t.characters.startsWith('💬 回覆：'));
  results.push({
    name: ann.name,
    id: ann.id,
    reply: replyText ? replyText.characters.replace('💬 回覆：', '').trim() : '（未回覆）'
  });
}

return results;
```

**根據回覆調整 Phase 3 計畫：**
- 回覆確認問題 → 在遷移計畫中加入修正項
- 回覆說明是刻意的 → 在計畫中標記為「已知，不處理」
- 未回覆 → 在 Phase 3 中標記為「待確認」，不自行決定

#### Annotation 清理

在 Phase 4 寫入完成後，**詢問使用者**是否要刪除來源頁面上的 annotation frame：

```
所有 annotation 已處理完畢。要刪除 Figma 中的 N 個標注框嗎？(Y/N)
```

---

### Phase 3：規劃重組（Plan Restructure）— HUMAN TOUCHPOINT

**目標：** 建立完整的遷移計畫，提交人工審核。

**3a. 定義命名規範**

提出新的命名規範建議：

```
建議命名格式：
  頁面：{FlowNumber}-{FlowName}
  畫面：{FlowNumber}-{StepNumber}_{ScreenName}[_{State}]
  元件：{ComponentName}/{VariantName}

範例：
  頁面：01-Authentication
  畫面：01-01_Login, 01-02_OTP-Verify, 01-03_Login-Success
  變體：01-01_Login_Error, 01-01_Login_Loading
  元件：Button/Primary, Card/Product
```

**3b. 建立遷移映射表**

```
## 遷移映射

### 目標結構
Page "01-Authentication"
  ├── 01-01_Login         ← 來源: Page 1 / "login screen"
  ├── 01-01_Login_Error   ← 來源: Page 1 / "login error"
  ├── 01-02_OTP-Verify    ← 來源: Page 1 / "Frame 37"
  └── 01-03_Login-Success ← 來源: Page 2 / "success page"

Page "02-Shopping"
  ├── 02-01_Product-List  ← 來源: Page 3 / "products"
  └── 02-02_Product-Detail ← 來源: Page 3 / "detail view"

Page "Components"
  ├── Button/Primary      ← 來源: Page 5 / "btn-primary"
  └── Card/Product        ← 來源: Page 5 / "product card"

### 排除項目（待確認）
- "old header v2" — 疑似已廢棄，有新版替代
- "test layout" — 疑似草稿
```

**3c. 提交人工審核**

將完整計畫呈現給使用者，確認以下項目：

- [ ] 流程推理是否正確
- [ ] 命名規範是否滿意
- [ ] 遷移映射是否完整（有無遺漏或錯誤歸類）
- [ ] 排除項目是否正確（不該排除的有沒有被排除）
- [ ] 目標檔案：提供 Figma B URL，或確認建立新檔

**等待使用者確認後才進入 Phase 4。不要自行跳過這一步。**

---

### Phase 4：寫入 Figma B（Write to Target）

**前置：** 每次呼叫 `use_figma` 前，必須先載入 `figma-use` skill。

**4a. 建立頁面結構**

使用 Phase 0 確認的 `targetFileKey` 和命名策略：

**獨立目標檔模式：**

```javascript
// use_figma(fileKey=B) — 先載入 figma-use skill
const pageNames = ["01-Authentication", "02-Shopping", "Components"];

for (const name of pageNames) {
  const page = figma.createPage();
  page.name = name;
}
```

**同檔新頁面模式：**

```javascript
// use_figma(fileKey=A) — 先載入 figma-use skill
// ⚠️ 不修改、不刪除任何原有頁面
const pageNames = [
  "[Restructured] 01-Authentication",
  "[Restructured] 02-Shopping",
  "[Restructured] Components"
];

for (const name of pageNames) {
  const page = figma.createPage();
  page.name = name;
}
```

**4b. 逐一遷移畫面**

對遷移映射表中的每個項目，**逐一處理**（`targetFileKey` = Phase 0 決定的目標 fileKey）：

```
For item in migration_map:
    1. 從 Figma A 讀取來源資料
       get_design_context(fileKey=A, nodeId=source.id)
       → 取得結構、顏色、字體、佈局資料

    2. 搜尋可複用的設計系統元件
       search_design_system(query=componentName, fileKey=targetFileKey)
       → 有匹配 → 後續用 importComponentByKeyAsync 引入
       → 無匹配 → 用 Plugin API 手動建立

    3. 在目標位置建立對應結構
       use_figma(fileKey=targetFileKey, code=...) — 載入 figma-use skill
       → 在目標頁面建立 frame
       → 設定尺寸、auto-layout、填充、邊框
       → 建立子元素（text、shape、imported components）
       → 套用新名稱

    4. ⚠️ 清理 context — 只保留 "✅ {newName} 已完成"
```

**4c. 元件處理順序**

```
1. 先遷移 Components 頁面中的獨立元件
2. 將元件建立為 Component node（figma.createComponent()）
3. 在 Screen 頁面中用 component.createInstance() 引用
```

**4d. 分批驗證**

每完成 3-5 個 frame 後，做一次中間驗證：

```
get_screenshot(fileKey=B, nodeId=latestFrame.id)
→ 確認結構正確、命名正確
→ 有問題立即修正
→ ⚠️ 截圖看完即丟
```

---

### Phase 5：驗證（Verify）

**5a. 結構驗證**

用 Plugin API 讀取目標的完整結構，比對遷移計畫：

```javascript
// use_figma(fileKey=targetFileKey) — 讀取結構
const pages = figma.root.children
  // 同檔模式下只檢查 [Restructured] 前綴的頁面
  .filter(p => outputMode === 'separate' || p.name.startsWith('[Restructured]'));

const structure = pages.map(page => ({
  name: page.name,
  children: page.children.map(c => ({ name: c.name, type: c.type }))
}));
console.log(JSON.stringify(structure, null, 2));
```

驗證項目：
- [ ] 頁面數量與計畫一致
- [ ] 每頁的 frame 數量與計畫一致
- [ ] 所有名稱符合命名規範
- [ ] 流程順序正確（frame 排列順序）

**5b. 抽樣視覺比對**

從每個流程中抽 1-2 個畫面做 A vs B 比對：

```
For sample in random_samples:
    get_screenshot(fileKey=A, nodeId=source.id)            → 觀察來源
    get_screenshot(fileKey=targetFileKey, nodeId=target.id) → 觀察目標
    → 記錄文字差異
    → ⚠️ 截圖看完即丟
```

**5c. 輸出最終報告**

```
## 遷移完成報告

**來源：** [Figma A URL]
**目標：** [Figma B URL]

### 統計
- 遷移頁面：N 個
- 遷移畫面：M 個
- 遷移元件：K 個
- 排除項目：J 個

### 命名映射
| 來源位置 | 來源名稱 | 目標位置 | 目標名稱 | 狀態 |
|---|---|---|---|---|
| Page 1 | "login screen" | 01-Authentication | 01-01_Login | ✅ |
| Page 1 | "Frame 37" | 01-Authentication | 01-02_OTP-Verify | ✅ |

### 品質
- 結構驗證：✅ / ❌
- 抽樣視覺比對：N/M 通過
- 已知差異：[列表]

### 需要人工處理
- [無法自動遷移的複雜項目]
- [視覺還原度不足的項目]
```

## Context Management

Figma 設計檔資料量極大。一個中等複雜的設計檔可能有數十個 frame，每個 `get_design_context` 回傳可達數千 token。必須嚴格控制 context window 使用。

### 核心原則：讀一個、摘要、丟掉

```
For each item:
  1. 取得資料（get_design_context / get_screenshot）
  2. 提取需要的資訊，寫成文字摘要
  3. ⚠️ 丟掉原始資料
  4. 繼續下一個
```

### 具體限制

- **不要**同時持有超過 1 個 frame 的 `get_design_context` 回傳
- **截圖**觀察後立即記錄文字描述，不保留圖片在 context 中
- Phase 1 掃描超過 20 個 frame 時，**分批處理**（每批 10 個）
- 遷移映射表用文字格式保留，不保留任何原始資料

### 大型檔案策略（50+ frames）

考慮用 subagent 並行處理：
- 每個 subagent 負責 1-2 個流程的掃描與遷移
- 主 agent 彙整結果
- 每個 subagent 的 prompt 中附上命名規範和遷移映射（該流程的部分）

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| 「先掃完所有截圖再分析比較有效率」 | 同時持有大量截圖會爆 context。逐一處理，摘要後丟掉，才是可持續的做法。 |
| 「先掃一下其他頁面，可能有相關內容」 | 一個 Figma 檔可能有幾十個不相關的頁面。只處理 URL 指定的節點範圍，需要擴大再由使用者提供新 URL。 |
| 「來源命名已經夠清楚，不需要重新命名」 | 重新命名是這個流程的核心價值。統一的命名規範才能確保長期維護性和可搜尋性。 |
| 「這個 frame 看起來像草稿，直接跳過」 | 不要自行決定排除。記錄在分析報告中，Phase 3 由人工確認是否排除。 |
| 「流程順序很明顯，不需要寫推理依據」 | 推理依據是給人驗證的。「很明顯」對你和對使用者可能完全不同。 |
| 「元件太複雜，在 B 裡大概做個樣子就好」 | 能精確就精確。用 `get_design_context` 取得具體數值，用 Plugin API 精確還原。做不到的部分明確標記「需人工處理」。 |
| 「目標檔已經有內容，我直接加在旁邊」 | 先用 `get_metadata` 確認目標檔現有內容。有衝突要在 Phase 3 報告中提出，等人確認。 |
| 「Phase 3 的計畫很完整了，不用等人確認」 | Phase 3 的人工確認是 **mandatory gate**。推理可能有誤，命名可能不符團隊慣例，排除項目可能判斷錯誤。 |
| 「輸出模式很明顯，不用問使用者」 | Phase 0 的輸出模式確認是 **mandatory gate**。agent 不能自行假設要寫到哪裡。 |
| 「同檔模式下原有頁面命名不好，順便改一下」 | 同檔模式只建立新頁面，**絕不修改或刪除原有頁面**。原始內容是使用者的參考基準。 |

## Red Flags

- 沒有在 Phase 0 詢問使用者輸出模式就開始掃描
- 掃描了 URL node-id 指定範圍以外的頁面或節點
- 發現問題但沒有在 Figma 中建立 annotation，只在對話中提到
- 沒有等使用者回覆 annotation 就直接進入 Phase 3
- 自行替使用者回覆未回覆的 annotation（應標記「待確認」）
- Phase 4 完成後沒有詢問使用者是否清理 annotation
- 同檔新頁面模式下修改或刪除了原有頁面
- 同時載入超過 3 個 frame 的 design_context 或截圖
- 沒有建立遷移映射表就直接開始寫入目標位置
- 跳過 Phase 3 的人工審核直接進入 Phase 4
- 推理流程順序時沒有逐條列出推理依據
- 自行決定排除 frame（應標記「待確認」交由人工決定）
- 呼叫 `use_figma` 前沒有載入 `figma-use` skill
- 一次性寫入大量內容而不做分批中間驗證
- 命名沒有遵循 Phase 3 確認的命名規範
- 遷移完成後沒有做結構驗證（Phase 5a）
- Phase 5 驗證發現問題但沒有修正就產出最終報告

## Verification

遷移完成後確認：

- [ ] Phase 0 已詢問並確認輸出模式（獨立目標檔 / 同檔新頁面）
- [ ] Phase 1 掃描範圍正確（只掃描 URL 指定的節點，未擴散到其他頁面）
- [ ] Phase 2 每個 frame 都有分類、內容描述、推理依據
- [ ] Phase 2.5 發現的問題已在 Figma 中建立 annotation frame
- [ ] Phase 2.5 使用者已回覆所有 annotation（或未回覆的已標記「待確認」）
- [ ] Phase 2.5 的回饋已反映在 Phase 3 的遷移計畫中
- [ ] Phase 3 遷移計畫已獲人工確認才開始 Phase 4
- [ ] Phase 4 遷移映射表中所有項目都已處理
- [ ] Phase 4 命名規範一致（無遺漏的重命名）
- [ ] Phase 4 每次呼叫 `use_figma` 前都有載入 `figma-use` skill
- [ ] Phase 5 結構驗證通過（頁面數、frame 數、命名一致性）
- [ ] Phase 5 抽樣視覺比對完成（每個流程至少 1 個畫面）
- [ ] 最終報告包含完整映射表和品質摘要
- [ ] 無法自動處理的項目已清楚標記為「需人工處理」
- [ ] 已詢問使用者是否清理來源頁面上的 annotation frame
