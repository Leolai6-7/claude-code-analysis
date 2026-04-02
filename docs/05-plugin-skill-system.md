# 第五章：Plugin / Skill 載入機制

## 三層模型

Claude Code 的 plugin 系統採用 **三層架構**：

```
Intent Layer (意圖層)          — settings.json 裡的宣告
    ↓ getDeclaredMarketplaces()
Materialization Layer (物化層)  — ~/.claude/plugins/ 磁碟上的檔案
    ↓ loadAllPlugins()
Active Components Layer (活化層) — AppState 裡的運行時元件
```

### Layer 1: Intent (設定宣告)

```json
// ~/.claude/settings.json
{
  "enabledPlugins": {
    "everything-claude-code@everything-claude-code": true,
    "financial-analysis@financial-services-plugins": false
  },
  "extraKnownMarketplaces": {
    "financial-services-plugins": {
      "source": { "source": "github", "repo": "anthropics/financial-services-plugins" }
    }
  }
}
```

### Layer 2: Materialization (磁碟物化)

```
~/.claude/plugins/
├── known_marketplaces.json      # 已知 marketplace 設定
├── installed_plugins.json       # 安裝 metadata（V2 格式）
├── install-counts-cache.json    # 安裝計數快取
├── blocklist.json               # 封鎖列表
├── marketplaces/                # Marketplace 快取
│   ├── everything-claude-code/  # Git clone
│   └── financial-services-plugins/
└── cache/                       # Plugin 安裝快取
    ├── everything-claude-code/
    │   └── everything-claude-code/56ff5d444bfe/
    ├── claude-plugins-official/
    │   ├── discord/0.0.4/
    │   └── ralph-loop/1.0.0/
    └── claude-code-workflows/
        ├── python-development/1.2.1/
        └── ...
```

### Layer 3: Active Components (運行時)

```typescript
// AppState.plugins
{
  enabled: LoadedPlugin[]        // 啟用的 plugins
  disabled: LoadedPlugin[]       // 停用的 plugins
  errors: PluginError[]          // 載入錯誤
  pluginCommands: Command[]      // 所有 plugin 的命令
  agentDefinitions: AgentDef[]   // 所有 plugin 的 agent 定義
}
```

## Plugin 目錄結構

```
my-plugin/
├── plugin.json           # 可選的 manifest（metadata、MCP servers、LSP servers）
├── commands/             # Slash 命令（.md 檔案）
│   ├── plan.md
│   └── verify.md
├── skills/               # Skills（同 commands，也可用 SKILL.md 目錄格式）
│   ├── coding-standards/
│   │   └── SKILL.md
│   └── tdd-workflow/
│       └── SKILL.md
├── agents/               # Agent 定義（.md 檔案含 system prompt）
│   ├── code-reviewer.md
│   └── planner.md
├── rules/                # 規則檔（注入 system prompt）
│   ├── coding-style.md
│   └── security.md
├── hooks/                # Hook 設定
│   └── hooks.json
├── contexts/             # Context 定義
│   ├── dev.md
│   └── review.md
└── output-styles/        # 輸出樣式設定
```

## Plugin 載入流程

**源碼**: `src/utils/plugins/pluginLoader.ts` (31KB)

```
1. 發現 plugin 來源
   ├── Marketplace plugins (enabledPlugins 設定)
   ├── Session plugins (--plugin-dir CLI flag)
   └── Builtin plugins (內建)

2. 載入 manifest
   └── plugin.json → PluginManifest 驗證

3. 載入元件
   ├── Commands: commands/ + skills/ 目錄的 .md 檔
   ├── Agents: agents/ 目錄的 .md 檔
   ├── Hooks: hooks/hooks.json
   ├── MCP Servers: manifest.mcpServers
   └── LSP Servers: manifest.lspServers

4. 命名空間化
   └── pluginName:namespace:commandName

5. 去重與衝突檢測
   └── 防止重複命令/agent 名稱

6. 錯誤收集
   └── 結構化 PluginError
```

## Skill 載入細節

**源碼**: `src/skills/loadSkillsDir.ts` (10KB), `src/skills/bundledSkills.ts`

### Bundled Skills（內建）

```typescript
registerBundledSkill({
  name: 'commit',
  description: 'Create git commits',
  whenToUse: 'When user asks to commit...',
  allowedTools: ['Bash(git:*)'],
  getPromptForCommand: async () => '...',
})
```

### Disk-Based Skills（Plugin/用戶自訂）

**SKILL.md 格式**：

```markdown
---
name: my-skill
description: What this skill does
effort: medium
allowed-tools: ["Bash(npm:*)", "FileEditTool"]
model: sonnet
user-invocable: true
argument-hint: "<project-name>"
when-to-use: "When the user asks to..."
---

# Skill Content

The actual prompt content...
```

### Frontmatter 欄位

| 欄位 | 用途 |
|------|------|
| `name` | Skill 名稱 |
| `description` | 描述（出現在 system prompt 的 skill 列表） |
| `effort` | 計算努力等級 |
| `allowed-tools` | 允許使用的工具白名單 |
| `model` | 指定使用的模型 |
| `user-invocable` | 是否可被用戶直接 `/skill-name` 調用 |
| `argument-hint` | 參數提示 |
| `when-to-use` | 觸發條件描述 |
| `context` | `inline`（當前對話）或 `fork`（子 agent） |
| `agent` | 關聯的 agent 定義 |
| `hide-from-slash-command-tool` | 是否從 SkillTool 隱藏 |

## Marketplace 系統

**源碼**: `src/utils/plugins/marketplaceManager.ts`

### Marketplace 來源類型

| 類型 | 格式 | 用途 |
|------|------|------|
| `github` | `{ repo: "owner/repo" }` | GitHub repo |
| `git` | `{ url: "https://..." }` | 任意 Git URL |
| `http` | `{ url: "https://..." }` | HTTP JSON |
| `local` | `{ path: "/path/to/dir" }` | 本地目錄 |
| `npm` | `{ package: "@org/name" }` | NPM 包 |

### 背景安裝管理

**源碼**: `src/services/plugins/PluginInstallationManager.ts`

```
Session 啟動
    ↓
performBackgroundPluginInstallations()
    ├── diffMarketplaces() — 比較宣告 vs 物化
    ├── reconcileMarketplaces() — Clone/更新
    │   ├── 新安裝 → 自動 refresh plugins + MCP 重連
    │   └── 更新 → 設定 needsRefresh flag
    └── 通知用戶 /reload-plugins
```

### 政策控制

**源碼**: `src/utils/plugins/pluginPolicy.ts`

- 官方 marketplace 名稱保護（只能從 `anthropics` org 來）
- 冒充偵測（`BLOCKED_OFFICIAL_NAME_PATTERN`）
- 來源白名單（`isSourceAllowedByPolicy()`）
- 封鎖列表支援（企業管理）

## 熱重載

```
/reload-plugins
    ↓
refreshActivePlugins(setAppState)
    ├── clearAllCaches()
    ├── loadAllPlugins()
    ├── getPluginCommands()
    ├── getAgentDefinitionsWithOverrides()
    ├── loadPluginHooks()
    ├── loadPluginMcpServers()
    ├── loadPluginLspServers()
    └── 更新 AppState
        ├── plugins.enabled/disabled/errors
        ├── plugins.pluginCommands
        ├── plugins.agentDefinitions
        └── mcp.pluginReconnectKey++ (觸發 MCP 重連)
```

## 變數替換

Plugin 的命令和 agent 支援變數替換：

| 變數 | 值 |
|------|---|
| `${CLAUDE_PLUGIN_ROOT}` | Plugin 安裝路徑 |
| `${user_config.X}` | 用戶自訂 plugin 設定值 |

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/utils/plugins/pluginLoader.ts` | 主載入器 (31KB) |
| `src/services/plugins/pluginOperations.ts` | 安裝/啟用/更新操作 |
| `src/services/plugins/PluginInstallationManager.ts` | 背景安裝管理 |
| `src/utils/plugins/marketplaceManager.ts` | Marketplace 來源管理 |
| `src/utils/plugins/installedPluginsManager.ts` | 安裝 metadata 持久化 |
| `src/utils/plugins/loadPluginCommands.ts` | 命令/skill 載入 |
| `src/utils/plugins/loadPluginAgents.ts` | Agent 載入 |
| `src/utils/plugins/loadPluginHooks.ts` | Hook 載入與註冊 |
| `src/utils/plugins/refresh.ts` | 熱重載 |
| `src/utils/plugins/schemas.ts` | Zod 驗證 schemas (16.7KB) |
| `src/types/plugin.ts` | 型別定義 |
| `src/skills/bundledSkills.ts` | 內建 skill 註冊 |
| `src/skills/loadSkillsDir.ts` | 磁碟 skill 載入 (10KB) |
