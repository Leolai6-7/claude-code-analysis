# 第二章：核心架構設計模式

## 入口點分派

Claude Code 有三個主要入口點：

```
cli.tsx ──→ 主 CLI（互動式/非互動式）
mcp.ts  ──→ MCP Server 模式（被其他工具調用）
sdk/    ──→ SDK 程式化 API（被其他程式嵌入）
```

### cli.tsx — 啟動流程

```typescript
// src/entrypoints/cli.tsx
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // Fast-path: --version 零模組載入
  if (args[0] === '--version') {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }

  // Fast-path: --dump-system-prompt（內部用）
  // Fast-path: --claude-in-chrome-mcp
  // Fast-path: --chrome-native-host

  // 完整啟動路徑
  const { init } = await import('../entrypoints/init.js');
  await init();
  // ... 渲染 React Ink App
}
```

**設計哲學**：Fast-path 分派。`--version` 只需讀一個常量，不載入任何模組。完整啟動時才動態 import。

### init.ts — 完整初始化

```
init()
  ├── enableConfigs()                    # 啟用設定系統
  ├── applySafeConfigEnvironmentVariables() # 安全環境變數
  ├── applyExtraCACertsFromConfig()      # TLS 憑證
  ├── configureGlobalAgents()            # HTTP 代理
  ├── configureGlobalMTLS()              # mTLS 設定
  ├── setShellIfWindows()                # Windows shell 設定
  ├── detectCurrentRepository()          # 偵測 Git repo
  ├── setupGracefulShutdown()            # 優雅關閉處理
  ├── initializeTelemetry()              # OpenTelemetry 初始化
  ├── populateOAuthAccountInfoIfNeeded() # OAuth 帳號資訊
  ├── initializePolicyLimitsLoadingPromise() # 政策限制
  └── preconnectAnthropicApi()           # API 預連線
```

## React Ink TUI 架構

Claude Code 的終端 UI 使用 **React Ink**（React 的終端渲染器）：

```
App (React Ink root)
├── AppState Provider (store)
├── MCP Connection Manager (hook)
├── Plugin Manager
├── Screen Router
│   ├── Onboarding Screen
│   ├── Main Chat Screen
│   │   ├── Message List
│   │   │   ├── UserMessage
│   │   │   ├── AssistantMessage
│   │   │   ├── ToolUseMessage
│   │   │   └── ToolResultMessage
│   │   ├── PromptInput
│   │   ├── StatusLine
│   │   └── Permission Dialog (conditional)
│   └── Settings Screen
└── Spinner / Progress indicators
```

### 狀態管理

使用自製的 **Zustand-like store**：

```typescript
// src/state/store.ts
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

`AppState` 是核心狀態樹，包含：

| 區塊 | 內容 |
|------|------|
| `settings` | 全域設定、模型選擇 |
| `mcp` | MCP 客戶端連線、工具、命令、資源 |
| `plugins` | 啟用/停用的 plugins、命令、agent 定義 |
| `tasks` | 任務狀態 map |
| `session` | Session ID、遠端 URL、bridge 狀態 |

## API 通訊層

```
User Input
    ↓
System Prompt Construction (constants/prompts.js)
    ↓
Message Assembly (utils/messages/)
    ↓
Anthropic Messages API (services/api/)
    ├── Streaming response
    ├── Extended thinking (interleaved)
    └── Tool use blocks
    ↓
Tool Execution (services/tools/)
    ↓
Result Assembly → append to messages → next API call
```

### System Prompt 組裝

System prompt 是動態組裝的，包含：

1. **Base prompt** — 核心行為指令
2. **Tool descriptions** — 所有啟用工具的描述
3. **CLAUDE.md files** — 用戶/專案指令
4. **Plugin rules** — Plugin 注入的規則
5. **Skill descriptions** — 所有可用 skill 的摘要
6. **MCP server instructions** — MCP 伺服器的使用說明
7. **Memory content** — MEMORY.md 索引
8. **Context reminders** — 系統提醒

## 設定系統層級

```
Policy Settings (企業管理，最高優先)
    ↓ override
Flag Settings (Feature flag)
    ↓ override
Local Settings (.claude/settings.local.json)
    ↓ override
Project Settings (.claude/settings.json)
    ↓ override
User Settings (~/.claude/settings.json)
    ↓ override
CLI Arguments (--model, --permission-mode, etc.)
    ↓ override
Session Settings (runtime 修改)
```

## 關鍵設計模式

### 1. Lazy Dynamic Import
所有重型模組使用 `await import()` 延遲載入，確保 fast-path 極速回應。

### 2. Feature Flag Dead Code Elimination
```typescript
import { feature } from 'bun:bundle';
if (feature('ABLATION_BASELINE')) { /* build time 決定是否保留 */ }
```

### 3. Memoized Initialization
```typescript
export const init = memoize(async () => { /* 只執行一次 */ });
```

### 4. Async Generator Pattern
Hook 執行和工具結果使用 async generator，支援 streaming 和 incremental progress。

### 5. Immutable State Updates
AppState 通過 `setState(prev => ({...prev, ...changes}))` 更新，所有更新觸發 subscriber。

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/entrypoints/cli.tsx` | 主入口、fast-path 分派 |
| `src/entrypoints/init.ts` | 完整初始化流程 |
| `src/state/store.ts` | Reactive store 實作 |
| `src/state/AppStateStore.ts` | AppState 型別定義 |
| `src/state/AppState.tsx` | React Context Provider |
| `src/constants/prompts.js` | System prompt 組裝 |
| `src/services/api/` | API 通訊層 |
