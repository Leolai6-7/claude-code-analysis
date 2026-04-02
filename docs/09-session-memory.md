# 第九章：Session 與 Memory 管理

> **源碼**: `src/state/`, `src/services/SessionMemory/`, `src/services/extractMemories/`

## 狀態管理

### Store 架構

**源碼**: `src/state/store.ts`

```typescript
// Zustand-like reactive store
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```

所有 state 更新通過 `setState(prev => newState)` 函式式更新，觸發所有 subscriber。

### AppState 主要區塊

| 區塊 | 內容 |
|------|------|
| `settings` | 全域設定、模型選擇、權限模式 |
| `mcp` | MCP 連接、工具、命令、資源 |
| `plugins` | 啟用/停用 plugins、命令、agent 定義 |
| `tasks` | `{ [taskId]: TaskState }` |
| `session` | Session ID、遠端 URL、bridge 狀態 |

## Session Memory

**源碼**: `src/services/SessionMemory/sessionMemory.ts` (16.5KB)

### 觸發條件（全部滿足）

1. **Token 閾值**：context window 增長超過閾值
   - 初始：10,000 tokens
   - 更新：5,000 tokens 增長
2. **工具呼叫閾值**：至少 3 次工具呼叫
3. **安全檢查**：最後一個 assistant 回合沒有工具呼叫

### 抽取流程

```
觸發條件滿足
    ↓
setupSessionMemoryFile()
    ↓
載入模板（~/.claude/session-memory/config/template.md）
    ↓
Fork 子 agent（不阻塞主線程）
    ├── 讀取當前 memory 和對話上下文
    ├── 接收更新 prompt
    └── 使用 FileEditTool 更新 markdown sections
    ↓
保留模板結構（標題和斜體描述）
```

### 模板 Sections

| Section | 內容 |
|---------|------|
| Session Title | 5-10 字標題 |
| Current State | 當前活動工作、待處理任務 |
| Task Specification | 設計決策、上下文 |
| Files and Functions | 重要檔案及原因 |
| Workflow | Bash 命令、解讀 |
| Errors & Corrections | 遇到的問題 |
| Codebase Documentation | 系統元件 |
| Learnings | 什麼有效/無效 |
| Key Results | 用戶請求的精確輸出 |
| Worklog | 逐步摘要 |

## Extract Memories（即時抽取）

**源碼**: `src/services/extractMemories/extractMemories.ts`

每次查詢結束後 fire-and-forget 執行：

### 與 AutoDream 的差異

| | Extract Memories | AutoDream |
|---|---|---|
| 觸發 | 每次查詢後 | 24h + 5 sessions |
| 範圍 | 當前對話 | 多個歷史 session |
| 輪數 | ≤5 | 無限 |
| 目的 | 即時記憶 | 整理/清理 |

### 游標追蹤

```typescript
let lastMemoryMessageUuid: string | undefined  // Session 內的游標

// 只處理游標之後的新訊息
countModelVisibleMessagesSince(messages, sinceUuid)

// 互斥：主 agent 已寫記憶 → extractor 跳過
hasMemoryWritesSince(messages, sinceUuid)
```

## 記憶系統架構

```
即時層: extractMemories (每次查詢後)
    │   寫入新記憶檔
    ↓
索引層: MEMORY.md (≤200 行, ≤25KB)
    │   指向記憶檔的索引
    ↓
整理層: AutoDream (每 24h)
    │   合併重複、刪除過時、修剪索引
    ↓
載入層: loadMemoryPrompt()
    │   每次對話時載入 MEMORY.md 到 system prompt
```

## Session 記憶 vs 長期記憶

```
Session Memory (~/.claude/session-memory.md)
├── 當前 session 的工作摘要
├── 每次查詢後可能更新
└── Session 結束後不再更新

Long-term Memory (~/.claude/projects/<path>/memory/)
├── MEMORY.md — 索引
├── user_*.md — 用戶資訊
├── feedback_*.md — 行為指導
├── project_*.md — 專案狀態
└── reference_*.md — 外部指針
```

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/state/store.ts` | Reactive store |
| `src/state/AppStateStore.ts` | AppState 型別 (22KB) |
| `src/state/AppState.tsx` | React Context Provider (23KB) |
| `src/services/SessionMemory/sessionMemory.ts` | Session memory 主邏輯 (17KB) |
| `src/services/SessionMemory/sessionMemoryUtils.ts` | Token 追蹤、設定 (6KB) |
| `src/services/SessionMemory/prompts.ts` | 抽取 prompt (13KB) |
| `src/services/extractMemories/extractMemories.ts` | 即時記憶抽取 |
| `src/memdir/memdir.ts` | MEMORY.md 管理 |
