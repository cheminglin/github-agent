# GitHub agent

一個示範以 Model Context Protocol (MCP) 為核心理念的代理（agent）儲存庫，README 針對 MCP 的背景、架構、能力、傳輸、實作要點與安全模型進行精要又完整的說明，協助你在 IDE、桌面應用或伺服器環境中快速對接 MCP。

---

## 目錄導覽
- [什麼是 Model Context Protocol (MCP)](#什麼是-model-context-protocol-mcp)
- [核心概念與能力模型](#核心概念與能力模型)
- [連線生命週期與握手（Handshake）](#連線生命週期與握手handshake)
- [傳輸層（Transports）](#傳輸層transports)
- [訊息與結構化模式](#訊息與結構化模式)
- [錯誤處理](#錯誤處理)
- [安全與信任邊界](#安全與信任邊界)
- [與 Agent/IDE 整合的實務建議](#與-agentide-整合的實務建議)
- [官方規格章節連結](#官方規格章節連結)
- [參考與延伸閱讀](#參考與延伸閱讀)

---

## 什麼是 Model Context Protocol (MCP)

MCP 是一個開放協議，用來標準化 LLM 應用如何向模型提供上下文與能力。你可以把 MCP 想像成「AI 應用的 USB-C」：
- **Host（主機）**：承載模型互動的應用（例如 IDE 外掛、桌面應用、聊天應用）。
- **Client（客戶端）**：主機內部與伺服器建立 1:1 連線的端點。
- **Server（伺服器）**：向客戶端暴露資料資源、工具與提示，讓模型在安全可控的前提下取得與操作外部世界。

MCP 基於 JSON-RPC 2.0 設計抽象的「能力」層，並支援多種傳輸方式（如 stdio 與 WebSocket），以適應不同的執行環境與隔離需求。

---

## 核心概念與能力模型

MCP 定義了數個一等公民（first-class）能力，主機與伺服器在初始化協商後，客戶端即可在安全邊界內呼叫：

- **Resources（資源）**：
  - 伺服器可宣告可讀的資料資源（例如：檔案、資料庫查詢、HTTP 端點聚合結果）。
  - 支援列舉（list）、讀取（read）、訂閱更新（watch/stream）等模式。
- **Tools（工具）**：
  - 可執行的函式/動作，由伺服器實作並經主機控管。
  - 參數採結構化模式（通常使用 JSON Schema），可定義同步或非同步回應。
  - 主機需在執行前落實用戶可見與可控（例如：顯示即將呼叫的工具與參數）。
- **Prompts（提示）**：
  - 伺服器可對外提供可參數化的提示模板，供主機/模型重複引用以提升一致性。
- **Sampling（取樣）/ Model Invocation（模型呼叫）橋接**：
  - MCP 將「工具/資源提供」與「模型取樣」分離，主機可自由接入模型供應商。

---

## 連線生命週期與握手（Handshake）

典型流程（以 JSON-RPC 為基礎）：
1. **啟動與握手**：
   - 客戶端呼叫 `initialize` 交換協議版本、支援的能力與伺服器資訊。
   - 雙方可透過 capability flags 宣告支援的模組（resources、tools、prompts 等）。
2. **能力發現（Discovery）**：
   - 客戶端可列舉資源、工具與提示，取得其結構化描述與 schema。
3. **操作期（Operational Phase）**：
   - 讀取資源、呼叫工具、渲染提示、訂閱更新流。
4. **關閉（Shutdown/Exit）**：
   - 優雅地釋放連線與背景任務。

---

## 傳輸層（Transports）

MCP 將「協議語義」與「傳輸實作」解耦，常見選項：
- **stdio**：
  - 以標準輸入/輸出承載 JSON-RPC，適合本機隔離與 sandboxed 啟動（如 CLI、子行程）。
- **WebSocket**：
  - 適用長連線、雙向通訊與雲端服務整合。
- （實務上亦可見）Server-Sent Events、HTTP/2、Named Pipes 等封裝，只要維持 JSON-RPC 語義。

---

## 訊息與結構化模式

- **JSON-RPC 2.0**：請求/回應/通知語意清晰，可追蹤 `id` 與錯誤碼。
- **結構化參數與回傳**：工具與資源使用明確 schema，利於驗證與 UI 呈現。
- **串流（Streaming）**：用於大型內容或長耗時操作的增量傳輸。

---

## 錯誤處理

- 沿用 JSON-RPC 標準錯誤碼，並擴展領域錯誤（如資源不存在、驗證失敗、權限不足）。
- 工具執行錯誤須具備可觀測性（message、code、data 以利除錯與審計）。

---

## 安全與信任邊界

- **最小特權**：伺服器只暴露必要資源/工具；主機僅在用戶同意下調用。
- **顯式同意**：主機在呼叫工具前需讓用戶可見與可取消。
- **資料外洩保護**：主機應在將資源內容提供給模型前提示與控管範圍。
- **隔離/沙箱**：透過傳輸（如 stdio 子行程）與容器隔離降低風險。
- **審計**：保留工具呼叫與資源讀取的審計日誌。

---

## 與 Agent/IDE 整合的實務建議

- 建立清楚的工具邊界與輸入驗證，避免任意指令注入。
- 將「資源」抽象為可列舉/可訂閱，利於在 IDE 側做檔案樹與資料視圖。
- 以 schema-first 設計工具參數，並為每個工具提供簡短說明與範例 payload。
- 對長流程動作使用串流回饋（progress events / incremental logs）。

---

## 官方規格章節連結
- 總覽（Overview）：`https://modelcontextprotocol.io`
- 具體規格（Specification）：`https://model-context-protocol.github.io/specification/`
- 概念與架構（Concepts / Architecture）：`https://modelcontextprotocol.io/concepts`
- 能力與模組（Capabilities: resources/tools/prompts）：`https://modelcontextprotocol.io/spec#capabilities`
- 連線生命週期（Lifecycle / Initialize）：`https://modelcontextprotocol.io/spec#initialize`
- 傳輸（Transports: stdio/WebSocket）：`https://modelcontextprotocol.io/spec#transports`
- 工具（Tools API）：`https://modelcontextprotocol.io/spec#tools`
- 資源（Resources API）：`https://modelcontextprotocol.io/spec#resources`
- 提示（Prompts API）：`https://modelcontextprotocol.io/spec#prompts`
- 錯誤碼（Error Codes）：`https://modelcontextprotocol.io/spec#errors`
- 安全考量（Security Considerations）：`https://modelcontextprotocol.io/spec#security`

> 以上連結可能因規格重構而調整，請以官方站點目錄為準。

---

## 參考與延伸閱讀

- 規範（英文）：`https://modelcontextprotocol.io`（或官方規格站 `https://model-context-protocol.github.io/specification/`）
- GitHub 參考實作與範例（可能持續演進）：
  - `https://github.com/modelcontextprotocol`（組織與範例倉庫）
- 社群與文章：
  - `https://news.ycombinator.com/from?site=modelcontextprotocol.io`

> 註：MCP 生態正快速演進，建議以官方規格站為最終依據，並留意版本相容性與破壞性變更公告。

