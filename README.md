# Claude Code CLI 源代碼深度分析

> 基於 Claude Code v2.1.90 反編譯源碼的完整架構分析與教學指南

## 目錄

| 章節 | 內容 | 檔案 |
|------|------|------|
| 1 | [專案結構總覽](docs/01-project-structure.md) | 1,902 個源碼檔、512,685 行 TypeScript |
| 2 | [核心架構設計模式](docs/02-architecture.md) | 入口點、啟動流程、React Ink TUI |
| 3 | [請求/回應生命週期](docs/03-request-lifecycle.md) | 從用戶輸入到 API 回應的完整流程 |
| 4 | [Tool 系統](docs/04-tool-system.md) | 43 個內建工具、註冊、執行、並發控制 |
| 5 | [Plugin/Skill 載入機制](docs/05-plugin-skill-system.md) | Marketplace、三層模型、熱重載 |
| 6 | [Hook 系統](docs/06-hook-system.md) | 27 種 hook 事件、5 種執行類型 |
| 7 | [Permission 模型](docs/07-permission-model.md) | 模式、規則匹配、設定階層 |
| 8 | [MCP 整合](docs/08-mcp-integration.md) | 伺服器連接、工具發現、Elicitation |
| 9 | [Session 與 Memory 管理](docs/09-session-memory.md) | 狀態管理、記憶抽取、壓縮 |

### 四大重點深度分析

| 章節 | 內容 | 檔案 |
|------|------|------|
| 10 | [**學 Anthropic 怎麼寫 System Prompt**](docs/10-focus-system-prompt-engineering.md) | 工具約束、風險控制、輸出規範、過度工程防護 |
| 11 | [**拆解多 Agent 協作架構（Swarm）**](docs/11-focus-multi-agent-swarm.md) | Coordinator、權限佇列、原子認領、Team Memory |
| 12 | [**上下文壓縮策略**](docs/12-focus-context-compression.md) | 三層壓縮：MicroCompact → AutoCompact → Full Compact |
| 13 | [**AutoDream 記憶整理機制**](docs/13-focus-autodream-memory.md) | 四階段：Orient → Gather → Consolidate → Prune |

## 源碼來源

- **NPM 包**: `@anthropic-ai/claude-code@2.1.90`（bundled 為單一 cli.js，16,911 行 minified）
- **反編譯源碼**: [Leolai6-7/claude-code-source-code](https://github.com/Leolai6-7/claude-code-source-code)

## 技術棧

- **語言**: TypeScript (strict mode)
- **Runtime**: Bun (primary) / Node.js 18+ (fallback)
- **UI 框架**: React Ink（終端 TUI）
- **打包工具**: Bun bundler → 單一 cli.js
- **Schema 驗證**: Zod
- **API 通訊**: Anthropic Messages API
- **遙測**: OpenTelemetry + gRPC exporters

## 模組規模 (按行數)

```
180,472  utils/          — 工具函式庫（權限、設定、Git、MCP、Shell...）
 81,546  components/     — React Ink UI 元件
 53,680  services/       — 核心服務（API、MCP、Plugin、LSP、OAuth...）
 50,828  tools/          — 43 個內建工具實作
 26,449  commands/       — 90+ 個 slash 命令
 19,842  ink/            — Ink 渲染引擎客製層
 19,204  hooks/          — Hook 執行與通知系統
 12,613  bridge/         — IDE bridge 通訊
 12,353  cli/            — CLI 解析與 transport
  5,977  screens/        — 全螢幕 UI（onboarding、settings）
  4,081  native-ts/      — 原生 TS 實作（color-diff、file-index、yoga）
  4,066  skills/         — Skill 載入與 bundled skills
  4,051  entrypoints/    — 入口點（CLI、SDK、MCP server）
  3,446  types/          — 型別定義
  3,286  tasks/          — 任務系統（Agent、Remote、Shell、Dream）
  3,159  keybindings/    — 鍵盤快捷鍵系統
  2,648  constants/      — 常量與 system prompt
  1,758  bootstrap/      — 啟動初始化狀態
  1,736  memdir/         — 記憶目錄管理
  1,513  vim/            — Vim 模式支援
  1,298  buddy/          — Buddy 系統
  1,190  state/          — 狀態管理 store
  1,127  remote/         — 遠端 session
  1,004  context/        — Context provider
```
