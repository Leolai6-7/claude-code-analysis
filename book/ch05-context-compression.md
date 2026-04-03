# Chapter 5: 三層上下文壓縮 — MicroCompact → AutoCompact → Full Compact

> 200K tokens 聽起來很多，但一個中型 codebase 的探索只需要 15-20 次工具呼叫就能填滿。
> 上下文壓縮不是可選的優化——它是長對話 Agent 能否正常運作的**生存機制**。
> Claude Code 用三層漸進式壓縮，從零成本的本地操作到完整的對話重組，
> 在每一個代價層級都有精確的工程方案。

---

## 5.1 為什麼 200K Tokens 不夠用

### 5.1.1 真實數學：上下文消耗速度

讓我們做一個不帶任何誇張的計算。假設你對一個中型 Node.js 專案（約 500 個檔案，15 萬行程式碼）提出一個中等複雜度的需求：「幫我重構 authentication middleware，改用 JWT，並更新相關測試。」

Agent 會這樣工作：

```
步驟 1: Glob 搜尋專案結構                    → ~500 tokens（結果列表）
步驟 2: Grep 搜尋 "auth" 相關檔案            → ~2,000 tokens（匹配行 + 上下文）
步驟 3: Read auth/middleware.ts (180 行)      → ~1,500 tokens
步驟 4: Read auth/session.ts (250 行)         → ~2,000 tokens
步驟 5: Read auth/types.ts (80 行)            → ~800 tokens
步驟 6: Read test/auth.test.ts (400 行)       → ~3,200 tokens
步驟 7: Grep 搜尋所有 import auth 的檔案     → ~3,000 tokens
步驟 8: Read routes/api.ts (300 行)           → ~2,400 tokens
步驟 9: Read routes/admin.ts (200 行)         → ~1,600 tokens
步驟 10: Read config/security.ts              → ~1,000 tokens
步驟 11: 第一次 Edit (middleware.ts)           → ~2,000 tokens（diff + 確認）
步驟 12: Read package.json                    → ~800 tokens
步驟 13: Shell: npm install jsonwebtoken      → ~500 tokens
步驟 14: 第二次 Edit (middleware.ts 續)        → ~2,500 tokens
步驟 15: Edit session.ts                      → ~2,000 tokens
步驟 16: Edit types.ts                        → ~1,500 tokens
步驟 17: Edit api.ts                          → ~2,000 tokens
步驟 18: Edit admin.ts                        → ~1,800 tokens
步驟 19: Edit auth.test.ts                    → ~3,000 tokens
步驟 20: Shell: npm test                      → ~5,000 tokens（測試輸出）
────────────────────────────────────────────────
                              工具結果小計：~35,800 tokens
```

但這只是**工具結果**。每一步還包含：

```
每步的 overhead：
├── Model 的推理文字（思考 + 回應）    ~300-800 tokens/步
├── Tool use block（呼叫參數）          ~100-200 tokens/步
├── System prompt（每次都要重送）        ~3,000-8,000 tokens（只算一次，但佔空間）
└── 歷史對話累積                        越來越大
```

20 步之後的真實 token 消耗：

```
System prompt:                    ~5,000 tokens
工具結果累積:                     ~35,800 tokens
Model 回應累積 (20步 × ~500):    ~10,000 tokens
Tool use blocks 累積:             ~3,000 tokens
User messages:                    ~1,000 tokens
────────────────────────────────────
                     總計：~54,800 tokens
```

這才一輪重構。如果測試失敗需要 debug，再加 10-15 步。如果需要探索其他模組的副作用，再加 10 步。

**一個真實的開發 session 通常是 40-80 次工具呼叫**，token 消耗輕鬆到達 100K-150K。

### 5.1.2 更殘酷的現實：System Prompt 膨脹

Claude Code 的 system prompt 不是固定大小的。它是動態組裝的（參見 Ch03），包含：

```
Base system prompt:        ~2,000 tokens
Permission rules:          ~500-2,000 tokens（視規則數量）
Active skills:             ~3,000-20,000 tokens（每個 skill 可達 5KB）
MCP server schemas:        ~2,000-10,000 tokens
Session memory:            ~500-5,000 tokens
Plan context:              ~500-3,000 tokens
Hook configurations:       ~200-1,000 tokens
────────────────────────────────────
              可能的 system prompt：~8,700-43,000 tokens
```

當你掛了 5 個 MCP server、載入了 3 個 skill、有完整的 session memory，system prompt 本身就可能佔去 30K-40K tokens。在 200K 的 context window 裡，**留給對話歷史的只剩 160K**。

### 5.1.3 為什麼不能「全部丟掉重來」

最樸素的方案——context 快滿了就清空重來——在 coding agent 場景完全不可行：

1. **工作意圖會丟失**：Agent 正在做一個五步的 refactoring，清空後不知道還差哪兩步
2. **檔案狀態會斷裂**：Agent 知道它剛改了哪些檔案、改了什麼，這些資訊清空後就要重新讀取
3. **使用者偏好會遺忘**：使用者在對話中表達的偏好（「不要用 lodash」「用 snake_case」）全部丟失
4. **debug 線索會消失**：之前嘗試過什麼方案、為什麼失敗，這些寶貴的除錯歷史歸零

所以需要的不是「丟掉」，而是「壓縮」——把 150K tokens 的對話歷史壓縮成一個精華摘要，保留關鍵資訊，釋放空間繼續工作。

這就是 Claude Code 三層壓縮機制的設計動機。

---

## 5.2 三層壓縮總覽

### 5.2.1 架構圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                        每次 API 呼叫                                 │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Layer 1: MicroCompact                               │
│                  ═══════════════════════                              │
│                                                                      │
│   成本：零（不呼叫 API）                                              │
│   時機：每次發送 API 請求之前                                         │
│                                                                      │
│   ┌──────────────────────┐    ┌──────────────────────┐              │
│   │  Cache-Based          │    │  Time-Based           │              │
│   │  ──────────           │    │  ──────────           │              │
│   │  cache_edits 指令      │    │  60 分鐘不活動觸發   │              │
│   │  API 側刪除舊結果      │    │  保留最近 5 個結果    │              │
│   │  本地訊息不變          │    │  替換為 cleared 標記  │              │
│   │  觸發：工具計數閾值    │    │  觸發：時間間隔       │              │
│   └──────────────────────┘    └──────────────────────┘              │
│                                                                      │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼ 發送 API 請求
                                 │
                                 ▼ 收到回應，計算 token 使用量
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
              tokens < 167K   167K ≤ t     t ≥ 180K
              (正常範圍)      < 180K      (ERROR 閾值)
                    │       (AUTO 觸發)       │
                    ▼            │            ▼
                 繼續工作         │       強制 Compact
                                 │       + 錯誤提示
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Layer 2: AutoCompact                                │
│                  ═══════════════════════                              │
│                                                                      │
│   成本：1 次 API 呼叫（摘要生成）                                     │
│   時機：token 使用量達到閾值時自動觸發                                │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────┐      │
│   │  斷路器檢查                                               │      │
│   │  consecutiveFailures < 3？                                │      │
│   │  ├── No → 跳過（"circuit breaker tripped"）              │      │
│   │  └── Yes ↓                                               │      │
│   │                                                           │      │
│   │  嘗試 1: Session Memory Compact（輕量）                   │      │
│   │  ├── minTokens: 10,000                                   │      │
│   │  ├── minTextBlockMessages: 5                             │      │
│   │  ├── maxTokens: 40,000                                   │      │
│   │  ├── 成功 → consecutiveFailures = 0, 結束               │      │
│   │  └── 失敗 ↓                                              │      │
│   │                                                           │      │
│   │  嘗試 2: Full Conversation Compact                        │      │
│   │  ├── 壓縮整段對話為結構化摘要                             │      │
│   │  ├── 成功 → consecutiveFailures = 0, 結束               │      │
│   │  └── 失敗 → consecutiveFailures++                        │      │
│   └──────────────────────────────────────────────────────────┘      │
│                                                                      │
└────────────────────────────────┬────────────────────────────────────┘
                                 │（當 Full Compact 執行時）
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Layer 3: Full Compact                               │
│                  ═══════════════════════                              │
│                                                                      │
│   成本：1 次 API 呼叫 + 檔案重新讀取                                 │
│   時機：AutoCompact 回退 或 使用者手動 /compact                      │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────┐      │
│   │  1. 發送壓縮 prompt（純文字，不帶工具）                    │      │
│   │  2. Model 生成 9 段結構化摘要                             │      │
│   │  3. 重新注入最近 5 個檔案（50K token 預算）               │      │
│   │  4. 重新注入使用過的 skill schemas（25K 子預算）          │      │
│   │  5. 保留進行中的 plan                                     │      │
│   │  6. 組裝新的訊息陣列                                      │      │
│   │  7. 執行 postCompactCleanup                               │      │
│   └──────────────────────────────────────────────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2.2 設計原則：代價遞增，介入遞深

三層壓縮的核心設計原則是**漸進式介入**：

| 層級 | 名稱 | API 成本 | 資訊損失 | 觸發頻率 | 可逆性 |
|------|------|---------|---------|---------|--------|
| Layer 1 | MicroCompact | **零** | 極低（只刪工具結果） | 很高（每次請求前） | Cache-based 可逆 |
| Layer 2 | AutoCompact | 1 次 API 呼叫 | 中等（摘要替代原文） | 中（到達閾值時） | 不可逆 |
| Layer 3 | Full Compact | 1 次 API + 檔案 I/O | 較高（重建整個上下文） | 低（回退或手動） | 不可逆 |

這個設計讓系統在大多數情況下使用最便宜的方案（MicroCompact），只在必要時才升級到更重的方案。就像作業系統的記憶體管理：先 trim cache，再 swap，最後才 OOM kill。

---

## 5.3 Layer 1: MicroCompact — 零 API 成本的上下文瘦身

**源碼位置**: `src/services/compact/microCompact.ts`

MicroCompact 是三層壓縮中最精巧的一層，因為它完全不需要呼叫 API——所有操作都是本地的或利用 API 的快取機制完成。這意味著**零額外成本、零延遲增加**。

### 5.3.1 COMPACTABLE_TOOLS：哪些工具結果可以被壓縮

在深入壓縮機制之前，首先要理解一個關鍵概念：不是所有工具結果都能被壓縮。

```typescript
// microCompact.ts, lines 38-50
const COMPACTABLE_TOOLS = new Set([
  'FileRead',     // 檔案內容——可以重新讀取
  'Shell',        // Shell 輸出——歷史執行結果
  'Grep',         // 搜尋結果——可以重新搜尋
  'Glob',         // 檔案列表——可以重新列舉
  'WebSearch',    // 搜尋結果——可以重新搜尋
  'WebFetch',     // 網頁內容——可以重新抓取
  'FileEdit',     // 編輯結果——只是確認訊息
  'FileWrite',    // 寫入結果——只是確認訊息
])
```

為什麼是這些工具？它們有一個共同特徵：**結果是可重現的或已失去時效性的**。

- `FileRead` 的結果是某個時間點的檔案內容，如果後來被 `FileEdit` 修改了，舊的讀取結果已經過時
- `Shell` 的輸出（如測試結果、build 輸出）是一次性的，model 已經看過並做出了反應
- `Grep` 和 `Glob` 的搜尋結果是探索性的，model 用它們來決定下一步，之後就不再需要

反過來，哪些**不能**被壓縮？像使用者的 text message、model 自己的推理、tool_use blocks——這些包含了意圖和決策鏈，壓縮它們會破壞上下文的連貫性。

### 5.3.2 Token 估算機制

MicroCompact 需要知道每個訊息佔多少 token 才能做出壓縮決策。但精確的 token 計數需要呼叫 tokenizer，對於高頻的本地操作來說太慢。所以 Claude Code 用了一個快速估算：

```typescript
// microCompact.ts, lines 164-205

function estimateMessageTokens(message: Message): number {
  let totalTokens = 0

  for (const block of message.content) {
    if (block.type === 'text') {
      totalTokens += roughTokenCountEstimation(block.text)
    } else if (block.type === 'tool_result') {
      totalTokens += roughTokenCountEstimation(block.content)
    } else if (block.type === 'image') {
      totalTokens += IMAGE_MAX_TOKEN_SIZE  // 常量 = 2000
    }
  }

  return totalTokens
}

function roughTokenCountEstimation(text: string): number {
  // 經驗法則：1 token ≈ 4 characters（英文）
  // 加上 4/3 的 padding 來應對非英文文字和 tokenizer 差異
  return Math.ceil(text.length / 4 * (4 / 3))
}
```

這裡有幾個值得注意的設計選擇：

1. **4/3 padding**：`text.length / 4` 是標準的英文 token 估算（平均每 token 4 個字元）。但乘以 `4/3`（約 1.33x）是一個安全係數，因為：
   - 非 ASCII 字元（中文、日文）每個字元可能消耗 2-3 個 token
   - 特殊格式（JSON、程式碼的縮排）可能有不同的 tokenization
   - 寧可高估也不要低估——低估可能導致 context overflow

2. **IMAGE_MAX_TOKEN_SIZE = 2000**：圖片的 token 消耗取決於解析度，但用固定值 2000 做上界估算，避免解析圖片 metadata 的開銷

3. **不使用真正的 tokenizer**：像 `tiktoken` 或 Claude 的 tokenizer 需要 ~50ms per call，在每次請求前對所有訊息做 token 計數太慢。粗估雖然不精確（誤差可達 20-30%），但對 MicroCompact 的觸發決策來說足夠

### 5.3.3 Cache-Based MicroCompact

Cache-Based MicroCompact 是整個壓縮系統中最巧妙的設計。它利用 Anthropic API 的 `cache_edits` 功能，在 **API 側**刪除快取中的舊工具結果，但**本地訊息完全不變**。

#### 原理：cache_edits 如何運作

要理解這個機制，先回顧 Anthropic API 的 prompt caching：

```
正常的 API 呼叫：
Client → 發送完整 messages 陣列 → API
                                    ↓
                              API 處理所有 tokens
                                    ↓
                              回傳結果

使用 prompt caching 的 API 呼叫：
Client → 發送完整 messages 陣列 → API
                                    ↓
                              API 檢查快取
                              「前 80% 跟上次一樣」
                              只處理後 20% 的新 tokens
                                    ↓
                              回傳結果（更快、更便宜）
```

`cache_edits` 在這之上加了一個操作：

```
使用 cache_edits 的 API 呼叫：
Client → 發送 messages + cache_edits 指令 → API
                                              ↓
                                        API 先執行 cache_edits：
                                        「刪除快取中第 3、7、12 個
                                          tool_result 的內容」
                                              ↓
                                        用編輯後的快取處理請求
                                              ↓
                                        回傳結果
```

**關鍵洞察**：`cache_edits` 只修改 API 側的快取，不影響 client 端的 messages 陣列。這意味著：

- 本地的 conversation state 完全不變
- 不需要重新計算任何東西
- 如果 cache miss，API 會用完整的本地 messages（包含未刪除的內容）作為 fallback
- 對 model 來說，那些舊的工具結果就好像從未存在過

#### cachedMicrocompactPath() 函數

```typescript
// microCompact.ts, line 305
function cachedMicrocompactPath(
  messages: Message[],
  compactableIds: Set<string>,
  toolCallCount: number,
  config: MicroCompactConfig
): CacheEditInstruction[] {
  // 1. 遍歷所有訊息，找到可壓縮的 tool_result blocks
  const editableBlocks: CacheEditTarget[] = []

  for (const [msgIdx, msg] of messages.entries()) {
    for (const [blockIdx, block] of msg.content.entries()) {
      if (block.type === 'tool_result' &&
          compactableIds.has(block.tool_use_id)) {
        editableBlocks.push({
          messageIndex: msgIdx,
          blockIndex: blockIdx,
          estimatedTokens: estimateBlockTokens(block)
        })
      }
    }
  }

  // 2. 保留最近的結果，只編輯較舊的
  // 3. 生成 cache_edits 指令陣列
  return editableBlocks
    .slice(0, -config.keepRecent)  // 保留最後 N 個
    .map(target => ({
      type: 'delete',
      path: [target.messageIndex, 'content', target.blockIndex]
    }))
}
```

#### consumePendingCacheEdits() 函數

```typescript
// microCompact.ts, line 88
function consumePendingCacheEdits(): CacheEditInstruction[] | null {
  // 取出待處理的 cache edits，清空佇列
  const pending = pendingCacheEdits
  pendingCacheEdits = null
  return pending
}
```

這是一個典型的 **consume-once** 模式：cache edits 被計算出來後放入佇列，下次 API 呼叫時取出並附加到請求中，取出後佇列清空。這確保同一組 edits 不會被重複發送。

#### 工具註冊與排序

MicroCompact 透過工具系統的 registration 機制來追蹤工具呼叫計數。每次工具被呼叫時，計數器遞增。但這裡有一個微妙的問題：**工具的執行順序影響哪些結果會被保留**。

```typescript
// 工具註冊順序決定了 compactableIds 的累積順序
// 較早呼叫的工具結果 → 較早進入 compactableIds → 較早被壓縮
// 較晚呼叫的工具結果 → 較晚進入 compactableIds → 被保留

// 例如：
// Call #1: FileRead("auth.ts")       → id_001 → 最早，最可能被壓縮
// Call #2: Grep("import auth")        → id_002 → 較早
// Call #3: FileRead("routes.ts")      → id_003 → 中間
// Call #4: FileEdit("auth.ts", ...)   → id_004 → 較晚
// Call #5: Shell("npm test")          → id_005 → 最晚，最可能被保留
```

這個 FIFO 策略是合理的：**越新的工具結果越可能與當前工作相關**。一個 30 步前讀取的檔案內容，很可能早就過時了。

#### GrowthBook 觸發配置

Cache-Based MicroCompact 的觸發不是硬編碼的——它由 GrowthBook feature flag 控制：

```typescript
// Feature flag: tengu_slate_heron
//
// 這個名稱是 GrowthBook 的隨機命名慣例（動物名組合），
// 不是有含義的名稱。
//
// 配置結構：
{
  enabled: boolean,              // 是否啟用 cache-based MicroCompact
  triggerAfterToolCalls: number, // 多少次工具呼叫後觸發
  keepRecent: number,            // 保留最近多少個結果
}
```

使用 feature flag 而非硬編碼的原因：

1. **A/B 測試**：可以對不同使用者群組測試不同的觸發閾值
2. **快速回滾**：如果 cache_edits 出現 bug，可以遠端關閉而不需要發佈新版
3. **漸進發布**：先對 5% 使用者啟用，觀察效果，再逐步擴大

### 5.3.4 Time-Based MicroCompact

Time-Based MicroCompact 解決的是另一個場景：**使用者離開了一段時間再回來**。

想像這個場景：你在上午 10 點讓 Agent 做了一些工作，然後去開會。下午 2 點回來，繼續在同一個 session 裡提問。上午的 context 裡有大量的 `FileRead` 和 `Grep` 結果，但 4 小時過去了，這些結果很可能已經不再相關（也許你或同事在這期間手動改了檔案）。

Time-Based MicroCompact 會在這種情況下自動清理舊的工具結果。

#### 配置

```typescript
// TimeBasedMCConfig（預設值）
// 源碼：src/services/compact/timeBasedMCConfig.ts

interface TimeBasedMCConfig {
  enabled: boolean               // 預設 false（由 feature flag 控制）
  gapThresholdMinutes: number    // 預設 60（60 分鐘不活動）
  keepRecent: number             // 預設 5（保留最近 5 個結果）
}
```

注意 `enabled` 預設是 `false`。這表示 Time-Based MicroCompact 是一個**實驗性功能**，需要透過 feature flag 或手動配置來啟用。這種保守的預設是合理的——在功能驗證充分之前，不應該預設對使用者的 context 做不可逆的修改。

#### evaluateTimeBasedTrigger() 函數

```typescript
// microCompact.ts, lines 422-444

function evaluateTimeBasedTrigger(
  messages: Message[],
  config: TimeBasedMCConfig,
  now: number = Date.now()
): TimeBasedTriggerResult {
  if (!config.enabled) {
    return { shouldTrigger: false }
  }

  // 找到最後一條使用者訊息的時間戳
  const lastUserMessageTime = findLastUserMessageTimestamp(messages)

  if (!lastUserMessageTime) {
    return { shouldTrigger: false }
  }

  // 計算距離上次活動的時間差
  const gapMinutes = (now - lastUserMessageTime) / (1000 * 60)

  // 是否超過閾值
  if (gapMinutes < config.gapThresholdMinutes) {
    return { shouldTrigger: false }
  }

  return {
    shouldTrigger: true,
    gapMinutes,
    lastActivityTime: lastUserMessageTime
  }
}
```

這個函數的邏輯非常直接：看最後一條使用者訊息是什麼時候發的，如果距現在超過 60 分鐘，就觸發清理。

#### 清理邏輯：keepSet 與 cleared marker

```typescript
// microCompact.ts, line 461

function applyTimeBasedCompaction(
  messages: Message[],
  compactableIds: string[],
  keepRecent: number
): Message[] {
  // 保留最後 keepRecent 個可壓縮的工具結果
  const keepSet = new Set(
    compactableIds.slice(-keepRecent)   // 取最後 5 個
  )

  // 遍歷所有訊息，替換不在 keepSet 中的工具結果
  return messages.map(msg => ({
    ...msg,
    content: msg.content.map(block => {
      if (block.type === 'tool_result' &&
          isCompactable(block.tool_use_id) &&
          !keepSet.has(block.tool_use_id)) {
        // 替換為清除標記
        return {
          ...block,
          content: '[Old tool result content cleared]'
        }
      }
      return block
    })
  }))
}
```

幾個設計細節值得注意：

1. **`compactableIds.slice(-keepRecent)`**：`slice(-5)` 取陣列最後 5 個元素。因為 compactableIds 是按時間順序累積的，最後 5 個就是最近的 5 個工具結果。

2. **`'[Old tool result content cleared]'`**：不是直接刪除 block，而是替換內容為一個人可讀的標記。這麼做的原因：
   - 保持 messages 陣列的結構不變（tool_use 和 tool_result 的配對關係不被破壞）
   - Model 能看到「這裡曾經有一個工具結果，但被清除了」，而不是一個莫名其妙的空洞
   - 如果 model 需要這個資訊，它知道可以重新呼叫該工具

3. **不可逆操作**：與 Cache-Based 不同，Time-Based 直接修改了本地 messages。一旦替換，原始內容就沒了（除非有外部日誌）。這也是為什麼它預設關閉。

### 5.3.5 MicroCompact 的效果評估

讓我們用一個具體例子估算 MicroCompact 的效果：

```
場景：25 次工具呼叫後，context 約 80,000 tokens
其中 tool_result 佔 ~45,000 tokens（20 個可壓縮結果）

Cache-Based MicroCompact（keepRecent=5）：
├── 壓縮 15 個舊結果
├── 每個平均 ~2,250 tokens
├── API 側釋放：~33,750 tokens
└── 本地 messages 不變

Time-Based MicroCompact（keepRecent=5）：
├── 壓縮 15 個舊結果
├── 替換為 ~40 tokens 的 cleared marker
├── 本地釋放：~33,150 tokens（33,750 - 15×40）
└── 本地 messages 被修改

效果：從 80K tokens 的 context 中釋放了 ~33K tokens
      相當於延長了 ~40% 的可用空間
      成本：零 API 呼叫
```

這就是 MicroCompact 的價值：**在問題還沒發生之前就預防性地釋放空間，而且完全免費**。

---

## 5.4 Layer 2: AutoCompact — 閾值驅動的自動壓縮

**源碼位置**: `src/services/compact/autoCompact.ts`

當 MicroCompact 不足以控制 context 大小時（通常是因為對話本身很長、model 回應很詳細、或 system prompt 很大），AutoCompact 接手。它會**自動偵測**何時 context 接近上限，並觸發壓縮。

### 5.4.1 閾值計算的精確數學

AutoCompact 的觸發時機由三個函數決定：

#### getEffectiveContextWindowSize()

```typescript
// autoCompact.ts

function getEffectiveContextWindowSize(model: Model): number {
  const contextWindow = getContextWindowForModel(model)  // 例如 200,000
  const maxOutput = getMaxOutputTokensForModel(model)    // 例如 128,000

  // 有效窗口 = 總窗口 - min(maxOutput, 20000)
  const reservedForSummary = Math.min(maxOutput, MAX_OUTPUT_TOKENS_FOR_SUMMARY)
  return contextWindow - reservedForSummary
}
```

為什麼要扣掉 `min(maxOutput, 20000)`？

這是為壓縮摘要的**輸出**預留空間。當 AutoCompact 觸發時，它要呼叫 API 生成一個摘要，這個摘要本身需要 output tokens。`MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20000` 是經驗值——一個好的對話摘要通常在 5K-15K tokens，20K 留了足夠的 headroom。

用 `Math.min` 是因為有些 model 的 `maxOutputTokens` 可能小於 20K（例如某些小模型），不能預留比它能輸出的更多。

#### getAutoCompactThreshold()

```typescript
// autoCompact.ts, line 62

const AUTOCOMPACT_BUFFER_TOKENS = 13000

function getAutoCompactThreshold(model: Model): number {
  const effectiveWindow = getEffectiveContextWindowSize(model)
  return effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS
}
```

這個 13,000 tokens 的 buffer 是什麼？

它是一個**安全餘量**，確保在觸發壓縮到壓縮完成之間，model 還有足夠的空間完成當前 turn。想像這個時間線：

```
Turn N: token count = 166,000（還沒觸發）
        → model 回應 + 工具結果 = +3,000 tokens
Turn N+1: token count = 169,000（超過 167,000 觸發線）
        → AutoCompact 觸發
        → 但 model 仍然需要完成摘要生成
        → 需要 ~13,000 tokens 的餘量來安全完成
```

如果 buffer 太小（比如 3,000），可能在觸發後 model 還沒來得及壓縮就 overflow 了。如果太大（比如 50,000），壓縮會觸發得太早，浪費可用空間。13,000 是 Anthropic 根據實際使用數據調整出來的平衡點。

#### 完整計算範例

以 200K context window 的 model 為例：

```
Step 1: contextWindow = 200,000
Step 2: maxOutput = 128,000（Sonnet 4 的 maxOutputTokens）
Step 3: reservedForSummary = min(128,000, 20,000) = 20,000
Step 4: effectiveWindow = 200,000 - 20,000 = 180,000
Step 5: autoCompactThreshold = 180,000 - 13,000 = 167,000

結論：當 token 使用量達到 167,000 時觸發 AutoCompact
      這是標稱 context window 的 83.5%
```

對比其他 model：

```
128K model:
  effectiveWindow = 128,000 - 20,000 = 108,000
  threshold = 108,000 - 13,000 = 95,000（74.2%）

64K model:
  effectiveWindow = 64,000 - 20,000 = 44,000
  threshold = 44,000 - 13,000 = 31,000（48.4%）
```

注意到一個有趣的現象：**context window 越小的 model，AutoCompact 觸發得越「早」**（佔比越低）。這是因為 20,000 + 13,000 = 33,000 的固定開銷在小 window 裡佔比更大。對 64K model 來說，將近一半的空間被保留了。這暗示了一個潛在的優化點——小 model 可能需要不同的常量。

### 5.4.2 所有常量一覽

```typescript
// autoCompact.ts — 完整常量列表

// Line 30
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000
// 壓縮摘要的最大輸出 token 數

// Line 62
const AUTOCOMPACT_BUFFER_TOKENS = 13_000
// 自動觸發閾值與有效窗口之間的緩衝

// Line 63
const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
// 顯示「context 快滿了」警告的緩衝
// 觸發點 = contextWindow - 20,000
// 例如 200K model → 180,000 時顯示警告

// Line 64
const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
// 顯示錯誤（強制 compact）的緩衝
// 注意：這與 WARNING 用的是同一個值（20,000）
// 但它們的計算基準不同

// Line 70
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
// 斷路器閾值：連續失敗 3 次後停止重試

// 未直接命名但在 Session Memory Compact 中使用的配置：
const SESSION_MEMORY_COMPACT_CONFIG = {
  minTokens: 10_000,          // session memory 至少要有 10K tokens 才值得壓縮
  minTextBlockMessages: 5,    // 至少要有 5 個文字 block 訊息
  maxTokens: 40_000,          // session memory compact 最多處理 40K tokens
}
```

### 5.4.3 斷路器模式（Circuit Breaker Pattern）

AutoCompact 有一個可能被忽略但極其重要的設計：**斷路器**。

#### 為什麼需要斷路器

壓縮本身是一次 API 呼叫。如果它失敗了（網路錯誤、API 超時、token 不足以生成摘要），會發生什麼？

沒有斷路器的情況：

```
Turn 1: token count = 168,000 → 觸發 AutoCompact → 失敗
Turn 2: token count = 170,000 → 觸發 AutoCompact → 失敗（花費更多 token）
Turn 3: token count = 172,000 → 觸發 AutoCompact → 失敗（更多 token）
Turn 4: token count = 174,000 → 觸發 AutoCompact → 失敗
...
Turn N: context overflow → 整個 session 崩潰
```

每次失敗的壓縮嘗試**反而消耗更多 token**（因為要發送壓縮 prompt 和接收錯誤回應），讓情況變得更糟。這是一個正反饋循環（死亡螺旋）。

#### 斷路器實作

```typescript
// autoCompact.ts, lines 257-265

type AutoCompactTrackingState = {
  compacted: boolean           // 是否已經成功壓縮過
  turnCounter: number          // turn 計數器
  consecutiveFailures?: number // 斷路器計數器
}

async function maybeAutoCompact(
  tracking: AutoCompactTrackingState,
  messages: Message[],
  model: Model
): Promise<AutoCompactResult> {

  // ===== 斷路器檢查 =====
  const prevFailures = tracking.consecutiveFailures ?? 0

  if (prevFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    // 斷路器已跳閘
    logWarning(
      "circuit breaker tripped after 3 consecutive failures" +
      " — skipping future attempts"
    )
    return { wasCompacted: false }
  }

  // ===== 嘗試壓縮 =====
  try {
    const result = await performCompaction(messages, model)

    // ===== 成功：重置斷路器 =====
    tracking.consecutiveFailures = 0
    return { wasCompacted: true, result }

  } catch (error) {
    // ===== 失敗：遞增斷路器計數 =====
    tracking.consecutiveFailures = prevFailures + 1

    logError(
      `AutoCompact failed (${tracking.consecutiveFailures}/${MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES})`
    )

    return { wasCompacted: false, error }
  }
}
```

斷路器的三個狀態：

```
              ┌──────────────┐
              │   CLOSED     │  ← 正常狀態
              │ (正常運作)    │
              └──────┬───────┘
                     │ 失敗
                     ▼
              ┌──────────────┐
              │  HALF-OPEN   │  ← 開始計數
              │ (failures=1) │     但仍嘗試
              └──────┬───────┘
                     │ 連續失敗到 3 次
                     ▼
              ┌──────────────┐
              │    OPEN      │  ← 停止嘗試
              │ (failures≥3) │     直到 session 結束
              └──────────────┘
                     │
                     │ 任何時候成功
                     ▼
              ┌──────────────┐
              │   CLOSED     │  ← 重置
              │ (failures=0) │
              └──────────────┘
```

注意一個細節：Claude Code 的斷路器**沒有自動恢復**（no timeout-based reset）。一旦跳閘，在整個 session 中都不會再嘗試自動壓縮。這是比較激進的策略——理由是如果壓縮連續 3 次失敗，問題很可能是系統性的（例如 model 無法處理當前的 context 狀態），繼續嘗試只會浪費資源。

使用者仍然可以手動執行 `/compact` 命令來強制壓縮，這不受斷路器限制。

### 5.4.4 壓縮順序：Session Memory Compact → Full Compact

當 AutoCompact 觸發時，它不是直接做 Full Compact——而是先嘗試一個**更輕量的方案**：

```
AutoCompact 觸發
    │
    ▼
Step 1: 嘗試 Session Memory Compact（輕量）
    │
    ├── 檢查 session memory 是否夠大（≥ 10,000 tokens）
    ├── 檢查是否有足夠的文字訊息（≥ 5 個 text block messages）
    ├── 限制處理量（≤ 40,000 tokens）
    │
    ├── 條件滿足 → 只壓縮 session memory 部分
    │   ├── 成功 → 結束（通常可以釋放 5K-20K tokens）
    │   └── 失敗 → 繼續到 Step 2
    │
    └── 條件不滿足 → 直接到 Step 2

Step 2: 回退到 Full Conversation Compact（重量級）
    │
    ├── 壓縮整段對話歷史為結構化摘要
    ├── 重新注入關鍵文件和 skill
    │
    ├── 成功 → 結束
    └── 失敗 → consecutiveFailures++
```

#### Session Memory Compact 的配置

```typescript
// Session Memory Compact 的觸發條件
{
  minTokens: 10_000,        // session memory 部分至少 10K tokens
  minTextBlockMessages: 5,  // 至少 5 個文字 block 訊息
  maxTokens: 40_000,        // 最多處理 40K tokens 的 session memory
}
```

為什麼先嘗試 Session Memory Compact？

1. **成本更低**：只處理 session memory 的一部分（10K-40K tokens），而非整個對話（可能 100K+ tokens）
2. **風險更低**：不會丟失任何對話歷史，只是壓縮了 session memory 中的舊資訊
3. **速度更快**：處理的 token 更少，API 呼叫更快返回
4. **常常就夠了**：如果 context 膨脹的主要原因是 session memory（累積的長期記憶），壓縮它就能釋放足夠空間

只有當 Session Memory Compact 不適用（session memory 太小、沒有足夠的文字訊息）或失敗時，才會升級到 Full Compact。

### 5.4.5 Warning 與 Error 閾值

除了 AutoCompact 觸發閾值之外，還有兩個使用者可見的閾值：

```
Context Window 使用量

0%          50%              83.5%     90%      100%
|─────────────|─────────────────|────────|────────|
              正常區間       AUTO觸發  WARNING  ERROR
                            (167K)    (180K)   (200K)

WARNING (contextWindow - WARNING_THRESHOLD_BUFFER_TOKENS):
  「Context window 快滿了」
  → 顯示警告訊息給使用者
  → 使用者可以選擇手動 /compact

ERROR (contextWindow - ERROR_THRESHOLD_BUFFER_TOKENS):
  「Context window 已滿」
  → 強制執行 compact
  → 顯示錯誤訊息
```

注意 WARNING 和 ERROR 的 buffer 都是 20,000 tokens，但它們的計算基準不同：

- WARNING threshold = `contextWindow - WARNING_THRESHOLD_BUFFER_TOKENS` = 200,000 - 20,000 = 180,000
- ERROR threshold 通常更接近 context window 的實際上限

在 AutoCompact 觸發（167K）和 WARNING（180K）之間有 13K 的間隔——這段空間是 AutoCompact 工作的「跑道」。

---

## 5.5 Layer 3: Full Compact — 完整的對話重組

**源碼位置**: `src/services/compact/compact.ts`, `src/services/compact/prompt.ts`

Full Compact 是最重量級的壓縮操作。它把整段對話歷史壓縮成一個結構化摘要，然後重新注入關鍵文件和配置，本質上是**重建整個 context**。

### 5.5.1 壓縮 Prompt：9 段結構化摘要

Full Compact 的核心是一個精心設計的 prompt，指導 model 如何壓縮對話歷史。這個 prompt 位於 `prompt.ts` 的 lines 19-143。

#### 關鍵約束：TEXT ONLY, NO TOOLS

```
// prompt.ts — 壓縮 prompt 的核心指令

CRITICAL INSTRUCTIONS:
- You MUST respond with TEXT ONLY
- Do NOT use any tools
- Do NOT ask for clarification
- Simply analyze the conversation and produce the summary
```

為什麼不能用工具？

1. **避免副作用**：壓縮是一個「只讀」操作，不應該觸發任何檔案修改、shell 命令或網路請求
2. **減少 token 消耗**：tool_use blocks 本身佔 token，壓縮操作應該盡可能節省
3. **提高可靠性**：工具呼叫可能失敗，純文字輸出幾乎不會失敗
4. **加速處理**：沒有工具呼叫的 roundtrip，一次呼叫就完成

#### 摘要結構：analysis + summary

Model 被要求產出兩個區塊：

```xml
<analysis>
分析對話歷史，識別關鍵資訊...
（這部分是 model 的「工作區」，用來整理思路）
</analysis>

<summary>
結構化的 9 段摘要...
（這部分是最終保留的壓縮結果）
</summary>
```

`<analysis>` 區塊類似 chain-of-thought——讓 model 先分析再總結，提高摘要品質。但最終只有 `<summary>` 的內容被保留在壓縮後的 context 中。

#### 9 個摘要段落

壓縮 prompt 要求 model 按照以下 9 個段落組織摘要：

```
1. Primary Request（主要需求）
   使用者的原始請求是什麼？核心目標是什麼？

2. Key Concepts and Decisions（關鍵概念與決策）
   在對話中做出了哪些重要的技術決策？
   例如：「決定用 JWT 而非 session-based auth」

3. Files and Code（檔案與程式碼）
   讀取、修改、建立了哪些檔案？
   每個檔案的狀態是什麼（已修改、已建立、只讀取）？
   關鍵的程式碼變更摘要

4. Errors and Debugging（錯誤與除錯）
   遇到了哪些錯誤？如何解決的？
   哪些嘗試失敗了？為什麼？

5. Problem Solving Approaches（問題解決方法）
   嘗試了哪些方法？哪些有效？哪些無效？

6. User Messages and Preferences（使用者訊息與偏好）
   使用者表達了哪些偏好？
   例如：「不要用 class，用 function」「prefer TypeScript」

7. Pending Tasks（待辦事項）
   還有什麼沒完成的？
   哪些任務被提到但還沒開始？

8. Current Work State（當前工作狀態）
   目前正在做什麼？做到哪裡了？
   最後一步做了什麼？

9. Next Step（下一步）
   接下來應該做什麼？
   是否有使用者指定的下一步？
```

這 9 個段落的設計非常用心。它們覆蓋了一個 coding agent 需要記住的所有維度：

- **意圖**（1, 6）：使用者要什麼、偏好什麼
- **知識**（2, 3）：技術決策、檔案狀態
- **歷史**（4, 5）：試過什麼、失敗過什麼
- **狀態**（7, 8, 9）：現在在哪、接下來做什麼

缺少任何一個維度都會導致壓縮後的 agent 行為異常：

- 沒有「意圖」→ 不知道目標，可能偏離方向
- 沒有「知識」→ 重新讀取已經讀過的檔案，浪費 token
- 沒有「歷史」→ 重複嘗試已經失敗的方案
- 沒有「狀態」→ 不知道做到哪裡，可能重做或跳過步驟

### 5.5.2 Post-Compact 重新注入預算

壓縮後的 context 不是只有一個摘要——Claude Code 會**主動重新注入**最近的關鍵檔案和 skill 配置，確保 model 在壓縮後仍然有足夠的上下文來繼續工作。

#### 預算常量

```typescript
// compact.ts, lines 122-130

const POST_COMPACT_TOKEN_BUDGET = 50_000        // 總重新注入預算
const POST_COMPACT_MAX_FILES_TO_RESTORE = 5     // 最多恢復 5 個檔案
const POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000  // 每個檔案最多 5,000 tokens
const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000 // 每個 Skill 最多 5,000 tokens
const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000 // Skill 重新注入子預算
```

#### 預算分配圖

```
POST_COMPACT_TOKEN_BUDGET = 50,000 tokens
│
├── 檔案區 (最多 25,000 tokens)
│   ├── File 1: auth/middleware.ts    → 最多 5,000 tokens
│   ├── File 2: routes/api.ts        → 最多 5,000 tokens
│   ├── File 3: test/auth.test.ts    → 最多 5,000 tokens
│   ├── File 4: config/security.ts   → 最多 5,000 tokens
│   └── File 5: auth/types.ts        → 最多 5,000 tokens
│       ────────────────────────────
│       最多 5 × 5,000 = 25,000 tokens
│
└── Skill 區 (最多 25,000 tokens)
    ├── Skill 1: verify skill         → 最多 5,000 tokens
    ├── Skill 2: claude-api skill     → 最多 5,000 tokens
    ├── Skill 3: deploy skill         → 最多 5,000 tokens
    ├── Skill 4: test skill           → 最多 5,000 tokens
    └── Skill 5: review skill         → 最多 5,000 tokens
        ────────────────────────────
        最多 5 × 5,000 = 25,000 tokens
```

#### 為什麼 Skill 需要截斷而不是丟棄

源碼中有一段重要的註釋：

```
// Skills can be large (verify=18.7KB, claude-api=20.1KB).
// Per-skill truncation beats dropping — instructions at the top
// are usually the critical part
```

一個 Skill 的 schema 可以高達 20KB（~5,000 tokens）。如果有 5 個活躍的 Skill，那就是 100KB（~25,000 tokens）。直覺上可能想「不重要的 Skill 就丟掉」，但問題是**你無法判斷哪個 Skill 不重要**——使用者隨時可能再次使用任何一個。

所以策略是**截斷而非丟棄**：每個 Skill 保留前 5,000 tokens。Skill 的描述和核心指令通常在開頭，截斷尾部損失的多半是次要的 edge case 說明。

### 5.5.3 檔案選擇策略

重新注入哪些檔案？不是隨機選的——有一個精確的選擇算法：

```typescript
// compact.ts — 檔案選擇邏輯

function selectFilesToRestore(
  conversationHistory: Message[],
  maxFiles: number = POST_COMPACT_MAX_FILES_TO_RESTORE
): FileToRestore[] {

  // 1. 從對話歷史中提取所有被讀取/修改過的檔案
  const touchedFiles = extractTouchedFiles(conversationHistory)

  // 2. 按最後訪問時間降序排序（最近的排前面）
  touchedFiles.sort((a, b) => b.lastAccessTimestamp - a.lastAccessTimestamp)

  // 3. 排除不需要重新注入的檔案
  const filtered = touchedFiles.filter(file => {
    // 排除 plan 檔案（會透過其他機制保留）
    if (isPlanFile(file.path)) return false

    // 排除 memory 檔案（由 session memory 系統管理）
    if (isMemoryFile(file.path)) return false

    return true
  })

  // 4. 取前 5 個
  return filtered.slice(0, maxFiles)
}
```

設計考量：

1. **按時間降序**：最近操作的檔案最可能與當前工作相關。你 30 步前讀的 `README.md` 遠不如 3 步前修改的 `auth.ts` 重要。

2. **排除 plan 檔案**：Plan（任務計劃）有自己的保留機制——它們會被作為 attachment 直接附加到壓縮後的 context 中，不需要佔用檔案恢復的額度。

3. **排除 memory 檔案**：Session memory（如 `CLAUDE.md`）同樣有專門的保留路徑，不應該和普通檔案競爭名額。

4. **硬上限 5 個**：不管觸及了多少檔案，最多恢復 5 個。這是在「恢復得越多越好」和「恢復太多會佔用太多 token」之間的平衡。5 × 5,000 = 25,000 tokens 是一個合理的投資——佔總預算的 50%，但通常能覆蓋當前工作最相關的檔案。

### 5.5.4 壓縮後的訊息組裝

Full Compact 完成後，新的 messages 陣列是這樣組裝的：

```typescript
// compact.ts — 訊息組裝順序

interface CompactionResult {
  boundaryMarker: SystemMessage       // 壓縮邊界標記
  summaryMessages: UserMessage[]      // Model 生成的結構化摘要
  messagesToKeep?: Message[]          // 保留的最近訊息（如果是 reactive compact）
  attachments: AttachmentMessage[]    // Plan、Skill schemas、MCP 配置
  hookResults: HookResultMessage[]    // PostCompact hook 的結果
}

// 最終的 messages 陣列：
const newMessages = [
  boundaryMarker,       // 1. 標記壓縮邊界
  ...summaryMessages,   // 2. 壓縮摘要
  ...messagesToKeep,    // 3. 保留的最近訊息
  ...attachments,       // 4. 重新注入的附件
  ...hookResults,       // 5. PostCompact hook 結果
]
```

#### 每個部分的角色

**1. boundaryMarker（邊界標記）**

```
[COMPACT] Conversation was compacted at this point.
Messages above this marker are from the compressed summary.
Messages below are from the current conversation.
```

這個標記讓 model 知道「上面的內容是壓縮摘要，不是真正的對話歷史」。如果沒有這個標記，model 可能會把摘要當成使用者的訊息來回應，導致奇怪的行為。

**2. summaryMessages（摘要訊息）**

Model 生成的 9 段結構化摘要，作為 User message 注入。這是壓縮的核心產出。

**3. messagesToKeep（保留的訊息）**

在某些壓縮模式中（如 reactive compact），最近的幾條訊息會被原封不動地保留，而不是被壓縮。這確保了正在進行的對話的連續性。

**4. attachments（附件）**

```
├── 最近的 5 個檔案內容（重新讀取）
├── 活躍的 Skill schemas（截斷後）
├── 進行中的 Plan
└── MCP server 配置
```

這些是透過前面描述的預算機制選擇和截斷的。

**5. hookResults（Hook 結果）**

PostCompact hook 的執行結果。例如，一個自定義 hook 可能在壓縮後注入一些額外的上下文（見 5.8 節）。

### 5.5.5 Prompt Cache Sharing

Full Compact 呼叫 API 生成摘要時，有一個重要的優化細節：

```typescript
// compact.ts — 壓縮呼叫的 API 配置

const compactResponse = await callAPI({
  messages: compactPromptMessages,
  model: compactModel,
  maxTokens: MAX_OUTPUT_TOKENS_FOR_SUMMARY,

  // 關鍵配置：
  forkedAgent: true,          // 使用獨立的 agent context
  skipCacheWrite: true,       // 不寫入 prompt cache
})
```

**`forkedAgent: true`**：壓縮呼叫使用一個「forked」（分叉的）agent context，不會影響主對話的狀態。如果壓縮失敗，主對話不受影響。

**`skipCacheWrite: true`**：壓縮 prompt 是一次性的——每次壓縮的 prompt 都不同（因為對話歷史不同）。把它寫入 cache 毫無意義，只會污染 cache 空間。跳過 cache write 節省了 cache 存儲成本。

但注意，壓縮呼叫仍然可以**讀取** cache（cache read 沒有被跳過）。如果壓縮 prompt 的前綴恰好匹配了之前的 cache，就能省下一些 input token 成本。

---

## 5.6 Token 估算的工程細節

**源碼位置**: `src/services/compact/microCompact.ts`, lines 164-205

Token 估算貫穿整個壓縮系統——MicroCompact 用它決定哪些結果值得壓縮，AutoCompact 用它判斷是否到達閾值。讓我們深入這個看似簡單但有很多 edge case 的函數。

### 5.6.1 estimateMessageTokens() 完整邏輯

```typescript
// microCompact.ts, lines 164-205

function estimateMessageTokens(message: Message): number {
  let totalTokens = 0

  // 基礎開銷：每條訊息有 message-level 的 overhead
  // （role 標記、content array 的 JSON 結構等）
  totalTokens += MESSAGE_OVERHEAD  // ~10 tokens

  for (const block of message.content) {
    switch (block.type) {
      case 'text':
        totalTokens += roughTokenCountEstimation(block.text)
        break

      case 'tool_use':
        // tool_use 包含 tool name + input JSON
        totalTokens += roughTokenCountEstimation(block.name)
        totalTokens += roughTokenCountEstimation(
          JSON.stringify(block.input)
        )
        break

      case 'tool_result':
        if (typeof block.content === 'string') {
          totalTokens += roughTokenCountEstimation(block.content)
        } else if (Array.isArray(block.content)) {
          for (const subBlock of block.content) {
            if (subBlock.type === 'text') {
              totalTokens += roughTokenCountEstimation(subBlock.text)
            } else if (subBlock.type === 'image') {
              totalTokens += IMAGE_MAX_TOKEN_SIZE  // 2000
            }
          }
        }
        break

      case 'image':
        totalTokens += IMAGE_MAX_TOKEN_SIZE  // 2000
        break
    }
  }

  return totalTokens
}
```

### 5.6.2 roughTokenCountEstimation() 與 4/3 Padding

```typescript
function roughTokenCountEstimation(text: string): number {
  if (!text) return 0
  return Math.ceil(text.length / 4 * (4 / 3))
  // 等價於 Math.ceil(text.length / 3)
}
```

數學簡化：`text.length / 4 * (4/3)` = `text.length * (1/4) * (4/3)` = `text.length * (1/3)` = `text.length / 3`。

所以這個函數本質上就是**字元數除以 3，向上取整**。

這比常見的「除以 4」更保守（估算值更高）：

```
文字範例                字元數    除以4估算    除以3估算    實際tokens
────────────────────────────────────────────────────────────────────
"Hello, world!"         13       4           5           4
"console.log('test')"   20       5           7           6
"const x = [1,2,3]"     18       5           6           5
中文「你好世界」         8*3=24   6           8           8-12
JSON {"key":"value"}     15       4           5           5
```

對中文和其他非 ASCII 文字來說，除以 3 的估算更接近實際值，因為這些字元的 tokenization 效率更低。

### 5.6.3 IMAGE_MAX_TOKEN_SIZE = 2000

圖片的 token 消耗在 Claude API 中取決於圖片的解析度和大小，範圍從幾百到幾千 tokens。使用固定值 2000 是一個上界估算，好處是避免了解析圖片 metadata 的開銷（需要讀取圖片文件頭來獲取解析度），壞處是可能高估小圖片的消耗。

在壓縮場景中，高估比低估安全——寧可早一點觸發壓縮，也不要因為低估而導致 context overflow。

---

## 5.7 Post-Compact 清理

**源碼位置**: `src/services/compact/postCompactCleanup.ts`

壓縮完成後，不只是 messages 陣列改變了——許多全域狀態也需要重置。如果忘記重置，會導致壓縮後的行為異常。

### 5.7.1 清理項目

```typescript
// postCompactCleanup.ts

async function postCompactCleanup(): Promise<void> {

  // 1. 重置 MicroCompact 狀態
  resetMicrocompactState()
  // - 清空 compactableIds 列表
  // - 重置工具呼叫計數器
  // - 清空 pendingCacheEdits 佇列
  // 為什麼：壓縮後的訊息已經是全新的，
  //         舊的 compactable IDs 已經不存在

  // 2. 清除 System Prompt Sections
  clearSystemPromptSections()
  // - 移除動態注入的 system prompt 片段
  // - 這些片段可能引用了已壓縮的上下文
  // 為什麼：壓縮後某些動態 section 可能過時
  //         （例如「最近修改的檔案列表」）

  // 3. 清除 Classifier Approvals
  clearClassifierApprovals()
  // - 重置 permission classifier 的快取
  // - 之前批准的工具使用權限被清除
  // 為什麼：壓縮改變了上下文，之前的
  //         permission 判斷基於舊上下文，
  //         不應該延續到新上下文

  // 4. 清除其他快取狀態
  clearToolResultCache()
  // - 某些工具結果有快取（避免重複計算）
  // - 壓縮後這些快取可能不一致

  // 5. 更新統計資訊
  updateCompactionStats({
    timestamp: Date.now(),
    preCompactTokens: beforeTokenCount,
    postCompactTokens: afterTokenCount,
    compressionRatio: afterTokenCount / beforeTokenCount
  })
}
```

### 5.7.2 為什麼 Permission Approvals 需要清除

這是一個容易被忽略但有安全意義的設計：

```
壓縮前的場景：
User: 「幫我修改 auth.ts」
Agent: [tool_use: FileEdit("auth.ts", ...)]
Permission System: ✅ 批准（基於使用者的明確請求）
  → 快取：FileEdit("auth.ts") = APPROVED

壓縮後的情況：
如果不清除快取 → Agent 仍然可以免批准地修改 auth.ts
但使用者可能在壓縮後的新對話中談的是完全不同的事情
→ 安全風險：Agent 可能在不適當的上下文中修改檔案
```

清除 classifier approvals 強制 Agent 在壓縮後**重新取得權限**，這是一個「寧可多問一次，也不要誤操作」的安全策略。

---

## 5.8 Pre/Post Compact Hooks

Claude Code 的 Hook 系統（參見 Ch09）與壓縮系統整合，允許自定義的壓縮前後邏輯。

### 5.8.1 PreCompact Hook

```
PreCompact Hook
觸發時機：壓縮開始之前
可用操作：
├── 注入額外上下文（確保被包含在摘要中）
├── 記錄壓縮前的狀態（用於審計或除錯）
├── 取消壓縮（返回特定錯誤碼）
└── 修改壓縮配置
```

使用場景：

```yaml
# .claude/hooks.yaml
hooks:
  preCompact:
    - command: "echo 'IMPORTANT: Current git branch is $(git branch --show-current)'"
      # 在壓縮前注入當前 git branch 資訊
      # 確保壓縮摘要包含 branch 上下文
```

### 5.8.2 PostCompact Hook

```
PostCompact Hook
觸發時機：壓縮完成之後，在新的 messages 組裝完成之後
可用操作：
├── 注入額外的 context（作為 hookResults 附加）
├── 記錄壓縮統計
├── 通知外部系統
└── 驗證壓縮品質
```

使用場景：

```yaml
# .claude/hooks.yaml
hooks:
  postCompact:
    - command: "cat .claude/project-context.md"
      # 壓縮後重新注入專案級別的重要上下文
      # 確保 Agent 不會因壓縮而忘記專案規則
```

Hook 結果被附加到壓縮後的 messages 陣列最後（參見 5.5.4 的組裝順序），確保它們是 Agent 最後看到的內容，給予最高的注意力權重。

---

## 5.9 設計模式總結

從 Claude Code 的三層壓縮系統中，可以提煉出以下可移植的設計模式：

### 5.9.1 模式一：漸進式壓縮（Progressive Compression）

```
原則：先用最便宜的方案，不夠再升級

Layer 0: 本地操作（零 API 成本）
         ↓ 不夠
Layer 1: 輕量壓縮（1 次 API 呼叫，局部壓縮）
         ↓ 不夠
Layer 2: 全量壓縮（1 次 API 呼叫 + I/O，整體重組）

每一層都有獨立的觸發條件、執行邏輯、和回退策略。
層與層之間是 fallback 關係，不是 pipeline。
```

這個模式的通用性很高。任何需要「處理資源限制」的系統都能用：

- **Web 應用的快取策略**：L1 cache（memory）→ L2 cache（Redis）→ 資料庫
- **影片串流的品質調整**：降低位元率 → 降低解析度 → 降低幀率
- **API 請求的 retry 策略**：快速重試 → 指數退避 → 斷路器

### 5.9.2 模式二：斷路器防死循環（Circuit Breaker）

```
原則：失敗 N 次後停止嘗試，避免正反饋循環

狀態轉移：
CLOSED（正常）
  ↓ 失敗
HALF-OPEN（計數中，但仍嘗試）
  ↓ 連續失敗達到閾值
OPEN（停止嘗試）
  ↓ 成功（或手動重置）
CLOSED

關鍵設計選擇：
- N = 3（Anthropic 的經驗值）
- 沒有 timeout-based reset（session 級別的持久跳閘）
- 手動操作（/compact）不受斷路器限制
```

為什麼選 3 而不是其他數字？

- N = 1：太激進，一次暫時性的 API 錯誤就會永久停止壓縮
- N = 5：太寬鬆，5 次失敗的壓縮嘗試可能累計消耗 10K+ 額外 tokens
- N = 3：合理的平衡——排除暫時性錯誤（1-2 次），但及時停止系統性問題

### 5.9.3 模式三：結構化重新注入（Structured Re-injection）

```
原則：壓縮後不是空白開始，而是精選重新注入

三個維度的恢復：
1. Context 恢復（最近的檔案）→ 讓 Agent 知道它在操作什麼
2. 能力恢復（Skill schemas）→ 讓 Agent 知道它能做什麼
3. 意圖恢復（Plan + 摘要） → 讓 Agent 知道它要做什麼

預算制：
- 總預算 50K tokens
- 子預算：檔案 25K + Skill 25K
- 單項上限：每檔案/每 Skill 5K tokens
- 數量上限：最多 5 個檔案
```

這個模式的核心洞察是：**壓縮不只是「丟掉舊的」，更重要的是「恢復關鍵的」**。一個好的壓縮系統花在「選擇重新注入什麼」上的設計精力，應該不少於花在「如何壓縮」上的。

### 5.9.4 模式四：Cache-Based 微優化（Zero-Cost Optimization）

```
原則：利用基礎設施的能力，在不改變語意的情況下優化

cache_edits 的巧妙之處：
- 本地狀態不變（完美的 fallback）
- API 側減少處理量（成本降低）
- 零額外 API 呼叫（零延遲增加）
- 有 cache miss 時自動降級（優雅退化）
```

這是一個更廣泛的原則：**不要只想著在應用層解決問題，看看基礎設施提供了什麼**。很多「免費」的優化隱藏在平台能力中——CDN 的 edge caching、資料庫的 partial index、API 的 conditional request。

### 5.9.5 模式五：預算制資源分配（Budget-Based Selection）

```
原則：用 token 預算來控制重新注入的規模

總預算 → 子預算 → 單項上限 → 數量上限

四層限制確保不會因為某個異常大的項目（比如一個 50KB 的檔案）
而耗盡所有預算。

排序策略：
- 檔案：按最後訪問時間降序
- Skill：按使用頻率降序
- 截斷策略：保留前 N tokens（而非完全丟棄）
```

---

## 5.10 關鍵數字速查表

以下是本章涉及的所有關鍵數字，方便快速查閱：

### MicroCompact 相關

| 數字 | 含義 | 來源 |
|------|------|------|
| 8 個工具 | COMPACTABLE_TOOLS 集合大小 | microCompact.ts:38-50 |
| 2,000 tokens | IMAGE_MAX_TOKEN_SIZE | microCompact.ts |
| 4/3 (≈1.33x) | Token 估算的 padding 係數 | microCompact.ts:roughTokenCountEstimation |
| 60 分鐘 | Time-Based 觸發的不活動閾值 | timeBasedMCConfig.ts:32 |
| 5 個 | Time-Based 保留最近結果數 | timeBasedMCConfig.ts:33 |
| `'[Old tool result content cleared]'` | 清除標記文字 | microCompact.ts |

### AutoCompact 相關

| 數字 | 含義 | 來源 |
|------|------|------|
| 13,000 tokens | AUTOCOMPACT_BUFFER_TOKENS | autoCompact.ts:62 |
| 20,000 tokens | MAX_OUTPUT_TOKENS_FOR_SUMMARY | autoCompact.ts:30 |
| 20,000 tokens | WARNING_THRESHOLD_BUFFER_TOKENS | autoCompact.ts:63 |
| 20,000 tokens | ERROR_THRESHOLD_BUFFER_TOKENS | autoCompact.ts:64 |
| 3,000 tokens | MANUAL_COMPACT_BUFFER_TOKENS | autoCompact.ts |
| 3 次 | MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES | autoCompact.ts:70 |
| 167,000 tokens | 200K model 的 AutoCompact 觸發點 | 計算值 |
| 83.5% | 200K model 的觸發佔比 | 計算值 |

### Session Memory Compact 相關

| 數字 | 含義 | 來源 |
|------|------|------|
| 10,000 tokens | minTokens（最小值得壓縮的量） | autoCompact.ts |
| 5 | minTextBlockMessages（最少文字訊息數） | autoCompact.ts |
| 40,000 tokens | maxTokens（最大處理量） | autoCompact.ts |

### Full Compact 相關

| 數字 | 含義 | 來源 |
|------|------|------|
| 50,000 tokens | POST_COMPACT_TOKEN_BUDGET | compact.ts:123 |
| 5 個 | POST_COMPACT_MAX_FILES_TO_RESTORE | compact.ts:122 |
| 5,000 tokens | POST_COMPACT_MAX_TOKENS_PER_FILE | compact.ts:124 |
| 5,000 tokens | POST_COMPACT_MAX_TOKENS_PER_SKILL | compact.ts:129 |
| 25,000 tokens | POST_COMPACT_SKILLS_TOKEN_BUDGET | compact.ts:130 |
| 9 段 | 壓縮摘要的結構化段落數 | prompt.ts:19-143 |

### 模型對照表

| Model | Context Window | Effective Window | AutoCompact 觸發 | 觸發佔比 |
|-------|---------------|-----------------|------------------|---------|
| 200K | 200,000 | 180,000 | 167,000 | 83.5% |
| 128K | 128,000 | 108,000 | 95,000 | 74.2% |
| 64K | 64,000 | 44,000 | 31,000 | 48.4% |

---

## 5.11 章節總結

### 5.11.1 核心要點回顧

**200K tokens 的上下文窗口為什麼不夠？**

因為一個中型 codebase 的探索在 15-20 次工具呼叫後就能消耗 50K+ tokens，加上 system prompt 可能佔 30K-40K tokens，一個正常的開發 session（40-80 次工具呼叫）很容易逼近上限。壓縮不是優化，是**生存必需**。

**三層壓縮的設計哲學？**

漸進式介入，代價遞增。MicroCompact 零成本但只能清理工具結果，AutoCompact 花一次 API 呼叫做輕量壓縮，Full Compact 花一次 API 呼叫加 I/O 做完整重組。每一層都有獨立的觸發條件和回退邏輯，形成一個有韌性的多層防線。

**最精妙的設計是什麼？**

三個亮點：

1. **Cache-Based MicroCompact**：利用 API 的 cache_edits 功能，在不修改本地狀態的情況下釋放 API 側的空間。零成本、零風險、優雅退化。這是「用基礎設施能力解決應用層問題」的典範。

2. **結構化重新注入**：壓縮後不是空白重來，而是精選注入最近的檔案、活躍的 Skill、進行中的 Plan。50K token 的預算分配體現了「context 恢復、能力恢復、意圖恢復」三個維度的考量。

3. **9 段摘要結構**：不是讓 model 隨意摘要，而是指定 9 個維度（意圖、知識、歷史、狀態），確保壓縮後的 Agent 在每個維度都有足夠的資訊繼續工作。

### 5.11.2 可帶走的經驗法則

1. **壓縮觸發點設在標稱上限的 ~80%**：留 20% 作為工作空間和安全餘量
2. **斷路器閾值 3 次**：平衡暫時性錯誤和系統性問題
3. **重新注入預算佔壓縮後空間的 ~25-30%**：太少恢復不了上下文，太多浪費空間
4. **按時間降序選擇恢復項目**：最近的最相關
5. **截斷優於丟棄**：保留部分資訊比完全丟失好，尤其是 schema 和 instructions 的開頭通常最重要
6. **Token 估算寧可高估**：高估 20% 導致提早壓縮（浪費一點空間），低估 20% 可能導致 overflow（session 崩潰）

### 5.11.3 與其他章節的關聯

- **Ch03 (System Prompt)**：system prompt 的動態組裝直接影響可用的 context 空間，理解 system prompt 的大小是理解壓縮閾值的前提
- **Ch04 (Multi-Agent Swarm)**：每個 Worker Agent 有自己獨立的 context，壓縮是 per-agent 的。Leader 的 context 因為要追蹤所有 Worker 的狀態，壓縮壓力更大
- **Ch07 (Tool System)**：COMPACTABLE_TOOLS 集合定義了哪些工具結果可以被 MicroCompact 清理
- **Ch08 (Permission Model)**：壓縮後 permission approvals 被清除，Agent 需要重新取得權限
- **Ch09 (Hook System)**：PreCompact/PostCompact hooks 允許自定義壓縮行為
- **Ch13 (Memory System)**：Session Memory Compact 是壓縮系統和記憶系統的交界點；AutoDream 的長期記憶可以減輕短期上下文的壓力

### 5.11.4 開放問題

在結束本章之前，留下幾個值得思考的問題：

1. **小 model 的閾值問題**：當 context window 只有 64K 時，固定的 33K 開銷（20K + 13K）佔了 51.5%，留給實際對話的空間非常有限。是否需要按 model 大小動態調整這些常量？

2. **壓縮品質評估**：目前沒有系統性的方式來評估壓縮摘要的品質。如果 model 遺漏了關鍵資訊怎麼辦？是否需要一個「壓縮品質檢查」的步驟？

3. **多輪壓縮的退化**：一個很長的 session 可能經歷多次 Full Compact。每次壓縮都是「摘要的摘要」，資訊必然逐漸退化。如何量化和緩解這種退化？

4. **跨 session 的連續性**：壓縮解決的是 session 內的問題。跨 session 的上下文恢復由 Memory 系統負責（Ch13）。兩個系統之間的銜接——壓縮摘要是否應該影響長期記憶——是一個有趣的設計空間。

---

> **本章核心洞察**：上下文壓縮是 long-running Agent 的生命線。Claude Code 的三層設計——從零成本的 cache 操作到完整的對話重組——展示了如何在「盡可能少丟資訊」和「不能超過 context 上限」之間找到精確的平衡。最好的壓縮不是讓使用者察覺不到壓縮發生了，而是讓壓縮後的 Agent 表現得好像它記得一切。
