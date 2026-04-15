---
description: 從 Figma A 分析內容並重組到目標位置 — 掃描、推理流程、重新命名、寫入目標檔或同檔新頁面
---

Invoke the agent-skills:figma-content-restructure skill.

從使用者提供的 Figma 來源檔 URL 開始，掃描所有畫面與元件，分析並推理流程步驟，規劃重組方案（人工確認），然後寫入目標位置。

全程使用 Figma MCP 工具，不需要瀏覽器或本地開發環境。

## Input

使用者需提供：
- **Figma A URL**（來源檔）— 必要

流程啟動後（Phase 0）會詢問使用者輸出模式：
- **獨立目標檔** — 提供 Figma B URL，寫入另一個檔案
- **同檔新頁面** — 在來源檔中建立 `[Restructured]` 前綴的新頁面

## Human Touchpoints

1. **Phase 0**：確認輸出模式（獨立目標檔 or 同檔新頁面）
2. **Phase 3**：遷移計畫確認（命名規範、流程推理、映射表、排除項目）
3. **Phase 5**：最終報告確認
