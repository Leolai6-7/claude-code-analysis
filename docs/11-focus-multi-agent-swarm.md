# 重點二：拆解多 Agent 協作架構（Swarm）

> **核心源碼**: `src/coordinator/`, `src/utils/swarm/`, `src/tools/shared/spawnMultiAgent.ts`
>
> 這是 Anthropic 自己的多 Agent 最佳實踐——如何在給 Agent 自主權的同時保持人類控制。

## 架構總覽

```
┌─────────────────────────────────────────────────┐
│                  Leader Agent                     │
│  (Coordinator Mode - 分配任務、審批權限)          │
│                                                   │
│  Tools: TeamCreateTool, SendMessageTool           │
│  權限: 完整 ToolUseConfirm 對話框                  │
└──────────┬──────────────┬──────────────┬──────────┘
           │              │              │
    ┌──────▼──────┐ ┌────▼──────┐ ┌────▼──────┐
    │  Worker A   │ │  Worker B │ │  Worker C │
    │ (in-process)│ │  (tmux)   │ │ (worktree)│
    │             │ │           │ │           │
    │ 標準工具    │ │ 標準工具   │ │ 標準工具   │
    │ 無法生成    │ │ 無法生成   │ │ 無法生成   │
    │ 新 Worker   │ │ 新 Worker  │ │ 新 Worker  │
    └──────┬──────┘ └─────┬─────┘ └─────┬─────┘
           │              │              │
           └──────────────┼──────────────┘
                          │
              ┌───────────▼───────────┐
              │    Mailbox (檔案系統)   │
              │ ~/.claude/teams/{name}/│
              │  ├── inboxes/          │
              │  ├── permissions/      │
              │  │   ├── pending/      │
              │  │   └── resolved/     │
              │  └── team.json         │
              └───────────────────────┘
```

## Coordinator Mode

**源碼**: `src/coordinator/coordinatorMode.ts`

```typescript
// 啟用條件
export function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE')
    && isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
}
```

**Worker 工具限制**：Worker 不能使用 TeamCreate、TeamDelete、SendMessage、SyntheticOutput——防止 Worker 自己生成新 Worker 或與其他 Worker 直接通訊。

## 權限佇列（Permission Queue）

這是整個 Swarm 系統最精妙的設計。

### 問題
Worker 執行危險操作（如 `rm -rf`）時，需要人類批准。但 Worker 在自己的進程/上下文中，沒有 UI。

### 解決方案：兩目錄佇列

**源碼**: `src/utils/swarm/permissionSync.ts`

```
~/.claude/teams/{teamName}/permissions/
  ├── pending/             # Worker 寫入，Leader 輪詢
  │   └── perm-{uuid}.json
  └── resolved/            # Leader 寫入，Worker 輪詢
      └── perm-{uuid}.json
```

**Permission Request 結構**：

```typescript
SwarmPermissionRequest = {
  id: string                // 唯一請求 ID
  workerId: string          // Worker 的 agent ID
  workerName: string        // Worker 的名稱
  teamName: string          // 團隊名稱（路由用）
  toolName: string          // 需要權限的工具
  toolUseId: string         // 原始 tool_use ID
  description: string       // 人可讀描述
  input: Record<string, unknown>  // 工具輸入
  status: 'pending' | 'approved' | 'rejected'
  feedback?: string         // Leader 的回饋
  updatedInput?: Record     // Leader 可以修改輸入
  permissionUpdates?: []    // 權限規則更新
  createdAt: number
}
```

### 流程

```
Worker 需要權限
    │
    ├── 1. 建立 SwarmPermissionRequest
    ├── 2. 寫入 pending/ 目錄（帶檔案鎖）
    ├── 3. 註冊回調到 pendingCallbacks Map
    ├── 4. 開始輪詢 resolved/ 目錄（每 500ms）
    │
    │   同時...
    │
    ├── Leader 輪詢 pending/ 目錄
    ├── 顯示 ToolUseConfirm 對話框
    ├── 用戶批准/拒絕
    └── Leader 寫入 resolved/ 目錄
            │
            ├── Worker 發現 resolved 檔案
            ├── 匹配 pendingCallbacks
            └── 觸發回調 → 繼續/中止執行
```

## 原子認領機制（createResolveOnce）

**源碼**: `src/hooks/toolPermission/PermissionContext.ts:63-93`

這解決了一個關鍵的競態問題：多個非同步事件（Leader 回應、abort 訊號、classifier 決策）可能同時嘗試解決同一個權限請求。

```typescript
function createResolveOnce<T>(resolve: (value: T) => void): ResolveOnce<T> {
  let claimed = false
  let delivered = false

  return {
    resolve(value: T) {
      if (delivered) return     // 已經 resolve 過了
      delivered = true
      claimed = true
      resolve(value)            // 實際 resolve promise
    },

    isResolved() {
      return claimed
    },

    claim() {
      if (claimed) return false // 已被其他人認領
      claimed = true            // 原子性：只有第一個 caller 回傳 true
      return true
    },
  }
}
```

### 使用模式——三方競爭

```typescript
const { resolve: resolveOnce, claim } = createResolveOnce(resolve)

// 競爭者 1: Leader 回應
registerPermissionCallback({
  onAllow(allowedInput, permissionUpdates) {
    if (!claim()) return  // ← 原子檢查，只有第一個贏
    clearPendingRequest()
    resolveOnce(await ctx.handleUserAllow(...))
  },
  onReject(feedback) {
    if (!claim()) return
    clearPendingRequest()
    resolveOnce(ctx.cancelAndAbort(feedback))
  },
})

// 競爭者 2: 發送請求到 Leader
void sendPermissionRequestViaMailbox(request)

// 競爭者 3: Abort 訊號
ctx.abortController.signal.addEventListener('abort', () => {
  if (!claim()) return  // ← 如果 Leader 已經回應，忽略 abort
  clearPendingRequest()
  resolveOnce(ctx.cancelAndAbort(undefined, true))
}, { once: true })
```

**這是 CAS（Compare-And-Swap）語意**：多個非同步源競爭，但只有第一個 `claim()` 成功的會實際執行。

## Mailbox 系統（檔案 IPC）

**源碼**: `src/utils/teammateMailbox.ts`

Agent 間通訊基於檔案系統：

```
~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

```typescript
type TeammateMessage = {
  from: string        // 發送者名稱
  text: string        // 訊息內容
  timestamp: string   // ISO 時間戳
  read: boolean       // 已讀標記
  color?: string      // 顯示顏色
  summary?: string    // 5-10 字預覽
}
```

### 原子寫入（檔案鎖）

```typescript
async function writeToMailbox(recipientName, message, teamName) {
  const lockFilePath = `${inboxPath}.lock`
  let release = await lockfile.lock(inboxPath, {
    lockfilePath: lockFilePath,
    retries: {
      retries: 10,      // 最多重試 10 次
      minTimeout: 5,     // 最短 5ms
      maxTimeout: 100,   // 最長 100ms
    },
  })

  try {
    const messages = await readMailbox(recipientName)
    messages.push({ ...message, read: false })
    await writeFile(inboxPath, JSON.stringify(messages, null, 2))
  } finally {
    await release()
  }
}
```

## In-Process Teammate

**源碼**: `src/tasks/InProcessTeammateTask/types.ts`

Worker 可以在同一個 Node.js 進程內運行，用 `AsyncLocalStorage` 隔離上下文：

```typescript
type InProcessTeammateTaskState = {
  type: 'in_process_teammate'
  identity: TeammateIdentity     // agentId, agentName, teamName, color
  prompt: string
  model?: string
  permissionMode: PermissionMode
  messages?: Message[]           // 上限 50 條（防記憶體爆炸）
  pendingUserMessages: string[]
  isIdle: boolean
  shutdownRequested: boolean
}
```

**訊息上限 50 條**——來自真實事故：一個 session 在 2 分鐘內生成 292 個 agent，RSS 達到 36.8GB。上限防止每個 agent 持有完整對話副本。

## Team Memory Sync

**源碼**: `src/services/teamMemorySync/index.ts`

跨 Agent 共享知識空間，通過 API 同步：

```typescript
type SyncState = {
  lastKnownChecksum: string | null    // ETag for conditional requests
  serverChecksums: Map<string, string> // 每個 key 的 sha256
  serverMaxEntries: number | null      // 從 413 回應學到的上限
}
```

**同步語意**：
- **Pull**: Server wins per-key（覆蓋本地）
- **Push**: Delta upload（只推 hash 不同的 key）
- **Atomic**: Server 在 entry-count overflow 時 reject 整個 PUT

## SendMessage 結構化訊息

**源碼**: `src/tools/SendMessageTool/SendMessageTool.ts`

```typescript
// 支援三種結構化訊息類型
StructuredMessage = discriminatedUnion('type', [
  { type: 'shutdown_request', reason?: string },
  { type: 'shutdown_response', request_id, approve, reason? },
  { type: 'plan_approval_response', request_id, approve, feedback? },
])
```

**路由**：
- `to: "*"` → 廣播給所有 teammate
- `to: "agent_name"` → 發給特定 agent
- `to: "uds:<socket>"` → 發給本地 peer
- `to: "bridge:<session-id>"` → 發給遠端 session

## 防競態的五個機制

| 機制 | 位置 | 問題 |
|------|------|------|
| `createResolveOnce` (CAS) | PermissionContext.ts | 多個非同步源同時解決同一請求 |
| 檔案鎖 (lockfile) | teammateMailbox.ts | 多個 agent 同時寫入同一 mailbox |
| 兩目錄佇列 | permissionSync.ts | 請求與回應解耦 |
| AsyncLocalStorage | inProcessRunner.ts | 進程內多 agent 上下文隔離 |
| 訊息上限 50 | InProcessTeammateTask | 292 agent 記憶體爆炸防護 |

## 可直接套用的模式

### 模式 1：檔案系統 IPC
不需要 Redis/RabbitMQ，用檔案鎖 + JSON 檔就能做可靠的 agent 間通訊。

### 模式 2：兩目錄佇列
`pending/` + `resolved/` 天然解耦請求者和處理者。

### 模式 3：CAS 原子認領
用一個簡單的 boolean flag 解決多源競爭，比 mutex 輕量得多。

### 模式 4：Leader 審批模式
Worker 自主執行安全操作，危險操作上報 Leader，Leader 顯示 UI 給人類。

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/coordinator/coordinatorMode.ts` | Coordinator 模式定義 |
| `src/utils/swarm/permissionSync.ts` | 權限同步佇列 |
| `src/utils/swarm/inProcessRunner.ts` | In-process 執行器 |
| `src/utils/swarm/backends/InProcessBackend.ts` | In-process 後端 |
| `src/utils/swarm/leaderPermissionBridge.ts` | Leader 權限橋接 |
| `src/utils/teammateMailbox.ts` | Mailbox IPC |
| `src/hooks/toolPermission/PermissionContext.ts` | createResolveOnce 原子認領 |
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | 團隊建立 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | Inter-agent 訊息 |
| `src/tasks/InProcessTeammateTask/types.ts` | In-process agent 狀態 |
| `src/services/teamMemorySync/index.ts` | Team memory 同步 |
