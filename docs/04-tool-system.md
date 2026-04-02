# 第四章：Tool 系統

## 工具接口定義

**源碼**: `src/Tool.ts`

Claude Code 使用 **型別接口 + 建構函式** 模式，而非 class 繼承：

```typescript
// Tool<Input, Output, Progress> — 泛型接口
interface Tool<I, O, P> {
  name: string
  aliases?: string[]                    // 重命名後的向後相容

  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<O>>
  description(): string                 // 動態描述（可依 context 變化）
  prompt(): string                      // System prompt 中的使用說明

  // Schema
  inputSchema: ZodSchema               // Zod schema 驗證輸入
  inputJSONSchema?: JSONSchema          // JSON Schema 給 API
  outputSchema?: ZodSchema              // 可選輸出 schema

  // 元數據方法
  isEnabled(): boolean                  // 是否啟用
  isConcurrencySafe(): boolean          // 是否可並行執行
  isReadOnly(): boolean                 // 是否唯讀
  isDestructive(): boolean              // 是否具破壞性

  // 權限
  checkPermissions(): PermissionResult  // 工具特定權限邏輯
  validateInput(): ValidationResult     // 執行前額外驗證

  // 渲染
  renderToolUseMessage()                // UI 渲染工具呼叫
  renderToolResultMessage()             // UI 渲染結果

  // 大結果處理
  maxResultSizeChars: number            // 超過此值持久化到磁碟
}
```

## 43 個內建工具

| 工具 | 類型 | 並發安全 | 用途 |
|------|------|---------|------|
| **FileReadTool** | 讀取 | ✅ | 讀取檔案內容 |
| **FileEditTool** | 寫入 | ❌ | 精確字串替換 |
| **FileWriteTool** | 寫入 | ❌ | 建立/覆寫檔案 |
| **BashTool** | 執行 | ❌ | Shell 命令執行 |
| **GlobTool** | 讀取 | ✅ | 檔案模式搜尋 |
| **GrepTool** | 讀取 | ✅ | 內容搜尋（內嵌 ripgrep） |
| **AgentTool** | 控制 | ❌ | 生成子 agent |
| **SendMessageTool** | 控制 | ❌ | 向子 agent 發訊息 |
| **WebFetchTool** | 網路 | ✅ | 抓取網頁內容 |
| **WebSearchTool** | 網路 | ✅ | 網路搜尋 |
| **SkillTool** | 控制 | ❌ | 執行 skill |
| **MCPTool** | 橋接 | 視情況 | 調用 MCP 工具 |
| **LSPTool** | 橋接 | ✅ | LSP 查詢 |
| **ToolSearchTool** | 讀取 | ✅ | 搜尋 deferred tool schema |
| **TaskCreateTool** | 控制 | ❌ | 建立任務 |
| **TaskUpdateTool** | 控制 | ❌ | 更新任務狀態 |
| **TaskGetTool** | 讀取 | ✅ | 取得任務資訊 |
| **TaskListTool** | 讀取 | ✅ | 列出任務 |
| **TaskStopTool** | 控制 | ❌ | 停止任務 |
| **TaskOutputTool** | 讀取 | ✅ | 讀取任務輸出 |
| **NotebookEditTool** | 寫入 | ❌ | 編輯 Jupyter notebook |
| **AskUserQuestionTool** | 互動 | ❌ | 向用戶提問 |
| **EnterPlanModeTool** | 控制 | ❌ | 進入計畫模式 |
| **ExitPlanModeTool** | 控制 | ❌ | 離開計畫模式 |
| **EnterWorktreeTool** | 控制 | ❌ | 進入 git worktree |
| **ExitWorktreeTool** | 控制 | ❌ | 離開 git worktree |
| **ConfigTool** | 控制 | ❌ | 修改設定 |
| **TeamCreateTool** | 控制 | ❌ | 建立團隊 |
| **TeamDeleteTool** | 控制 | ❌ | 刪除團隊 |
| **RemoteTriggerTool** | 控制 | ❌ | 遠端觸發 |
| **ScheduleCronTool** | 控制 | ❌ | 排程 cron 任務 |
| **SleepTool** | 控制 | ❌ | 休眠（自主模式） |
| **BriefTool** | 輸出 | ✅ | 簡短輸出 |
| **TodoWriteTool** | 寫入 | ❌ | 寫入 TODO |
| **REPLTool** | 執行 | ❌ | REPL 環境 |
| **PowerShellTool** | 執行 | ❌ | PowerShell 命令 |
| **SyntheticOutputTool** | 內部 | ❌ | 結構化輸出 |
| **McpAuthTool** | 認證 | ❌ | MCP OAuth |
| **ListMcpResourcesTool** | 讀取 | ✅ | 列出 MCP 資源 |
| **ReadMcpResourceTool** | 讀取 | ✅ | 讀取 MCP 資源 |

## 工具註冊流程

**源碼**: `src/tools.ts`

```typescript
// 1. 收集所有基礎工具
function getAllBaseTools(): Tools {
  return [AgentTool, BashTool, FileReadTool, FileEditTool, ...]
}

// 2. 依權限模式過濾
function getTools(permissionContext): Tools {
  // Simple mode: 減少工具數量
  // REPL mode: 特殊工具集
  // Deny rules: filterToolsByDenyRules()
}

// 3. 組裝最終工具池（含 MCP 工具）
function assembleToolPool(permissionContext, mcpTools): Tools {
  // 合併內建 + MCP，去重
}
```

## 工具執行生命週期

**源碼**: `src/services/tools/toolExecution.ts` (60KB)

```
runToolUse(toolUseBlock)
  │
  ├── 1. 工具查找：在 tools[] 中找工具（含 alias 匹配）
  │
  ├── 2. 輸入驗證
  │   ├── Zod schema 解析: tool.inputSchema.safeParse(input)
  │   └── 工具特定驗證: tool.validateInput(parsedInput)
  │
  ├── 3. PreToolUse Hooks
  │   └── 可阻止/修改輸入/自動批准
  │
  ├── 4. 權限檢查
  │   ├── tool.checkPermissions(input, context)
  │   ├── checkRuleBasedPermissions() — deny/allow/ask 規則
  │   ├── resolveHookPermissionDecision() — hook 覆蓋
  │   └── canUseTool() — 互動式提示（如需要）
  │
  ├── 5. 工具執行
  │   └── tool.call(input, context, canUseTool, message, onProgress)
  │
  ├── 6. PostToolUse Hooks
  │   └── 可修改/阻止輸出
  │
  └── 7. 結果處理
      ├── mapToolResultToToolResultBlockParam()
      └── processToolResultBlock() — 大結果持久化
```

## 並發控制

**源碼**: `src/services/tools/toolOrchestration.ts`

```typescript
// 當 model 回傳多個 tool_use block
function partitionToolCalls(toolCalls) {
  const concurrent = []  // isConcurrencySafe() === true
  const serial = []      // isConcurrencySafe() === false

  // 同一批裡，concurrent 先並行跑完
  // 然後 serial 一個一個跑
}

async function runToolsConcurrently(tools) {
  // Promise.all() 並行執行
  // Context modifiers 佇列化，批次結束後依序套用
}
```

## 大結果持久化

當工具輸出超過 `maxResultSizeChars` 時：

```
1. 將完整結果寫入磁碟暫存檔
2. 回傳給 model 的只有預覽 + 檔案路徑
3. Model 可選擇用 FileReadTool 讀取完整內容
```

這避免了大型 grep/read 結果佔滿 context window。

## 源碼參考

| 檔案 | 大小 | 角色 |
|------|------|------|
| `src/Tool.ts` | — | 工具接口定義、buildTool() 建構器 |
| `src/tools.ts` | — | 工具註冊、過濾、組裝 |
| `src/services/tools/toolExecution.ts` | 60KB | 工具執行主流程 |
| `src/services/tools/toolOrchestration.ts` | 5.5KB | 並發控制、批次執行 |
| `src/services/tools/toolHooks.ts` | 22KB | Hook 觸發邏輯 |
| `src/services/tools/StreamingToolExecutor.ts` | 17KB | Streaming 工具執行 |
