---
name: figma-content-restructure
description: 從 Figma A 掃描、分析並推理畫面結構與流程步驟，重新組織、命名並寫入 Figma B。全程使用 Figma MCP 完成跨檔案的內容遷移與重組。Use when reorganizing Figma files, migrating design content between files, cleaning up messy designs into structured flows, or extracting and documenting user flows from unorganized Figma files.
---

# Figma Content Restructure

## Overview

從使用者提供的 Figma URL 指定的節點（頁面或 frame）出發，只掃描該節點及其子項，分析內容結構、推理使用者流程與步驟順序，然後在目標位置建立重新命名與組織過的乾淨版本。不掃描檔案中的其他頁面。全程透過 Figma MCP 工具操作，不需要瀏覽器或本地開發環境。

**若重組目標包含後續生成 code，這個 skill 的任務不是單純把版面整理得「比較好看」，而是要把 Figma 重組成更適合 Figma MCP 讀取與後續 agent 實作的結構。** 重組後的結果應讓 `get_design_context`、`get_screenshot`、必要時的 `get_context_for_code_connect` 更容易取得單一職責、邊界清楚、命名穩定的節點。

## When to Use

- Figma 檔案內容雜亂，需要重新整理命名與組織結構
- 需要從一個 Figma 檔案提取畫面，推理流程後寫入另一個檔案
- 將草稿/原型檔整理成正式的設計文件
- 從設計稿中推理出使用者流程步驟並建立有序的畫面集
- 跨團隊交接時，需要將設計內容遷移到新檔案並重新命名
- 需要先把 Figma 整理成 codegen-friendly 的 screen/state/reference 結構，再交給後續 design-to-code agent

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

### Figma 容器階層（Pages vs Sections）

Figma 只有兩層「容器」可用來做資料夾語意，必須清楚區分：

| 層級 | 元素 | 能當資料夾嗎？ | 用途 |
|---|---|---|---|
| File sidebar | **Page** | ❌ 扁平結構，無法巢狀 | 大區塊切分（流程群、裝置類、notes） |
| Canvas | **Section**（⇧S） | ✅ 正式群組容器，可巢狀 | 在同一 page 內做資料夾式分組 |
| 視覺輔助 | Page divider（命名 `--- ---`） | ❌ 只是命名慣例，無 node ID | 在 sidebar 視覺分群 |

**預設策略：以 Section 為資料夾主幹。** 原因是 Section 有穩定 node ID，可被 `get_design_context`、`export_nodes`、`batch_get` 直接鎖定；page divider 只是視覺上的分隔線，工具無法引用。

### Codegen-Oriented Goal

若使用者明示或暗示「之後要生成 code」，則重組時**必須**滿足以下目標：

- 每個正式畫面都對應單一、可直接讀取的 frame，不與 arrows、notes、spec blocks 混在一起
- 同一流程中的不同狀態拆成獨立 frame，而不是放在同一個大畫布中靠位置暗示
- 正式 screen、state variant、reference、notes 分頁或分區隔離
- 命名要 machine-readable 且**採 kebab-case**，與後續 code 的資料夾路徑 1:1 對應，避免 mapping table
- 讓後續 agent 可以直接對重組後節點呼叫 `get_design_context`，避免再次在雜亂頁面中做二次清理
- **Section 名稱即 code 路徑**：`flow-a/step-1-login` ↔ `src/flows/flow-a/step-1-login/`

建議的結果應接近這種層級（Section 巢狀 + kebab-case）：

```text
Page "[Restructured] flows"
  Section "flow-a-authentication"
    Section "step-1-login"
      Frame  "screen"
      Frame  "screen--error"       ← state 用 BEM-like 後綴
      Frame  "note"
    Section "step-2-otp-verify"
      Frame  "screen"
      Frame  "note"

Page "[Restructured] reference"
  Section "device-tablet"
    Frame  "flow-a-login"
  Section "device-mobile"
    Frame  "flow-a-login"

Page "[Restructured] notes"
  Section "decisions"
    Frame  "flow-a-summary"
```

**何時仍用 Page 切分：** 單一流程內容超過一個 canvas 可負擔的資訊量、或流程彼此完全不共享元件時。預設是單 page + Section 巢狀；僅在量級失控時才升級成多 page。

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
Phase 5: 驗證 ──→ 結構比對 + 抽樣視覺比對 ──→ 初版最終報告
    │
    ▼
Phase 6: 對抗式 review ──→ 透過 Codex 獨立質疑 ──→ 處理紅旗 ──→ 定稿最終報告
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
1.5. Figma prototype actions / interactions（如果存在，優先視為互動事實來源）
2. Frame 名稱中的序號或步驟標記（Step 1, 2, 3…）
3. 頁面內 frame 的空間位置（左→右 = 先→後）
4. 內容邏輯（登入→首頁→詳情→結帳…）
5. UI 狀態關係（預設→填寫中→完成→錯誤）
```

**每條推理必須寫明依據。** 例如：「Step 2 在 Step 1 右邊且內容為 OTP 輸入，推斷為登入後的驗證步驟」。

#### 關於 action / prototype connection 的判斷原則

- **可以不依賴 action 先做高可信推理。** 若畫面本身已有明確訊號（stepper、畫布排列、命名、內容狀態差異、規格補充），可先建立流程與狀態推理。
- **但沒有 action 時，不要把推理誤寫成互動事實。** 例如「按這個按鈕一定會開 modal」或「此按鈕一定跳到某頁」這種互動級結論，若沒有 prototype/action 或明確註解，只能標記為推測。
- **對後續 code generation 而言，action 是加分但不是必要條件。** 沒有 action 時，應優先產出「畫面級」與「狀態級」的乾淨結構、穩定命名與明確 notes，讓後續 agent 或工程師能補互動邏輯。
- **需要精準互動規格時，主動提示風險。** 尤其是以下類型若沒有 action，應在分析報告或 annotation 中明示「互動未確認」：
  - 按鈕跳轉目標
  - modal / drawer 開關
  - back / next 的返回規則
  - 條件分支流程
  - 同頁內的 tab / accordion / progressive reveal 狀態切換
- **若重組目標是之後要生成 code，必須整理成 code-friendly 結構。** 這是確定要求，不是偏好。即使沒有 action，也必須把正式 screen、state variant、reference、notes 分離，並用 machine-readable 命名表達狀態差異。

**2c. 命名分析**

檢查現有命名的問題：
- 命名不一致（混合中英文、駝峰/底線混用）
- 無意義名稱（Frame 1, Group 3, Rectangle 12）
- 缺少分類前綴或流程標記
- 重複或衝突的名稱

**2d. Codegen readiness 分析**

除了「流程是否合理」，還要判斷「這份設計能不能直接拿來生成 code」。至少檢查：

| 類型 | 風險 |
|---|---|
| 大型畫布混雜多個正式畫面 | `get_design_context` 會回傳過大且邊界不清 |
| 正式畫面與註解/箭頭/規格框混在同層 | 後續 agent 會誤讀為 UI 結構 |
| 一個 frame 同時承載多個狀態 | 無法穩定映射成 component state / route state |
| 命名只反映視覺位置，不反映功能 | 難以映射到 route / screen / state 名稱 |
| 正式 desktop 與 RWD/reference 混在一起 | 後續 agent 難以判斷哪個是 source of truth |

若發現上述問題，Phase 3 與 Phase 4 必須以「可直接生成 code」為目標來拆解與重組。

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

**預設（codegen 模式）：kebab-case + Section 階層直接對應 code 路徑**

```
建議命名格式：
  頁面：[Restructured] {kind}            ← kind ∈ flows | reference | notes
  Section（資料夾）：{flow-name} / {step-name}
  畫面 frame：{screen}[--{state}]        ← state 用 BEM-like 後綴
  note frame：note 或 note--{topic}

範例：
  頁面：[Restructured] flows
    Section "flow-a-authentication"
      Section "step-1-login"
        Frame "screen"
        Frame "screen--error"
        Frame "screen--loading"
        Frame "note"
      Section "step-2-otp-verify"
        Frame "screen"
```

**所有名稱必須 kebab-case**，因為它會直接成為後續 code 的路徑或檔名。

**非 codegen 模式（純設計整理）可退回傳統命名：**

```
頁面：01-Authentication
畫面：01-01_Login, 01-02_OTP-Verify
變體：01-01_Login_Error
```

在 Phase 0 之後、Phase 3 提交審核前，確認使用者選擇哪種命名規範；codegen 為預設。

**如果目標是後續生成 code，命名規範必須額外滿足：**

- 所有名稱採 kebab-case，與 code 資料夾/檔名 1:1 對應
- state 名稱直接寫入 frame 名稱（`screen--error`），不依賴空間位置暗示
- 避免 `Desktop_Header`、`Frame 37`、`final-final-v2` 這種無功能語義的名稱
- 裝置型 reference 必須標明 `device-desktop` / `device-tablet` / `device-mobile`
- 補充說明必須放在 `[Restructured] notes` 頁面的 Section 中，不得混入正式 screen

範例：

```text
flow-a-checkout/step-3-payment/screen
flow-a-checkout/step-3-payment/screen--error
flow-a-checkout/step-3-payment/screen--success
device-tablet/flow-a-checkout
notes/flow-a-decisions
```

**3b. 建立遷移映射表**

```
## 遷移映射（codegen 預設格式）

### 目標結構
Page "[Restructured] flows"
  Section "flow-a-authentication"
    Section "step-1-login"
      Frame "screen"         ← 來源: Page 1 / "login screen"
      Frame "screen--error"  ← 來源: Page 1 / "login error"
      Frame "note"           ← 來源: Phase 2.5 annotation 結論
    Section "step-2-otp-verify"
      Frame "screen"         ← 來源: Page 1 / "Frame 37"
    Section "step-3-success"
      Frame "screen"         ← 來源: Page 2 / "success page"
  Section "flow-b-shopping"
    Section "step-1-product-list"
      Frame "screen"         ← 來源: Page 3 / "products"
    Section "step-2-product-detail"
      Frame "screen"         ← 來源: Page 3 / "detail view"

Page "[Restructured] components"
  Section "button"
    Frame "primary"          ← 來源: Page 5 / "btn-primary"
  Section "card"
    Frame "product"          ← 來源: Page 5 / "product card"

### 排除項目（待確認）
- "old header v2" — 疑似已廢棄，有新版替代
- "test layout" — 疑似草稿
```

**若目標是 codegen，遷移映射表還必須明確標示節點角色：**

| 角色 | 定義 | 放置位置 |
|---|---|---|
| `screen` | 正式畫面來源，後續可直接拿去 `get_design_context` | `[Restructured] flows` 內的 step Section |
| `state` | 同一 screen 的狀態變體（error/loading/expanded） | 與 screen 同 Section，命名 `screen--{state}` |
| `reference` | 裝置參考、補充示意、非正式主要來源 | `[Restructured] reference` |
| `notes` | 決策、規格、annotation 回覆摘要 | `[Restructured] notes` |
| `exclude` | 不進入重組結果的內容 | — |

範例（角色標示 + Section 階層）：

```text
Page "[Restructured] flows"
  Section "flow-a-authentication"
    Section "step-1-login"
      screen  frame "screen"          ← 來源: "login screen"
      state   frame "screen--error"   ← 來源: "login error"
    Section "step-2-otp-verify"
      screen  frame "screen"          ← 來源: "Frame 37"

Page "[Restructured] reference"
  Section "device-tablet"
    reference frame "flow-a-login"    ← 來源: "tablet login"

Page "[Restructured] notes"
  Section "flow-a-decisions"
    notes frame "summary"             ← 來源: annotations / spec blocks
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

**4a. 建立頁面 + Section 階層**

使用 Phase 0 確認的 `targetFileKey` 和命名策略。**Codegen 預設結構：少量 page + 巢狀 Section 當資料夾**。

**獨立目標檔模式（codegen 預設）：**

```javascript
// use_figma(fileKey=B) — 先載入 figma-use skill
// 1. 建立三個頂層 page：flows / reference / notes
const pages = {};
for (const kind of ["flows", "reference", "notes"]) {
  const p = figma.createPage();
  p.name = `[Restructured] ${kind}`;
  pages[kind] = p;
}

// 2. 在 flows page 中建立 Section 階層
await figma.setCurrentPageAsync(pages.flows);

function createSection(parent, name, x = 0, y = 0, w = 2000, h = 1500) {
  const s = figma.createSection();
  s.name = name;                        // kebab-case，對應 code 路徑
  s.resizeWithoutConstraints(w, h);
  s.x = x; s.y = y;
  if (parent && parent.type === 'SECTION') parent.appendChild(s);
  return s;
}

// 範例：flow-a-authentication / step-1-login
const flowA = createSection(null, 'flow-a-authentication', 0, 0, 6000, 3000);
const step1 = createSection(flowA, 'step-1-login', 100, 100, 2800, 1400);
const step2 = createSection(flowA, 'step-2-otp-verify', 3000, 100, 2800, 1400);
// 後續 4b 會把 screen/state frame 塞入對應 Section
```

**若目標是 codegen，結構策略：**

1. `flows` page 內以 Section 表達「流程 → 步驟」兩層資料夾；screen 與 state frame 放在最內層 Section
2. `reference` page 內以 Section 依 `device-{desktop|tablet|mobile}` 分組
3. `notes` page 內以 Section 依流程或主題分組
4. **不要**在 `flows` 頁面保留 arrows、浮動 spec box、巨大章節標題、示意背景文字
5. Section 名稱必須可直接當作 code 路徑（kebab-case、無空白、無特殊字元）

這樣做的理由是：後續 agent 可以直接對 Section node ID 呼叫 `get_design_context`（取整個 step 的設計脈絡）或對內層 frame 取單一 screen，不需要二次清理。

**同檔新頁面模式：**

```javascript
// use_figma(fileKey=A) — 先載入 figma-use skill
// ⚠️ 不修改、不刪除任何原有頁面
for (const kind of ["flows", "reference", "notes"]) {
  const p = figma.createPage();
  p.name = `[Restructured] ${kind}`;
}
// Section 建立邏輯同上
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

**Codegen-friendly 寫入規則：**

- 一個 frame 只表達一個 screen 或 state
- **Frame 必須 `appendChild` 進對應的 Section**（資料夾），不要散落在 Section 外
- 若來源是超大畫布中的局部內容，遷移後要建立獨立 frame，不要把整張畫布原封不動搬過去
- 若一個流程步驟有多個漸進狀態，用 BEM-like 後綴：`screen--expanded`、`screen--selected`、`screen--error`
- 正式 frame 在 Section 內保持一致排列（通常左到右），避免交錯
- 保留實作需要的元件與層級，但刪除與 code 無關的展示性包裝
- 若 annotation 已形成決策，將決策摘要寫進 `notes` 頁對應 Section，而不是散落在正式畫面旁

**重要：不要把「大畫布展示結構」當成最終輸出。**

在原始設計中，為了展示流程，常見做法是把很多畫面、箭頭、規格框放在一個超大型 frame/page。這種結構通常**不適合**後續 code generation。重組後應改成「每個正式畫面都是獨立、可讀、可命名、可直接取 design context 的節點」。

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
- [ ] 每頁的 Section / frame 數量與計畫一致
- [ ] 所有名稱符合命名規範（codegen 模式為 kebab-case）
- [ ] Section 階層正確（flow → step → screen/state），無孤兒 frame 在 Section 外
- [ ] 流程順序正確（Section 排列順序）
- [ ] `flows` 頁面中沒有混入 notes / arrows / spec blocks
- [ ] 正式 screen/state frame 可單獨作為 `get_design_context` 的輸入
- [ ] Section 也可作為 `get_design_context` 的輸入（取整個 step 的脈絡）
- [ ] `reference` 與 `notes` 已與正式 screen 隔離

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

> 這是**初版最終報告**。Phase 6 對抗式 review 之後才算定稿。

---

### Phase 6：對抗式 Review（Adversarial Review via Codex）

**目標：** 讓一個獨立、與主 agent 沒有共享 context 的 reviewer（Codex）質疑重組決策。主 agent 對自己的推理容易產生合理化偏誤；對抗式 review 的價值在於找出流程誤判、錯誤分類、遺漏畫面、命名不一致等主 agent 自己看不到的問題。

**這不是「更嚴格的 implementation defect review」**，而是「重組方向對不對」的質疑：
- 流程分支是否正確？
- 分類（screen/state/reference/notes/exclude）是否有誤判？
- 命名是否真的能 1:1 映射到 code 路徑？
- 被排除的元素是否其實藏著流程線索？
- 對抗式 review 的預設立場是「你錯了，證明我是對的」，不是「check 一下對不對」。

**Prerequisite：** 需要 `codex-plugin-cc` 已安裝且認證（可透過 `/codex:setup` 檢查）。

#### 6a. 組裝 review package

把整個重組結果壓縮成一份 ≤ 2000 字的「review package」文字。**不要**夾帶截圖或 design context 原始回傳（Codex 無法讀 Figma），全部以文字呈現。

必要欄位：

| 區塊 | 內容 |
|---|---|
| **Source** | fileKey、nodeId、來源 page 名稱、頂層 child 數、掃描日期 |
| **Target** | 同檔新頁面 / 獨立目標檔、`[Restructured]` 頁面清單 |
| **Output mode** | codegen kebab-case / design-only |
| **Final structure tree** | Phase 5a 輸出（壓縮到 ≤ 100 行；保留 page → section → frame/popup 三層） |
| **Flow inference** | 每條 flow 的 step 數、跳過邏輯、推理依據（≤ 10 行） |
| **Phase 2.5 Q&A** | 每個 annotation 的問題 + 使用者回覆重點（≤ 10 行） |
| **Classification decisions** | screen / state / reference / notes / exclude 的統計與分類規則 |
| **Exclusions** | 排除項目分類統計（例如 "8 arrows, 10 rectangles, 24 spec blocks"）+ 排除理由 |
| **Known gaps** | 初版報告中標示「需人工處理」的項目 |
| **Codegen readiness self-check** | Phase 5 驗證清單的 pass/fail 狀態 |

#### 6b. 呼叫 Codex adversarial review

**首選：** 使用 `/codex:rescue` slash command（foreground / `--wait`），prompt 內容用下方模板。

**備選：** 直接用 `Agent` tool 呼叫 codex 子代理：
```
Agent({
  subagent_type: "codex:codex-rescue",
  description: "Adversarial review of figma restructure",
  prompt: "<the review prompt, see template below>"
})
```

**不使用 `/codex:adversarial-review`**：那個命令是 git-diff 導向的，此 skill 沒有 code diff。

**Prompt 模板：**

```
You are an adversarial reviewer of a Figma content restructure.
Your job is to find flaws, not to agree.
Challenge the restructure direction. Default stance: "something is wrong here — prove otherwise."

## What was done
- Source: Figma {fileKey}, nodeId {nodeId}
  Source page: "{sourceName}" ({N} top-level children)
- Target: {same-file new pages | separate file {fileKey2}}
- Output mode: {codegen-kebab | design-only}

## Final structure (target, from Phase 5a)
{structure tree — ≤ 100 lines, indented, showing page → section → frame/popup}

## Flow inference
{each flow with step count, skip logic, reasoning basis}

## Phase 2.5 annotations — user Q&A
{each annotation issue + user reply bullet}

## Classification decisions
{screen / state / reference / notes / exclude counts and rules}

## Exclusions
{count by category + reasoning, e.g., "8 arrows: decorative flow pointers replaced by Section order"}

## Known gaps (already declared)
{items labeled 需人工處理}

## Your task

Challenge the above with 8 angles:

1. **Flow inference** — could the flow branching be wrong?
   Any step mis-assigned to a mode that actually skips it, or missing from a mode that needs it?

2. **Frame classification** — any screen / state / reference / notes / exclude mislabel?
   Especially: could any "reference" or "notes" actually be a primary screen?
   Could any "screen" actually be a state variant of another screen?

3. **Naming** — is the kebab-case path really 1:1 with code?
   Are state suffixes consistent across modes (e.g., --state-2 vs --expanded vs --error)?
   Any name that a code reviewer would reject?

4. **Exclusions** — could any excluded arrow / rectangle / text note carry flow info now lost?
   Were "無此步驟" markers or "此處跳轉" notes actually flow metadata that should be in notes/?

5. **Codegen readiness** — if someone runs get_design_context on every final Section,
   would each return a single-responsibility result?
   Any Section that mixes unrelated screens / popups / references?

6. **Consistency** — same-purpose screens across modes — do they have parallel names and structure?
   If mode-a has step-3-question-range with 5 sub-steps, does mode-b (if reusing step-3) match?

7. **Section depth** — did nesting exceed 2 levels (flow → step → frame)?
   Any sub-section that should be flattened or promoted?

8. **Row-4 / misc zone** — did variants get split correctly between primary flow (as states of a step)
   and misc-variants (as separate variant tracks)? Any variant misclassified?

## Output format

Return a structured adversarial report:

### 🚩 RED FLAGS (must-fix before declaring done)
For each: cite specific frame/section name or ID, explain what's wrong, propose concrete fix.

### ⚠️ WARNINGS (should reconsider)
Inconsistencies, risky shortcuts, or edge-case gaps.

### 💡 SUGGESTIONS (optional polish)
Non-blocking improvements to naming or organization.

### ✅ PASSED (explicit OK)
What you checked and found acceptable.

Be concrete. Cite specific frame/section names or node IDs. No generic advice.
If you find nothing wrong, explicitly say so — but only after probing all 8 angles.
```

#### 6c. 處理 Codex 回饋

Codex 回傳後（原樣呈現給使用者），依據嚴重度分流：

| 嚴重度 | 處理方式 |
|---|---|
| 🚩 **Red flag（必修）** | 回到 Phase 3 或 Phase 4 做局部修正（不重做整個 migration）。修正後再跑一輪 Phase 5 結構驗證。修正完可再次呼叫 Codex 複驗（視嚴重度決定）。 |
| ⚠️ **Warning（建議）** | 與使用者確認要處理或忽略。處理的做為小幅修正；忽略的記入「已知權衡」清單。 |
| 💡 **Suggestion（選配）** | 附在最終報告的「後續改善」段落，不阻斷完成。 |
| ✅ **Pass** | 在最終報告加上 `adversarial review: passed`。 |

**重要：** 絕不自行替 Codex 說「看起來 OK」或過度合理化紅旗。使用者是最終決策者，但必須清楚看到 Codex 原文質疑，才能做決定。

#### 6d. 輸出定稿最終報告

在 Phase 5 初版報告之後，追加：

```
## Adversarial Review（Codex）

**Status:** passed / fixed / partial（n 項未處理）/ skipped（Codex 不可用）

### 紅旗處理紀錄
| 項目 | 修正方式 | 確認 |
|---|---|---|
| ... | ... | ✅ / ❌ |

### 未處理項目（已知權衡）
- ...

### Codex 原文
<逐字附上 Codex 回傳內容，不要改寫>
```

#### 6e. 跳過情境

以下情況允許跳過 Phase 6，但必須明示：

- **Codex 未安裝 / 未認證：** 在最終報告寫 `Adversarial review: skipped — Codex plugin unavailable`，建議使用者 `/codex:setup` 後再跑。
- **使用者明確要求跳過：** 在最終報告寫 `Adversarial review: skipped by user request`，不取代成假的 pass。
- **結構極小（< 5 個 frame）：** 可選擇跳過，但在報告說明規模與跳過理由。

**不允許跳過的情況：** 使用者沒說跳過、Codex 可用、規模超過 5 frame — 必須執行。

---

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

### Codegen-Friendly Heuristics

若重組結果是為了之後生成 code，優先遵守以下啟發式：

- **Prefer screen boundary over visual grouping**  
  重視「哪個 node 可以成為單一畫面來源」，高於「原始畫布怎麼排版展示」

- **Prefer state decomposition over mega-canvas preservation**  
  把狀態拆開，比保留巨型流程總覽更重要

- **Prefer explicit names over positional meaning**  
  用命名說明語義，不要要求後續 agent 靠位置猜

- **Prefer notes isolation over inline annotations**  
  將決策與規格集中到 `Notes`，不要貼在正式 UI 旁邊

- **Prefer a canonical source of truth per implementation target**  
  同類畫面若有多份示意，必須明確指定哪一份是後續實作的正式來源

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
| 「沒有 action 就不能整理流程」 | 多數情況仍可依畫面訊號做高可信重組；只是互動級結論要明確區分為「已確認」或「推測」。 |
| 「沒有 action 也能 100% 確認所有按鈕行為」 | 不行。沒有 prototype/action 或文字註記時，只能精準整理 screen/state，不能假裝掌握完整互動規格。 |
| 「把原始大畫布整塊搬過去就比較完整」 | 對 codegen 來說通常更差。應拆成邊界清楚、單一職責的 frame，讓 MCP 工具可直接讀取。 |
| 「註解留在正式畫面旁邊比較有脈絡」 | 對人可能方便，對後續 agent 容易誤判。應移到 `Notes` 頁或獨立區。 |
| 「Desktop/Tablet/Mobile 放一起比較好比較」 | 對展示可能合理，但對 codegen 應明確區分 primary source 與 reference。 |
| 「元件太複雜，在 B 裡大概做個樣子就好」 | 能精確就精確。用 `get_design_context` 取得具體數值，用 Plugin API 精確還原。做不到的部分明確標記「需人工處理」。 |
| 「目標檔已經有內容，我直接加在旁邊」 | 先用 `get_metadata` 確認目標檔現有內容。有衝突要在 Phase 3 報告中提出，等人確認。 |
| 「Phase 3 的計畫很完整了，不用等人確認」 | Phase 3 的人工確認是 **mandatory gate**。推理可能有誤，命名可能不符團隊慣例，排除項目可能判斷錯誤。 |
| 「輸出模式很明顯，不用問使用者」 | Phase 0 的輸出模式確認是 **mandatory gate**。agent 不能自行假設要寫到哪裡。 |
| 「同檔模式下原有頁面命名不好，順便改一下」 | 同檔模式只建立新頁面，**絕不修改或刪除原有頁面**。原始內容是使用者的參考基準。 |
| 「Phase 5 都通過了，Phase 6 只是多餘」 | Phase 5 是**主 agent 對自己工作的自我驗證**，無法抓主 agent 沒意識到的合理化偏誤。Phase 6 是獨立 reviewer — 這兩件事本質不同，不能互相替代。 |
| 「Codex 的建議看起來不太對，我覺得原本設計就好」 | Codex 的原文必須逐字呈現給使用者；不要替 Codex 軟化或替使用者過濾紅旗。判斷權在使用者，不在主 agent。 |
| 「Codex 沒 install 就算了，品質差不多」 | 明確在最終報告寫 `skipped — Codex unavailable` 並建議使用者 `/codex:setup`。不要裝作 Phase 6 過了。 |

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
- 明知後續要生成 code，卻仍把正式畫面、reference、notes 混在同一頁
- 沒有把可作為實作來源的正式 frame 拆成獨立節點
- 保留大量 arrows/spec blocks 在正式 `flows` 頁面，導致後續 `get_design_context` 難以使用
- Codegen 模式下使用 PascalCase / snake_case / 中文 / 含空白的命名（應為 kebab-case）
- 用 page divider（`--- ---` 命名）當資料夾，而非 Section（divider 無 node ID，工具無法鎖定）
- Frame 直接建在 page root，未 `appendChild` 進對應的 Section
- Section 階層超過三層（flow → step → ...），導致路徑過深難對應 code
- Phase 5 通過後就直接宣稱完成，未執行 Phase 6 對抗式 review（且 Codex 可用、規模超過 5 frame）
- Phase 6 的 Codex 回覆被主 agent 摘要、重寫或過濾，而非逐字呈現給使用者
- 對 Codex 紅旗自行合理化為「看起來沒關係」而跳過修正
- Codex 未安裝時假裝 Phase 6 通過，而非明確標示 `skipped`
- 使用 `/codex:adversarial-review` 來做這個 skill 的 review（該命令是 git-diff 導向，不適用）

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
- [ ] 若目標是 codegen，正式實作來源已集中在 `[Restructured] flows` 頁面
- [ ] 若目標是 codegen，所有正式 screen/state 都可單獨命名並直接取得 design context
- [ ] 若目標是 codegen，`reference` 與 `notes` 沒有污染正式實作來源
- [ ] 若目標是 codegen，所有命名為 kebab-case，Section 階層為 `flow-* → step-*`
- [ ] 若目標是 codegen，Section 名稱可直接當作 code 資料夾路徑（無需 mapping table）
- [ ] Phase 6 對抗式 review 已執行（或明確標示 skipped 並說明理由）
- [ ] Phase 6 review package 包含：structure tree / flow inference / Phase 2.5 Q&A / exclusions / known gaps
- [ ] Phase 6 呼叫的是 `/codex:rescue` 或 `codex:codex-rescue` 子代理，**不是** `/codex:adversarial-review`
- [ ] Phase 6 Codex 原文已逐字呈現給使用者，未被主 agent 摘要或過濾
- [ ] Phase 6 紅旗已修正或已與使用者確認權衡
- [ ] 定稿最終報告包含 `Adversarial Review` 區塊（status + 紅旗處理紀錄 + Codex 原文）
