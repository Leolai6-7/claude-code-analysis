# Chapter 1: 什麼是 Coding Agent

> 從自動補全到自主工程師——理解 Claude Code 的定位，以及為什麼它的源碼值得你讀。

---

## 1.1 從 Copilot 到 Agent：三代演進

### 第一代：自動補全（2021-2022）

GitHub Copilot 開啟了 AI 寫程式的時代。它的模型很簡單：

```
[你打的程式碼上下文] → [模型預測下一行]
```

**特徵**：
- 被動觸發（你打字，它補全）
- 沒有工具使用能力
- 沒有記憶
- 作用範圍：一個檔案內的幾行程式碼

### 第二代：對話式助手（2023-2024）

ChatGPT、Claude.ai 讓你可以「聊」程式問題。

```
[你的問題] → [模型回答，可能包含程式碼]
```

**特徵**：
- 主動提問
- 可以解釋概念
- 仍然沒有工具——不能讀你的檔案、不能跑命令
- 作用範圍：一段對話

### 第三代：Coding Agent（2024-2025）

Claude Code 代表的是第三代——不只是回答問題，而是**自主完成工程任務**。

```
[你的目標] → [Agent 規劃] → [讀檔案] → [執行命令] → [修改程式碼]
                    ↑                                    │
                    └────────── 觀察結果，決定下一步 ─────┘
```

**特徵**：
- 擁有 43 個工具（讀檔案、寫檔案、執行命令、搜尋、網路...）
- 迴圈執行：觀察 → 思考 → 行動 → 觀察
- 有記憶系統（跨 session 記住你的偏好）
- 有權限系統（知道什麼能做、什麼要問你）
- 可以生成子 Agent 並行工作
- 作用範圍：整個程式碼庫

---

## 1.2 Claude Code 的規模

先給你一個直觀的數字感受。

### 源碼規模

| 指標 | 數值 |
|------|------|
| TypeScript 源檔 | 1,902 個 |
| 總行數 | 512,685 行 |
| 內建工具 | 43 個 |
| Slash 命令 | 90+ 個 |
| Hook 事件類型 | 27 種 |
| MCP Transport 類型 | 8 種 |
| 支援平台 | 6 個 (macOS/Linux/Windows × arm64/x64) |

### 發布形態

Claude Code 以兩種形式發布：

**NPM 包**（`@anthropic-ai/claude-code`）：
```
cli.js              13MB    Bun-bundled 單一檔案，16,911 行 minified
sdk-tools.d.ts      117KB   SDK 公開型別
vendor/ripgrep/     ~5MB    內嵌 ripgrep（6 平台）
vendor/audio-capture/ ~3MB  內嵌語音擷取（6 平台）
```

**Homebrew Cask**：193MB Mach-O 原生二進位（Node.js SEA 打包）。

有趣的是，`package.json` 的 `dependencies: {}` 是空的——所有依賴在 build time 被 Bun bundler 打包進單一 `cli.js`。

### 技術棧

| 技術 | 用途 |
|------|------|
| TypeScript | 主語言（strict mode） |
| Bun | 主 runtime + bundler |
| React Ink | 終端 TUI 渲染 |
| Zod | Schema 驗證 |
| Anthropic Messages API | LLM 通訊 |
| OpenTelemetry | 遙測 |
| Yoga (WASM) | Flexbox 佈局引擎 |
| ripgrep (vendor) | 快速文字搜尋 |

---

## 1.3 模組地圖：512K 行怎麼組織

```
src/
├── utils/          180,472 行  ← 最大模組，所有底層工具
│   ├── permissions/            權限系統核心
│   ├── plugins/                Plugin 載入器
│   ├── skills/                 Skill 載入器
│   ├── hooks/                  Hook 執行器
│   ├── mcp/                    MCP 工具
│   ├── settings/               設定管理
│   ├── git/                    Git 操作
│   ├── bash/                   Bash 執行
│   ├── shell/                  Shell 工具
│   ├── memory/                 記憶工具
│   ├── model/                  模型選擇
│   ├── messages/               訊息處理
│   ├── sandbox/                沙箱管理
│   ├── swarm/                  多 Agent 協調
│   └── telemetry/              遙測
│
├── components/      81,546 行  ← React Ink UI
│   ├── permissions/            權限對話框
│   ├── messages/               訊息渲染
│   ├── diff/                   Diff 顯示
│   ├── PromptInput/            輸入框
│   └── design-system/          設計系統
│
├── services/        53,680 行  ← 核心服務
│   ├── api/                    API 通訊（含重試、streaming）
│   ├── mcp/                    MCP 客戶端（119KB 核心檔）
│   ├── plugins/                Plugin 操作
│   ├── tools/                  工具執行編排
│   ├── compact/                對話壓縮
│   ├── SessionMemory/          Session 記憶
│   ├── extractMemories/        記憶萃取
│   ├── autoDream/              AutoDream 記憶整理
│   ├── lsp/                    LSP 伺服器管理
│   └── oauth/                  OAuth 認證
│
├── tools/           50,828 行  ← 43 個內建工具
│   ├── BashTool/               Shell 命令執行
│   ├── FileReadTool/           讀檔（含 PDF、圖片、Notebook）
│   ├── FileEditTool/           精確字串替換
│   ├── FileWriteTool/          建立/覆寫檔案
│   ├── GlobTool/               檔案模式搜尋
│   ├── GrepTool/               內容搜尋（用 vendored ripgrep）
│   ├── AgentTool/              生成子 Agent
│   ├── SendMessageTool/        Agent 間通訊
│   ├── SkillTool/              執行 Skill
│   ├── MCPTool/                MCP 工具橋接
│   ├── WebFetchTool/           抓取網頁
│   ├── WebSearchTool/          網路搜尋
│   ├── ToolSearchTool/         Deferred tool 搜尋
│   └── Task*Tool/              任務管理工具群
│
├── commands/        26,449 行  ← 90+ Slash 命令
│   ├── compact/   config/   cost/   diff/   doctor/
│   ├── help/   hooks/   mcp/   memory/   model/
│   ├── permissions/   plan/   plugin/   session/
│   └── skills/   tasks/   theme/   vim/   voice/ ...
│
├── hooks/           19,204 行  ← Hook 系統
├── ink/             19,842 行  ← Ink 渲染引擎客製層
├── bridge/          12,613 行  ← IDE Bridge
├── cli/             12,353 行  ← CLI 解析
├── entrypoints/      4,051 行  ← 入口點
│   ├── cli.tsx                 主 CLI 入口
│   ├── init.ts                 完整初始化
│   ├── mcp.ts                  MCP Server 模式
│   └── sdk/                    SDK API
│
├── state/            1,190 行  ← 狀態管理
│   ├── store.ts                Reactive store（Zustand-like）
│   ├── AppStateStore.ts        AppState 型別（22KB）
│   └── AppState.tsx            React Context Provider
│
├── constants/        2,648 行  ← System prompt + 常量
│   └── prompts.ts              System prompt 組裝（54KB）
│
├── tasks/            3,286 行  ← 任務類型
│   ├── LocalAgentTask/         本地子 Agent
│   ├── RemoteAgentTask/        遠端 Agent
│   ├── InProcessTeammateTask/  進程內 Teammate
│   ├── LocalShellTask/         Shell 任務
│   └── DreamTask/              記憶整理任務
│
├── skills/           4,066 行  ← Skill 系統
├── plugins/          (少量)    ← 內建 Plugin
├── coordinator/        369 行  ← Coordinator 模式
├── voice/              ---     ← 語音輸入
├── vim/              1,513 行  ← Vim 模式
├── keybindings/      3,159 行  ← 鍵盤快捷鍵
├── native-ts/        4,081 行  ← 原生實作（color-diff, yoga）
└── memdir/           1,736 行  ← 記憶目錄管理
```

---

## 1.4 Agent 設計的五大核心問題

Claude Code 的 512K 行程式碼，本質上是在回答五個問題：

### 問題一：如何讓 AI 可靠地使用工具？

不是「能不能」用工具，而是**可靠地**用。

- 43 個工具需要統一接口（`Tool<Input, Output, Progress>`）
- 輸入要用 Zod schema 嚴格驗證
- 並發安全要區分（讀取工具可並行，寫入工具必須串行）
- 大結果要持久化到磁碟，不能塞滿 context
- 錯誤要結構化回傳給 model，讓它自己修正

→ 詳見 **Chapter 3: System Prompt 工程化** 和 **Chapter 2: 請求生命週期**

### 問題二：如何在自主性與安全之間取得平衡？

Agent 需要足夠的自主權才有用，但太多自主權會造成危險。

- Permission 模型：6 種模式（default → bypass）
- 規則匹配：allow / deny / ask 三種行為
- 設定階層：8 層優先順序（企業管理 > 用戶 > session）
- Hook 系統：27 種事件可攔截/修改/阻止
- 危險模式偵測：自動移除過於寬鬆的規則

→ 詳見 **Chapter 3: System Prompt 工程化**（風險控制框架）

### 問題三：如何管理有限的上下文窗口？

200K token 聽起來很大，但一個稍微複雜的任務就能塞滿。

- 三層壓縮：MicroCompact（零 API 成本）→ AutoCompact（摘要）→ Full Compact（重組）
- 精確的閾值計算：167K token 觸發，13K 緩衝，3 次失敗斷路器
- 壓縮後重新注入：50K token 預算，最近 5 個檔案，使用中的 skill schema

→ 詳見 **Chapter 5: 三層上下文壓縮**

### 問題四：如何讓 AI 有長期記憶？

單次對話的記憶會消失。跨 session 的記憶需要系統設計。

- 即時抽取（extractMemories）：每次對話後 fire-and-forget
- 定期整理（AutoDream）：24 小時 + 5 個新 session 觸發
- 四類記憶：user / feedback / project / reference
- MEMORY.md 索引：200 行 / 25KB 硬限制
- 過時偵測：超過 1 天的記憶附加警告

→ 詳見 **Chapter 6: AutoDream 記憶系統**

### 問題五：如何讓多個 Agent 協作？

一個 Agent 的能力有限。複雜任務需要多個 Agent 分工。

- Coordinator 模式：Leader 分配任務，Worker 平行執行
- 權限佇列（Mailbox）：Worker 需要危險操作時向 Leader 請求
- 原子認領（createResolveOnce）：防止多個源同時解決同一請求
- Team Memory：跨 Agent 共享知識
- 記憶體保護：50 條訊息上限，防止 292 agent 記憶體爆炸

→ 詳見 **Chapter 4: 多 Agent 協作架構**

---

## 1.5 為什麼讀這份源碼？

### 對產品經理

理解 AI Agent 產品的**設計約束**：安全不是附加功能，是核心架構決策。權限模型、Hook 系統、壓縮策略——這些決定了產品能不能上線。

### 對架構師

學習 Anthropic 如何組織 **512K 行的 TypeScript 專案**：模組邊界在哪裡切、狀態怎麼管理、Plugin 系統怎麼分三層、工具接口怎麼統一。

### 對做 Agent 產品的工程師

直接可以帶走的設計模式：
- System prompt 的工程化寫法（不是「寫得好」，是行為規格書）
- 三層壓縮策略（MicroCompact 零成本最聰明）
- 原子認領機制（CAS 語意解決多源競爭）
- AutoDream 記憶整理（便宜 gate 優先的觸發策略）

### 對所有 AI 從業者

這是 Anthropic **自己做的** Agent。不是學術論文，不是 demo，是每天被數十萬開發者使用的生產系統。它的每個設計決策都經過真實使用的打磨。

---

## 1.6 本書結構

| Part | 章節 | 顆粒度 | 目標讀者 |
|------|------|--------|---------|
| **I 基礎** | Ch01: 什麼是 Coding Agent | 概念 | 所有人 |
| | Ch02: 一個問題的完整旅程 | 架構 | 工程師 |
| **II 設計模式** | Ch03: System Prompt 工程化 | 逐行 | Agent 建造者 |
| | Ch04: 多 Agent 協作架構 | 逐行 | Agent 建造者 |
| | Ch05: 三層上下文壓縮 | 逐行 | Agent 建造者 |
| | Ch06: AutoDream 記憶系統 | 逐行 | Agent 建造者 |

Part I 給你地圖和全貌。Part II 帶你進入四個最值得學習的系統，逐行拆解源碼。

---

## 1.7 如何閱讀本書

### 源碼引用格式

本書所有源碼引用格式為 `檔案路徑:行號`，例如：

```
src/constants/prompts.ts:444    → prompts.ts 第 444 行
src/services/compact/autoCompact.ts:62  → autoCompact.ts 第 62 行
```

源碼來自 Claude Code v2.1.90 的反編譯版本，可在 GitHub 上取得：
- [Leolai6-7/claude-code-source-code](https://github.com/Leolai6-7/claude-code-source-code)

### 建議閱讀路徑

**快速了解**（30 分鐘）：Ch01 → Ch02
**深入設計模式**（2-3 小時）：挑 Ch03-06 中最相關的一章
**完整閱讀**（半天）：按順序讀完全部 6 章

### 每章結構

每章都包含：
1. **概念說明** — 為什麼需要這個系統
2. **架構圖** — 視覺化理解
3. **源碼片段** — 附 file:line 引用
4. **設計模式** — 可直接帶走的模式
5. **章節總結** — 關鍵要點

---

## 章節總結

| 要點 | 內容 |
|------|------|
| Claude Code 是第三代 AI 工具 | 從補全 → 對話 → 自主 Agent |
| 512K 行 TypeScript | 1,902 檔、43 工具、90+ 命令、27 種 hook |
| 零 runtime dependencies | 全部 Bun bundle 進單一 cli.js |
| 五大核心問題 | 工具可靠性、安全平衡、上下文管理、長期記憶、多 Agent 協作 |
| 本書聚焦四大設計模式 | Prompt 工程、Swarm、壓縮、記憶 |
