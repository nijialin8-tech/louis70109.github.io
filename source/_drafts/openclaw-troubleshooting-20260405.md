---
title: OpenClaw 部署最佳實踐：從 3.1 邁向 4.2 的完整指南
tags:
  - OpenClaw
  - Ollama
  - Gemini
  - 技術筆記
  - 升級指南
  - Discord
categories: 技術
date: 2026-04-05 20:30:00 +0800
---

![](https://images.pexels.com/photos/230325/pexels-photo-230325.jpeg)

# 前言

在部署 **OpenClaw** 並整合 Ollama Cloud 或第三方轉接層（如 LiteLLM）時，設定檔的精準度直接影響了服務的穩定性。本篇文章彙整了從 2026.3.1 到 2026.4.2 升級過程中的核心問題及其技術對策，提供給需要進行相關架設或升級的開發者參考。

<!-- more -->

---

# 1. Agent 權限層級配置

在 `openclaw.json` 中，全局工具權限（`tools.profile`）並不完全等同於個別 Agent 的權限。為確保工具可用，必須在 Agent 配置層級進行確認。

### 問題描述
全局以開啟 `full` 模式，但在 Discord 或特定環境調用時回報 `Tool not found`。

### 技術解決方案
在 `agents.list` 下的特定 Agent 區塊中，需使用 `alsoAllow` 明確列出需要的敏感權限（如：`exec`、`read`、`image`、`web_search`）。

```json
"list": [
  {
    "id": "personal",
    "tools": {
      "alsoAllow": ["exec", "read", "image", "web_search"]
    }
  }
]
```

---

# 2. 第三方端點的 API 模式適配

使用非原生的 `/v1` 兼容介面時，API 的 schema 處理機制是連線成功的關鍵。

### 問題描述
使用 Ollama Cloud 或代理轉接層時，回報 HTTP 404 Model Not Found 或驗證失敗。

### 技術解決方案
1. **Endpoint 確認**：確保 `baseUrl` 正確（例如 `https://ollama.com/v1`），無重複斜線。
2. **API 模式轉換**：必須將 `api` 欄位由預設改為 **`openai-compatible`**。

```json
"ollama": {
  "baseUrl": "https://ollama.com/v1",
  "api": "openai-compatible"
}
```

---

# 3. Discord Gateway 與頻道特定設定

針對 Discord 平台的部署，必須注意頻道規則與觸發機制，否則 Agent 會出現「已連線但無反應」的現象。

### 技術細節
1. **提到 (Mention) 規則**：在群組頻道中，若 `requireMention` 設為 `true`，Agent 僅在被標註時才會回應。
2. **Guild 選項配置**：
   ```json
   "guilds": {
     "YOUR_GUILD_ID": {
       "requireMention": true,
       "channels": {
         "ALLOWED_CHANNEL_ID": { "allow": true }
       }
     }
   }
   ```
3. **權限回傳**：確保 Discord Bot 在該伺服器擁有足夠的「嵌入連結」與「上傳檔案」權限，否則分析結果（如看圖或搜尋網址）將無法正確顯示。

---

# 4. 解決 Thought Signature 錯誤 (重要)

在 2026.3.2 至 2026.3.7 之間的版本中，Gemini 3 Flash 在執行 `function_calling` 時會觸發傳輸協定衝突。

### 解決方案
1. **關閉推理開關**：模型列表中將 `reasoning` 設為 `false`。
2. **版本回退**：強烈建議使用 **2026.3.1** 穩定版本（若尚未準備好升級 4.2）。

---

# 5. 版本降級 (2026.3.1) 安裝指令

```bash
openclaw gateway stop
pnpm add -g openclaw@2026.3.1
openclaw gateway start
```

---

# 6. 模型 ID 字尾的精確匹配 (:cloud)

使用 Ollama Cloud 時，確保 `openclaw.json` 中的 `id` 與 `primary` 設定完整包含 **`:cloud`** 字尾（註：此規則在 4.2 之後有變動，詳見下方升級章節）。

---

# 🚀 升級章節：OpenClaw 4.2 升級與 Ollama 配置修正 (2026-04-05)

當我們將環境從 `2026.3.1` 升級至 `2026.4.2` 時，發現了模型提供商命名與 ID 引用上的重大變動，以下是本次升級的核心修正內容。

### 1. 提供商名稱統一 (Provider Name)
在 4.2 版本中，為了將雲端 Ollama 與本地實例明確切分，我們將 Ollama Cloud 的訪問權限統一歸類在 `ollama-cloud` 提供商 ID 下。
- **變動**：將原本混合的 `ollama` 配置拆分，確保雲端調用路徑清晰。

### 2. 模型 ID 更新 (Model IDs)
升級後發現原本帶有 `:cloud` 字尾的模型 ID（例如 `gemini-3-flash-preview:cloud`）在新的雲端部署命名規則下失效。
- **修正**：將模型 ID 從 `gemini-3-flash-preview:cloud` 更新為 `gemini-3-flash-preview:latest`。
- **預設配置更新**：同步更新 `openclaw.json` 中的 `defaults.model.primary` 以及各個 Agent 的 `model` 設定為主機路徑 `ollama-cloud/gemini-3-flash-preview:latest`。

### 3. 配置清理與環境優化
為了避免對於沒有運行本地 Ollama 實例的機器造成誤導，我們在 `models.json` 中移出了指向 `127.0.0.1` 的本地 Ollama 配置。
- **驗證**：此修正已在 `writer-assistant` 與 `openclaw-config` 倉庫中驗證通過。

---

# 🍪 2026.4.2 建議配置配方

目前最穩定的升級路徑建議：

- **版本**：`2026.4.2`
- **提供商**：使用 `ollama-cloud`
- **模型**：`gemini-3-flash-preview:latest`
- **API 模式**：維持 `openai-compatible`
- **推理功能**：建議暫時設為 `Reasoning: false` 以確保 Function Calling 穩定性。

---

# 結語：持續優化的 OpenClaw 體驗

不論是維持在穩定的 3.1 版本，還是邁向具備更多新功能的 4.2，正確的配置都是發揮 AI 助手最大效能的關鍵。

整合紀錄於 2026-04-05 By 筆耕餅乾 (Writer Assistant)
