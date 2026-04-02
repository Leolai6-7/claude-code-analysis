# 重點四：AutoDream 記憶整理機制

> **核心源碼**: `src/services/autoDream/`, `src/services/extractMemories/`, `src/memdir/`
>
> 任何需要長期記憶的 AI 應用都能用這個模式——記憶需要定期整理，不能只增不減。

## AutoDream 是什麼？

Claude Code 會在**背景自動整理記憶**——讀取最近的對話記錄，更新記憶檔案，清理過時資訊，保持 MEMORY.md 精簡。

這不是每次對話都做的（那是 extractMemories），而是一個**低頻、高價值的維護任務**。

## 觸發條件（全部滿足才執行）

**源碼**: `src/services/autoDream/autoDream.ts`

```
Gate 1: Feature Gate (最便宜)
    └── tengu_onyx_plover 開關 + CLAUDE_CODE_DISABLE_AUTO_MEMORY 未設定

Gate 2: Environment Gate
    ├── 非 KAIROS 模式
    ├── 非 Remote 模式
    └── Auto-memory 已啟用

Gate 3: Time Gate (一次 stat() 呼叫)
    └── 距上次整理 ≥ 24 小時                    ← minHours: 24

Gate 4: Scan Throttle (防止頻繁掃描)
    └── 距上次掃描 ≥ 10 分鐘                    ← SESSION_SCAN_INTERVAL_MS

Gate 5: Session Gate (需要掃描檔案系統)
    └── 上次整理之後有 ≥ 5 個新 session          ← minSessions: 5

Gate 6: Lock Acquisition (互斥)
    └── 成功取得 .consolidate-lock 檔案鎖
```

**設計哲學**：Gate 從便宜到貴排列。Feature/Environment gate 只查環境變數（0 成本）。Time gate 只需一次 `stat()` 呼叫。只有通過所有便宜的 gate 後，才執行昂貴的 session 掃描。

### 閾值速查

| 閾值 | 值 | 出處 |
|------|---|------|
| minHours | 24 小時 | autoDream.ts:64 |
| minSessions | 5 個 | autoDream.ts:65 |
| SESSION_SCAN_INTERVAL_MS | 10 分鐘 | autoDream.ts:56 |
| HOLDER_STALE_MS | 1 小時 | consolidationLock.ts:19 |

## 四階段整理流程

整理工作由一個 **forked agent** 執行，接收以下 prompt：

**源碼**: `src/services/autoDream/consolidationPrompt.ts:15-64`

### Phase 1: Orient（定位）

```
- ls 記憶目錄，看現在有什麼
- 讀 MEMORY.md 理解當前索引
- 略讀現有主題檔，避免建立重複檔案
- 如果有 logs/ 或 sessions/ 子目錄，檢查最近的條目
```

### Phase 2: Gather（收集）

```
1. Daily logs (logs/YYYY/MM/YYYY-MM-DD.md) 如果有的話
2. 已漂移的記憶 — 與現狀矛盾的事實
3. Transcript 搜尋 — 只在有具體懷疑時才 grep：
   grep -rn "<narrow term>" ${transcriptDir}/ --include="*.jsonl" | tail -50

⚠️ 不要窮舉讀取 transcript。只搜尋你已經懷疑重要的東西。
```

### Phase 3: Consolidate（整合）

```
對每個值得記住的事：寫入或更新記憶檔案。
重點：
- 合併新訊號到現有主題檔，不要建近似重複
- 把相對日期轉為絕對日期（"昨天" → "2026-04-02"）
- 刪除被推翻的事實 — 如果今天的資訊否定了舊記憶，在源頭修正
```

### Phase 4: Prune（修剪）

```
更新 MEMORY.md，保持在 200 行 AND ~25KB 以內。
每條一行，~150 字元以內：
  - [Title](file.md) — one-line hook

- 移除指向過時/錯誤/被取代的記憶的指針
- 降級冗長條目：超過 ~200 字元的，把細節移到主題檔
- 加入新重要記憶的指針
- 解決矛盾
```

## MEMORY.md 約束

**源碼**: `src/memdir/memdir.ts:34-38`

```typescript
const ENTRYPOINT_NAME = 'MEMORY.md'
const MAX_ENTRYPOINT_LINES = 200      // 行數上限
const MAX_ENTRYPOINT_BYTES = 25_000   // 位元組上限 (25KB)
```

### 截斷邏輯

```typescript
// memdir.ts:57-103
function truncateMemoryIndex(content: string): string {
  // 1. 先按行截斷到 200 行
  // 2. 再按位元組截斷到 25KB（在最後一個換行處切）
  // 3. 附加警告：
  //    "> WARNING: MEMORY.md is [reason]. Only part of it was loaded."
}
```

**截斷原因**：
- 只超行數：`"250 lines (limit: 200)"`
- 只超位元組：`"25.0 KB (limit: 25 KB) — index entries are too long"`
- 兩者都超：`"250 lines and 25.0 KB"`

## 記憶檔案格式

**源碼**: `src/memdir/memoryTypes.ts`

```markdown
---
name: User Role
type: user
description: Senior engineer, 10 years Python, new to Go
---

Leo is a senior engineer with deep Python expertise.
Currently learning Go for the first time.
Prefers concise explanations with backend analogies.
```

### 四種記憶類型

| 類型 | 用途 | 壽命 |
|------|------|------|
| `user` | 用戶角色、目標、偏好 | 長期 |
| `feedback` | 行為修正（做/不做） | 長期 |
| `project` | 進行中的工作、目標、截止日 | 中期（會過時） |
| `reference` | 外部系統指針 | 長期 |

### 過時偵測

**源碼**: `src/memdir/memoryAge.ts`

```typescript
function memoryFreshnessText(mtimeMs: number): string {
  const days = memoryAgeDays(mtimeMs)
  if (days <= 1) return ''  // 新鮮記憶不警告

  return (
    `This memory is ${days} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

## extractMemories（即時抽取）vs AutoDream（背景整理）

| 特性 | extractMemories | AutoDream |
|------|----------------|-----------|
| **觸發時機** | 每次查詢結束後 | 24 小時 + 5 個新 session |
| **執行方式** | Fire-and-forget | Forked agent |
| **最大輪數** | 5 | 無限（agent 自行決定） |
| **作用域** | 當前對話 | 多個歷史 session |
| **互斥** | 與主 agent 互斥 | 與其他 dream 互斥 |

### extractMemories 節流

```typescript
// extractMemories.ts
// 游標追蹤：只處理上次抽取後的新訊息
let lastMemoryMessageUuid: string | undefined

// 互斥：如果主 agent 已經寫了記憶，extractor 跳過
function hasMemoryWritesSince(messages, sinceUuid): boolean
```

## 鎖機制

**源碼**: `src/services/autoDream/consolidationLock.ts`

```
.consolidate-lock 檔案
├── 內容：持有者的 PID
├── mtime：上次整理的時間戳
└── 過期：1 小時（HOLDER_STALE_MS）
```

### 取得鎖

```typescript
async function tryAcquireConsolidationLock(): Promise<number | null> {
  // 1. stat() 取得 mtime 和 PID
  // 2. 如果 (now - mtime < 1hr) && PID 還活著 → 回傳 null（被阻塞）
  // 3. 否則：寫入自己的 PID
  // 4. 讀回驗證（偵測併發 reclaim 競爭）
  // 5. 回傳 priorMtime（用於回滾）
}
```

### 回滾鎖（kill/失敗時）

```typescript
async function rollbackConsolidationLock(priorMtime: number) {
  if (priorMtime === 0) {
    unlink(lockFile)  // 恢復「無檔案」狀態
  } else {
    writeFile(lockPath, '')   // 清除 PID
    utimes(lockPath, priorMtime)  // 倒回 mtime
  }
  // → 下個 session 可以立即重試
}
```

## Dream Agent 工具限制

Dream agent 不能隨意操作——有嚴格的工具白名單：

```
✅ 允許：
  - FileRead — 無限制
  - Grep, Glob — 無限制
  - Bash — 只允許唯讀命令：ls, find, grep, cat, stat, wc, head, tail
  - FileEdit, FileWrite — 只限記憶目錄內的路徑

❌ 禁止：
  - 記憶目錄外的任何寫入
  - 可寫的 Bash 命令
  - MCP 工具
  - Agent/Fork 工具
```

## DreamTask UI 狀態

**源碼**: `src/tasks/DreamTask/DreamTask.ts`

```typescript
type DreamTaskState = {
  type: 'dream'
  phase: 'starting' | 'updating'  // 第一次寫檔案時從 starting → updating
  sessionsReviewing: number        // 正在檢視的 session 數
  filesTouched: string[]           // 接觸過的檔案
  turns: DreamTurn[]               // 最後 30 個 assistant 回合
  priorMtime: number               // 用於 kill 時回滾鎖
}
```

## 可直接套用的模式

### 模式 1：便宜 Gate 優先
```
觸發條件從便宜到貴排列：
1. 環境變數查詢（0 成本）
2. 檔案 stat()（微成本）
3. 目錄掃描（中等成本）
4. 鎖競爭（需要寫檔案）

只有通過所有便宜 gate 後，才執行貴的操作。
```

### 模式 2：記憶只增不減 → 定期整理
```
- 每次對話抽取新記憶（extractMemories）
- 定期整理：合併重複、刪除過時、轉換相對日期
- 硬限制：200 行 / 25KB，強制精簡
```

### 模式 3：Forked Agent 隔離
```
整理工作由子 agent 執行，不是主 agent：
- 共享父進程的 prompt cache（省 token）
- 工具白名單限制（防止亂改）
- Kill handler + 鎖回滾（可安全中斷）
```

### 模式 4：四類記憶分類
```
user: 用戶是誰（長期）
feedback: 做/不做什麼（長期）
project: 正在做什麼（會過時）
reference: 資訊在哪裡（長期）

每種類型有不同的過時特性，整理時區別對待。
```

### 模式 5：過時警告而非刪除
```
超過 1 天的記憶不是立即刪除，而是附加警告：
「This memory is N days old. Verify against current code.」
讓 agent 自行判斷是否信任。
```

## 關鍵數字速查

| 數字 | 含義 | 出處 |
|------|------|------|
| 24 小時 | 最小整理間隔 | autoDream.ts:64 |
| 5 個 session | 最小新 session 數 | autoDream.ts:65 |
| 10 分鐘 | 掃描節流間隔 | autoDream.ts:56 |
| 1 小時 | 鎖過期時間 | consolidationLock.ts:19 |
| 200 行 | MEMORY.md 行數上限 | memdir.ts:35 |
| 25 KB | MEMORY.md 位元組上限 | memdir.ts:38 |
| 30 行 | Frontmatter 掃描深度 | memoryScan.ts:22 |
| 200 個 | 記憶檔案掃描上限 | memoryScan.ts:21 |
| 5 輪 | extractMemories 最大輪數 | extractMemories.ts:426 |
| 30 輪 | Dream UI 顯示上限 | DreamTask.ts:12 |

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/services/autoDream/autoDream.ts` | AutoDream 主流程 |
| `src/services/autoDream/consolidationPrompt.ts` | 四階段 prompt |
| `src/services/autoDream/consolidationLock.ts` | 鎖機制 |
| `src/services/autoDream/config.ts` | 設定與 feature gate |
| `src/services/extractMemories/extractMemories.ts` | 即時記憶抽取 |
| `src/memdir/memdir.ts` | MEMORY.md 管理 |
| `src/memdir/memoryTypes.ts` | 記憶類型定義 |
| `src/memdir/memoryAge.ts` | 過時偵測 |
| `src/memdir/memoryScan.ts` | 記憶掃描 |
| `src/tasks/DreamTask/DreamTask.ts` | Dream 任務 UI 狀態 |
