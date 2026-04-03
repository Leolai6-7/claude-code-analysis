# Chapter 4: 多 Agent 協作架構（Swarm）— Coordinator、Mailbox、原子認領

> 當一個 Agent 不夠用，你需要的不是更大的模型，而是更好的協作協議。
> Claude Code 的 Swarm 架構用檔案系統做 IPC、用兩目錄佇列做權限、用 CAS 做原子認領——
> 沒有 Redis，沒有 Kafka，只有 JSON 檔案和精心設計的併發語意。

---

## 4.1 為什麼單一 Agent 不夠

### 4.1.1 單一 Agent 的天花板

想像你請一個 Coding Agent 做這件事：

> 「重構整個認證模組，同時更新所有受影響的測試，並且修改 API 文件。」

一個 Agent 做這件事的流程是：

```
1. 讀取認證模組源碼          (3 秒)
2. 分析依賴關係               (2 秒)
3. 修改認證模組               (5 秒)
4. 讀取第一個測試檔           (1 秒)
5. 修改第一個測試             (3 秒)
6. 讀取第二個測試檔           (1 秒)
7. 修改第二個測試             (3 秒)
... 重複 20 個測試 ...
8. 讀取 API 文件              (1 秒)
9. 修改 API 文件              (3 秒)
────────────────────────────────
                      總計：~90 秒串行
```

問題不在於模型的能力，而在於**執行模型**。這是一個 I/O-bound 的工作，理想狀態下應該三個人同時做：

```
Worker A: 修改認證模組         (10 秒)
Worker B: 修改 20 個測試檔     (30 秒，可並行讀寫不同檔案)
Worker C: 修改 API 文件        (10 秒)
────────────────────────────────
                      總計：~30 秒並行
```

但引入多個 Agent 並不是簡單地「多開幾個 process」。你立刻會遇到五個核心問題：

### 4.1.2 多 Agent 的五個核心挑戰

| # | 挑戰 | 具體場景 | 嚴重程度 |
|---|------|---------|---------|
| 1 | **權限審批** | Worker A 要跑 `rm -rf build/`，誰來批准？Worker 自己沒有 UI | 致命 |
| 2 | **通訊可靠性** | Worker A 改了 `auth.ts`，Worker B 在測試還在用舊版——怎麼通知？ | 高 |
| 3 | **競態條件** | Leader 批准了權限，但 abort 訊號同時到達——到底該執行還是中止？ | 高 |
| 4 | **資源控制** | 2 分鐘內產生 292 個 Agent，RSS 達到 36.8GB | 高 |
| 5 | **狀態一致性** | 三個 Agent 的記憶要同步——一個 Agent 學到的規則，其他人也要知道 | 中 |

Claude Code 的 Swarm 架構對每一個問題都有具體的工程方案。本章會逐一拆解。

### 4.1.3 設計哲學：檔案系統即通訊層

在介紹具體元件之前，先理解 Swarm 最根本的設計選擇：

**用檔案系統做 IPC（Inter-Process Communication）。**

不是 Redis。不是 RabbitMQ。不是 gRPC。是 JSON 檔案 + 檔案鎖。

這個選擇背後的推理：

1. **零依賴**：Claude Code 是 CLI 工具，使用者不該為了多 Agent 去裝 Redis
2. **可檢查性**：所有通訊都是可讀的 JSON 檔，debug 時直接 `cat` 就能看
3. **持久化免費**：檔案天然持久，process crash 後狀態還在
4. **跨進程安全**：`lockfile` 套件提供跨進程檔案鎖，語意清楚

代價是什麼？吞吐量低、延遲高（poll-based, ~500ms）。但對 Agent 協作來說，這完全夠用——Agent 之間的通訊頻率是每秒幾則訊息，不是每秒幾萬則。

---

## 4.2 架構總覽

### 4.2.1 完整架構圖

```
┌──────────────────────────────────────────────────────────────────┐
│                        Human (使用者)                             │
│                     透過 Terminal UI 互動                         │
└───────────────────────────┬──────────────────────────────────────┘
                            │ ToolUseConfirm 對話框
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Leader Agent (Coordinator)                    │
│                                                                   │
│  功能:                                                            │
│  ├── 分析任務、拆分子任務                                          │
│  ├── 派遣 Worker（TeamCreateTool）                                │
│  ├── 傳送訊息（SendMessageTool）                                  │
│  ├── 輪詢 Permission Queue → 顯示 UI → 批准/拒絕                  │
│  └── 同步 Team Memory                                             │
│                                                                   │
│  工具集: TeamCreate, TeamDelete, SendMessage, SyntheticOutput     │
│         + 所有標準工具                                             │
│                                                                   │
│  權限: 完整 ToolUseConfirm（人類審批 UI）                          │
└──────┬──────────────┬──────────────┬──────────────┬──────────────┘
       │              │              │              │
       │    ┌─────────┘              └─────────┐    │
       │    │                                  │    │
       ▼    ▼                                  ▼    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   Worker A    │  │   Worker B    │  │   Worker C    │
│  (in-process) │  │    (tmux)     │  │  (worktree)   │
│               │  │               │  │               │
│ 標準工具集    │  │ 標準工具集    │  │ 標準工具集    │
│ - Read        │  │ - Read        │  │ - Read        │
│ - Write       │  │ - Write       │  │ - Write       │
│ - Bash        │  │ - Bash        │  │ - Bash        │
│ - Grep        │  │ - Grep        │  │ - Grep        │
│               │  │               │  │               │
│ ✗ TeamCreate  │  │ ✗ TeamCreate  │  │ ✗ TeamCreate  │
│ ✗ SendMessage │  │ ✗ SendMessage │  │ ✗ SendMessage │
│ ✗ TeamDelete  │  │ ✗ TeamDelete  │  │ ✗ TeamDelete  │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
              ┌────────────▼────────────┐
              │   Mailbox (檔案系統 IPC)  │
              │                          │
              │  ~/.claude/teams/{name}/ │
              │  ├── inboxes/            │
              │  │   ├── leader.json     │
              │  │   ├── worker-a.json   │
              │  │   ├── worker-b.json   │
              │  │   └── worker-c.json   │
              │  ├── permissions/        │
              │  │   ├── pending/        │  ← Worker 寫入
              │  │   │   └── perm-*.json │
              │  │   └── resolved/       │  ← Leader 寫入
              │  │       └── perm-*.json │
              │  ├── memory/             │  ← Team Memory
              │  └── team.json           │  ← Team 元資料
              └──────────────────────────┘
```

### 4.2.2 資料流

從資料流的角度，整個系統有四條主要管線：

```
管線 1: 任務分派
Leader ──[TeamCreateTool]──→ 產生 Worker process/thread
                              └──→ Worker 開始執行任務

管線 2: 訊息通訊
Agent A ──[writeToMailbox]──→ inboxes/agent-b.json
                              └──→ Agent B 輪詢讀取

管線 3: 權限審批
Worker ──[createPermissionRequest]──→ pending/perm-{uuid}.json
                                          │
Leader ──[pollPendingRequests]──→ 讀取 pending/
         [resolvePermission]──→ resolved/perm-{uuid}.json
                                     │
Worker ──[pollResolvedRequests]──→ 讀取 resolved/ → 觸發回調

管線 4: 記憶同步
Any Agent ──[pushMemory]──→ API Server ──[pullMemory]──→ Other Agents
                              └── ETag conditional request
```

### 4.2.3 三種 Worker 後端

Claude Code 支援三種方式執行 Worker：

| 後端 | 實作 | 隔離等級 | 效能 | 使用場景 |
|------|------|---------|------|---------|
| **In-Process** | `AsyncLocalStorage` 隔離 | 共享記憶體 | 最快（無 IPC 開銷） | 預設，大多數情況 |
| **tmux** | 獨立 tmux session | 進程隔離 | 中等 | 需要獨立 terminal |
| **worktree** | 獨立 git worktree | 進程 + 檔案系統隔離 | 最慢 | 需要修改不同分支 |

每種後端共用同一套 Mailbox 和 Permission Queue——這是架構的優雅之處。

---

## 4.3 Coordinator Mode

### 4.3.1 啟用條件

**源碼**：`src/coordinator/coordinatorMode.ts:36-41`

```typescript
// src/coordinator/coordinatorMode.ts:36-41
export function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE')
    && isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
}
```

兩個條件必須同時滿足：

1. **Feature flag 開啟**：`feature('COORDINATOR_MODE')` — 伺服端控制，用於漸進式發布
2. **環境變數啟用**：`CLAUDE_CODE_COORDINATOR_MODE=1` — 使用者端控制

這是典型的**雙開關模式**：產品團隊可以在伺服端開啟 feature flag 讓功能可用，但使用者仍然需要明確 opt-in。這比單一開關安全得多——即使 feature flag 意外開啟，如果使用者沒有設定環境變數，功能不會生效。

### 4.3.2 Worker 工具限制

**源碼**：`src/coordinator/coordinatorMode.ts:29-34`

```typescript
// src/coordinator/coordinatorMode.ts:29-34
const INTERNAL_WORKER_TOOLS: Set<string> = new Set([
  'TeamCreate',
  'TeamDelete',
  'SendMessage',
  'SyntheticOutput',
])
```

這四個工具被明確禁止在 Worker 中使用：

| 被禁工具 | 禁止原因 |
|---------|---------|
| `TeamCreate` | Worker 不能自己生成 Worker，防止無限遞迴 |
| `TeamDelete` | Worker 不能殺掉其他 Worker |
| `SendMessage` | Worker 不能直接和其他 Worker 通訊，必須透過 Leader 路由 |
| `SyntheticOutput` | Worker 不能偽造輸出 |

**為什麼 Worker 不能直接通訊？**

這是一個關鍵的架構決策。想像如果 Worker A 可以直接發訊息給 Worker B：

```
Worker A: 「嘿 B，幫我跑 rm -rf /」
Worker B: 「收到，執行中...」
```

Leader 完全不知道發生了什麼。所有 Worker 間通訊必須透過 Leader 路由，Leader 就是**中央控制點**。這是 hub-and-spoke 拓撲，不是 mesh 拓撲。

```
✗ 不允許：Mesh 拓撲                  ✓ 允許：Hub-and-Spoke 拓撲

  Worker A ←──→ Worker B               Worker A ←──→ Leader ←──→ Worker B
     ↕              ↕                                  ↕
  Worker C ←──→ Worker D               Worker C ←──→ Leader ←──→ Worker D
                                        (所有通訊都經過 Leader)
```

### 4.3.3 Coordinator System Prompt

**源碼**：`src/coordinator/coordinatorMode.ts:120-368`

Coordinator 的 system prompt 是 Swarm 架構的「行為規格書」。它告訴 Leader Agent 如何管理團隊。以下是 system prompt 的關鍵段落結構：

```
System Prompt 結構（約 248 行）
├── 角色定義
│   └── "You are a coordinator agent that manages a team of worker agents."
│
├── 核心規則
│   ├── 任務分析：先理解任務全貌，再拆分
│   ├── 分派策略：每個 Worker 一個明確的子任務
│   ├── 獨立性：Worker 之間不應有依賴關係
│   └── 最小權限：每個 Worker 只拿到完成任務需要的資訊
│
├── 工具使用指南
│   ├── TeamCreate：何時建立新 Worker
│   ├── SendMessage：何時與 Worker 溝通
│   ├── 權限處理：當 Worker 請求權限時的行為
│   └── 訊息路由：如何轉發 Worker 之間的訊息
│
├── 錯誤處理
│   ├── Worker 超時：如何偵測並處理
│   ├── Worker 失敗：重試策略
│   └── 權限被拒：通知 Worker 修改方案
│
└── 結束條件
    ├── 所有 Worker 完成：匯總結果回報人類
    ├── 部分失敗：回報成功與失敗的部分
    └── 全部失敗：回報原因，建議替代方案
```

幾個特別值得注意的設計：

**任務拆分指導原則**：

```
Prompt 中的關鍵指令：

1. "Break the task into independent, parallelizable subtasks."
   → 強調獨立性。如果 Worker B 需要等 Worker A 的輸出，
     就不應該拆成兩個 Worker。

2. "Each worker should have a clear, specific objective."
   → 不是「幫我寫 code」，而是「修改 src/auth/login.ts，
     將 JWT 驗證邏輯從 synchronous 改為 async」。

3. "Workers cannot communicate directly with each other."
   → 明確告訴 LLM 架構限制，避免它嘗試讓 Worker 互傳訊息。

4. "When a worker requests permission, present it to the user
    with context about what the worker is trying to do."
   → Leader 不能盲目轉發權限請求，必須加上上下文。
```

**權限處理的特殊邏輯**：

Coordinator prompt 中有大段篇幅處理「當 Worker 需要權限時」的行為。這是因為 Worker 沒有 UI，它們的權限請求必須由 Leader「翻譯」給人類看。

```
Worker 權限請求 → Leader 接收
    │
    ├── 加上上下文：「Worker A (負責重構認證) 想要執行 rm -rf build/」
    ├── 顯示在 Leader 的 ToolUseConfirm UI 中
    ├── 人類批准/拒絕
    └── Leader 將結果寫回 resolved/ 目錄
```

### 4.3.4 Coordinator 的完整生命週期

把 Coordinator Mode 啟動後的完整流程串起來：

```
Phase 1: 接收任務
━━━━━━━━━━━━━━━━
User: 「重構認證模組，更新測試和文件」
  │
  ▼
Leader 分析任務 → 識別三個可並行子任務

Phase 2: 派遣 Worker
━━━━━━━━━━━━━━━━━━━
Leader 呼叫 TeamCreateTool × 3
  │
  ├── Worker A: prompt = "重構 src/auth/*.ts 的認證邏輯"
  ├── Worker B: prompt = "更新 tests/auth/*.test.ts"
  └── Worker C: prompt = "更新 docs/auth.md"

  每個 Worker 啟動後立即開始工作。

Phase 3: 監控與協調
━━━━━━━━━━━━━━━━━━
Leader 進入事件迴圈：
  │
  while (有 Worker 仍在執行) {
  │
  ├── 輪詢 Mailbox：有沒有 Worker 發訊息？
  │   └── Worker B: 「auth.ts 的 interface 改了嗎？」
  │       └── Leader 轉發給 Worker A，或自己回答
  │
  ├── 輪詢 Permission Queue：有沒有需要審批的？
  │   └── Worker A: 「要 bash rm -rf build/」
  │       └── Leader 顯示 UI → 人類批准 → 寫入 resolved/
  │
  └── 檢查 Worker 狀態：有沒有超時或失敗？
  }

Phase 4: 匯總結果
━━━━━━━━━━━━━━━━
Leader 收到所有 Worker 完成通知
  │
  ├── 匯總：「認證模組已重構，20 個測試已更新，文件已同步」
  └── 回報給 User
```

---

## 4.4 Permission Queue — 最精巧的設計

Permission Queue 是整個 Swarm 架構中最值得深入研究的子系統。它解決了一個本質上困難的問題：**無 UI 的 Worker 如何安全地獲取人類授權。**

### 4.4.1 問題定義

```
┌─────────────────────────────────────────┐
│             Worker Process               │
│                                          │
│  Agent 想執行: bash("rm -rf build/")     │
│                                          │
│  Permission 系統判定: 需要人類批准        │
│                                          │
│  但是... Worker 沒有 Terminal UI！       │
│  它不能彈出確認對話框。                    │
│  它甚至可能不在同一個進程裡。              │
│                                          │
│  怎麼辦？                                 │
└─────────────────────────────────────────┘
```

傳統方案的問題：

| 方案 | 問題 |
|------|------|
| RPC 呼叫 Leader | Leader 和 Worker 可能不在同一進程，甚至不在同一台機器 |
| 共享記憶體 | 只適用於 in-process，tmux/worktree 不行 |
| WebSocket | 需要持久連線，增加複雜度 |
| 資料庫 | 需要外部依賴 |

Claude Code 的方案：**檔案系統兩目錄佇列**。

### 4.4.2 兩目錄佇列

**源碼**：`src/utils/swarm/permissionSync.ts`

```
~/.claude/teams/{teamName}/permissions/
├── pending/                    ← Worker 寫入，Leader 讀取
│   ├── perm-a1b2c3d4.json    ← Worker A 的權限請求
│   ├── perm-e5f6g7h8.json    ← Worker B 的權限請求
│   └── perm-i9j0k1l2.json    ← Worker C 的權限請求
│
└── resolved/                   ← Leader 寫入，Worker 讀取
    ├── perm-a1b2c3d4.json    ← Worker A 的請求已批准
    └── perm-e5f6g7h8.json    ← Worker B 的請求已拒絕
```

**為什麼是兩個目錄，而不是一個？**

如果只有一個目錄，Worker 寫入 `status: 'pending'`，Leader 把它改成 `status: 'approved'`——這需要 Leader 對 Worker 的檔案做 read-modify-write。多個 Agent 同時操作同一個檔案，即使有鎖，邏輯也很複雜。

兩個目錄的好處：

```
                pending/              resolved/
Worker 寫入:    ✓ (只寫，不讀)         ✓ (只讀，不寫)
Leader 寫入:    ✗ (只讀，不寫)         ✓ (只寫，不讀)
                ↑                      ↑
                單向寫入               單向寫入
                沒有競爭               沒有競爭
```

每個目錄只有一個寫入者，讀取者可以有多個。這是 **SPSC（Single-Producer Single-Consumer）佇列** 的變形——寫入端永遠只有一方，讀取端輪詢即可。

### 4.4.3 Permission Request Schema

**源碼**：`src/utils/swarm/permissionSync.ts:49-86`

```typescript
// src/utils/swarm/permissionSync.ts:49-86
// SwarmPermissionRequestSchema 的完整結構

const SwarmPermissionRequestSchema = z.object({
  // === 識別欄位 ===
  id: z.string(),                    // UUID，全局唯一
  workerId: z.string(),              // Worker 的 agent ID
  workerName: z.string(),            // Worker 的人類可讀名稱
  teamName: z.string(),              // 團隊名稱（決定檔案路徑）

  // === 工具欄位 ===
  toolName: z.string(),              // 需要權限的工具名稱
  toolUseId: z.string(),             // 原始 tool_use block 的 ID
  description: z.string(),           // 人類可讀的操作描述
  input: z.record(z.unknown()),      // 工具的完整輸入參數

  // === 狀態欄位 ===
  status: z.enum([
    'pending',                       // 等待 Leader 處理
    'approved',                      // 已批准
    'rejected',                      // 已拒絕
  ]),

  // === Leader 回應欄位 ===
  feedback: z.string().optional(),           // Leader/人類的文字回饋
  updatedInput: z.record(z.unknown())        // Leader 可以修改輸入！
    .optional(),                             // 例：把 "rm -rf /" 改成 "rm -rf build/"
  permissionUpdates: z.array(                // 權限規則更新
    z.object({                               // 例：「以後允許這個 Worker
      toolName: z.string(),                  //   對 build/ 目錄執行 rm」
      permission: z.enum(['allow', 'deny']), // → 避免重複詢問
    })
  ).optional(),

  // === 時間戳 ===
  createdAt: z.number(),             // Unix timestamp，用於超時檢測
})
```

幾個值得注意的設計：

**1. `updatedInput` — Leader 可以修改 Worker 的輸入**

這不只是 approve/reject 的二元決策。Leader 可以**修改 Worker 想做的事**：

```
Worker 請求: bash("rm -rf /")
Leader 回應: approved, updatedInput = { command: "rm -rf build/" }
             ↑
             Leader 修正了命令，而不是拒絕
```

這讓人類有更細緻的控制——不是只能說「可以」或「不行」，還能說「可以，但改一下」。

**2. `permissionUpdates` — 動態權限規則更新**

每次批准都可以附帶一條規則：

```json
{
  "permissionUpdates": [
    {
      "toolName": "Bash",
      "permission": "allow"
    }
  ]
}
```

效果：「以後 Worker A 執行 Bash 命令不用再問我了。」這避免了反覆審批相同類型的操作。

**3. `createdAt` — 超時檢測**

如果一個 pending 請求超過一定時間（例如 5 分鐘）沒有 resolved，系統可以判定 Leader 可能已經 crash，並做相應處理。

### 4.4.4 createPermissionRequest() 函數

**源碼**：`src/utils/swarm/permissionSync.ts:167-207`

```typescript
// src/utils/swarm/permissionSync.ts:167-207
export async function createPermissionRequest(
  teamName: string,
  request: Omit<SwarmPermissionRequest, 'id' | 'createdAt' | 'status'>
): Promise<SwarmPermissionRequest> {

  // 1. 組裝完整的 request 物件
  const fullRequest: SwarmPermissionRequest = {
    ...request,
    id: randomUUID(),           // 生成唯一 ID
    status: 'pending',          // 初始狀態
    createdAt: Date.now(),      // 時間戳
  }

  // 2. 確保 pending 目錄存在
  const pendingDir = path.join(
    getTeamDir(teamName),
    'permissions',
    'pending'
  )
  await fs.mkdir(pendingDir, { recursive: true })

  // 3. 計算檔案路徑
  const filePath = path.join(pendingDir, `perm-${fullRequest.id}.json`)

  // 4. 帶檔案鎖寫入
  const lockPath = `${filePath}.lock`
  const release = await lockfile.lock(filePath, {
    lockfilePath: lockPath,
    retries: {
      retries: 5,               // 最多重試 5 次
      minTimeout: 10,           // 最短等待 10ms
      maxTimeout: 200,          // 最長等待 200ms
      factor: 2,                // 指數退避：10, 20, 40, 80, 160
    },
  })

  try {
    // 5. 寫入 JSON 檔案
    await fs.writeFile(
      filePath,
      JSON.stringify(fullRequest, null, 2),
      'utf-8'
    )
  } finally {
    // 6. 釋放鎖（finally 確保一定釋放）
    await release()
  }

  return fullRequest
}
```

**檔案鎖的重試退避策略**：

```
重試 1: 等待  10ms
重試 2: 等待  20ms   (10 × 2)
重試 3: 等待  40ms   (20 × 2)
重試 4: 等待  80ms   (40 × 2)
重試 5: 等待 160ms   (80 × 2)
────────────────────
最壞情況總等待: 310ms
```

這是指數退避（exponential backoff），它解決了一個微妙的問題：如果多個 Worker 同時建立權限請求，它們不會全部在同一時間重試（因為隨機偏移），而是會逐漸分散。

### 4.4.5 完整的權限流程圖

把 Permission Queue 的完整流程視覺化：

```
時間 ──────────────────────────────────────────────────────→

Worker A                    檔案系統                    Leader
────────                    ────────                    ──────
    │                                                      │
    │ 呼叫 bash("rm -rf")                                  │
    │                                                      │
    │ Permission 系統攔截                                    │
    │                                                      │
    │ createPermissionRequest()                             │
    │──── 寫入 ───→ pending/perm-abc.json                   │
    │                     │                                │
    │ 註冊回調到            │                                │
    │ pendingCallbacks     │                                │
    │                     │           pollPendingRequests() │
    │ 開始輪詢             │ ←──── 讀取 ────────────────────│
    │ resolved/            │                                │
    │                     │           顯示 ToolUseConfirm   │
    │    (等待中...)       │           UI 給人類              │
    │                     │                                │
    │    (等待中...)       │           人類點擊 [Allow]      │
    │                     │                                │
    │                     │           resolvePermission()   │
    │                     resolved/perm-abc.json ←── 寫入 ──│
    │                     │ { status: 'approved' }          │
    │                     │                                │
    │ 發現 resolved 檔案！  │                                │
    │ ←──── 讀取 ──────────│                                │
    │                                                      │
    │ 觸發回調                                               │
    │ → 繼續執行 bash("rm -rf")                              │
    ▼                                                      ▼
```

---

## 4.5 createResolveOnce — 原子認領機制

### 4.5.1 問題：三方競爭

Permission Queue 解決了 Worker-Leader 之間的通訊問題。但還有一個更微妙的問題：**同一個權限請求可能被多個非同步事件同時解決。**

```
                    ┌─── Leader 回應「approved」
                    │
Permission Request ─┼─── Abort 訊號（使用者按了 Ctrl+C）
                    │
                    └─── Permission Classifier 自動判定
```

這三個事件是非同步的，任何一個都可能先到達。如果沒有原子性保證，可能發生：

```
時間 0ms: Leader 回應 approved → 開始執行操作
時間 1ms: Abort 訊號到達 → 中止操作
結果：操作開始了但又被中止 → 不一致狀態
```

或者更糟：

```
時間 0ms: Leader 回應 approved → resolve promise (繼續執行)
時間 1ms: Abort 訊號到達 → resolve promise 再次 (???)
結果：promise 被 resolve 兩次 → 未定義行為
```

### 4.5.2 解決方案：CAS（Compare-And-Swap）語意

**源碼**：`src/hooks/toolPermission/PermissionContext.ts:63-93`

```typescript
// src/hooks/toolPermission/PermissionContext.ts:63-93
// 完整源碼，一字不改

type ResolveOnce<T> = {
  resolve(value: T): void
  isResolved(): boolean
  claim(): boolean
}

function createResolveOnce<T>(
  resolve: (value: T) => void
): ResolveOnce<T> {
  let claimed = false
  let delivered = false

  return {
    resolve(value: T) {
      if (delivered) return       // 已經 deliver 過，忽略
      delivered = true
      claimed = true
      resolve(value)              // 實際 resolve 底層 promise
    },

    isResolved() {
      return claimed
    },

    claim() {
      if (claimed) return false   // 已被認領，回傳 false
      claimed = true              // 原子認領
      return true                 // 回傳 true，表示認領成功
    },
  }
}
```

### 4.5.3 為什麼這是 CAS

CAS（Compare-And-Swap）是並發程式設計的基本操作：

```
CAS(memory_location, expected_value, new_value):
  if memory_location == expected_value:
    memory_location = new_value
    return true
  else:
    return false
```

`claim()` 的語意完全對應 CAS：

```typescript
claim():
  if claimed == false:    // Compare: 是不是還沒被認領？
    claimed = true        // Swap: 設為已認領
    return true           // 認領成功
  else:
    return false          // 已被其他人認領
```

**為什麼在 JavaScript 單線程模型中還需要 CAS？**

JavaScript 是單線程的，但它有**非同步**。考慮這個時序：

```
microtask queue: [leaderCallback, abortCallback]

1. Event loop 取出 leaderCallback
2. leaderCallback 呼叫 claim() → true（成功）
3. leaderCallback 呼叫 resolveOnce(approved)
4. Event loop 取出 abortCallback
5. abortCallback 呼叫 claim() → false（已被認領）
6. abortCallback 什麼都不做
```

雖然 JavaScript 不會真的「同時」執行兩個 callback，但兩個 callback 可能在同一個 microtask batch 中排隊等待。`claim()` 確保無論排隊順序如何，只有第一個 caller 能成功。

### 4.5.4 三方競爭的完整程式碼

**源碼**：`src/hooks/toolPermission/PermissionContext.ts`

```typescript
// 建立 promise 和 resolveOnce 包裝器
const { promise, resolve } = Promise.withResolvers<PermissionResult>()
const resolveOnce = createResolveOnce(resolve)

// ═══════════════════════════════════════════
// 競爭者 1: Leader 回應（via Permission Queue）
// ═══════════════════════════════════════════
registerPermissionCallback(request.id, {
  onAllow(allowedInput, permissionUpdates) {
    if (!resolveOnce.claim()) return    // ← CAS：只有第一個贏

    clearPendingRequest(request.id)

    // 如果 Leader 修改了輸入，用新的輸入
    const finalInput = allowedInput ?? request.input

    // 如果 Leader 附帶了權限規則更新，套用它
    if (permissionUpdates) {
      applyPermissionUpdates(permissionUpdates)
    }

    resolveOnce.resolve(
      await ctx.handleUserAllow(finalInput, permissionUpdates)
    )
  },

  onReject(feedback) {
    if (!resolveOnce.claim()) return    // ← CAS

    clearPendingRequest(request.id)
    resolveOnce.resolve(
      ctx.cancelAndAbort(feedback)
    )
  },
})

// ═══════════════════════════════════════════
// 競爭者 2: 發送請求到 Leader
// ═══════════════════════════════════════════
// void 表示不等待這個 promise——它是「射後不理」(fire-and-forget)
void sendPermissionRequestViaMailbox(request)

// ═══════════════════════════════════════════
// 競爭者 3: Abort 訊號（使用者按了 Ctrl+C）
// ═══════════════════════════════════════════
ctx.abortController.signal.addEventListener('abort', () => {
  if (!resolveOnce.claim()) return      // ← CAS

  clearPendingRequest(request.id)
  resolveOnce.resolve(
    ctx.cancelAndAbort(undefined, /* isAbort */ true)
  )
}, { once: true })  // once: true 確保 listener 只觸發一次

// 等待三方競爭的結果
return await promise
```

**時序分析——三種可能的情況**：

```
情況 A: Leader 先回應
───────────────────
  t=0    Worker 建立 request，開始輪詢
  t=200  Leader 讀到 pending，顯示 UI
  t=500  人類按下 Allow
  t=501  onAllow() → claim() → true ✓ → resolve(approved)
  t=502  abort listener 還在 → 不會觸發

  結果：操作被批准並執行 ✓


情況 B: 使用者按 Ctrl+C
──────────────────────
  t=0    Worker 建立 request，開始輪詢
  t=100  使用者按 Ctrl+C
  t=101  abort event → claim() → true ✓ → resolve(cancelled)
  t=300  Leader 讀到 pending，但...
  t=500  onAllow() → claim() → false ✗ → return（不做任何事）

  結果：操作被安全取消 ✓


情況 C: Classifier 搶先判定
────────────────────────────
  t=0    Worker 建立 request
  t=10   Classifier 分析 toolName + input → 自動批准
  t=11   onAllow() → claim() → true ✓ → resolve(approved)
  t=200  Leader 讀到 pending，但已經 resolved

  結果：安全操作被快速自動批准 ✓
```

### 4.5.5 為什麼不用 Mutex？

你可能會問：為什麼不直接用 Mutex 或 Semaphore？

```
// 用 Mutex 的版本（更重量級）
const mutex = new Mutex()

async function handlePermissionResponse(value) {
  const release = await mutex.acquire()  // 可能會阻塞
  try {
    if (!resolved) {
      resolved = true
      resolve(value)
    }
  } finally {
    release()
  }
}
```

差異：

| 特性 | createResolveOnce (CAS) | Mutex |
|------|------------------------|-------|
| 阻塞 | 永不阻塞 | 可能阻塞 |
| 記憶體 | 2 個 boolean | Mutex 物件 + 等待佇列 |
| 語意 | 嘗試認領，失敗就放棄 | 等待輪到我 |
| 適用場景 | 多方競爭，只需一個贏家 | 需要所有方都執行（排隊） |

在 Permission 場景中，CAS 是更好的選擇——因為我們不需要讓失敗的 caller 排隊等待，我們需要的是「只有一個贏家，其他人直接忽略」。

---

## 4.6 Mailbox 系統

### 4.6.1 概念

Mailbox 是 Agent 之間的異步通訊通道。每個 Agent 有一個 inbox 檔案，其他 Agent 可以往裡面寫訊息。

**源碼**：`src/utils/teammateMailbox.ts`

```
~/.claude/teams/{team_name}/inboxes/
├── leader.json          ← Leader 的收件箱
├── worker-auth.json     ← Worker A 的收件箱
├── worker-test.json     ← Worker B 的收件箱
└── worker-docs.json     ← Worker C 的收件箱
```

### 4.6.2 TeammateMessage 型別

```typescript
// src/utils/teammateMailbox.ts

type TeammateMessage = {
  from: string          // 發送者名稱，例如 "leader" 或 "worker-auth"
  text: string          // 訊息內容（純文字或 JSON 字串）
  timestamp: string     // ISO 8601 時間戳，例如 "2025-01-15T10:30:00Z"
  read: boolean         // 已讀標記，收件者讀取後設為 true
  color?: string        // 顯示顏色，用於 TUI 區分不同 Agent
  summary?: string      // 5-10 字的訊息摘要，用於 UI 預覽
}
```

**為什麼有 `summary` 欄位？**

Agent 的 inbox 可能有很多未讀訊息。在 TUI（Terminal UI）中顯示完整內容會佔太多空間。`summary` 讓 UI 可以顯示：

```
┌─ Inbox ──────────────────────────────────┐
│ [worker-auth] 認證邏輯已重構完成           │
│ [worker-test] 測試全部通過                 │
│ [worker-docs] 需要確認 API 端點名稱        │  ← 這是 summary
└──────────────────────────────────────────┘
```

### 4.6.3 writeToMailbox() — 帶鎖的原子寫入

```typescript
// src/utils/teammateMailbox.ts

async function writeToMailbox(
  recipientName: string,
  message: TeammateMessage,
  teamName: string
): Promise<void> {

  // 1. 計算 inbox 檔案路徑
  const inboxPath = path.join(
    getTeamDir(teamName),
    'inboxes',
    `${recipientName}.json`
  )

  // 2. 確保目錄存在
  await fs.mkdir(path.dirname(inboxPath), { recursive: true })

  // 3. 取得檔案鎖
  const lockFilePath = `${inboxPath}.lock`
  let release = await lockfile.lock(inboxPath, {
    lockfilePath: lockFilePath,
    retries: {
      retries: 10,        // 最多重試 10 次
      minTimeout: 5,       // 最短等待 5ms
      maxTimeout: 100,     // 最長等待 100ms
    },
  })

  try {
    // 4. 讀取現有訊息
    const messages = await readMailbox(recipientName, teamName)

    // 5. 追加新訊息
    messages.push({
      ...message,
      read: false,          // 新訊息預設未讀
    })

    // 6. 寫回整個陣列
    await fs.writeFile(
      inboxPath,
      JSON.stringify(messages, null, 2),
      'utf-8'
    )
  } finally {
    // 7. 釋放鎖
    await release()
  }
}
```

### 4.6.4 為什麼是 Read-Modify-Write 而不是 Append？

你可能想問：為什麼不直接 append 到檔案尾部？那樣連鎖都不需要。

```
// 簡單 append（不需要鎖）
await fs.appendFile(inboxPath, JSON.stringify(message) + '\n')
```

原因是**JSON 格式不支援 append**。一個有效的 JSON 陣列是 `[{...}, {...}]`，你不能直接在 `]` 後面加 `,{...}]`。

你可以改用 NDJSON（每行一個 JSON），但那會失去：
1. 易讀性（`cat inbox.json | jq` 不能用了）
2. `read` 標記更新（需要修改已有的 entry）

所以 Claude Code 選擇了 read-modify-write + 檔案鎖，犧牲一點效能換取資料格式的清晰。

### 4.6.5 鎖的重試策略

```
retries: 10, minTimeout: 5ms, maxTimeout: 100ms

重試時間序列（指數退避 + 抖動）：
  嘗試 1:   5ms
  嘗試 2:  10ms
  嘗試 3:  20ms
  嘗試 4:  40ms
  嘗試 5:  80ms
  嘗試 6: 100ms  ← 到達 maxTimeout，不再增長
  嘗試 7: 100ms
  嘗試 8: 100ms
  嘗試 9: 100ms
  嘗試 10: 100ms
  ────────────────
  最壞情況: ~655ms
```

10 次重試、最長 100ms。這意味著如果鎖被持有超過 ~655ms（很不正常——寫一個 JSON 檔不應超過 10ms），就會放棄。這是一個合理的超時：

- 正常情況下，鎖在第一次就能取得（<5ms）
- 有輕微競爭時，1-3 次重試就能取得（<35ms）
- 嚴重競爭（不太可能，因為 Agent 數量有限）時，仍然在 1 秒內完成
- 如果超時，代表有 bug（例如進程 crash 但鎖沒釋放），需要錯誤處理

### 4.6.6 Mailbox vs Permission Queue 的差異

| 特性 | Mailbox | Permission Queue |
|------|---------|------------------|
| 用途 | 通用訊息傳遞 | 權限審批 |
| 資料結構 | 一個 JSON 陣列（所有訊息） | 每個請求一個 JSON 檔案 |
| 寫入模式 | Read-Modify-Write | Write-Only（新建檔案） |
| 讀取模式 | 讀取整個陣列 | 掃描目錄中的檔案 |
| 回應機制 | 寫入對方的 Mailbox | 寫入 resolved/ 目錄 |
| 關聯性 | 無（每則訊息獨立） | request ↔ response 配對 |

---

## 4.7 In-Process Teammate

### 4.7.1 概念

大多數情況下，Worker 不需要獨立的進程。在同一個 Node.js 進程內用 `AsyncLocalStorage` 隔離上下文就夠了。這省去了 IPC 的開銷，但也帶來了記憶體共享的風險。

**源碼**：`src/tasks/InProcessTeammateTask/types.ts`

### 4.7.2 InProcessTeammateTaskState 型別

```typescript
// src/tasks/InProcessTeammateTask/types.ts

type InProcessTeammateTaskState = {
  type: 'in_process_teammate'          // 型別標記（區分其他 task type）

  // 身份資訊
  identity: TeammateIdentity           // { agentId, agentName, teamName, color }

  // 任務資訊
  prompt: string                       // Leader 給的任務描述
  model?: string                       // 可以指定不同的模型
  permissionMode: PermissionMode       // 權限模式（通常是 swarm-worker）

  // 訊息狀態
  messages?: Message[]                 // 對話歷史，上限 50 條
  pendingUserMessages: string[]        // 尚未處理的傳入訊息

  // 生命週期
  isIdle: boolean                      // 是否閒置（等待新訊息）
  shutdownRequested: boolean           // 是否收到關閉請求
}
```

### 4.7.3 50 條訊息上限：從真實事故學到的教訓

```typescript
// src/tasks/InProcessTeammateTask/types.ts

const TEAMMATE_MESSAGES_UI_CAP = 50
```

這個數字看起來很小，但它背後有一個真實的事故紀錄。

**事故回顧**：

```
時間軸：
  00:00  使用者啟動一個 Coordinator 任務
  00:10  Leader 產生了 5 個 Worker
  00:30  每個 Worker 又嘗試產生更多 Worker（bug：INTERNAL_WORKER_TOOLS 限制沒生效）
  01:00  Worker 數量：50
  01:30  Worker 數量：150
  02:00  Worker 數量：292

記憶體：
  00:00  RSS: 400MB（基線）
  00:30  RSS: 2.1GB
  01:00  RSS: 8.4GB
  01:30  RSS: 22.3GB
  02:00  RSS: 36.8GB ← Node.js 被 OOM killer 殺掉

根本原因分析：
  每個 Agent 持有完整的對話歷史副本
  假設每個 Agent 平均 200 則訊息 × 5KB/則 = 1MB/Agent
  292 個 Agent × 1MB = 292MB 光是訊息
  加上 LLM 回應快取、工具輸出 → 每個 Agent 實際占用 ~120MB
  292 × 120MB ≈ 34.2GB（與觀測值 36.8GB 吻合）
```

**修復方案**：

```
修復 1：INTERNAL_WORKER_TOOLS 限制（根本原因）
  → Worker 不能產生新 Worker
  → 防止無限遞迴

修復 2：TEAMMATE_MESSAGES_UI_CAP = 50（防禦性措施）
  → 即使 bug 導致大量 Agent，每個 Agent 的記憶體佔用被限制
  → 50 條 × 5KB = 250KB/Agent（可接受）

修復 3：appendCappedMessage() 函數
  → 當訊息超過 50 條時，丟棄最舊的訊息
  → 類似 ring buffer 的行為
```

### 4.7.4 appendCappedMessage() 函數

```typescript
// src/tasks/InProcessTeammateTask/types.ts

function appendCappedMessage(
  state: InProcessTeammateTaskState,
  message: Message
): Message[] {
  const messages = state.messages ?? []
  messages.push(message)

  // 如果超過上限，從頭部移除
  if (messages.length > TEAMMATE_MESSAGES_UI_CAP) {
    // 保留最近的 TEAMMATE_MESSAGES_UI_CAP 則訊息
    return messages.slice(-TEAMMATE_MESSAGES_UI_CAP)
  }

  return messages
}
```

**為什麼丟棄最舊的，而不是壓縮？**

壓縮（如 context compression）需要 LLM 呼叫，成本高。對 Worker 來說，它的任務是短期的、聚焦的。最舊的訊息通常是任務初始化的上下文，中間的執行過程才是重要的。而且 Worker 的 prompt 已經包含了完整的任務描述，所以即使最舊的訊息被丟棄，Worker 仍然知道自己在做什麼。

### 4.7.5 AsyncLocalStorage 隔離

In-Process Teammate 使用 Node.js 的 `AsyncLocalStorage` 隔離每個 Agent 的上下文：

```typescript
// 概念程式碼

import { AsyncLocalStorage } from 'node:async_hooks'

const agentContext = new AsyncLocalStorage<AgentContext>()

// 每個 Agent 在自己的 context 中執行
function runAgent(identity: TeammateIdentity, task: () => Promise<void>) {
  const ctx: AgentContext = {
    agentId: identity.agentId,
    agentName: identity.agentName,
    teamName: identity.teamName,
    messages: [],
    abortController: new AbortController(),
  }

  // AsyncLocalStorage.run() 確保所有非同步操作
  // 都在同一個 context 中執行
  return agentContext.run(ctx, task)
}

// 任何程式碼都可以取得目前 Agent 的 context
function getCurrentAgent(): AgentContext {
  return agentContext.getStore()!
}
```

`AsyncLocalStorage` 的關鍵特性：

```
Agent A 的 context          Agent B 的 context
        │                           │
        ▼                           ▼
  async function() {          async function() {
    getCurrentAgent()           getCurrentAgent()
    → { agentId: "A" }         → { agentId: "B" }

    await doSomething()         await doSomething()
    // 即使在 await 之後        // 即使在 await 之後
    // context 仍然正確          // context 仍然正確

    getCurrentAgent()           getCurrentAgent()
    → { agentId: "A" } ✓       → { agentId: "B" } ✓
  }
```

即使兩個 Agent 的非同步操作在 event loop 中交錯執行，`AsyncLocalStorage` 仍然能正確追蹤每個操作屬於哪個 Agent。

---

## 4.8 SendMessage 結構化訊息

### 4.8.1 三種結構化訊息類型

**源碼**：`src/tools/SendMessageTool/SendMessageTool.ts`

Agent 之間的訊息不只是純文字。SendMessage 工具支援結構化訊息，用於特定的協作協議：

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts

const StructuredMessage = z.discriminatedUnion('type', [

  // 類型 1: 關閉請求
  z.object({
    type: z.literal('shutdown_request'),
    reason: z.string().optional(),       // 為什麼要關閉
  }),

  // 類型 2: 關閉回應
  z.object({
    type: z.literal('shutdown_response'),
    request_id: z.string(),              // 對應哪個 shutdown_request
    approve: z.boolean(),                // 是否同意關閉
    reason: z.string().optional(),       // 原因
  }),

  // 類型 3: 計畫審批回應
  z.object({
    type: z.literal('plan_approval_response'),
    request_id: z.string(),              // 對應哪個 plan_approval 請求
    approve: z.boolean(),                // 是否批准
    feedback: z.string().optional(),     // 回饋意見
  }),
])
```

**為什麼需要結構化訊息？**

純文字訊息的問題：

```
Worker → Leader: 「我做完了，可以關閉了嗎？」
Leader 需要做 NLP 來理解這是一個 shutdown 請求
→ 不可靠、延遲高、可能誤判
```

結構化訊息的好處：

```
Worker → Leader: { type: "shutdown_request", reason: "任務完成" }
Leader 直接 switch(message.type) → 處理
→ 可靠、快速、無歧義
```

### 4.8.2 訊息路由

SendMessage 的 `to` 欄位支援四種路由模式：

```typescript
// 路由模式 1: 廣播
{ to: "*", text: "所有 Worker 請暫停" }
// → 寫入每個 teammate 的 inbox

// 路由模式 2: 指名傳送
{ to: "worker-auth", text: "認證模組重構完了嗎？" }
// → 只寫入 worker-auth 的 inbox

// 路由模式 3: UDS（Unix Domain Socket）
{ to: "uds:/tmp/claude-peer.sock", text: "..." }
// → 透過 Unix socket 發送，用於本地 peer-to-peer

// 路由模式 4: Bridge（遠端 session）
{ to: "bridge:session-abc123", text: "..." }
// → 透過 bridge 發送到遠端 Claude Code session
```

路由決策的流程：

```
to 欄位的值
    │
    ├── "*"              → 遍歷所有 teammate，逐一 writeToMailbox
    │
    ├── "agent_name"     → 查找 team.json 中的 agent 列表
    │                      → writeToMailbox(agent_name, ...)
    │
    ├── "uds:..."        → 解析 socket 路徑
    │                      → net.connect(socketPath) → 寫入
    │
    └── "bridge:..."     → 解析 session ID
                           → 透過 API relay 發送到遠端
```

### 4.8.3 訊息生命週期

```
發送方                    檔案系統                    接收方
────────                  ────────                    ────────
    │                                                    │
    │ SendMessage({                                      │
    │   to: "worker-auth",                               │
    │   text: "進度如何？",                               │
    │   summary: "詢問進度"                               │
    │ })                                                 │
    │                                                    │
    │ writeToMailbox()                                    │
    │──→ 取鎖 ──→ 讀取 inbox ──→ append ──→ 寫回 ──→ 解鎖 │
    │                                                    │
    │                    inboxes/worker-auth.json         │
    │                    [{                               │
    │                      from: "leader",                │
    │                      text: "進度如何？",             │
    │                      timestamp: "...",              │
    │                      read: false,    ← 未讀         │
    │                      summary: "詢問進度"             │
    │                    }]                               │
    │                                                    │
    │                              pollMailbox()          │
    │                    ←──────── 讀取 inbox ────────────│
    │                                                    │
    │                              發現新訊息！            │
    │                              注入到 Agent 的對話中   │
    │                              標記 read: true         │
    │                                                    │
    │                    inboxes/worker-auth.json         │
    │                    [{                               │
    │                      from: "leader",                │
    │                      text: "進度如何？",             │
    │                      read: true,     ← 已讀         │
    │                      ...                            │
    │                    }]                               │
    │                                                    │
    ▼                                                    ▼
```

---

## 4.9 Team Memory Sync

### 4.9.1 問題

多個 Agent 在處理同一個專案時，會各自學到東西：

- Worker A 發現 `tsconfig.json` 有特殊設定
- Worker B 發現測試需要特定的 env 變數
- Worker C 發現文件格式有特定規範

如果這些知識只存在各自的上下文中，其他 Agent 就要重新發現一次。Team Memory Sync 讓知識可以在 Agent 之間共享。

**源碼**：`src/services/teamMemorySync/index.ts`

### 4.9.2 SyncState 型別

```typescript
// src/services/teamMemorySync/index.ts

type SyncState = {
  lastKnownChecksum: string | null      // 整體 ETag，用於條件請求
  serverChecksums: Map<string, string>  // 每個 key 的 sha256 hash
  serverMaxEntries: number | null       // 從 413 回應學到的 entry 數上限
}
```

**三個欄位的用途**：

**`lastKnownChecksum`**：HTTP ETag，用於 conditional GET。

```
GET /team-memory
If-None-Match: "abc123"        ← 上次拿到的 ETag

→ 304 Not Modified              ← 沒有變化，省掉傳輸
→ 200 OK, ETag: "def456"        ← 有變化，回傳新資料
```

**`serverChecksums`**：每個 key 的 hash，用於 delta push。

```
本地 memory:
  "tsconfig_rule"  → sha256: "aaa"
  "env_setup"      → sha256: "bbb"    ← 新的！
  "doc_format"     → sha256: "ccc"

Server checksums:
  "tsconfig_rule"  → sha256: "aaa"    ← 相同，不用推
  "doc_format"     → sha256: "ccc"    ← 相同，不用推

Delta: 只推 "env_setup"（sha256 不在 server checksums 中）
```

**`serverMaxEntries`**：從 413 回應學到的上限。

```
PUT /team-memory
Body: { entries: [... 1001 entries ...] }

→ 413 Payload Too Large
  { maxEntries: 1000 }

下次推送時，先檢查 entries.length <= serverMaxEntries
超過的部分根據優先級丟棄
```

### 4.9.3 同步語意

Team Memory Sync 有三種操作模式：

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  Pull（拉取）— Server Wins Per-Key                       │
│  ─────────────────────────────────────                   │
│  從 server 拉取所有 entries                               │
│  對每個 key：server 版本覆蓋本地版本                       │
│  衝突解決策略：last-writer-wins                            │
│                                                          │
│  適用時機：Agent 啟動時、定期同步                           │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Push（推送）— Delta Upload                              │
│  ─────────────────────────────────────                   │
│  比對本地 checksums 和 server checksums                    │
│  只推送 hash 不同的 entries（delta）                       │
│  大幅減少網路流量                                          │
│                                                          │
│  適用時機：Agent 學到新知識後                               │
│                                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Atomic Rejection（原子拒絕）                             │
│  ─────────────────────────────────────                   │
│  PUT 請求是全有或全無                                      │
│  如果 entry 數量超過上限 → 整個 PUT 被拒絕（413）          │
│  不會部分寫入                                              │
│                                                          │
│  適用時機：Server 端保護                                   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 4.9.4 同步流程

```
Agent A 學到新規則                       Server                       Agent B
──────────────────                       ──────                       ─────────
    │                                       │                            │
    │ memory.set("no_rm_rf",                │                            │
    │   "永遠不要 rm -rf /")                │                            │
    │                                       │                            │
    │ Push: delta upload                    │                            │
    │──────────────────────────────────────→│                            │
    │ { entries: [                           │                            │
    │   { key: "no_rm_rf",                  │                            │
    │     value: "永遠不要 rm -rf /",        │                            │
    │     checksum: "sha256:abc" }          │                            │
    │ ] }                                   │                            │
    │                                       │                            │
    │ ← 200 OK, ETag: "new-etag"           │                            │
    │                                       │                            │
    │                                       │  Pull: conditional GET     │
    │                                       │←─────────────────────────────│
    │                                       │  If-None-Match: "old-etag" │
    │                                       │                            │
    │                                       │  200 OK (有新資料)          │
    │                                       │──────────────────────────→│
    │                                       │  { entries: [...] }         │
    │                                       │                            │
    │                                       │  Agent B 現在也知道        │
    │                                       │  "永遠不要 rm -rf /" 了    │
    ▼                                       ▼                            ▼
```

---

## 4.10 Leader Permission Bridge

### 4.10.1 問題

In-Process Teammate 跑在和 Leader 同一個 Node.js 進程中。這時，Worker 不需要透過檔案系統的 Permission Queue 來請求權限——它可以直接使用 Leader 的 ToolUseConfirm 對話框。

但怎麼讓 Worker 的程式碼「看到」Leader 的 UI？

**源碼**：`src/utils/swarm/leaderPermissionBridge.ts`

### 4.10.2 Module-Level Registry Pattern

```typescript
// src/utils/swarm/leaderPermissionBridge.ts

// Module-level 變數——整個進程共享
let leaderToolUseConfirm:
  | ((request: ToolUseConfirmRequest) => Promise<ToolUseConfirmResult>)
  | null = null

// Leader 啟動時註冊自己的 ToolUseConfirm
export function registerLeaderToolUseConfirm(
  confirm: (request: ToolUseConfirmRequest) => Promise<ToolUseConfirmResult>
): void {
  leaderToolUseConfirm = confirm
}

// Worker 需要權限時，取得 Leader 的 ToolUseConfirm
export function getLeaderToolUseConfirm():
  | ((request: ToolUseConfirmRequest) => Promise<ToolUseConfirmResult>)
  | null {
  return leaderToolUseConfirm
}

// Leader 結束時清除
export function clearLeaderToolUseConfirm(): void {
  leaderToolUseConfirm = null
}
```

**這是 Module-Level Registry Pattern**——用 ES module 的 singleton 特性來做跨上下文的服務定位。

```
Node.js 進程
┌──────────────────────────────────────────────┐
│                                              │
│  leaderPermissionBridge.ts (module scope)     │
│  ┌──────────────────────────────────┐        │
│  │ leaderToolUseConfirm = fn(...)   │        │
│  └──────────┬───────────────────────┘        │
│             │                                │
│    ┌────────┴────────┐                       │
│    │                 │                       │
│    ▼                 ▼                       │
│  Leader            Worker A                  │
│  ┌──────────┐     ┌──────────┐              │
│  │ register │     │ get      │              │
│  │ ToolUse  │     │ ToolUse  │              │
│  │ Confirm  │     │ Confirm  │              │
│  │          │     │          │              │
│  │ (擁有 UI)│     │ (沒有 UI)│              │
│  └──────────┘     └──────────┘              │
│                                              │
│  Worker A 呼叫 getLeaderToolUseConfirm()      │
│  → 取得 Leader 的 confirm 函數                 │
│  → 直接呼叫，UI 顯示在 Leader 的 terminal      │
│  → 不需要檔案系統 IPC！                        │
└──────────────────────────────────────────────┘
```

### 4.10.3 Bridge 的使用流程

```
In-Process Worker                    Leader Permission Bridge          Leader UI
────────────────                    ──────────────────────            ──────────
    │                                       │                            │
    │ 需要執行 bash("rm -rf build/")         │                            │
    │                                       │                            │
    │ getLeaderToolUseConfirm()             │                            │
    │──────────────────────────────────────→│                            │
    │                                       │                            │
    │ ← fn(request)                         │                            │
    │                                       │                            │
    │ 呼叫 fn({                              │                            │
    │   toolName: "Bash",                   │                            │
    │   input: { command: "rm -rf build/" }, │                            │
    │   workerName: "worker-auth"           │                            │
    │ })                                    │                            │
    │                                       │                            │
    │                                       │ 顯示 ToolUseConfirm       │
    │                                       │──────────────────────────→│
    │                                       │                            │
    │                                       │        [Allow] [Deny]     │
    │                                       │              │             │
    │                                       │        人類按 Allow       │
    │                                       │              │             │
    │                                       │ ← { approved: true }      │
    │                                       │←─────────────────────────│
    │                                       │                            │
    │ ← { approved: true }                  │                            │
    │                                       │                            │
    │ 繼續執行 bash("rm -rf build/")          │                            │
    ▼                                       ▼                            ▼
```

### 4.10.4 為什麼不直接 import Leader 的 UI？

你可能會想：Worker 和 Leader 在同一個進程，為什麼不直接 import Leader 的 UI 模組？

因為**耦合**。如果 Worker 直接依賴 Leader 的 UI 模組：

```
// ✗ 壞的設計
import { showToolUseConfirm } from '../leader/ui'  // Worker 依賴 Leader 的 UI

// Worker 必須知道 Leader 的 UI 實作
// Leader 的 UI 改了，Worker 也要改
// 如果 Worker 跑在 tmux 裡（不同進程），這個 import 不存在
```

Registry pattern 解耦了這個依賴：

```
// ✓ 好的設計
import { getLeaderToolUseConfirm } from '../swarm/leaderPermissionBridge'

const confirm = getLeaderToolUseConfirm()
if (confirm) {
  // In-process：直接用 Leader 的 UI
  return await confirm(request)
} else {
  // Out-of-process：fallback 到 Permission Queue
  return await createPermissionRequest(request)
}
```

Worker 不需要知道 UI 是什麼樣子，它只需要一個 `(request) => Promise<result>` 的函數。

---

## 4.11 五個競態條件及其緩解機制

Swarm 架構中存在至少五個已知的競態條件。每一個都有具體的緩解機制。

### 4.11.1 總覽表

| # | 競態條件 | 位置 | 機制 | 嚴重程度 |
|---|---------|------|------|---------|
| 1 | 多源同時解決同一權限請求 | `PermissionContext.ts:63-93` | `createResolveOnce` (CAS) | 致命 |
| 2 | 多 Agent 同時寫入同一 Mailbox | `teammateMailbox.ts` | `lockfile` 檔案鎖 | 高 |
| 3 | 權限請求/回應的生產消費解耦 | `permissionSync.ts` | 兩目錄佇列（pending/resolved） | 高 |
| 4 | In-process 多 Agent 上下文混淆 | `inProcessRunner.ts` | `AsyncLocalStorage` | 高 |
| 5 | Agent 無限增殖導致 OOM | `InProcessTeammateTask` | 訊息上限 50 + Worker 工具限制 | 致命 |

### 4.11.2 競態 1：多源同時解決權限請求

**場景**：

```
時間軸：
  t=0    Worker 請求權限
  t=100  Leader 讀到請求，開始顯示 UI
  t=150  使用者按了 Ctrl+C（abort）
  t=151  abort handler 開始執行
  t=152  Leader 的 UI 收到 abort，但 onAllow callback 已經在 microtask queue
  t=153  onAllow 和 abort handler 都嘗試 resolve 同一個 promise
```

**不處理的後果**：Promise 被 resolve 兩次。在 JavaScript 中第二次 resolve 會被靜默忽略，但業務邏輯可能已經執行了兩次（例如工具被執行然後又被 abort）。

**緩解機制**：`createResolveOnce` 的 CAS 語意（見 4.5 節）。

### 4.11.3 競態 2：多 Agent 同時寫入 Mailbox

**場景**：

```
Worker A: 完成任務，寫訊息到 leader.json
Worker B: 同一時刻也完成，也寫訊息到 leader.json

不加鎖的結果：
  Worker A 讀取 leader.json: [msg1, msg2]
  Worker B 讀取 leader.json: [msg1, msg2]          ← 讀到相同的舊資料
  Worker A 寫入: [msg1, msg2, msgA]
  Worker B 寫入: [msg1, msg2, msgB]                 ← 覆蓋了 Worker A 的寫入！

  最終 leader.json: [msg1, msg2, msgB]              ← msgA 丟失了！
```

**緩解機制**：`lockfile` 檔案鎖。

```
Worker A: lock() → 成功
Worker B: lock() → 等待（重試機制：10次, 5-100ms）

Worker A: read → append → write → unlock
Worker B: lock() → 成功（Worker A 已經 unlock）
Worker B: read → append → write → unlock

最終 leader.json: [msg1, msg2, msgA, msgB] ← 兩個訊息都在 ✓
```

### 4.11.4 競態 3：請求/回應的生產消費問題

**場景**：

```
如果用單一檔案（而不是兩個目錄）：

Worker: 寫入 request.json, status: "pending"
Leader: 讀取 request.json → 開始處理
Worker: 重新讀取 request.json（想確認是否已 resolved）
Leader: 寫入 request.json, status: "approved"
  ↑
  同一個檔案，Worker 和 Leader 都在讀寫
  需要 read-modify-write 原子性
  鎖的粒度和持有時間都成問題
```

**緩解機制**：兩目錄佇列。

```
pending/ 目錄：只有 Worker 寫入，Leader 只讀
resolved/ 目錄：只有 Leader 寫入，Worker 只讀

沒有兩方同時寫同一個檔案的情況
→ 不需要 read-modify-write 的原子性
→ 簡單的 poll + 新建檔案就夠了
```

### 4.11.5 競態 4：In-Process 上下文混淆

**場景**：

```
Agent A 開始執行工具 → await llmCall()
Event loop 切換到 Agent B → Agent B 開始執行
Agent B 呼叫 getCurrentContext() → ？？？

如果用全域變數存上下文：
  → Agent B 拿到的是 Agent A 的 context
  → Agent B 用 Agent A 的權限執行操作
  → 安全漏洞
```

**緩解機制**：`AsyncLocalStorage`。

```
每個 Agent 在自己的 AsyncLocalStorage context 中執行
即使 event loop 交錯，每個 Agent 總是拿到自己的 context
這是 Node.js 原生支援的——類似 Java 的 ThreadLocal
```

### 4.11.6 競態 5：Agent 無限增殖

**場景**：

```
如果 Worker 可以呼叫 TeamCreate：
  Leader → Worker A
  Worker A → Worker A1, Worker A2
  Worker A1 → Worker A1a, Worker A1b
  Worker A2 → Worker A2a, Worker A2b
  ...
  指數增長 → 2 分鐘 292 個 Agent → 36.8GB RSS → OOM
```

**緩解機制**：雙重防護。

```
防護 1: INTERNAL_WORKER_TOOLS
  Worker 的工具集不包含 TeamCreate
  → Worker 物理上無法產生新 Worker
  → 阻止指數增長的根本原因

防護 2: TEAMMATE_MESSAGES_UI_CAP = 50
  即使繞過了防護 1（bug），每個 Agent 的記憶體被限制
  → 50 條訊息 × 5KB ≈ 250KB/Agent
  → 即使 292 個 Agent，訊息部分只佔 ~73MB
  → 不會 OOM（雖然仍然很多，但不致命）
```

---

## 4.12 可直接套用的設計模式

本章分析的五個設計模式，可以直接應用於任何多 Agent 系統——不限於 Claude Code。

### 4.12.1 模式 1：檔案系統 IPC

**適用場景**：中低頻率的 Agent 間通訊，不想引入外部依賴。

```
┌──────────────────────────────────────────────────┐
│ 模式：檔案系統 IPC                                 │
├──────────────────────────────────────────────────┤
│                                                  │
│ 核心組件：                                        │
│ - JSON 檔案作為訊息載體                            │
│ - lockfile 套件提供跨進程檔案鎖                     │
│ - 輪詢（poll）讀取新訊息                           │
│                                                  │
│ 適用條件：                                        │
│ ✓ 訊息頻率 < 100/秒                               │
│ ✓ 延遲容忍 > 100ms                                │
│ ✓ 不想安裝 Redis/RabbitMQ                         │
│ ✓ 需要可檢查性（可以直接 cat 訊息）                  │
│                                                  │
│ 不適用：                                          │
│ ✗ 高頻率通訊（> 1000/秒）                          │
│ ✗ 低延遲需求（< 10ms）                             │
│ ✗ 分散式系統（跨機器）                              │
│                                                  │
│ 實作要點：                                        │
│ 1. 用 lockfile 保護寫入                            │
│ 2. 用指數退避重試                                  │
│ 3. finally 確保鎖釋放                              │
│ 4. 輪詢間隔根據場景調整（100ms-1000ms）             │
│                                                  │
└──────────────────────────────────────────────────┘
```

**實作模板**：

```typescript
import lockfile from 'proper-lockfile'
import fs from 'fs/promises'

async function atomicAppend<T>(
  filePath: string,
  newItem: T
): Promise<void> {
  // 確保檔案存在
  try {
    await fs.access(filePath)
  } catch {
    await fs.writeFile(filePath, '[]', 'utf-8')
  }

  // 帶鎖的 read-modify-write
  const release = await lockfile.lock(filePath, {
    retries: { retries: 10, minTimeout: 5, maxTimeout: 100 },
  })

  try {
    const data = JSON.parse(await fs.readFile(filePath, 'utf-8'))
    data.push(newItem)
    await fs.writeFile(filePath, JSON.stringify(data, null, 2), 'utf-8')
  } finally {
    await release()
  }
}
```

### 4.12.2 模式 2：兩目錄佇列

**適用場景**：請求-回應模式，需要解耦生產者和消費者。

```
┌──────────────────────────────────────────────────┐
│ 模式：兩目錄佇列                                   │
├──────────────────────────────────────────────────┤
│                                                  │
│ 結構：                                            │
│ queue/                                           │
│ ├── pending/       ← 生產者寫入，消費者讀取        │
│ │   └── req-{uuid}.json                          │
│ └── completed/     ← 消費者寫入，生產者讀取        │
│     └── req-{uuid}.json                          │
│                                                  │
│ 關鍵性質：                                        │
│ - 每個目錄只有一方寫入 → 無寫入競爭                 │
│ - 每個請求一個檔案 → 天然並行                       │
│ - UUID 命名 → 不需要序號                           │
│ - 輪詢目錄 → 簡單可靠                              │
│                                                  │
│ 對比傳統佇列：                                     │
│ ┌──────────┬──────────┬──────────┐               │
│ │          │ 兩目錄    │ Redis    │               │
│ ├──────────┼──────────┼──────────┤               │
│ │ 依賴     │ 無       │ Redis    │               │
│ │ 持久化   │ 免費     │ 需配置    │               │
│ │ 可檢查   │ cat/jq   │ redis-cli│               │
│ │ 吞吐量   │ 低       │ 高       │               │
│ │ 延遲     │ 高       │ 低       │               │
│ └──────────┴──────────┴──────────┘               │
│                                                  │
└──────────────────────────────────────────────────┘
```

**實作模板**：

```typescript
import fs from 'fs/promises'
import path from 'path'
import { randomUUID } from 'crypto'

class TwoDirectoryQueue<TReq, TRes> {
  constructor(
    private baseDir: string,
    private pollIntervalMs: number = 500
  ) {}

  private get pendingDir() { return path.join(this.baseDir, 'pending') }
  private get completedDir() { return path.join(this.baseDir, 'completed') }

  // 生產者：提交請求
  async submit(request: TReq): Promise<string> {
    const id = randomUUID()
    const filePath = path.join(this.pendingDir, `${id}.json`)
    await fs.mkdir(this.pendingDir, { recursive: true })
    await fs.writeFile(filePath, JSON.stringify({ id, ...request }))
    return id
  }

  // 消費者：掃描 pending 目錄
  async pollPending(): Promise<Array<TReq & { id: string }>> {
    try {
      const files = await fs.readdir(this.pendingDir)
      const requests = []
      for (const file of files) {
        const content = await fs.readFile(
          path.join(this.pendingDir, file), 'utf-8'
        )
        requests.push(JSON.parse(content))
      }
      return requests
    } catch {
      return []
    }
  }

  // 消費者：寫入回應
  async complete(id: string, response: TRes): Promise<void> {
    await fs.mkdir(this.completedDir, { recursive: true })
    await fs.writeFile(
      path.join(this.completedDir, `${id}.json`),
      JSON.stringify({ id, ...response })
    )
  }

  // 生產者：等待回應
  async waitForCompletion(id: string): Promise<TRes> {
    const filePath = path.join(this.completedDir, `${id}.json`)
    while (true) {
      try {
        const content = await fs.readFile(filePath, 'utf-8')
        return JSON.parse(content)
      } catch {
        await new Promise(r => setTimeout(r, this.pollIntervalMs))
      }
    }
  }
}
```

### 4.12.3 模式 3：CAS 原子認領

**適用場景**：多個非同步事件競爭同一個結果，只有一個應該勝出。

```
┌──────────────────────────────────────────────────┐
│ 模式：CAS 原子認領                                 │
├──────────────────────────────────────────────────┤
│                                                  │
│ 核心程式碼（可直接複製使用）：                       │
│                                                  │
│  function createResolveOnce<T>(                  │
│    resolve: (value: T) => void                   │
│  ) {                                             │
│    let claimed = false                           │
│    return {                                      │
│      claim() {                                   │
│        if (claimed) return false                 │
│        claimed = true                            │
│        return true                               │
│      },                                          │
│      resolve(value: T) {                         │
│        if (claimed) return                       │
│        claimed = true                            │
│        resolve(value)                            │
│      },                                          │
│    }                                             │
│  }                                               │
│                                                  │
│ 使用模式：                                        │
│  const { claim, resolve } = createResolveOnce(r) │
│                                                  │
│  // 競爭者們各自：                                 │
│  if (!claim()) return  // 只有第一個成功           │
│  resolve(myValue)      // 只有贏家的值被採用       │
│                                                  │
│ 適用場景：                                        │
│ - 超時 vs 回應 vs 取消 的三方競爭                   │
│ - 多個資料來源的 first-response-wins               │
│ - 非同步事件的去重                                  │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 4.12.4 模式 4：Leader 審批模式

**適用場景**：Worker 需要執行危險操作，但沒有 UI 來獲得人類授權。

```
┌──────────────────────────────────────────────────┐
│ 模式：Leader 審批                                  │
├──────────────────────────────────────────────────┤
│                                                  │
│ 架構：                                            │
│                                                  │
│  Worker ──→ Permission Request ──→ Leader         │
│                                       │           │
│                                  顯示 UI 給人類   │
│                                       │           │
│  Worker ←── Permission Response ←── Leader         │
│    │                                              │
│    └── 根據回應繼續或中止                           │
│                                                  │
│ 關鍵設計決策：                                     │
│                                                  │
│ 1. Worker 分類操作為「安全」和「危險」              │
│    → 安全操作直接執行，不需要審批                   │
│    → 只有危險操作才進入審批流程                     │
│    → 減少審批疲勞                                  │
│                                                  │
│ 2. Leader 可以修改 Worker 的輸入                   │
│    → 不只是 approve/reject                        │
│    → 人類可以「改一下再批准」                       │
│                                                  │
│ 3. 審批可以附帶規則更新                             │
│    → 「以後這類操作不用問了」                       │
│    → 減少重複審批                                  │
│                                                  │
│ 4. 超時處理                                       │
│    → 請求帶有 createdAt 時間戳                     │
│    → 超時的請求可以被自動拒絕                       │
│    → 防止 Worker 永遠等待                          │
│                                                  │
└──────────────────────────────────────────────────┘
```

### 4.12.5 模式 5：記憶體上限保護

**適用場景**：動態產生的 Agent 可能失控增長。

```
┌──────────────────────────────────────────────────┐
│ 模式：記憶體上限保護                                │
├──────────────────────────────────────────────────┤
│                                                  │
│ 防禦層次：                                        │
│                                                  │
│ 層 1: 限制 Agent 數量                              │
│   → Worker 不能產生新 Worker                       │
│   → 只有 Leader 可以                               │
│   → 從源頭控制                                     │
│                                                  │
│ 層 2: 限制每個 Agent 的狀態大小                     │
│   → 訊息上限：50 條                                │
│   → 超過上限丟棄最舊的                              │
│   → 即使 Agent 數量失控，每個 Agent 佔用有限        │
│                                                  │
│ 層 3: 監控與告警                                    │
│   → 追蹤 RSS 使用量                                │
│   → 達到閾值時 graceful shutdown                    │
│                                                  │
│ 數學模型：                                        │
│ max_memory = num_agents × messages_per_agent      │
│            × avg_message_size                     │
│                                                  │
│ 有上限保護：                                       │
│   = num_agents × 50 × 5KB                         │
│   = num_agents × 250KB                             │
│                                                  │
│ 無上限保護（事故情境）：                             │
│   = 292 × 無限 × 5KB                               │
│   → 36.8GB → OOM                                   │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## 4.13 設計模式對照表

把本章所有的設計模式做一個完整的對照：

```
┌─────────────────┬─────────────────────┬──────────────────────────────┐
│ 問題             │ Claude Code 方案     │ 傳統方案（對比）              │
├─────────────────┼─────────────────────┼──────────────────────────────┤
│ Agent 間通訊     │ 檔案 Mailbox        │ Redis Pub/Sub, RabbitMQ     │
│                 │ + lockfile           │ gRPC streams                │
│                 │                     │                              │
│ 權限審批        │ 兩目錄佇列           │ RPC callback                │
│                 │ pending/ + resolved/ │ WebSocket channel           │
│                 │                     │                              │
│ 多源競爭        │ CAS (claim())       │ Mutex / Semaphore           │
│                 │                     │ Channel select              │
│                 │                     │                              │
│ 上下文隔離      │ AsyncLocalStorage    │ 獨立進程 / Worker Thread    │
│                 │                     │                              │
│ 記憶體保護      │ 訊息上限 50         │ 進程記憶體限制 (cgroup)      │
│                 │ + Worker 工具限制    │                              │
│                 │                     │                              │
│ 跨 Agent 記憶   │ Team Memory Sync    │ 共享資料庫                   │
│                 │ + ETag + Delta Push │ 分散式快取 (Redis)           │
│                 │                     │                              │
│ Worker ← Leader │ Module-level        │ Dependency Injection        │
│ UI 橋接        │ Registry Pattern    │ Service Locator             │
└─────────────────┴─────────────────────┴──────────────────────────────┘
```

Claude Code 的選擇一致地偏向**簡單、零依賴、可檢查**。這是 CLI 工具的正確選擇——使用者不該為了一個開發工具去管理 Redis 集群。

---

## 4.14 進階話題：從 Swarm 看分散式系統基本功

Swarm 架構雖然跑在單機上，但它的設計模式完全對應分散式系統的經典問題。對照一下：

### 4.14.1 CAP 定理的體現

```
Consistency（一致性）
├── Team Memory Sync: Server wins → 強一致
├── Permission Queue: 兩目錄佇列 → 最終一致
└── createResolveOnce: CAS → 線性一致

Availability（可用性）
├── Mailbox: 檔案存在就能讀寫
├── Permission: pending 目錄隨時可寫
└── Worker: crash 後可重啟

Partition Tolerance（分區容忍）
├── 檔案系統是本地的 → 不會分區
├── 但 Agent crash ≈ 分區
└── 超時機制處理 Agent 無回應
```

### 4.14.2 Saga Pattern

Coordinator 的任務分派和結果收集，本質上是一個 Saga：

```
Saga: 重構認證模組
├── Step 1: Worker A 修改 auth 模組       → 成功
├── Step 2: Worker B 修改測試              → 失敗！
└── Step 3: Worker C 修改文件              → 成功

補償：
├── Leader 通知人類 Step 2 失敗
├── 人類決定是否手動修復或重試
└── 不自動回滾（Agent 操作的回滾太複雜）
```

### 4.14.3 Event Sourcing

Mailbox 的 JSON 陣列本質上是 event log：

```json
[
  { "from": "leader", "text": "開始修改 auth", "timestamp": "t1" },
  { "from": "worker-a", "text": "auth.ts 已完成", "timestamp": "t2" },
  { "from": "leader", "text": "收到，繼續", "timestamp": "t3" }
]
```

你可以重播這個 log 來理解 Agent 之間的完整對話歷史。這對 debug 極其有用——不需要看 log，直接 `cat inboxes/worker-a.json | jq` 就能看到完整的通訊記錄。

---

## 4.15 章節總結

### 本章涵蓋的內容

```
4.1  為什麼單一 Agent 不夠
     → 串行 vs 並行、五個核心挑戰、檔案系統 IPC 的設計哲學

4.2  架構總覽
     → Leader-Worker 拓撲、四條資料管線、三種 Worker 後端

4.3  Coordinator Mode
     → isCoordinatorMode() 雙開關、INTERNAL_WORKER_TOOLS 限制
     → System prompt 結構、完整生命週期

4.4  Permission Queue
     → 兩目錄佇列、SwarmPermissionRequestSchema
     → createPermissionRequest() 帶鎖寫入、完整流程圖

4.5  createResolveOnce
     → CAS 語意、完整源碼、三方競爭時序分析
     → 為什麼不用 Mutex

4.6  Mailbox 系統
     → TeammateMessage、writeToMailbox() 帶鎖寫入
     → 為什麼是 Read-Modify-Write、鎖的重試策略

4.7  In-Process Teammate
     → InProcessTeammateTaskState、50 條訊息上限
     → 36.8GB OOM 事故分析、AsyncLocalStorage 隔離

4.8  SendMessage 結構化訊息
     → shutdown_request/response、plan_approval_response
     → 四種路由模式

4.9  Team Memory Sync
     → SyncState 型別、ETag 條件請求
     → Pull（server wins）、Push（delta）、Atomic（413 rejection）

4.10 Leader Permission Bridge
     → Module-level registry pattern
     → In-process Worker 直接使用 Leader 的 UI

4.11 五個競態條件
     → CAS、檔案鎖、兩目錄佇列、AsyncLocalStorage、訊息上限

4.12 可直接套用的設計模式
     → 檔案系統 IPC、兩目錄佇列、CAS 原子認領
     → Leader 審批模式、記憶體上限保護
```

### 核心啟示

**1. 簡單勝過複雜。** 整個 Swarm 系統不依賴 Redis、Kafka、gRPC。JSON 檔案 + 檔案鎖就解決了 Agent 間通訊。簡單的東西比較不會壞。

**2. 防禦性設計是必須的。** 292 個 Agent、36.8GB RSS 的事故說明了一件事：如果你的系統允許動態產生 Agent，你**必須**有資源上限。TEAMMATE_MESSAGES_UI_CAP = 50 和 INTERNAL_WORKER_TOOLS 限制是用血的教訓換來的。

**3. CAS 在非同步世界中依然有用。** JavaScript 是單線程的，但非同步 callback 仍然會競爭。`createResolveOnce` 用 20 行程式碼解決了一個可能導致未定義行為的競態問題。

**4. 分離寫入端消除競爭。** 兩目錄佇列的核心洞見：如果你讓每個目錄只有一個寫入者，你就不需要處理寫入競爭。這比用鎖保護共享檔案簡單得多。

**5. Hub-and-Spoke 拓撲適合有人類參與的系統。** Mesh 拓撲效率高但難以監控。當系統需要人類審批時，所有通訊經過一個中心點（Leader）讓人類有清晰的控制力。

### 下一章預告

> **Chapter 5: 狀態管理——Reactive Store 與 React Ink TUI**
>
> Swarm 產生的大量狀態（Worker 狀態、Mailbox 訊息、Permission 請求）怎麼在 Terminal UI 中即時顯示？
> 下一章深入 Claude Code 的 Reactive Store 和 React Ink 渲染管線。

---

## 原始碼參考索引

| 檔案路徑 | 本章涉及內容 | 關鍵行號 |
|---------|-------------|---------|
| `src/coordinator/coordinatorMode.ts` | Coordinator 模式定義、Worker 工具限制 | L29-34, L36-41, L120-368 |
| `src/utils/swarm/permissionSync.ts` | Permission Queue、兩目錄佇列 | L49-86, L167-207 |
| `src/hooks/toolPermission/PermissionContext.ts` | createResolveOnce 原子認領 | L63-93 |
| `src/utils/teammateMailbox.ts` | Mailbox 系統、writeToMailbox | — |
| `src/tasks/InProcessTeammateTask/types.ts` | InProcessTeammateTaskState、訊息上限 | — |
| `src/tools/SendMessageTool/SendMessageTool.ts` | 結構化訊息、路由 | — |
| `src/services/teamMemorySync/index.ts` | Team Memory Sync、SyncState | — |
| `src/utils/swarm/leaderPermissionBridge.ts` | Leader Permission Bridge | — |
| `src/utils/swarm/inProcessRunner.ts` | In-Process 執行器 | — |
| `src/utils/swarm/backends/InProcessBackend.ts` | In-Process 後端 | — |
