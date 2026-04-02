# 第一章：專案結構總覽

## 打包與發布

Claude Code 以兩種形式發布：

1. **NPM 包** (`@anthropic-ai/claude-code`) — 單一 `cli.js` bundled 檔 (13MB, 16,911 行 minified)
2. **Homebrew Cask** — Mach-O 64-bit 原生二進位 (193MB, 內嵌 Node.js runtime via SEA)

```
@anthropic-ai/claude-code/
├── cli.js              # Bun-bundled 入口（所有 TS 編譯為此）
├── package.json        # 只有 optionalDependencies (sharp for 圖片處理)
├── sdk-tools.d.ts      # SDK 公開型別定義 (117KB)
├── bun.lock            # Bun lockfile
├── vendor/
│   ├── ripgrep/        # 內嵌 rg 二進位（6 平台）
│   └── audio-capture/  # 內嵌語音擷取二進位（6 平台）
└── LICENSE.md
```

## 反編譯源碼結構

```
src/
├── entrypoints/        # 入口點
│   ├── cli.tsx         # 主 CLI 入口（fast-path 分派）
│   ├── init.ts         # 完整初始化流程
│   ├── mcp.ts          # MCP server 模式入口
│   └── sdk/            # SDK programmatic API
│
├── bootstrap/          # 啟動時全域狀態
│   └── state.js        # 全域 counter、meter、session 狀態
│
├── state/              # 應用狀態管理
│   ├── store.ts        # Zustand-like reactive store
│   ├── AppStateStore.ts # AppState 型別定義 (21.8KB)
│   └── AppState.tsx    # React Context Provider
│
├── tools/              # 43 個內建工具
│   ├── AgentTool/
│   ├── BashTool/
│   ├── FileReadTool/
│   ├── FileEditTool/
│   ├── FileWriteTool/
│   ├── GlobTool/
│   ├── GrepTool/
│   ├── WebFetchTool/
│   ├── WebSearchTool/
│   ├── SkillTool/
│   ├── MCPTool/
│   ├── NotebookEditTool/
│   ├── LSPTool/
│   ├── TaskCreateTool/   ...TaskGetTool, TaskUpdateTool, TaskListTool, TaskStopTool, TaskOutputTool
│   ├── AgentTool/        # 子 agent 生成
│   ├── SendMessageTool/  # 發訊息給子 agent
│   ├── TeamCreateTool/   # 團隊建立
│   ├── EnterPlanModeTool/ ...ExitPlanModeTool
│   ├── EnterWorktreeTool/ ...ExitWorktreeTool
│   ├── ConfigTool/
│   ├── RemoteTriggerTool/
│   ├── ScheduleCronTool/
│   ├── SleepTool/
│   ├── ToolSearchTool/   # Deferred tool schema 搜尋
│   ├── BriefTool/
│   ├── TodoWriteTool/
│   ├── REPLTool/
│   ├── PowerShellTool/
│   ├── SyntheticOutputTool/
│   ├── McpAuthTool/
│   ├── ListMcpResourcesTool/
│   ├── ReadMcpResourceTool/
│   └── shared/           # 共用工具（git tracking、multi-agent spawn）
│
├── services/           # 核心服務
│   ├── api/            # Anthropic API 通訊
│   ├── mcp/            # MCP 客戶端（119KB 核心）
│   ├── plugins/        # Plugin 操作（安裝、啟用、更新）
│   ├── tools/          # 工具執行編排
│   ├── lsp/            # LSP 伺服器管理
│   ├── oauth/          # OAuth 認證流程
│   ├── compact/        # 對話壓縮服務
│   ├── SessionMemory/  # Session 記憶抽取
│   ├── extractMemories/ # 記憶萃取邏輯
│   ├── analytics/      # 分析追蹤
│   ├── policyLimits/   # 政策限制
│   ├── remoteManagedSettings/ # 遠端管理設定
│   ├── settingsSync/   # 設定同步
│   ├── tips/           # 提示系統
│   └── MagicDocs/      # 文件生成服務
│
├── commands/           # 90+ slash 命令
│   ├── compact/        # /compact
│   ├── config/         # /config
│   ├── cost/           # /cost
│   ├── diff/           # /diff
│   ├── doctor/         # /doctor
│   ├── help/           # /help
│   ├── hooks/          # /hooks
│   ├── mcp/            # /mcp
│   ├── memory/         # /memory
│   ├── model/          # /model
│   ├── permissions/    # /permissions
│   ├── plan/           # /plan
│   ├── plugin/         # /plugin
│   ├── session/        # /session
│   ├── skills/         # /skills
│   ├── tasks/          # /tasks
│   ├── theme/          # /theme
│   ├── vim/            # /vim
│   ├── voice/          # /voice
│   └── ...（另外 70+ 個命令）
│
├── components/         # React Ink UI 元件
│   ├── permissions/    # 權限對話框
│   ├── messages/       # 訊息渲染
│   ├── diff/           # Diff 顯示
│   ├── skills/         # Skill UI
│   ├── mcp/            # MCP 狀態 UI
│   ├── memory/         # 記憶 UI
│   ├── tasks/          # 任務 UI
│   ├── design-system/  # 設計系統元件
│   └── PromptInput/    # 輸入框
│
├── hooks/              # Hook 執行系統
│   ├── notifs/         # 通知 hook
│   └── toolPermission/ # 工具權限 hook
│
├── utils/              # 工具函式庫（180K 行）
│   ├── permissions/    # 權限系統核心
│   ├── plugins/        # Plugin 載入器
│   ├── skills/         # Skill 載入器
│   ├── hooks/          # Hook 執行器
│   ├── mcp/            # MCP 工具
│   ├── settings/       # 設定管理
│   ├── git/            # Git 操作
│   ├── bash/           # Bash 執行
│   ├── shell/          # Shell 工具
│   ├── memory/         # 記憶工具
│   ├── model/          # 模型選擇
│   ├── messages/       # 訊息處理
│   ├── sandbox/        # 沙箱管理
│   ├── telemetry/      # 遙測
│   └── ...
│
├── skills/             # 內建 Skill 系統
│   ├── bundledSkills.ts # Skill 註冊
│   └── loadSkillsDir.ts # 從磁碟載入 Skills
│
├── tasks/              # 任務類型
│   ├── LocalAgentTask/   # 本地子 agent
│   ├── RemoteAgentTask/  # 遠端 agent
│   ├── LocalShellTask/   # 本地 shell 任務
│   ├── InProcessTeammateTask/ # 進程內隊友
│   └── DreamTask/        # Dream 任務（離線處理）
│
├── ink/                # Ink 渲染引擎客製
│   ├── components/     # 底層 Ink 元件
│   ├── hooks/          # Ink React hooks
│   ├── layout/         # 佈局系統
│   ├── events/         # 事件系統
│   └── termio/         # 終端 I/O
│
├── cli/                # CLI 解析
│   ├── handlers/       # 命令處理器
│   └── transports/     # 輸入/輸出 transport
│
├── types/              # TypeScript 型別
│   └── generated/      # 自動生成型別
│
├── constants/          # 常量
│   └── prompts.js      # System prompt 生成
│
├── voice/              # 語音輸入系統
├── vim/                # Vim 模式
├── keybindings/        # 鍵盤快捷鍵
├── bridge/             # IDE Bridge 通訊
├── remote/             # 遠端 session
├── coordinator/        # 協調器
├── server/             # 內建 server
├── schemas/            # Zod schemas
├── migrations/         # 設定遷移
├── query/              # 查詢工具
├── memdir/             # 記憶目錄管理
├── buddy/              # Buddy 系統
├── context/            # Context providers
├── outputStyles/       # 輸出樣式
├── native-ts/          # 原生 TS 實作
│   ├── color-diff/     # 彩色 diff
│   ├── file-index/     # 檔案索引
│   └── yoga-layout/    # Yoga 佈局引擎
└── upstreamproxy/      # 上游代理
```

## 關鍵數據

| 指標 | 數值 |
|------|------|
| TypeScript 源檔 | 1,902 個 |
| 總行數 | 512,685 行 |
| 內建工具 | 43 個 |
| Slash 命令 | 90+ 個 |
| Hook 事件類型 | 27 種 |
| MCP Transport 類型 | 8 種 |
| 支援平台 | 6 個 (macOS/Linux/Windows × arm64/x64) |

## 依賴關係

Claude Code **零 runtime dependencies**（package.json 的 `dependencies: {}`），所有依賴在 build time 被 Bun bundler 打包進 `cli.js`。

`optionalDependencies` 只有 `@img/sharp-*`（9 個平台變體），用於圖片處理。

`vendor/` 目錄包含預編譯的平台原生二進位：
- **ripgrep** — 用於 Grep tool 的快速搜尋
- **audio-capture** — 用於語音輸入功能
