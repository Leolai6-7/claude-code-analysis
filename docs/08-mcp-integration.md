# 第八章：MCP 整合

> **源碼**: `src/services/mcp/` (22 個檔案, ~12.2MB)

## MCP 伺服器設定

### 設定來源

| 來源 | 用途 |
|------|------|
| `local` | 本地設定 |
| `user` | 用戶全域 |
| `project` | 專案級別 |
| `dynamic` | 動態添加 |
| `enterprise` | 企業管理 |
| `claudeai` | Claude AI 官方 |
| `managed` | 遠端管理 |

### Transport 類型

| Transport | 機制 |
|-----------|------|
| `stdio` | 本地 subprocess via stdin/stdout |
| `sse` | Server-Sent Events |
| `http` | HTTP Streaming |
| `ws` | WebSocket |
| `sse-ide` / `ws-ide` | IDE 橋接變體 |
| `sdk` | In-process SDK server |
| `claudeai-proxy` | Claude AI 代理 |

## 連接管理

**源碼**: `src/services/mcp/client.ts` (119KB)

```
Session 啟動
    ↓
getAllMcpConfigs() — 聚合所有來源的設定
    ↓
connectToServer(config) — 建立 transport 連接
    ├── StdioClientTransport → 本地 subprocess
    ├── SSEClientTransport → SSE 連接
    ├── StreamableHTTPClientTransport → HTTP streaming
    ├── WebSocketTransport → WebSocket
    └── SdkControlTransport → In-process
    ↓
ensureConnectedClient() — 驗證 + 自動重連
    ├── 指數退避重試
    ├── 最多 5 次重連
    ├── 初始退避 1s，最大 30s
    └── 錯誤去重（防重複 toast）
```

## 工具發現與註冊

**源碼**: `src/services/mcp/client.ts:1743`

```typescript
async function fetchToolsForClient(server) {
  // 1. 調用 MCP tools/list endpoint
  const rawTools = await client.listTools()

  // 2. 轉換為內部 Tool 格式
  // 3. 命名前綴: mcp__{serverName}__{toolName}
  // 4. 分類:
  //    - readOnlyHint → isReadOnly()
  //    - destructiveHint → isDestructive()
  //    - openWorldHint → 開放世界偵測
}
```

### 批次發現

```typescript
async function getMcpToolsCommandsAndResources(servers) {
  // 1. 分區: local (低並發) vs remote (高並發)
  // 2. 並行抓取:
  await Promise.all([
    fetchToolsForClient(server),
    fetchCommandsForClient(server),
    fetchResourcesForClient(server),
  ])
  // 3. 快取 auth 失敗 (15 分鐘 TTL)
  // 4. 加入 ListMcpResourcesTool + ReadMcpResourceTool
}
```

## 工具調用流程

```
Model 調用 MCP 工具
    ↓
MCPTool.call()
    ↓
callMCPToolWithUrlElicitationRetry()
    ├── 認證重試邏輯
    ├── Session 過期處理
    └── URL elicitation 支援（OAuth 流程）
    ↓
實際 MCP 工具執行
    ↓
結果處理
    ├── mcpContentNeedsTruncation() — 檢查是否需要截斷
    ├── persistToolResult() — 大結果持久化
    └── 回傳給 model
```

## Elicitation Handler

**源碼**: `src/services/mcp/elicitationHandler.ts` (10KB)

當 MCP 伺服器需要用戶互動時（如 OAuth 授權）：

```
MCP Server → elicit_request → Handler
    ↓
Queue ElicitationRequestEvent in AppState
    ↓
UI 渲染 modal/dialog
    ├── form 模式: 內嵌表單
    └── url 模式: OAuth/確認流程
    ↓
用戶回應
    ↓
Handler → completion notification → MCP Server
```

### Elicitation Hooks

```
Elicitation hook → 程式化回應（不需 UI）
ElicitationResult hook → 回應後處理
```

## AppState 中的 MCP 狀態

```typescript
mcp: {
  clients: MCPServerConnection[]           // 連接狀態
  tools: Tool[]                           // 所有 MCP 工具
  commands: Command[]                     // MCP 命令 + skills
  resources: Record<string, ServerResource[]>  // 可用資源
  pluginReconnectKey: number              // Plugin 重載觸發器
}
```

## 連接生命週期 Hook

**源碼**: `src/services/mcp/useManageMCPConnections.ts` (44.8KB)

```
Config 變更 / Plugin 重載
    ↓
useManageMCPConnections React Hook
    ├── 初始化連接
    ├── 註冊生命週期事件處理器
    ├── 同步連接狀態到 AppState
    ├── 自動重連（指數退避）
    ├── Plugin reconnect key 監控
    └── Auth version 追蹤
```

## 源碼參考

| 檔案 | 大小 | 角色 |
|------|------|------|
| `src/services/mcp/client.ts` | 119KB | MCP 客戶端核心 |
| `src/services/mcp/config.ts` | 51KB | 設定載入/驗證 |
| `src/services/mcp/useManageMCPConnections.ts` | 45KB | 連接管理 Hook |
| `src/services/mcp/auth.ts` | 89KB | 認證 |
| `src/services/mcp/elicitationHandler.ts` | 10KB | Elicitation 處理 |
| `src/services/mcp/types.ts` | 6KB | 型別定義 |
