# Chapter 6: AutoDream 記憶系統 — 即時抽取 + 定期整理的雙層架構

> 記憶需要定期整理，不能只增不減。人類如此，AI 也是。
> Claude Code 的記憶系統不只是「把東西存起來」——它有即時抽取、定期整理、自動淘汰三個動作，
> 用雙層架構解決了長期記憶最核心的問題：**什麼該記、什麼該忘、什麼時候整理**。

---

## 6.1 為什麼 AI 需要「記憶管理」，而不只是「記憶儲存」

### 6.1.1 天真方案：全部存起來

最直覺的做法是：使用者說了什麼重要的東西，就寫進一個檔案。下次對話再載入。

```
Session 1: 使用者說「我用 Python」        → 存入 memory.txt
Session 2: 使用者說「用 pytest 做測試」    → 追加 memory.txt
Session 3: 使用者說「專案改用 TypeScript」 → 追加 memory.txt
...
Session 50: memory.txt 已經 500 行
Session 100: memory.txt 已經 2000 行
```

問題來了：

| 問題 | 具體影響 |
|------|---------|
| **容量爆炸** | 2000 行記憶全部塞進 system prompt，浪費 token |
| **資訊衝突** | 「我用 Python」和「專案改用 TypeScript」同時存在，agent 不知道信哪個 |
| **噪音淹沒** | 重要的偏好被淹沒在細碎的對話片段中 |
| **過時不自知** | 三個月前的專案狀態還被當成事實引用 |

這就像一個人從不整理書桌——一開始很方便，但最終什麼都找不到。

### 6.1.2 正確的抽象：記憶 = 儲存 + 抽取 + 整理 + 淘汰

Claude Code 的設計者認識到：記憶管理是一個**完整的生命週期**，不是一個存取動作。

```
┌─────────────────────────────────────────────────────────────┐
│                    記憶的完整生命週期                          │
│                                                             │
│  ① 產生    使用者在對話中透露偏好、修正行為、提及專案狀態       │
│      ↓                                                      │
│  ② 抽取    extractMemories 在每次查詢後自動抽取重要資訊        │
│      ↓                                                      │
│  ③ 儲存    寫入分類的記憶檔案，更新索引 MEMORY.md             │
│      ↓                                                      │
│  ④ 載入    下次對話時，MEMORY.md 被載入 system prompt          │
│      ↓                                                      │
│  ⑤ 整理    AutoDream 定期合併重複、轉換日期、刪除矛盾          │
│      ↓                                                      │
│  ⑥ 淘汰    過時的記憶被警告標記，最終在整理階段被移除           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

這個生命週期的關鍵洞見：**抽取和整理是兩個不同頻率的操作**。抽取要即時（每次查詢後），整理要低頻（每天一次）。這就是雙層架構的來源。

### 6.1.3 人類記憶的類比

如果你學過認知科學，Claude Code 的記憶系統和人類記憶有驚人的對應：

| 人類記憶 | Claude Code | 功能 |
|---------|-------------|------|
| 工作記憶 | Context window | 當前對話的短期存儲 |
| 海馬體（編碼） | extractMemories | 把短期記憶轉為長期記憶 |
| 睡眠整合 | AutoDream | 定期整理、鞏固、淘汰記憶 |
| 長期記憶 | MEMORY.md + 記憶檔案 | 跨 session 的持久儲存 |
| 遺忘曲線 | memoryFreshnessText | 越久的記憶越不可靠 |

AutoDream 這個名字不是隨便取的——它就是 AI 的「作夢」，在使用者不活躍的時候整理白天學到的東西。

---

## 6.2 雙層架構總覽

### 6.2.1 架構圖

```
使用者對話
    │
    ├──────────── 即時層 ──────────────────────────────────┐
    │                                                      │
    │  ┌──────────────────────────────────────────────┐    │
    │  │         extractMemories (每次查詢後)            │    │
    │  │                                                │    │
    │  │  觸發: fire-and-forget，不阻塞回應              │    │
    │  │  範圍: 只看當前對話的新訊息                      │    │
    │  │  輪數: ≤5 輪                                    │    │
    │  │  動作: 抽取 → 寫入記憶檔 → 更新 MEMORY.md       │    │
    │  │  互斥: 如果主 agent 已寫記憶 → 跳過              │    │
    │  │                                                │    │
    │  └──────────────────┬───────────────────────────┘    │
    │                     │                                │
    │                     ▼                                │
    │  ┌──────────────────────────────────────────────┐    │
    │  │              記憶儲存層                         │    │
    │  │                                                │    │
    │  │  ~/.claude/projects/<path>/memory/              │    │
    │  │  ├── MEMORY.md        索引 (≤200行, ≤25KB)     │    │
    │  │  ├── user_role.md     用戶記憶                  │    │
    │  │  ├── feedback_*.md    行為修正記憶               │    │
    │  │  ├── project_*.md     專案狀態記憶               │    │
    │  │  └── reference_*.md   參考資料記憶               │    │
    │  │                                                │    │
    │  └──────────────────┬───────────────────────────┘    │
    │                     │                                │
    ├──────────── 定期層 ──┼───────────────────────────────┘
    │                     │
    │  ┌──────────────────▼───────────────────────────┐
    │  │         AutoDream (每 24h + ≥5 新 session)     │
    │  │                                                │
    │  │  觸發: 6 道 gate 全部通過                       │
    │  │  範圍: 所有歷史 session 的 transcript           │
    │  │  輪數: 由 agent 自行決定（UI 顯示上限 30）       │
    │  │  動作: 定位 → 收集 → 整合 → 修剪                │
    │  │  互斥: .consolidate-lock 檔案鎖                 │
    │  │                                                │
    │  └────────────────────────────────────────────────┘
    │
    ▼
下次對話載入 MEMORY.md → 注入 system prompt
```

### 6.2.2 為什麼是兩層，不是一層或三層？

**一層不夠**：如果只有即時抽取，記憶會越來越多、越來越雜。沒有人整理，衝突和冗餘會持續累積。

**三層太多**：增加中間層（比如「每小時快速整理」）會帶來額外的併發問題和 token 消耗，但收益有限——記憶的變化速度不需要這麼高頻的整理。

**兩層剛好**：
- 即時層保證**不遺漏**——每次對話的重要資訊都會被捕獲
- 定期層保證**不混亂**——累積的記憶會被定期清理和結構化

這就像公司的文件管理：每天開完會都要做會議紀錄（即時抽取），但每季度要整理一次知識庫（定期整理）。兩者缺一不可。

---

## 6.3 記憶檔案格式與四類型分類

### 6.3.1 Frontmatter 格式

**源碼**: `src/memdir/memoryTypes.ts`

每個記憶檔案都是一個 Markdown 檔案，帶有 YAML frontmatter：

```markdown
---
name: User Programming Background
type: user
description: Leo is a theme-driven investor and senior engineer
---

Leo has deep experience with Python and TypeScript.
Prefers concise, example-driven explanations.
Currently focused on AI hardware supply chain analysis.
Uses Notion for investment rules and research management.
```

三個必填欄位：

| 欄位 | 用途 | 限制 |
|------|------|------|
| `name` | 記憶的人類可讀名稱 | 簡短描述性標題 |
| `type` | 記憶分類 | `user` / `feedback` / `project` / `reference` 四選一 |
| `description` | 一句話摘要 | 用於索引和快速瀏覽 |

### 6.3.2 四種記憶類型的完整解析

#### 類型一：`user`（用戶記憶）

**什麼該存**：用戶是誰、背景、偏好、技能水準、溝通風格。

**特徵**：長期穩定，很少需要更新。

**典型內容**：
```markdown
---
name: User Profile
type: user
description: Senior engineer focused on AI hardware supply chain
---

- 主要使用 Python 和 TypeScript
- 偏好簡潔解釋，不需要基礎概念說明
- 台股投資者，關注 AI 硬體供應鏈
- 習慣用 Notion 管理投資規則
```

**什麼時候觸發保存**：
- 使用者說「我是...」「我做...」「我偏好...」
- 使用者持續表現出某種偏好（連續三次都選擇某種風格）
- 使用者明確要求「記住這個」

#### 類型二：`feedback`（行為修正記憶）

**什麼該存**：使用者對 agent 行為的修正——「做這個」「不要做那個」。

**特徵**：長期穩定，是 agent 行為的「法律」。

**典型內容**：
```markdown
---
name: Code Style Feedback
type: feedback
description: Always check quarterly data, not annual, for inflection points
---

- 分析股票時，用季度數據而非年度數據判斷拐點
- 計算子公司價值時，要換算到每股持有量
- 回答前先確認數據是否為當日數據，不要混用不同日期
- 風險追蹤時，永遠不要用過期數據
```

**什麼時候觸發保存**：
- 使用者修正了 agent 的錯誤：「不是這樣，應該用...」
- 使用者設定規則：「以後都要先確認...」
- 使用者表示不滿：「太長了，以後簡短一點」

**為什麼 feedback 和 user 要分開**：

feedback 是行為指令（「做/不做」），user 是背景資訊（「我是誰」）。整理時策略不同：
- user 記憶通常只需要合併更新
- feedback 記憶需要檢查是否有矛盾（用戶先說要詳細，後來又說要簡短）

#### 類型三：`project`（專案記憶）

**什麼該存**：當前正在進行的工作、目標、截止日、進度狀態。

**特徵**：中期存在，最容易過時。

**典型內容**：
```markdown
---
name: Portfolio Risk Tracking
type: project
description: Tracking US-Iran/Hormuz crisis risk signals for re-entry timing
---

Started: 2026-03-23
Status: Active monitoring

Key signals to watch:
- US-Iran diplomatic developments
- Hormuz strait shipping traffic data
- Oil price movements
- Defense sector stock reactions

Current assessment: Elevated risk, holding position
Next review: 2026-04-07
```

**什麼時候觸發保存**：
- 使用者開始一個新任務：「幫我追蹤...」
- 使用者設定截止日期：「這個月底前要完成...」
- 專案狀態改變：「這個 PR 已經合併了」

**為什麼 project 需要特別關注過時性**：

project 記憶描述的是「正在進行的事」，天然有時效性。三個月前的「目前進度：Phase 2」可能早就結束了。AutoDream 整理時會重點檢查 project 類型的記憶。

#### 類型四：`reference`（參考資料記憶）

**什麼該存**：外部系統的指針、文件位置、API 端點、設定位置。

**特徵**：長期穩定，指向外部實體。

**典型內容**：
```markdown
---
name: Project References
type: reference
description: Key file locations and external system pointers
---

- 投資規則：Notion database (linked in bookmarks)
- 股票篩選器：~/scripts/stock-screener/
- 每日風險追蹤：~/claude-code-analysis/docs/risk-tracking/
- API keys: stored in ~/.env (never commit)
```

**什麼時候觸發保存**：
- 使用者提到重要文件位置：「規則都在 Notion 裡」
- 使用者指向外部系統：「CI/CD 用 GitHub Actions」
- 使用者建立資源連結：「測試資料放在...」

### 6.3.3 四類型的生命週期比較

```
穩定性 ──────────────────────────────────────────▶

  ┌─────────┐
  │  user   │ ████████████████████████████████████  幾乎不變
  └─────────┘

  ┌──────────┐
  │ feedback │ ██████████████████████████████████   很少變，但可能被後續修正覆蓋
  └──────────┘

  ┌───────────┐
  │ reference │ ████████████████████████████        穩定，但外部系統可能搬遷
  └───────────┘

  ┌──────────┐
  │ project  │ ██████████████                      最容易過時，需要頻繁更新
  └──────────┘
```

### 6.3.4 記憶目錄結構

完整的記憶目錄長這樣：

```
~/.claude/projects/<project-path-hash>/memory/
├── MEMORY.md                          # 索引檔（入口點）
├── user_profile.md                    # 用戶背景
├── user_communication_preferences.md  # 溝通風格偏好
├── feedback_code_style.md             # 程式碼風格修正
├── feedback_data_accuracy.md          # 數據準確性要求
├── project_risk_tracking_mar2026.md   # 風險追蹤專案
├── project_agent_growth_goals.md      # Agent 成長目標
├── reference_external_systems.md      # 外部系統指針
└── reference_file_locations.md        # 檔案位置
```

MEMORY.md 是索引，內容像這樣：

```markdown
# Memory Index

- [user_profile.md](user_profile.md) — Leo is a senior engineer and theme-driven investor
- [feedback_data_accuracy.md](feedback_data_accuracy.md) — Always verify data is current-day before reporting
- [project_risk_tracking_mar2026.md](project_risk_tracking_mar2026.md) — Tracking US-Iran crisis risk signals
```

每條索引一行，控制在約 150 字元以內。指向記憶檔案的相對路徑，加上一句話的描述。

---

## 6.4 MEMORY.md 的硬限制與截斷邏輯

### 6.4.1 為什麼需要硬限制

MEMORY.md 會被載入到 system prompt 中。如果不限制大小：
- 10KB 的記憶 = 約 2,500 tokens
- 50KB 的記憶 = 約 12,500 tokens
- 每次對話都要支付這些 token，即使大部分記憶跟當前任務無關

而且越大的 system prompt，model 的注意力越分散。精簡的記憶比冗長的記憶更有效。

### 6.4.2 三個常數

**源碼**: `src/memdir/memdir.ts:34-38`

```typescript
const ENTRYPOINT_NAME = 'MEMORY.md'
const MAX_ENTRYPOINT_LINES = 200       // 行數上限
const MAX_ENTRYPOINT_BYTES = 25_000    // 位元組上限 (25KB)
```

為什麼是這兩個數字？

**200 行**：假設每行 ~150 字元（一條索引 entry），200 行足以索引 150-180 個記憶檔案（扣除標題、空行、分隔線）。這對絕大多數使用者來說綽綽有餘。

**25KB**：這是位元組上限，和行數上限是互補的。如果有人把索引 entry 寫得很長（比如每行 500 字元），行數限制攔不住，但位元組限制可以。25KB 大約是 6,000 tokens，在可接受範圍內。

### 6.4.3 截斷邏輯：行優先，再看位元組

**源碼**: `src/memdir/memdir.ts:57-103`

```typescript
function truncateMemoryIndex(content: string): string {
  const lines = content.split('\n')
  let result = content
  let reasons: string[] = []

  // 第一步：行截斷
  if (lines.length > MAX_ENTRYPOINT_LINES) {
    result = lines.slice(0, MAX_ENTRYPOINT_LINES).join('\n')
    reasons.push(`${lines.length} lines (limit: ${MAX_ENTRYPOINT_LINES})`)
  }

  // 第二步：位元組截斷
  const bytes = Buffer.byteLength(result, 'utf8')
  if (bytes > MAX_ENTRYPOINT_BYTES) {
    // 找到不超過 25KB 的最後一個換行符位置
    // 在那裡截斷，保持行的完整性
    let truncated = Buffer.from(result, 'utf8')
      .slice(0, MAX_ENTRYPOINT_BYTES)
      .toString('utf8')
    const lastNewline = truncated.lastIndexOf('\n')
    if (lastNewline > 0) {
      truncated = truncated.slice(0, lastNewline)
    }
    result = truncated
    reasons.push(
      `${(bytes / 1024).toFixed(1)} KB (limit: ${MAX_ENTRYPOINT_BYTES / 1024} KB)` +
      ` — index entries are too long`
    )
  }

  // 附加警告
  if (reasons.length > 0) {
    result += '\n\n> WARNING: MEMORY.md is ' +
      reasons.join(' and ') +
      '. Only part of it was loaded.'
  }

  return result
}
```

### 6.4.4 三種截斷場景

**場景 1：只超行數**

```
記憶條目數量：250 行（超過 200 行上限）
記憶大小：18KB（未超過 25KB 上限）

截斷結果：取前 200 行
警告訊息：
> WARNING: MEMORY.md is 250 lines (limit: 200). Only part of it was loaded.
```

**場景 2：只超位元組**

```
記憶條目數量：180 行（未超過 200 行上限）
記憶大小：30KB（超過 25KB 上限）—— 每行平均 170 bytes

截斷結果：在 25KB 處找到最後一個換行符，在那裡截斷
警告訊息：
> WARNING: MEMORY.md is 30.0 KB (limit: 25 KB) — index entries are too long. Only part of it was loaded.
```

**場景 3：兩者都超**

```
記憶條目數量：300 行
先行截斷到 200 行，但位元組仍有 28KB

截斷結果：先截行，再截位元組
警告訊息：
> WARNING: MEMORY.md is 300 lines and 28.0 KB. Only part of it was loaded.
```

### 6.4.5 截斷警告的用途

這個警告不只是給使用者看的——它被注入到 system prompt 中。當 agent 看到這個警告，它會知道：

1. 記憶索引不完整，可能有遺漏
2. 應該建議使用者或 AutoDream 整理記憶
3. 不要假設記憶的完整性

這是一個**self-aware 機制**——讓 agent 知道自己的記憶有限。

---

## 6.5 記憶過時偵測

### 6.5.1 問題：記憶不等於事實

記憶是**某個時間點的觀察**，不是**永恆的事實**。

```
Day 1: 使用者的 main branch 有 bug → 記憶：「main 有 bug」
Day 5: bug 已修復 → 但記憶還寫著「main 有 bug」
Day 10: Agent 引用記憶，告訴使用者「main 有 bug」→ 錯誤！
```

最天真的解法是：過了 N 天就刪除。但這太激進了——「使用者偏好簡潔回答」這種記憶，30 天後一樣有效。

### 6.5.2 Claude Code 的解法：警告而非刪除

**源碼**: `src/memdir/memoryAge.ts`

```typescript
function memoryAgeDays(mtimeMs: number): number {
  return Math.floor((Date.now() - mtimeMs) / (1000 * 60 * 60 * 24))
}

function memoryFreshnessText(mtimeMs: number): string {
  const days = memoryAgeDays(mtimeMs)
  if (days <= 1) return ''  // 一天以內：新鮮，不警告

  return (
    `This memory is ${days} days old. ` +
    `Memories are point-in-time observations, not live state — ` +
    `claims about code behavior or file:line citations may be outdated. ` +
    `Verify against current code before asserting as fact.`
  )
}
```

邏輯非常簡潔：

```
記憶年齡 ≤ 1 天  → 不附加任何警告（新鮮）
記憶年齡 > 1 天  → 附加警告文字
```

### 6.5.3 警告文字的精心設計

讓我們拆解這段警告文字，每個詞都有工程考量：

```
"This memory is 15 days old."
```
↑ 量化年齡，讓 agent 自己判斷嚴重程度。15 天的 project 記憶很可能過時，但 15 天的 user 記憶大概還有效。

```
"Memories are point-in-time observations, not live state —"
```
↑ 最關鍵的一句。把記憶框架從「事實」重新定義為「觀察」。這改變了 agent 對記憶的信任等級。

```
"claims about code behavior or file:line citations may be outdated."
```
↑ 具體指出最容易過時的內容類型。code behavior 和 file:line 是變化最快的。

```
"Verify against current code before asserting as fact."
```
↑ 給出行動指令：先驗證再引用。不是「忽略這個記憶」，而是「去確認一下」。

### 6.5.4 為什麼選擇「警告」而非「刪除」

| 方案 | 優點 | 缺點 |
|------|------|------|
| **自動刪除** | 乾淨、無過時資訊 | 可能刪掉仍然有效的記憶（「用戶偏好簡潔」30天後仍有效） |
| **自動降級** | 靈活 | 需要定義降級規則，複雜度高 |
| **警告 + agent 判斷** | 保留資訊，由 agent 根據上下文決定 | 依賴 agent 的判斷力 |

Claude Code 選了第三個——因為**記憶是否過時取決於上下文**。

- 「使用者是 Python 工程師」→ 30 天後大概還有效
- 「main branch 的 CI 壞了」→ 3 天後大概已經修了
- 「使用者在做 Q4 報告」→ 90 天後肯定過時了

沒有一個通用的過期規則能處理所有情況。讓 agent 根據類型和上下文判斷，是最務實的方案。

### 6.5.5 mtime 作為唯一的「鮮度信號」

注意 `memoryAgeDays()` 用的是檔案的 `mtime`（最後修改時間），不是檔案的建立時間或內容中的日期。

這很重要：
- 如果 AutoDream 整理了一個記憶檔案（即使只是改了格式），mtime 會更新
- mtime 更新 = 記憶被「驗證過」= 重置鮮度
- 這意味著定期整理有一個副作用：**碰過的記憶會自動續期**

---

## 6.6 Extract Memories — 即時抽取層

### 6.6.1 核心定位

**源碼**: `src/services/extractMemories/extractMemories.ts`

extractMemories 是記憶系統的**第一層**——在每次查詢結束後，自動從對話中抽取值得記住的資訊。

```
使用者提問 → Agent 回答 → 回應送出 → extractMemories 啟動（背景）
                                            │
                                            ├── 掃描新訊息
                                            ├── 判斷有無值得記憶的內容
                                            ├── 寫入記憶檔案
                                            └── 更新 MEMORY.md 索引
```

關鍵特性：**fire-and-forget**。它在回應送出後啟動，不阻塞使用者的下一次輸入。使用者完全感受不到它的存在。

### 6.6.2 游標追蹤：lastMemoryMessageUuid

```typescript
let lastMemoryMessageUuid: string | undefined
```

這是一個 module-level 變數，追蹤「上次抽取記憶時處理到哪個訊息」。

```
對話訊息流：
  msg-001 (user)     ← 上次已處理
  msg-002 (assistant) ← 上次已處理
  msg-003 (user)     ← 新的！需要處理
  msg-004 (assistant) ← 新的！需要處理
                       ↑
                lastMemoryMessageUuid = "msg-002"
```

每次 extractMemories 執行時：
1. 從 `lastMemoryMessageUuid` 之後開始掃描
2. 只處理新訊息，避免重複抽取
3. 完成後更新游標到最新訊息

### 6.6.3 互斥機制：hasMemoryWritesSince()

```typescript
function hasMemoryWritesSince(messages: Message[], sinceUuid: string): boolean
```

這個函式檢查：在 `sinceUuid` 之後的訊息中，主 agent 是否已經寫過記憶檔案。

為什麼需要這個？

```
場景：
  使用者說「記住：我偏好 TypeScript」
  → 主 agent 直接寫了記憶檔案（因為使用者明確要求）
  → 查詢結束
  → extractMemories 啟動
  → 看到「記住：我偏好 TypeScript」
  → 又寫了一次？！→ 重複！
```

`hasMemoryWritesSince()` 防止這種情況：如果主 agent 已經在這輪對話中寫了記憶，extractor 就跳過這輪。

互斥的判斷方式是檢查訊息中是否有對記憶目錄的寫入操作（FileEdit 或 FileWrite tool use 指向記憶路徑）。

### 6.6.4 節流控制

```
Feature flag: tengu_bramble_lintel
Default behavior: 每個 turn 都可能觸發
```

extractMemories 透過 feature flag 控制觸發頻率。預設情況下，每次 agent 回應後都有機會觸發抽取。但 feature flag 可以調整為更低的頻率（比如每 N 個 turn 觸發一次），用於控制 token 消耗。

### 6.6.5 最大輪數：5 輪硬上限

```
const MAX_EXTRACT_TURNS = 5
```

extractMemories 的 forked agent 最多執行 5 個 tool-use 輪次。這是一個安全閥門：

- 通常 1-2 輪就夠了（讀現有記憶 + 寫入/更新）
- 複雜情況需要 3-4 輪（需要先讀多個檔案再決定寫到哪裡）
- 5 輪是硬上限——超過就強制停止，防止失控

為什麼是 5 而不是 10？因為 extractMemories 是 fire-and-forget 的背景任務，每次查詢都可能觸發。如果允許太多輪次，token 消耗會失控。

### 6.6.6 工具限制：createAutoMemCanUseTool()

**這是記憶系統安全性的核心機制。**

extractMemories 的 forked agent 不是一個全功能的 agent——它被嚴格限制了可用工具：

```typescript
function createAutoMemCanUseTool(memoryDir: string): ToolFilter {
  return (toolName: string, toolInput: any) => {
    // ✅ 允許：讀取類工具，無限制
    if (toolName === 'Read')  return { allowed: true }
    if (toolName === 'Grep')  return { allowed: true }
    if (toolName === 'Glob')  return { allowed: true }

    // ✅ 允許：Bash，但只限唯讀命令
    if (toolName === 'Bash') {
      const cmd = toolInput.command?.trim() || ''
      const readOnlyPrefixes = [
        'ls', 'find', 'grep', 'cat', 'stat', 'wc', 'head', 'tail'
      ]
      const isReadOnly = readOnlyPrefixes.some(prefix =>
        cmd.startsWith(prefix + ' ') || cmd === prefix
      )
      if (isReadOnly) return { allowed: true }
      return { allowed: false, reason: 'Only read-only bash commands allowed' }
    }

    // ✅ 允許：Edit/Write，但只限記憶目錄
    if (toolName === 'Edit' || toolName === 'Write') {
      const filePath = toolInput.file_path || ''
      if (filePath.startsWith(memoryDir)) return { allowed: true }
      return {
        allowed: false,
        reason: `Can only write to memory directory: ${memoryDir}`
      }
    }

    // ❌ 禁止：其他所有工具
    return { allowed: false, reason: 'Tool not available for memory extraction' }
  }
}
```

用表格整理：

| 工具 | 權限 | 限制條件 |
|------|------|---------|
| Read | 允許 | 無限制 |
| Grep | 允許 | 無限制 |
| Glob | 允許 | 無限制 |
| Bash | 部分允許 | 只限 `ls`, `find`, `grep`, `cat`, `stat`, `wc`, `head`, `tail` |
| Edit | 部分允許 | 只限記憶目錄內的路徑 |
| Write | 部分允許 | 只限記憶目錄內的路徑 |
| Agent/Fork | 禁止 | — |
| MCP 工具 | 禁止 | — |
| 其他寫入工具 | 禁止 | — |

**為什麼 Bash 要白名單而不是黑名單**：

黑名單（「禁止 rm, mv, chmod...」）永遠不完整——有無數方式可以透過 Bash 造成破壞（`python -c "import os; os.remove(...)"`）。白名單（「只允許這 8 個唯讀命令」）簡單、安全、沒有遺漏。

**為什麼 Read 不限制但 Write 限制**：

記憶 agent 需要讀取專案的任何檔案來理解上下文（比如讀程式碼來確認記憶是否過時），但它不應該修改任何專案檔案——它的唯一職責是管理記憶。

### 6.6.7 extractMemories 的完整執行流程

```
extractMemories() 被呼叫
    │
    ├── 檢查 feature flag (tengu_bramble_lintel)
    │   └── 未啟用？→ return
    │
    ├── 計算新訊息數量 (countModelVisibleMessagesSince)
    │   └── 0 條新訊息？→ return
    │
    ├── 檢查互斥 (hasMemoryWritesSince)
    │   └── 主 agent 已寫記憶？→ return（更新游標，跳過抽取）
    │
    ├── 建立 forked agent
    │   ├── 注入 prompt：「分析以下對話，抽取值得記憶的資訊」
    │   ├── 設定工具限制：createAutoMemCanUseTool()
    │   ├── 設定 maxTurns: 5
    │   └── 設定 skipTranscript: true
    │
    ├── 執行 agent
    │   ├── Agent 讀取現有 MEMORY.md
    │   ├── Agent 讀取相關記憶檔案
    │   ├── Agent 決定是否需要新增/更新記憶
    │   └── Agent 寫入記憶（如果需要）
    │
    └── 更新游標 lastMemoryMessageUuid
```

---

## 6.7 AutoDream — 定期整理層

### 6.7.1 核心定位

**源碼**: `src/services/autoDream/autoDream.ts`

AutoDream 是記憶系統的**第二層**——一個低頻、高價值的維護任務，在背景整理累積的記憶。

如果 extractMemories 是「每天做筆記」，AutoDream 就是「週末整理書桌」：

```
extractMemories:                    AutoDream:
┌─────────────┐                    ┌─────────────────────┐
│ 高頻、低成本  │                    │ 低頻、高成本           │
│ 每次查詢後    │                    │ 每 24h + 5 session   │
│ ≤5 輪        │                    │ Agent 自行決定輪數    │
│ 只看當前對話  │                    │ 掃描多個歷史 session  │
│ 新增記憶      │                    │ 合併、清理、重組記憶   │
└─────────────┘                    └─────────────────────┘
```

### 6.7.2 六道觸發 Gate——從便宜到貴

AutoDream 不是每次對話都觸發的——它有六道 gate，全部通過才執行。而且 gate 的排列順序是精心設計的：**從成本最低到成本最高**。

```
                    成本
Gate 1: Feature flag          ← 0（讀環境變數）
Gate 2: Environment checks    ← 0（讀環境變數）
Gate 3: Time ≥24h             ← ~0（一次 stat() 呼叫）
Gate 4: Scan throttle ≥10min  ← 0（讀 module 變數）
Gate 5: ≥5 new sessions       ← 中（目錄掃描）
Gate 6: Lock acquisition      ← 高（檔案寫入 + 驗證）
    │
    ▼
  執行 AutoDream              ← 最高（啟動 forked agent，消耗 token）
```

如果在 Gate 1 就失敗了（feature flag 關閉），後面的 gate 都不需要檢查。這省下了不必要的 stat() 呼叫和目錄掃描。

#### Gate 1：Feature Flag（成本：0）

```typescript
if (!isFeatureEnabled('tengu_onyx_plover')) return
if (process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY) return
```

最便宜的檢查——讀一個環境變數。如果 feature flag 關閉或使用者明確禁用了自動記憶，立刻退出。

#### Gate 2：Environment Checks（成本：0）

```typescript
if (isKairosMode()) return      // Kairos 模式不需要
if (isRemoteMode()) return      // 遠端模式不需要
if (!isAutoMemoryEnabled()) return  // 使用者關閉了自動記憶
```

仍然只是讀環境變數——成本為零。但它們過濾掉了幾種不需要記憶整理的場景。

**為什麼 Kairos 和 Remote 模式不需要？** Kairos 是無狀態的 API 模式，沒有持久會話。Remote 模式的記憶由遠端管理。

#### Gate 3：Time Gate — ≥24 小時（成本：一次 stat()）

```typescript
const lockStat = await stat(consolidateLockPath).catch(() => null)
const lastRunMs = lockStat?.mtimeMs ?? 0
const hoursSince = (Date.now() - lastRunMs) / (1000 * 60 * 60)
if (hoursSince < 24) return  // minHours: 24
```

這裡巧妙地利用了 `.consolidate-lock` 檔案的 **mtime** 來判斷上次整理的時間。只需要一次 `stat()` 系統呼叫——比讀檔案內容便宜得多。

**為什麼是 24 小時？** 記憶整理是「重量級」操作——啟動一個 forked agent、掃描多個 session 的 transcript、可能消耗數千 token。一天一次是合理的頻率。

#### Gate 4：Scan Throttle — ≥10 分鐘（成本：0）

```typescript
const SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000  // 10 分鐘

let lastScanTimestamp = 0  // module-level 變數

if (Date.now() - lastScanTimestamp < SESSION_SCAN_INTERVAL_MS) return
lastScanTimestamp = Date.now()
```

這是一個**記憶體內的節流**。即使通過了 Time Gate（24 小時以上沒整理），也不希望每秒都掃描 session 目錄。10 分鐘的掃描間隔確保 Gate 5 的目錄掃描不會太頻繁。

注意：這個 gate 用的是 module-level 變數（`lastScanTimestamp`），不是檔案系統——成本幾乎為零。

#### Gate 5：Session Gate — ≥5 個新 Session（成本：中等）

```typescript
const newSessions = await listSessionsTouchedSince(lastRunMs)
if (newSessions.length < 5) return  // minSessions: 5
```

這是第一個需要掃描檔案系統的 gate。它讀取 sessions 目錄，計算上次整理之後有多少個新 session。

**為什麼是 5 個？** 太少（1-2 個）可能觸發不必要的整理——新的資訊量不夠。太多（10+）可能錯過重要的記憶更新。5 是一個平衡點。

#### Gate 6：Lock Acquisition（成本：高）

```typescript
const priorMtime = await tryAcquireConsolidationLock()
if (priorMtime === null) return  // 有人正在整理
```

最後一道 gate——嘗試取得互斥鎖。如果另一個 Claude Code 實例正在整理，這裡會失敗。

這道 gate 放在最後，因為取得鎖涉及檔案寫入和驗證（見 6.8 節），是最昂貴的操作。

### 6.7.3 Gate 設計的工程哲學

六道 gate 的排列不是隨意的，它體現了一個重要的設計模式：**progressive cost filtering**（漸進成本過濾）。

```
100% 的呼叫到達 Gate 1 (成本: 0)
 ├── 5% 被過濾（feature flag 關閉）
 ▼
95% 到達 Gate 2 (成本: 0)
 ├── 2% 被過濾（特殊環境）
 ▼
93% 到達 Gate 3 (成本: ~0)
 ├── 90% 被過濾（24小時內已整理過）
 ▼
3% 到達 Gate 4 (成本: 0)
 ├── 2% 被過濾（10分鐘內已掃描過）
 ▼
1% 到達 Gate 5 (成本: 中)
 ├── 0.5% 被過濾（新 session 不夠）
 ▼
0.5% 到達 Gate 6 (成本: 高)
 ├── 0.1% 被過濾（鎖被占用）
 ▼
~0.4% 真正執行 AutoDream (成本: 最高)
```

絕大多數呼叫在 Gate 3 就被過濾了（因為 24 小時內肯定已經檢查過很多次了）。只有極少數呼叫需要執行昂貴的目錄掃描和鎖操作。

**如果反過來排列呢？**

如果把 Lock Acquisition 放在第一個 gate：
- 每次呼叫都要嘗試取得鎖（檔案寫入）
- 即使在 feature flag 關閉的情況下也要寫檔案
- 完全是浪費

這個模式可以推廣到任何有多個前置條件的系統：**把便宜的檢查放前面，貴的放後面**。

---

## 6.8 鎖機制

### 6.8.1 為什麼需要鎖

使用者可能同時開多個 terminal，每個 terminal 都跑著 Claude Code。如果兩個實例同時觸發 AutoDream：

```
Terminal A: Gate 1 ✓ → Gate 2 ✓ → ... → Gate 5 ✓ → 開始整理！
Terminal B: Gate 1 ✓ → Gate 2 ✓ → ... → Gate 5 ✓ → 也開始整理！

結果：
- 兩個 agent 同時讀寫 MEMORY.md
- 一個把 entry 刪了，另一個正在更新同一個 entry
- 記憶檔案損壞
```

所以需要互斥——在同一時間只有一個 AutoDream 實例可以執行。

### 6.8.2 .consolidate-lock 檔案

**源碼**: `src/services/autoDream/consolidationLock.ts`

```
~/.claude/projects/<path>/memory/.consolidate-lock
├── 內容：持有者的 PID（Process ID）
├── mtime：上次成功整理的時間戳
└── 如果超過 1 小時且 PID 不在了 → 視為過期
```

這個檔案承載了兩個角色：
1. **互斥鎖**：內容是持有者的 PID
2. **時間戳記**：mtime 記錄上次整理的時間

### 6.8.3 鎖的取得：tryAcquireConsolidationLock()

```typescript
const HOLDER_STALE_MS = 60 * 60 * 1000  // 1 小時

async function tryAcquireConsolidationLock(): Promise<number | null> {
  const lockPath = path.join(memoryDir, '.consolidate-lock')

  // Step 1: stat() 取得 mtime 和 PID
  const lockStat = await stat(lockPath).catch(() => null)

  if (lockStat) {
    const lockContent = await readFile(lockPath, 'utf8')
    const holderPid = parseInt(lockContent, 10)
    const ageMs = Date.now() - lockStat.mtimeMs

    // Step 2: 檢查鎖是否還有效
    if (ageMs < HOLDER_STALE_MS && isPidAlive(holderPid)) {
      return null  // 鎖有效且持有者還活著 → 取得失敗
    }
    // 否則：鎖過期或持有者死亡 → 可以搶占
  }

  const priorMtime = lockStat?.mtimeMs ?? 0

  // Step 3: 寫入自己的 PID
  await writeFile(lockPath, String(process.pid))

  // Step 4: 讀回驗證（防止併發寫入競爭）
  const verification = await readFile(lockPath, 'utf8')
  if (parseInt(verification, 10) !== process.pid) {
    return null  // 有人在我之後寫入了 → 取得失敗
  }

  // Step 5: 成功！回傳 priorMtime（用於回滾）
  return priorMtime
}
```

### 6.8.4 四步驗證的必要性

為什麼要讀回驗證（Step 4）？考慮這個 race condition：

```
時間線：
  T1: Process A: writeFile(lockPath, "1001")
  T2: Process B: writeFile(lockPath, "1002")  ← 覆蓋了 A
  T3: Process A: readFile(lockPath) → "1002"  ← 發現不是自己！→ 放棄
  T4: Process B: readFile(lockPath) → "1002"  ← 確認是自己 → 取得成功
```

沒有 Step 4 的話，Process A 和 B 都會認為自己取得了鎖。

**注意**：這不是一個完美的鎖。在極端情況下（T1 和 T3 之間有其他 process 寫入然後又恢復），仍然可能有問題。但對於 AutoDream 的使用場景，這種極端情況發生的概率極低，而且最壞的結果只是兩個 agent 同時整理記憶——不是資料遺失。

### 6.8.5 為什麼用 mtime 而不是檔案內容

鎖的時間戳存在 mtime 裡，而不是檔案內容的一部分。為什麼？

回顧 Gate 3：

```typescript
const lockStat = await stat(consolidateLockPath).catch(() => null)
const lastRunMs = lockStat?.mtimeMs ?? 0
```

Gate 3 只需要 `stat()` 就能判斷上次整理時間——不需要 `readFile()`。

如果時間戳存在檔案內容裡：

```typescript
// 如果用檔案內容：
const content = await readFile(lockPath, 'utf8')  // 需要 readFile()
const lastRunMs = JSON.parse(content).lastRunTime  // 需要解析
```

`stat()` 是一個系統呼叫，`readFile()` 是兩個（open + read）。對於每次查詢都要檢查的 gate，這個差異累積起來很可觀。

**mtime 作為時間戳的巧妙之處**：
1. Gate 3 只需 stat()，成本最低
2. 鎖取得成功後，AutoDream 完成時更新 mtime → 自動刷新「上次整理時間」
3. 不需要在鎖檔案中維護額外的時間戳欄位

### 6.8.6 鎖的回滾：rollbackConsolidationLock()

如果 AutoDream 被中斷（使用者按 Ctrl+C）或失敗（agent 出錯），需要回滾鎖狀態，讓下一個 session 可以重試。

```typescript
async function rollbackConsolidationLock(priorMtime: number) {
  const lockPath = path.join(memoryDir, '.consolidate-lock')

  if (priorMtime === 0) {
    // 之前沒有鎖檔案 → 刪除
    await unlink(lockPath).catch(() => {})
  } else {
    // 之前有鎖檔案 → 恢復 mtime
    await writeFile(lockPath, '')              // 清除 PID（釋放鎖）
    await utimes(lockPath, priorMtime / 1000)  // 倒回 mtime
  }
}
```

**為什麼要倒回 mtime？**

如果不倒回：
- AutoDream 在 Gate 6 寫入了鎖（mtime 更新為「現在」）
- 然後 AutoDream 失敗
- 下次 Gate 3 檢查時，看到 mtime 是「剛才」→ 認為 24 小時內已整理 → 跳過
- 結果：直到 24 小時後才會重試

倒回 mtime 到之前的值，讓 Gate 3 看到的仍然是上次成功整理的時間。下一個 session 會立刻重試。

### 6.8.7 HOLDER_STALE_MS = 1 小時的考量

為什麼鎖過期時間是 1 小時？

- **太短（5 分鐘）**：AutoDream 可能需要 10-20 分鐘來處理大量記憶。5 分鐘的過期時間會導致還在執行中的 AutoDream 被搶占。
- **太長（24 小時）**：如果 process crash 了（沒有執行回滾），鎖會卡住 24 小時。
- **1 小時**：大部分 AutoDream 在 30 分鐘內完成。如果 1 小時還沒完成，很可能是 crash 了。

配合 `isPidAlive()` 檢查：即使鎖還沒過期（比如才 5 分鐘），如果持有者的 PID 已經不存在了，也可以搶占。1 小時只是 PID 檢查失敗時的備用方案（比如 PID 被另一個 process 重用了）。

---

## 6.9 四階段整理 Prompt

### 6.9.1 Prompt 的角色

**源碼**: `src/services/autoDream/consolidationPrompt.ts:15-64`

AutoDream 的 forked agent 接收一個精心設計的 prompt，指導它按四個階段整理記憶。這個 prompt 不是「幫我整理記憶」這麼模糊——它是一份**操作手冊**。

### 6.9.2 Phase 1: Orient（定位）

```
Phase 1: Orient
- ls the memory directory to see what files exist
- Read MEMORY.md to understand the current index
- Skim existing topic files to avoid creating duplicates
- If there are logs/ or sessions/ subdirectories, check recent entries
```

這個階段的目的是**建立現狀認知**。agent 需要知道：
- 現在有哪些記憶檔案
- MEMORY.md 的索引結構
- 現有記憶涵蓋了哪些主題

**為什麼不直接開始整理？**

如果跳過 Orient，agent 可能會：
- 建立一個 `project_deployment.md`，但其實已經有 `project_deploy_status.md` 了
- 把使用者偏好寫到 project 類型的檔案裡
- 不知道索引的格式，寫出不一致的 MEMORY.md

### 6.9.3 Phase 2: Gather（收集）

```
Phase 2: Gather
1. Daily logs (logs/YYYY/MM/YYYY-MM-DD.md) if they exist
2. Drifted memories — facts that contradict current state
3. Transcript search — only grep when you have a specific suspicion:
   grep -rn "<narrow term>" ${transcriptDir}/ --include="*.jsonl" | tail -50

⚠️ Do NOT exhaustively read transcripts. Only search for things you
   already suspect are important.
```

收集階段有一個關鍵約束：**不要窮舉讀取 transcript**。

Transcript 是對話的完整記錄，可能有數百 MB。如果 agent 嘗試全部讀取：
- Token 消耗爆炸
- 大部分內容是無用的（日常 Q&A、已經記住的東西）
- 可能超出 context window

所以 prompt 明確指示：只在「有具體懷疑」的時候 grep 搜尋。比如：
- 記憶裡寫著「用戶用 Python」，但最近的 session 可能改了語言 → grep "TypeScript" 或 "migration"
- 記憶裡寫著某個截止日期，可能已經延期了 → grep 具體的日期或專案名

### 6.9.4 Phase 3: Consolidate（整合）

```
Phase 3: Consolidate
For each thing worth remembering: write or update a memory file.
Key rules:
- Merge new signals into existing topic files; don't create near-duplicates
- Convert relative dates to absolute dates ("yesterday" → "2026-04-02")
- Delete contradicted facts — if today's info negates old memory, fix at source
```

三條核心規則：

**規則 1：合併而非新增**

```
❌ 錯誤做法：
  已有：project_web_app.md (描述 Web App 專案狀態)
  新增：project_web_app_progress.md (描述同一專案的最新進度)
  → 兩個檔案涵蓋同一主題

✅ 正確做法：
  更新：project_web_app.md，把新進度合併進去
  → 一個檔案，一個主題
```

**規則 2：相對日期 → 絕對日期**

```
❌ 記憶裡寫：「昨天開始做 migration」
  → 30 天後讀到，「昨天」是哪天？

✅ 記憶裡寫：「2026-04-02 開始做 migration」
  → 任何時候讀到都清楚明確
```

這是一個非常實用的規則。extractMemories 在即時抽取時可能直接記下「昨天」「上週」，AutoDream 的整理階段負責把它們轉換成絕對日期。

**規則 3：刪除被推翻的事實**

```
舊記憶：「CI pipeline 用 Jenkins」
新發現：最近的 session 裡使用者說「已經遷移到 GitHub Actions」

→ 不是在舊記憶旁邊加註「已改用 GitHub Actions」
→ 而是直接把「用 Jenkins」改成「用 GitHub Actions」
→ 在源頭修正，不留矛盾
```

### 6.9.5 Phase 4: Prune（修剪）

```
Phase 4: Prune
Update MEMORY.md. Keep it under 200 lines AND ~25KB.
Each entry should be one line, ~150 chars:
  - [Title](file.md) — one-line hook

- Remove pointers to outdated/wrong/superseded memories
- Demote verbose entries: if >200 chars, move details to topic file
- Add pointers to important new memories
- Resolve contradictions
```

修剪階段是**品質控制**。它確保 MEMORY.md 始終是一個精簡、有用的索引。

索引 entry 的格式規範：

```markdown
# 好的 entry（~80 字元）：
- [feedback_data_accuracy.md](feedback_data_accuracy.md) — Always verify data is current-day

# 不好的 entry（~300 字元）：
- [feedback_data_accuracy.md](feedback_data_accuracy.md) — When reporting stock prices or risk metrics, always verify that the data is from today's date and never mix data from different dates or use stale cached data in any tracking or analysis tasks

# 修正方式：把詳細內容移到 feedback_data_accuracy.md，索引只保留 hook
```

~150 字元的限制是一個 soft guideline，確保 200 行的 MEMORY.md 不會因為過長的 entry 而超過 25KB 的位元組限制。

---

## 6.10 Forked Agent 的執行環境

### 6.10.1 runForkedAgent() 的特點

AutoDream 不是在主 agent 的上下文中執行的——它啟動一個**獨立的 forked agent**。

```typescript
const result = await runForkedAgent({
  prompt: consolidationPrompt,
  canUseTool: createAutoMemCanUseTool(memoryDir),
  skipTranscript: true,    // 不記錄到 transcript
  // ... 其他設定
})
```

三個重要參數：

#### prompt：consolidation prompt

就是 6.9 節描述的四階段 prompt。

#### canUseTool：工具限制

和 extractMemories 使用完全相同的 `createAutoMemCanUseTool()` 函式。AutoDream agent 和 extractMemories agent 的工具權限一模一樣。

#### skipTranscript: true

**這個參數至關重要**。

AutoDream agent 的動作不會被記錄到 session transcript 中。為什麼？

1. **避免循環**：如果 AutoDream 的動作被記錄，下次 extractMemories 會看到這些動作，可能嘗試「從 AutoDream 的輸出中抽取記憶」——形成循環。

2. **避免 race condition**：transcript 檔案可能正在被主 agent 寫入。兩個 process 同時寫同一個 JSONL 檔案會導致資料損壞。

3. **減少噪音**：AutoDream 的動作（「我讀了 MEMORY.md」「我更新了 project_*.md」）對使用者沒有價值，不需要佔用 transcript 空間。

### 6.10.2 Prompt Cache 共享

```
forked agent 共享父進程的 prompt cache
```

這是一個重要的效能優化。Claude 的 API 支援 prompt caching——如果兩次請求的 system prompt 相同，第二次可以使用快取，大幅減少延遲和成本。

forked agent 和父 agent 共享相同的基礎 system prompt，所以它可以命中父 agent 的 prompt cache。這意味著 AutoDream 的啟動成本比想像中低得多。

### 6.10.3 Progress Watcher：addDreamTurn()

AutoDream 執行時，UI 需要顯示進度。每次 forked agent 完成一個 turn（一次 assistant 回應），都會透過 `addDreamTurn()` 通知 UI。

```typescript
function addDreamTurn(turn: DreamTurn) {
  // 更新 DreamTaskState.turns 陣列
  // UI 會顯示最近的 turn 內容
}
```

這讓使用者在等待時能看到 AutoDream 在做什麼：「正在讀取 MEMORY.md...」「正在更新 project_status.md...」

---

## 6.11 DreamTask UI 狀態

### 6.11.1 DreamTaskState 的完整結構

**源碼**: `src/tasks/DreamTask/DreamTask.ts`

```typescript
type DreamTaskState = {
  type: 'dream'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]
  turns: DreamTurn[]
  priorMtime: number
}
```

每個欄位的用途：

| 欄位 | 類型 | 用途 |
|------|------|------|
| `phase` | `'starting'` \| `'updating'` | 目前階段：初始化 or 更新中 |
| `sessionsReviewing` | `number` | 正在檢視的 session 數量 |
| `filesTouched` | `string[]` | 被修改過的記憶檔案路徑 |
| `turns` | `DreamTurn[]` | 最近的 assistant 回合（最多 30 個） |
| `priorMtime` | `number` | 取得鎖之前的 mtime（用於回滾） |

### 6.11.2 Phase 狀態轉換

```
              啟動 AutoDream
                    │
                    ▼
            phase: 'starting'
            「正在分析記憶...」
                    │
                    │  第一次寫入記憶檔案
                    ▼
            phase: 'updating'
            「正在更新記憶...」
                    │
                    │  完成或中斷
                    ▼
              Task 結束
```

phase 從 `'starting'` 到 `'updating'` 的轉換時機是**第一次對記憶檔案的寫入操作**。這讓 UI 可以顯示有意義的狀態訊息：

- `starting`：Agent 還在讀取和分析，還沒改任何東西
- `updating`：Agent 已經開始修改記憶了

### 6.11.3 Kill Handler

如果使用者在 AutoDream 執行過程中按 Ctrl+C 或關閉 terminal：

```typescript
// Kill handler
const cleanup = () => {
  abortForkedAgent()                              // 中止 forked agent
  rollbackConsolidationLock(state.priorMtime)     // 回滾鎖
}

process.on('SIGINT', cleanup)
process.on('SIGTERM', cleanup)
```

兩步清理：
1. **中止 agent**：停止 forked agent 的執行，釋放 API 連接
2. **回滾鎖**：恢復 `.consolidate-lock` 的 mtime，讓下次 session 可以重試

### 6.11.4 MAX_TURNS = 30

```typescript
const MAX_TURNS = 30  // DreamTask.ts:12
```

DreamTask 的 `turns` 陣列最多保留 30 個 turn。這不是限制 AutoDream 的執行輪數——agent 可以執行超過 30 輪——而是限制 **UI 顯示的歷史深度**。

超過 30 個 turn 時，舊的 turn 會從陣列頭部移除。使用者永遠只能在 UI 中看到最近 30 個 turn。

---

## 6.12 Session 掃描機制

### 6.12.1 listSessionsTouchedSince()

AutoDream 的 Gate 5 需要知道「上次整理後有多少個新 session」。這個資訊由 `listSessionsTouchedSince()` 提供。

```typescript
async function listSessionsTouchedSince(sinceMs: number): Promise<string[]> {
  const sessionsDir = path.join(projectDir, 'sessions')
  const files = await readdir(sessionsDir)

  const touchedSessions: string[] = []

  for (const file of files) {
    // 排除非 JSONL 檔案
    if (!file.endsWith('.jsonl')) continue

    // 排除當前 session
    if (file === `${currentSessionId}.jsonl`) continue

    // 排除 agent-*.jsonl（forked agent 的 transcript）
    if (file.startsWith('agent-')) continue

    // 檢查 mtime
    const fileStat = await stat(path.join(sessionsDir, file))
    if (fileStat.mtimeMs > sinceMs) {
      touchedSessions.push(file)
    }
  }

  return touchedSessions
}
```

### 6.12.2 三個排除規則

**排除當前 session**：正在進行的對話不應該被 AutoDream 整理——它的記憶由 extractMemories 即時處理。

**排除 agent-*.jsonl**：這些是 forked agent（sub-agent）的 transcript。它們是主 session 的子任務，不是獨立的對話，不需要單獨整理。

**排除非 JSONL**：sessions 目錄可能有其他檔案（比如暫存檔），只看 `.jsonl` 格式的。

### 6.12.3 為什麼用 mtime 而不是 birthtime

```typescript
if (fileStat.mtimeMs > sinceMs)  // 用 mtime
// 而不是
if (fileStat.birthtimeMs > sinceMs)  // 不用 birthtime
```

兩個原因：

1. **跨平台兼容性**：某些 Linux 文件系統不支援 birthtime（建立時間）。mtime（最後修改時間）在所有平台都可靠。

2. **語意更準確**：一個 session 可能在週一建立（birthtime = 週一），但使用者在週三又回來繼續對話（mtime = 週三）。如果用 birthtime，週二的 AutoDream 不會把它算為「新 session」——但它確實有新內容需要整理。

---

## 6.13 Analytics 追蹤

### 6.13.1 三種事件

AutoDream 的執行結果會被追蹤為 analytics 事件：

```typescript
// 觸發時
trackEvent('tengu_auto_dream_fired', {
  sessionsSince: newSessions.length,
  hoursSinceLast: hoursSince,
})

// 成功完成時
trackEvent('tengu_auto_dream_completed', {
  turns: turnCount,
  filesTouched: filesTouched.length,
  durationMs: Date.now() - startTime,
  tokens: {
    cache_read: cacheReadTokens,
    cache_creation: cacheCreationTokens,
    output: outputTokens,
  },
})

// 失敗時
trackEvent('tengu_auto_dream_failed', {
  error: error.message,
  phase: currentPhase,
  turnsCompleted: turnCount,
})
```

### 6.13.2 Token 追蹤

AutoDream 的 token 消耗被細分為三類：

| Token 類型 | 含義 | 成本 |
|-----------|------|------|
| `cache_read` | 從 prompt cache 讀取的 token | 最低（~0.1x） |
| `cache_creation` | 建立新 cache 的 token | 中等（~1x） |
| `output` | Agent 生成的 output token | 最高（~3x） |

這個分類讓工程團隊可以分析：
- AutoDream 是否有效利用了 prompt cache
- 平均每次整理消耗多少 token
- 有沒有異常昂貴的整理（可能 agent 做了太多不必要的 grep）

### 6.13.3 失敗追蹤

失敗事件記錄了失敗的階段（`phase`）和已完成的輪數。這幫助診斷：
- 如果大量失敗在 Phase 1（Orient），可能是記憶目錄結構有問題
- 如果大量失敗在 Phase 2（Gather），可能是 transcript 搜尋有問題
- 如果大量失敗在 Phase 3（Consolidate），可能是工具限制太嚴格

---

## 6.14 設計模式總結

### 模式 1：Cheap Gates First（便宜 Gate 優先）

```
適用場景：有多個前置條件的系統（定時任務、觸發器、事件處理）

核心原則：
  檢查條件的順序 = 成本遞增的順序
  大部分呼叫在最便宜的 gate 就被過濾

Claude Code 實例：
  AutoDream 的六道 gate
  Gate 1-2: 環境變數 (0)
  Gate 3-4: stat() + 記憶體 (~0)
  Gate 5: 目錄掃描 (中)
  Gate 6: 檔案寫入 (高)
```

### 模式 2：Two-Layer Memory（雙層記憶）

```
適用場景：任何需要長期記憶的 AI 應用

核心原則：
  即時層保證不遺漏（每次對話都抽取）
  定期層保證不混亂（定時清理和重組）

Claude Code 實例：
  extractMemories (即時) + AutoDream (定期)
```

### 模式 3：Lock with Stale Detection（帶過期偵測的鎖）

```
適用場景：分散式/多進程的背景任務互斥

核心原則：
  PID + mtime 雙重驗證
  過期門檻 (1hr) 防止死鎖
  回滾機制確保失敗後可重試

Claude Code 實例：
  .consolidate-lock + HOLDER_STALE_MS + rollback
```

### 模式 4：Four-Type Memory Taxonomy（四類記憶分類）

```
適用場景：需要對記憶差異化管理的系統

核心原則：
  不同類型的記憶有不同的壽命和更新策略
  分類讓整理時可以差異化對待

Claude Code 實例：
  user (長期) / feedback (長期) / project (中期) / reference (長期)
```

### 模式 5：Hard Limits with Graceful Truncation（硬限制 + 優雅截斷）

```
適用場景：需要控制資源使用的系統

核心原則：
  設定硬限制（200 行、25KB）
  超限時截斷而非拒絕
  截斷時附加警告，讓系統自知不完整

Claude Code 實例：
  truncateMemoryIndex() + WARNING 訊息
```

### 模式 6：Warning > Deletion for Staleness（警告優於刪除）

```
適用場景：資訊有效性隨時間遞減但不確定何時失效的系統

核心原則：
  不確定是否過時？→ 警告，不刪除
  讓消費者（agent）根據上下文判斷

Claude Code 實例：
  memoryFreshnessText() 附加「N days old」警告
```

---

## 6.15 關鍵數字速查表

| 數字 | 含義 | 源碼位置 |
|------|------|---------|
| 200 | MEMORY.md 行數上限 | `memdir.ts:35` |
| 25,000 | MEMORY.md 位元組上限 (25KB) | `memdir.ts:38` |
| ~150 | 單條索引 entry 建議字元數 | `consolidationPrompt.ts` |
| 4 | 記憶類型數量 (user/feedback/project/reference) | `memoryTypes.ts` |
| 1 天 | 記憶鮮度警告的閾值 | `memoryAge.ts` |
| 5 | extractMemories 最大輪數 | `extractMemories.ts:426` |
| 24 小時 | AutoDream 最小觸發間隔 | `autoDream.ts:64` |
| 5 | AutoDream 需要的最少新 session 數 | `autoDream.ts:65` |
| 10 分鐘 | Session 掃描節流間隔 | `autoDream.ts:56` |
| 1 小時 | 鎖過期時間 (HOLDER_STALE_MS) | `consolidationLock.ts:19` |
| 30 | DreamTask UI 顯示的最大 turn 數 | `DreamTask.ts:12` |
| 200 | 記憶檔案掃描上限 | `memoryScan.ts:21` |
| 30 | Frontmatter 掃描深度（行） | `memoryScan.ts:22` |
| 8 | Bash 白名單命令數 (ls/find/grep/cat/stat/wc/head/tail) | `extractMemories.ts` |
| 6 | AutoDream 觸發 gate 數量 | `autoDream.ts` |
| 4 | AutoDream 整理階段數 | `consolidationPrompt.ts` |

---

## 6.16 章節總結：extractMemories vs AutoDream 完整比較

### 6.16.1 對比表

| 維度 | extractMemories | AutoDream |
|------|----------------|-----------|
| **角色** | 即時記錄者 | 定期整理者 |
| **觸發時機** | 每次查詢結束後 | 24 小時 + ≥5 新 session |
| **觸發方式** | Fire-and-forget | 六道 gate 全部通過 |
| **執行環境** | Forked agent | Forked agent |
| **最大輪數** | 5（硬上限） | Agent 自行決定（UI 顯示上限 30） |
| **作用範圍** | 當前對話的新訊息 | 多個歷史 session 的 transcript |
| **主要動作** | 新增記憶、更新索引 | 合併、清理、轉換日期、修剪索引 |
| **互斥對象** | 主 agent 的記憶寫入 | 其他 AutoDream 實例 |
| **互斥機制** | hasMemoryWritesSince() | .consolidate-lock 檔案鎖 |
| **工具限制** | createAutoMemCanUseTool() | 同上（完全相同） |
| **Transcript 記錄** | skipTranscript: true | skipTranscript: true |
| **Token 消耗** | 低（≤5 輪，小範圍） | 中高（多輪，大範圍 grep） |
| **失敗影響** | 低（下次查詢會重試） | 中（回滾鎖，下次 session 重試） |
| **UI 反饋** | 無（完全靜默） | DreamTask 顯示進度 |

### 6.16.2 協作方式

兩者不是競爭關係，而是互補關係：

```
extractMemories 負責「記下來」：
  Session 1: 記下「用戶偏好簡潔」
  Session 2: 記下「專案用 TypeScript」
  Session 3: 記下「CI 遷移到 GitHub Actions」
  Session 4: 記下「不要用年度數據」
  Session 5: 記下「子公司要算每股持有」

AutoDream 負責「整理好」：
  → 合併 Session 1 和 4 到 feedback_communication.md
  → 把 Session 2 和 3 更新到 project_tech_stack.md
  → 把 Session 5 合併到 feedback_data_accuracy.md
  → 修剪 MEMORY.md，確保 ≤200 行 / ≤25KB
  → 轉換所有相對日期為絕對日期
  → 刪除被後續 session 推翻的舊記憶
```

### 6.16.3 如果只有其中一層

**只有 extractMemories，沒有 AutoDream**：
- 記憶會持續增長
- MEMORY.md 最終會超過 200 行，被截斷
- 重複、矛盾的記憶會累積
- 相對日期（「昨天」）永遠不會被轉換
- 系統仍然可用，但記憶品質會逐漸下降

**只有 AutoDream，沒有 extractMemories**：
- 使用者的即時偏好不會被捕獲
- 要等到 24 小時後才會從 transcript 中抽取
- 短暫的、只在一次對話中出現的重要資訊可能被遺漏
- 記憶的「鮮度」嚴重落後

**兩者都有**：
- extractMemories 確保即時捕獲 → 不遺漏
- AutoDream 確保定期整理 → 不混亂
- 記憶系統始終處於「乾淨、準確、精簡」的狀態

### 6.16.4 帶走的工程洞見

回到本章開頭的命題：**記憶需要定期整理，不能只增不減**。

Claude Code 的記憶系統告訴我們：

1. **儲存不等於記憶**。把資訊存進檔案只是第一步。真正的記憶系統需要抽取、分類、合併、淘汰。

2. **頻率分層是必要的**。不同的操作有不同的最佳頻率——即時抽取要快，定期整理要慢。硬塞到同一頻率會造成浪費或遺漏。

3. **硬限制是記憶品質的保障**。200 行 / 25KB 的限制強迫系統保持精簡。沒有限制，記憶會像垃圾場一樣膨脹。

4. **安全比功能更重要**。記憶 agent 用白名單限制工具，只能讀寫記憶目錄。一個能修改程式碼的「記憶整理 agent」是一個安全災難。

5. **可回滾性是背景任務的基本要求**。AutoDream 的鎖回滾機制確保任何中斷都不會留下髒狀態。

6. **便宜的檢查放前面**。六道 gate 的漸進成本排列，是一個通用的效能模式。

這些原則不只適用於 AI 記憶系統——任何需要「持續收集 + 定期整理」的系統都可以借鑒。日誌聚合、資料倉儲 ETL、甚至個人的知識管理，都是同樣的模式。

---

> **本章源碼索引**
>
> | 檔案 | 角色 |
> |------|------|
> | `src/services/autoDream/autoDream.ts` | AutoDream 主流程與六道 gate |
> | `src/services/autoDream/consolidationPrompt.ts` | 四階段整理 prompt |
> | `src/services/autoDream/consolidationLock.ts` | 鎖機制（取得、驗證、回滾） |
> | `src/services/autoDream/config.ts` | Feature flag 與設定 |
> | `src/services/extractMemories/extractMemories.ts` | 即時記憶抽取 |
> | `src/memdir/memdir.ts` | MEMORY.md 管理與截斷邏輯 |
> | `src/memdir/memoryTypes.ts` | 四種記憶類型定義 |
> | `src/memdir/memoryAge.ts` | 過時偵測與鮮度警告 |
> | `src/memdir/memoryScan.ts` | 記憶檔案掃描 |
> | `src/tasks/DreamTask/DreamTask.ts` | Dream 任務 UI 狀態 |
