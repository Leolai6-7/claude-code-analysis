# 重點三：上下文壓縮策略

> **核心源碼**: `src/services/compact/`
>
> 三層壓縮機制——從不呼叫 API 的微壓縮到完整對話摘要，這是任何長對話 AI 應用都能直接用的架構。

## 三層壓縮總覽

```
請求進入
    │
    ▼
[Layer 1: MicroCompact] ← 不呼叫 API，本地操作
├── Cache-Based: cache_edits 指令刪除舊工具結果
└── Time-Based: 超過 60 分鐘未活動，清除舊結果
    │
    ▼
發送 API 請求
    │
    ├── Tokens < 167,000 (13,000 below threshold) ✓ 正常繼續
    │
    └── Tokens ≥ 167,000 (THRESHOLD HIT) ✗
        │
        ▼
    [Layer 2: AutoCompact] ← 呼叫 API 生成摘要
    ├── 先嘗試 Session Memory Compact（輕量）
    ├── 回退到 Full Conversation Compact
    ├── 斷路器：連續失敗 3 次停止重試
    │
        ▼
    [Layer 3: Full Compact] ← 完整重組
    ├── 50,000 token 預算
    ├── 重新注入最近 5 個檔案（每檔 5,000 token）
    ├── 重新注入使用過的 skill schema（25,000 token 子預算）
    └── 保留進行中的 plan
```

## Layer 1: MicroCompact（微壓縮）

**源碼**: `src/services/compact/microCompact.ts`

**核心價值**：零 API 呼叫就能釋放上下文空間。

### Cache-Based MicroCompact

利用 Anthropic API 的 `cache_edits` 功能，在不修改本地訊息的情況下刪除快取中的工具結果：

```typescript
// microCompact.ts:305
function cachedMicrocompactPath() {
  // 建立 cache_edits blocks
  // 告訴 API：「刪除這些舊的工具結果」
  // 本地訊息不變，但 API 側的快取被編輯了
}
```

**觸發**：基於工具呼叫計數（GrowthBook 設定 `tengu_slate_heron`）

**可壓縮的工具類型**：
- FileRead, Bash (Shell), Grep, Glob
- WebSearch, WebFetch
- FileEdit, FileWrite

### Time-Based MicroCompact

**源碼**: `src/services/compact/timeBasedMCConfig.ts`

```typescript
// 預設設定
{
  enabled: false,              // 預設關閉
  gapThresholdMinutes: 60,     // 60 分鐘未活動觸發
  keepRecent: 5,               // 保留最近 5 個工具結果
}
```

**邏輯**：

```typescript
// microCompact.ts:461-462
const keepSet = new Set(compactableIds.slice(-keepRecent))  // 保留最後 5 個

// 被清除的結果替換為：
'[Old tool result content cleared]'
```

## Layer 2: AutoCompact（自動壓縮）

**源碼**: `src/services/compact/autoCompact.ts`

### 關鍵閾值

| 常量 | 值 | 用途 |
|------|---|------|
| `AUTOCOMPACT_BUFFER_TOKENS` | **13,000** | 觸發點距有效上下文窗口的緩衝 |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | **20,000** | 保留給摘要輸出的 token |
| `WARNING_THRESHOLD_BUFFER_TOKENS` | **20,000** | 用戶警告閾值 |
| `ERROR_THRESHOLD_BUFFER_TOKENS` | **20,000** | 錯誤閾值 |
| `MANUAL_COMPACT_BUFFER_TOKENS` | **3,000** | 手動 /compact 命令的緩衝 |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | **3** | 斷路器閾值 |

### 觸發計算

```typescript
// autoCompact.ts:72-90
function getAutoCompactThreshold(model) {
  const contextWindow = getContextWindowForModel(model)
  const reservedForSummary = Math.min(maxOutputTokens, 20_000)
  const effectiveWindow = contextWindow - reservedForSummary
  return effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS
}

// 範例：200K context window model
// 200,000 - 20,000 = 180,000 有效 tokens
// 180,000 - 13,000 = 167,000 觸發點
// ≈ 83.5% 的標稱上下文窗口
```

### 斷路器模式

```typescript
// autoCompact.ts:257-265
type AutoCompactTrackingState = {
  compacted: boolean
  turnCounter: number
  consecutiveFailures?: number   // 斷路器計數
}

// 檢查
if (tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  // "circuit breaker tripped after 3 consecutive failures
  //  — skipping future attempts"
  return { wasCompacted: false }
}

// 失敗時遞增
consecutiveFailures = prevFailures + 1

// 成功時重置
consecutiveFailures = 0
```

**為什麼需要斷路器**：如果壓縮本身失敗（API 錯誤、token 不足），反覆重試會浪費更多 token 並陷入死循環。3 次失敗後放棄。

### 壓縮順序

```
AutoCompact 觸發
    │
    ├── 1. 嘗試 Session Memory Compact（輕量）
    │   └── 只壓縮 session memory 部分
    │
    ├── 2. 回退到 Full Conversation Compact（重量級）
    │   └── 壓縮整段對話
    │
    └── 3. 如果兩者都失敗 → consecutiveFailures++
```

## Layer 3: Full Compact（全量壓縮）

**源碼**: `src/services/compact/compact.ts`

### 重新注入預算

```typescript
// compact.ts:122-130
POST_COMPACT_MAX_FILES_TO_RESTORE = 5          // 最多恢復 5 個檔案
POST_COMPACT_TOKEN_BUDGET = 50_000             // 總重新注入預算
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000       // 每檔案截斷上限
POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000      // 每 Skill 截斷上限
POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000      // Skill 子預算
```

### 預算分配

```
總預算: 50,000 tokens
├── 檔案區 (最近 5 個編輯過的檔案): 5 × 5,000 = 25,000 tokens max
└── Skill 區 (使用過的 skill schemas): ~5 × 5,000 = 25,000 tokens max
```

### 保留策略

壓縮不是簡單地丟棄歷史，而是**結構化保留**：

```typescript
interface CompactionResult {
  boundaryMarker: SystemMessage       // 壓縮邊界標記
  summaryMessages: UserMessage[]      // Model 生成的摘要
  attachments: AttachmentMessage[]    // Plan、Skill、MCP 指令
  hookResults: HookResultMessage[]    // PostCompact hook 結果
  messagesToKeep?: Message[]          // 保留的最近訊息（Reactive compact）
}

// 訊息組裝順序
[boundaryMarker, ...summaryMessages, ...messagesToKeep, ...attachments, ...hookResults]
```

### Skill 截斷原理

```
// 源碼註釋：
// "Skills can be large (verify=18.7KB, claude-api=20.1KB).
//  Per-skill truncation beats dropping — instructions at the top
//  are usually the critical part"
```

## 壓縮觸發的完整決策樹

```
每次 API 回應後
    │
    ├── MicroCompact 評估
    │   ├── Cache-based: 工具計數達到閾值？ → cache_edits
    │   └── Time-based: 上次活動 > 60min？ → 清除舊結果
    │
    ├── Token 計數
    │   ├── < 警告閾值 → 正常繼續
    │   ├── ≥ 警告閾值 → 顯示警告
    │   ├── ≥ 觸發閾值 → AutoCompact
    │   └── ≥ 錯誤閾值 → 強制 Compact + 錯誤提示
    │
    └── AutoCompact（如果觸發）
        ├── 斷路器檢查（連續失敗 < 3？）
        ├── Session Memory Compact
        ├── Full Compact（回退）
        └── 重新注入（50K budget）
```

## Hooks 整合

```
PreCompact Hook
    │   可以在壓縮前注入額外資訊
    │   例如：保留某些關鍵上下文
    ▼
壓縮執行
    ▼
PostCompact Hook
    │   可以在壓縮後執行操作
    │   例如：記錄壓縮統計
```

## 可直接套用的模式

### 模式 1：三層漸進壓縮
```
Level 0: 本地操作（不呼叫 API）— 最便宜
Level 1: 輕量摘要 — 中等成本
Level 2: 完整重組 — 最貴但最徹底
每一層都有獨立的觸發條件和回退機制。
```

### 模式 2：斷路器防死循環
```
壓縮本身可能失敗。連續失敗 N 次後停止重試。
N = 3 是 Anthropic 的經驗值。
```

### 模式 3：結構化重新注入
```
壓縮後不是空白開始，而是精選重新注入：
- 最近編輯的 N 個檔案（context 恢復）
- 使用中的 schema（能力恢復）
- 進行中的 plan（意圖恢復）
```

### 模式 4：Cache-Based 微壓縮
```
利用 API 的快取編輯功能，不修改本地狀態就能釋放空間。
這是最聰明的優化——零成本。
```

## 關鍵數字速查

| 數字 | 含義 | 出處 |
|------|------|------|
| 13,000 tokens | AutoCompact 緩衝區 | autoCompact.ts:62 |
| 20,000 tokens | 摘要輸出保留 | autoCompact.ts:30 |
| 50,000 tokens | Full Compact 重新注入預算 | compact.ts:123 |
| 5,000 tokens | 每檔案/每 Skill 截斷上限 | compact.ts:124,129 |
| 5 個檔案 | 最多恢復檔案數 | compact.ts:122 |
| 3 次 | 斷路器閾值 | autoCompact.ts:70 |
| 60 分鐘 | Time-based 微壓縮觸發 | timeBasedMCConfig.ts:32 |
| 5 個 | Time-based 保留最近結果數 | timeBasedMCConfig.ts:33 |

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/services/compact/microCompact.ts` | MicroCompact（Cache + Time-based） |
| `src/services/compact/autoCompact.ts` | AutoCompact（閾值觸發 + 斷路器） |
| `src/services/compact/compact.ts` | Full Compact（完整重組 + 重新注入） |
| `src/services/compact/timeBasedMCConfig.ts` | Time-based 設定 |
