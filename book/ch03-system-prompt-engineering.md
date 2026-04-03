# Chapter 3: System Prompt 工程化 — 不是寫 prompt，是寫行為規格書

> 一般人寫 prompt 像寫求職信——希望 AI「盡量表現好」。Anthropic 寫 prompt 像寫 RFC——每條規則對應一個可測試的行為，每個措辭都有工程理由。

---

## 3.1 為什麼傳統 prompt 會失敗

### 「盡量幫助使用者」的問題

打開任何一個 prompt engineering 教學，你大概會看到這種開場：

```
You are a helpful AI assistant. Try your best to help the user
with their questions. Be detailed and thorough in your responses.
```

這句話有什麼問題？**每一個詞都是問題**。

| 詞彙 | 問題 |
|------|------|
| `helpful` | 沒有定義什麼是「有幫助」——多做一點算不算？重構順便的程式碼算不算？ |
| `Try your best` | 沒有邊界——model 會把「盡力」理解為「做越多越好」 |
| `detailed` | 與效率矛盾——使用者可能只要一行答案 |
| `thorough` | 鼓勵 over-engineering——每個邊界案例都要處理 |

結果就是：

```
使用者：「幫我修這個 bug」

傳統 prompt 的 AI：
1. 修好了 bug                     ← 你要求的
2. 順便重構了周圍的程式碼           ← 你沒要求的
3. 加了完整的錯誤處理              ← 你沒要求的
4. 補了 docstring                  ← 你沒要求的
5. 建議了三個「可以進一步改善的地方」  ← 你沒要求的
6. 寫了 2000 字的解釋               ← 你不想看的
```

**LLM 的天性是「多做」**。沒有明確的邊界約束，它會傾向於展示自己的能力，而不是精準完成任務。

### Anthropic 的工程化方法

Claude Code 的 system prompt 位於 `src/constants/prompts.ts`，共 915 行。它不是一段散文，而是一份**行為規格書（behavioral specification）**。每一條規則都遵循同樣的模式：

```
┌──────────────────────────────────────────────────┐
│  1. 明確的行為禁令或要求（Do / Do NOT）            │
│  2. 強調等級標記（CRITICAL / IMPORTANT / Note）    │
│  3. 具體的例子清單（不留模糊空間）                  │
│  4. 替代方案（禁止 A 時，告訴 model 該用 B）        │
│  5. 數值錨定（≤25 words，≤100 words）              │
└──────────────────────────────────────────────────┘
```

這和軟體工程中的 **Design by Contract** 是同一種思維：不是「希望」程式行為正確，而是**用規格定義什麼是正確**。

---

## 3.2 Prompt 組裝管線：動態生成的行為規格書

### 整體架構

System prompt 不是一個靜態字串。它是一個**動態組裝管線**，根據當前環境、用戶設定、可用工具等，在每次 API 呼叫前重新組裝。

```
                    ┌─────────────────────────┐
                    │   getSystemPrompt()     │
                    │   prompts.ts:444-577    │
                    └────────┬────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ 靜態區    │  │ 邊界標記  │  │ 動態區    │
        │ (快取)    │  │          │  │ (每次重組) │
        └──────────┘  └──────────┘  └──────────┘
              │              │              │
              ▼              ▼              ▼
        ┌─────────────────────────────────────┐
        │    完整 system prompt string[]      │
        │    → 傳給 Anthropic Messages API    │
        └─────────────────────────────────────┘
```

### getSystemPrompt() — 主組裝函式

```typescript
// src/constants/prompts.ts:444-577
async function getSystemPrompt(
  tools: Tool[],
  model: string,
  dirs: string[],
  mcpClients: McpClient[],
  options: SystemPromptOptions
): Promise<string[]> {

  // 1. Simple mode 快速路徑
  if (CLAUDE_CODE_SIMPLE) {
    return [`CWD: ${getCwd()}\nDate: ${date}`]
  }

  // 2. 解析各 section（帶 memoization）
  const sections = resolveSystemPromptSections(tools, model, options)

  // 3. 靜態區（跨請求可快取）
  const staticParts = [
    sections.intro,              // 角色定義 + 安全指令
    sections.system,             // 系統行為規則
    sections.doingTasks,         // 任務執行規則（含 over-engineering 防護）
    sections.actions,            // 風險控制框架
    sections.usingTools,         // 工具使用約束
    sections.toneAndStyle,       // 風格規範
    sections.outputEfficiency,   // 輸出效率規則
  ]

  // 4. 動態邊界標記
  staticParts.push(SYSTEM_PROMPT_DYNAMIC_BOUNDARY)

  // 5. 動態區（每次重組）
  const dynamicParts = [
    sections.agentTool,          // Agent 工具指導
    sections.skillTool,          // Skill 工具指導
    await loadMemoryPrompt(),    // MEMORY.md 內容
    computeSimpleEnvInfo(model), // 運行環境資訊
    getMcpInstructions(mcpClients), // MCP 伺服器指令
    sections.outputStyle,        // 輸出樣式
  ]

  return [...staticParts, ...dynamicParts]
}
```

### 靜態 vs 動態邊界

```typescript
// src/constants/prompts.ts:114
const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

這個邊界標記是整個 prompt 成本工程的關鍵。Anthropic Messages API 支援 **prompt caching**——如果 system prompt 的前綴沒有變化，API 不會重新處理那一段，而是從快取中讀取。

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  靜態區（scope: 'global'）                               │
│  ─────────────────────────                               │
│  • 角色定義         不依賴 session/user                    │
│  • 行為規則         每次構建（build）時確定                  │
│  • 工具約束         只依賴工具名稱常量                      │
│  • 風格規範         固定                                   │
│  • 風險控制         固定                                   │
│                                                         │
│  ──── SYSTEM_PROMPT_DYNAMIC_BOUNDARY ────               │
│                                                         │
│  動態區（每次 API call 重組）                              │
│  ─────────────────────────                               │
│  • MEMORY.md        用戶可能隨時修改                       │
│  • MCP 指令         MCP server 可能連線/斷線               │
│  • 環境資訊         CWD、日期、模型可能變化                  │
│  • Agent/Skill      可用的 agent/skill 可能變化            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**成本影響**：靜態區大約佔 system prompt 的 60-70%。在一個典型的多輪對話中（比如 20 輪），這意味著靜態區只被處理一次，節省了約 **60% 的 system prompt token 成本**。

### resolveSystemPromptSections() 與 Memoization

```typescript
// src/constants/prompts.ts
function resolveSystemPromptSections(
  tools: Tool[],
  model: string,
  options: SystemPromptOptions
): ResolvedSections {
  // 每個 section 函式都是 memoized 的
  // 只在參數變化時重新計算
  return {
    intro:            getSimpleIntroSection(),
    system:           getSimpleSystemSection(),
    doingTasks:       getSimpleDoingTasksSection(),
    actions:          getActionsSection(),
    usingTools:       getUsingYourToolsSection(tools),
    toneAndStyle:     getSimpleToneAndStyleSection(),
    outputEfficiency: getOutputEfficiencySection(model),
    // ...
  }
}
```

為什麼要 memoize？因為在同一個 session 中，工具列表和模型通常不會變化。如果 `getUsingYourToolsSection(tools)` 的 `tools` 參數沒變，就直接返回上次計算的結果，避免了字串拼接的重複計算。

這是一個微妙但重要的設計：**system prompt 的組裝本身也需要效能優化**。在一個活躍的對話中，每次 API 呼叫都要重新組裝 system prompt，memoization 確保只有真正變化的部分被重新計算。

---

## 3.3 Section-by-Section 拆解

以下我們逐一拆解 system prompt 的每個 section，從源碼層級理解 Anthropic 如何精確控制 Agent 行為。

### 3.3.1 工具約束：getUsingYourToolsSection()

**源碼位置**：`src/constants/prompts.ts:269-314`

這是整個 system prompt 中最具強制力的一段。

```typescript
// src/constants/prompts.ts:269-314
function getUsingYourToolsSection(tools: Tool[]): string {
  const toolNames = tools.map(t => t.name)

  let section = `# Using your tools

- Do NOT use the ${BASH_TOOL_NAME} to run commands when a relevant dedicated tool is provided.
  This is CRITICAL to assisting the user:
  - To read files use ${FILE_READ_TOOL_NAME} instead of cat, head, tail, or sed
  - To edit files use ${FILE_EDIT_TOOL_NAME} instead of sed or awk
  - To create files use ${FILE_WRITE_TOOL_NAME} instead of cat with heredoc or echo redirection
  - To search for files use ${GLOB_TOOL_NAME} instead of find or ls
  - To search the content of files, use ${GREP_TOOL_NAME} instead of grep or rg
  - Reserve using the ${BASH_TOOL_NAME} exclusively for system commands and terminal operations`

  // REPL mode 變體
  if (isReplMode()) {
    section += `\n  - For REPL operations, use ${REPL_TOOL_NAME} instead of ${BASH_TOOL_NAME}`
  }

  // 額外的工具使用指引
  section += `

- Do NOT use ${FILE_EDIT_TOOL_NAME} for creating new files—use ${FILE_WRITE_TOOL_NAME} instead.
- Do NOT use ${BASH_TOOL_NAME} for reading files unless you need to process them with unix commands.`

  return section
}
```

#### 設計解剖

**1. 絕對禁令，不是建議**

注意措辭是 `Do NOT`，不是 `try to avoid` 或 `prefer`。這裡有精確的語言學工程：

| 措辭 | model 理解的強制力 | 遵守率（估計） |
|------|-------------------|---------------|
| `try to avoid` | 軟建議 | ~60% |
| `prefer X over Y` | 偏好 | ~75% |
| `Do NOT use` | 禁令 | ~90% |
| `Do NOT ... This is CRITICAL` | 強制禁令 | ~95% |

Anthropic 在 `Do NOT` 之後又加了 `This is CRITICAL to assisting the user`——這個追加語不是廢話，而是告訴 model **違反此規則等於無法幫助用戶**，把它和 model 的核心目標掛鉤。

**2. 工具名稱常量替換**

```typescript
const BASH_TOOL_NAME = 'Bash'
const FILE_READ_TOOL_NAME = 'Read'
const FILE_EDIT_TOOL_NAME = 'Edit'
const FILE_WRITE_TOOL_NAME = 'Write'
const GLOB_TOOL_NAME = 'Glob'
const GREP_TOOL_NAME = 'Grep'
const REPL_TOOL_NAME = 'REPL'
```

為什麼不直接寫字串？因為工具可能被重新命名。如果 `BashTool` 改名為 `ShellTool`，只需要改一個常量，所有 prompt 自動更新。這是標準的 **DRY 原則**——即使在 prompt 裡也是如此。

**3. 每條禁令配替代方案**

```
❌ 不要用 Bash 讀檔案
✅ 用 Read 讀檔案

❌ 不要用 sed 編輯
✅ 用 Edit 編輯
```

只禁止不給替代，model 會困惑。每條 `Do NOT` 都跟著 `instead of`，形成了完整的**行為重定向**。

**4. REPL Mode 動態變體**

```typescript
if (isReplMode()) {
  section += `\n  - For REPL operations, use ${REPL_TOOL_NAME} instead of ${BASH_TOOL_NAME}`
}
```

system prompt 不是一成不變的——它根據運行模式動態調整。REPL mode 下新增一條規則，引導 model 使用 REPL 工具而非 Bash。

#### 為什麼這很重要？

你可能會問：model 不是已經知道自己有什麼工具了嗎？為什麼還需要明確告訴它「用 Read 不要用 cat」？

答案在於 **training data 的慣性**。LLM 在訓練時讀過大量的 shell 操作範例——`cat file.txt`、`grep pattern file`、`sed -i 's/old/new/'`。如果不明確禁止，model 會**自然傾向**使用這些熟悉的 shell 命令。

但專用工具有結構化的輸入/輸出、權限控制、進度報告、錯誤處理——這些都是 Bash 命令做不到的。工具約束的本質是：**用紀律覆蓋慣性**。

---

### 3.3.2 風險控制框架：getActionsSection()

**源碼位置**：`src/constants/prompts.ts:255-267`

```typescript
// src/constants/prompts.ts:255-267
function getActionsSection(): string {
  return `# Executing actions with care

Carefully consider the reversibility and blast radius of actions.

Local, reversible actions (freely allowed):
- editing files
- running tests
- creating branches

Hard-to-reverse actions (require confirmation):
- Destructive operations: deleting files/branches, dropping tables, rm -rf
- Force-push, git reset --hard, amending published commits
- Actions visible to others: pushing code, creating/closing PRs, sending messages
- Uploading content to third-party tools

Always prefer reversible actions. Always measure twice, cut once.`
}
```

#### 可逆性 x 影響範圍矩陣

這段 prompt 的核心是一個隱含的 **2x2 決策矩陣**：

```
                    影響範圍
                 本地        共享/外部
              ┌──────────┬──────────────┐
    可逆      │ 自由執行  │ 需要確認      │
              │          │              │
  可逆性      │ 編輯檔案  │ push 程式碼   │
              │ 跑測試    │ 建立 PR      │
              │ 建立分支  │ 發訊息        │
              ├──────────┼──────────────┤
    不可逆    │ 需要確認  │ 強烈警告      │
              │          │              │
              │ 刪除檔案  │ force-push   │
              │ rm -rf   │ drop table   │
              │          │ amend 已推送  │
              └──────────┴──────────────┘
```

**關鍵設計**：

1. **只有左上角是「自由執行」**——可逆 + 本地。其他三個象限全部需要確認或警告。

2. **不是模糊的「小心一點」**——給出了精確的例子清單。model 不需要自己判斷「git reset --hard 算不算危險」，prompt 直接告訴它。

3. **「measure twice, cut once」**——這句話不是格言引用，而是行為指令。它告訴 model：在執行不可逆操作前，先驗證自己的計劃。實際行為表現為 model 會先輸出「我打算做 X，確定嗎？」

#### 例子清單的精確性

注意 prompt 中列出的不可逆操作：

```
- Destructive operations: deleting files/branches, dropping tables, rm -rf
- Force-push, git reset --hard, amending published commits
- Actions visible to others: pushing code, creating/closing PRs, sending messages
- Uploading content to third-party tools
```

這份清單不是隨便寫的——它覆蓋了 Coding Agent 最常遇到的危險場景：

| 場景 | 為什麼危險 | 實際案例 |
|------|-----------|---------|
| `rm -rf` | 刪除不可恢復 | 誤刪整個目錄 |
| `git reset --hard` | 丟棄所有未提交改動 | 用戶花了一小時的改動全沒了 |
| `force-push` | 覆蓋遠端歷史 | 同事的 commit 被覆蓋 |
| `amend published commits` | 改寫已推送的歷史 | 破壞所有人的 rebase |
| `creating/closing PRs` | 對團隊可見 | 未經授權建立 PR |
| `sending messages` | 對外部可見 | 意外發送訊息 |
| `dropping tables` | 資料庫不可恢復 | 生產環境資料遺失 |

每一條都是從**真實的 incident** 中學到的教訓。

---

### 3.3.3 過度工程防護：getSimpleDoingTasksSection()

**源碼位置**：`src/constants/prompts.ts:199-253`

這是我認為整個 system prompt 中**最有洞察力**的一段。它不是在控制 model 做什麼，而是在控制 model **不做什麼**。

```typescript
// src/constants/prompts.ts:199-253
function getSimpleDoingTasksSection(): string {
  return `# Doing tasks

## Avoid over-engineering

- Don't add features, refactor code, or make "improvements" beyond what was asked
- A bug fix doesn't need surrounding code cleaned up
- A simple feature doesn't need extra configurability
- Don't add docstrings, comments, or type annotations to code you didn't change
- Don't add error handling for scenarios that can't happen
- Don't create helpers for one-time operations
- Three similar lines of code is better than a premature abstraction

## Making code changes

- Always read the relevant code before editing
- Use the most appropriate tool for each file operation
- After editing, verify your changes compile/parse correctly
- Don't leave placeholder comments like "// ... rest of code here"
- Always write complete code — never use TODO or placeholder implementations

## Comments

- Do NOT add comments to the code you write, unless:
  - The user explicitly asks for comments
  - The code implements a complex algorithm that benefits from explanation
  - The comment explains WHY, not WHAT (the code should be self-explanatory)
- ${IS_ANT ? 'For ANT code: follow existing comment style' : ''}
`
}
```

#### 七條反模式禁令

讓我們逐條分析為什麼每條都很重要：

**第一條：「Don't add features, refactor code, or make "improvements" beyond what was asked」**

```
使用者：「修這個 null pointer bug」

❌ 過度工程的 AI：
   - 修了 bug
   - 順便把 var 改成 const
   - 重新命名了幾個變數
   - 提取了一個 helper 函式
   - 加了三個新的 error boundary

✅ 受約束的 AI：
   - 修了 bug
   - 結束
```

**第二條：「A bug fix doesn't need surrounding code cleaned up」**

這條更具體——即使你覺得周圍的程式碼很醜，也不要動它。原因：
- 改動越多，review 越難
- 改動越多，引入新 bug 的機率越高
- 改動範圍應該與任務範圍匹配

**第三條：「A simple feature doesn't need extra configurability」**

```typescript
// ❌ 過度工程
function createUser(name: string, options: {
  validateEmail?: boolean
  sendWelcomeEmail?: boolean
  defaultRole?: string
  maxRetries?: number
}) { ... }

// ✅ 剛好夠用
function createUser(name: string) { ... }
```

使用者要求「建立用戶」，不需要預設五個配置項目。

**第四條：「Don't add docstrings, comments, or type annotations to code you didn't change」**

這條非常精確——它不是禁止所有 comment，而是禁止**對你沒改的程式碼**加 comment。如果你改了一個函式，加 docstring 合理；如果你改了隔壁的函式，不要順手幫它加 docstring。

**第五條：「Don't add error handling for scenarios that can't happen」**

```typescript
// ❌ 過度防禦
function divide(a: number, b: number): number {
  if (typeof a !== 'number') throw new TypeError('a must be number')
  if (typeof b !== 'number') throw new TypeError('b must be number')
  if (!Number.isFinite(a)) throw new RangeError('a must be finite')
  if (!Number.isFinite(b)) throw new RangeError('b must be finite')
  if (b === 0) throw new Error('Division by zero')
  return a / b
}

// ✅ TypeScript 已經保證了類型
function divide(a: number, b: number): number {
  if (b === 0) throw new Error('Division by zero')
  return a / b
}
```

TypeScript 的型別系統已經保證 `a` 和 `b` 是 `number`，再加 runtime check 就是過度防禦。

**第六條：「Don't create helpers for one-time operations」**

```typescript
// ❌ 過度抽象
function formatUserName(user: User): string {
  return `${user.firstName} ${user.lastName}`
}
console.log(formatUserName(user))

// ✅ 一行搞定
console.log(`${user.firstName} ${user.lastName}`)
```

只用一次的操作不需要抽象成函式。

**第七條：「Three similar lines of code is better than a premature abstraction」**

```typescript
// ❌ 過早抽象
function processItem(item: Item, type: 'create' | 'update' | 'delete') {
  const endpoint = type === 'create' ? '/api/items'
    : type === 'update' ? `/api/items/${item.id}`
    : `/api/items/${item.id}`
  const method = type === 'create' ? 'POST'
    : type === 'update' ? 'PUT'
    : 'DELETE'
  return fetch(endpoint, { method, body: JSON.stringify(item) })
}

// ✅ 三行重複但清晰
await fetch('/api/items', { method: 'POST', body: JSON.stringify(item) })
await fetch(`/api/items/${item.id}`, { method: 'PUT', body: JSON.stringify(item) })
await fetch(`/api/items/${item.id}`, { method: 'DELETE' })
```

這是對 **DRY 原則的精準反駁**——DRY 說「Don't Repeat Yourself」，但 Claude Code 的 prompt 說：**重複比錯誤的抽象好**。只有當重複達到一定程度（暗示三次以上），才值得抽象。

#### 為什麼 LLM 特別需要這些規則？

這七條規則針對的是 LLM 的一個核心特性：**model 在 training 時讀過大量的「好的工程實踐」文章**——寫 clean code、加 docstring、DRY、defensive programming。這些建議在人類工程師的日常工作中通常是對的，但在 Agent 的上下文中，它們的副作用被放大了：

```
人類工程師                          AI Agent
────────────                      ────────
改動後自己 review                   改動後直接提交
知道專案上下文                      只看到眼前的檔案
知道什麼時候「夠了」                 不知道，所以傾向多做
改動成本低（自己的時間）              改動成本高（影響整個 PR）
```

這些規則本質上是在說：**你不是在寫部落格教學，你是在執行精確的工程任務**。

#### ANT-only 註釋規則

```typescript
// prompts.ts
- ${IS_ANT ? 'For ANT code: follow existing comment style' : ''}
```

`IS_ANT` 是一個 build-time 宏，標記是否為 Anthropic 內部構建。內部版本有額外的註釋規則——跟隨現有程式碼的註釋風格，而不是應用通用規則。這是因為 Anthropic 內部的程式碼庫有自己的慣例。

---

### 3.3.4 輸出規範：getOutputEfficiencySection()

**源碼位置**：`src/constants/prompts.ts:403-428`

```typescript
// src/constants/prompts.ts:403-428
function getOutputEfficiencySection(model: string): string {
  const isAnt = IS_ANT

  if (isAnt) {
    return `# Output efficiency

Go straight to the point. Try the simplest approach first without going in circles.

Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan

If you can say it in one sentence, don't use three.

Writing style:
- Brief statement before your first tool call
- Short update only at key moments (load-bearing finds, direction changes, progress)
- Flowing prose, avoid fragments/em dashes/notation
- Avoid semantic backtracking — linear meaning building
- Only tables for short enumerable facts or quantitative data
- Inverted pyramid: most important info first`
  }

  // 外部版
  return `# Output efficiency

Go straight to the point. Try the simplest approach first without going in circles.

Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan

If you can say it in one sentence, don't use three.`
}
```

#### ANT 內部版 vs 外部版的差異

| 面向 | 外部版 | ANT 內部版 |
|------|--------|-----------|
| 基本原則 | 直奔重點，簡單方法優先 | 同左 |
| 內容聚焦 | 決策/進度/錯誤 | 同左 |
| 寫作風格 | 「一句能說完不用三句」 | 更精確的散文規範 |
| 結構規則 | 無 | flowing prose, 倒金字塔 |
| 反模式 | 無 | 禁止 fragments/em dashes |
| 語意規則 | 無 | 禁止 semantic backtracking |

ANT 內部版多了幾條非常精確的寫作指令：

**「Flowing prose, avoid fragments/em dashes/notation」**

```
❌ Fragments:
   "Found the bug. In utils.ts. Line 42. Null reference."

❌ Em dashes:
   "The issue — which I found in utils.ts — relates to null handling"

✅ Flowing prose:
   "I found a null reference bug on line 42 of utils.ts."
```

**「Avoid semantic backtracking — linear meaning building」**

```
❌ 語意回溯:
   "Let me check the file... Actually, wait, first let me look at the imports.
    No, actually the issue is in the exports. Let me go back to..."

✅ 線性語意建構:
   "The exports in utils.ts have a type mismatch on line 42."
```

這條規則禁止 model「自言自語」式的探索過程出現在輸出中。用戶不需要看到 model 的思考過程，只需要看到結論。

**「Inverted pyramid: most important info first」**

這是新聞寫作的經典原則——最重要的資訊放最前面。在 Agent 的上下文中：

```
❌ 正金字塔（先背景後結論）:
   "I examined the test files, looked at the configuration,
    checked the CI pipeline, and reviewed the dependencies.
    After thorough analysis, the issue is a missing import."

✅ 倒金字塔（先結論後解釋）:
   "The issue is a missing import in app.ts line 12.
    I verified this by checking the test output and CI logs."
```

#### 數值長度錨定

```typescript
// src/constants/prompts.ts:528-537
const outputLengthConstraints = `
- Keep text between tool calls to ≤25 words
- Keep final responses to ≤100 words unless task requires more detail
`
```

這是整個 system prompt 中**最有效的約束之一**。

為什麼用數字？因為 model 對「簡短」的理解因上下文而異——有時 200 字算「簡短」，有時 50 字算「太長」。給出精確的數字就消除了模糊空間：

| 約束 | 數值 | 效果 |
|------|------|------|
| 工具呼叫間的文字 | ≤25 words | 阻止 model 在工具之間輸出冗長的自言自語 |
| 最終回應 | ≤100 words | 確保回答精簡，不會寫出千字長文 |

**真實對比**：

```
沒有數值約束的 model：
"I'll start by reading the file to understand its structure.
 This will help me identify where the issue might be located.
 Let me use the Read tool to examine the contents."
[Read tool call]
"Now I can see the file contents. Let me analyze what's happening
 here. It looks like there might be an issue with the import
 statement on line 12. Let me search for related files to
 understand the full picture."
[Grep tool call]

有數值約束的 model：
"Reading the file."
[Read tool call]
"Found the issue on line 12."
[Grep tool call]
```

前者每次工具呼叫前說了 30-40 字的廢話；後者控制在 25 字以內，直奔主題。

---

### 3.3.5 網路安全風險指令

**源碼位置**：`src/constants/cyberRiskInstruction.ts`

```typescript
// src/constants/cyberRiskInstruction.ts
const CYBER_RISK_INSTRUCTION = `
IMPORTANT: Assist with authorized security testing, defensive security,
CTF challenges, and educational contexts.

Refuse requests for:
- Destructive techniques
- DoS attacks
- Mass targeting
- Supply chain compromise
- Detection evasion for malicious purposes

Dual-use security tools require clear authorization context:
pentesting engagements, CTF competitions, security research,
or defensive use cases.
`
```

#### 不是一刀切禁止，而是要求上下文

這段指令的設計哲學和大多數 AI 安全方法不同：

```
傳統方法:
「不要討論任何跟 hacking 有關的東西」
→ 結果：連合法的安全研究都做不了

Anthropic 方法:
「可以協助安全相關操作，但需要合法的上下文」
→ 結果：合法使用不受影響，惡意使用被攔截
```

具體來說，prompt 列出了**四種合法上下文**：

| 上下文 | 例子 |
|--------|------|
| 滲透測試合約 | 「我們被 X 公司授權測試他們的 API」 |
| CTF 競賽 | 「這是 DEF CON CTF 的題目」 |
| 安全研究 | 「我在研究這類漏洞的防禦方法」 |
| 防禦用途 | 「幫我寫一個 WAF 規則來擋這種攻擊」 |

同時明確禁止了**五種惡意場景**：

| 禁止場景 | 為什麼 |
|----------|--------|
| 破壞性技術 | 目的是造成損害 |
| DoS 攻擊 | 影響服務可用性 |
| 大規模目標攻擊 | 不是授權的單一目標 |
| 供應鏈攻擊 | 影響面太廣 |
| 惡意規避偵測 | 明確的惡意意圖 |

這種 **dual-use authorization context** 的設計，讓 Agent 能夠在安全領域正常工作（許多開發者確實需要寫安全工具、做滲透測試），同時設下明確的紅線。

---

### 3.3.6 風格規範：getSimpleToneAndStyleSection()

**源碼位置**：`src/constants/prompts.ts:430-442`

```typescript
// src/constants/prompts.ts:430-442
function getSimpleToneAndStyleSection(): string {
  return `# Tone and style

- Do not use emoji unless the user explicitly requests it
- Do not use a colon before tool calls
  ❌ "Let me read the file:"
  ✅ "Let me read the file."
- When referring to file locations, use the format file_path:line_number
  ✅ src/utils/helper.ts:42
  ❌ line 42 of src/utils/helper.ts
  ❌ src/utils/helper.ts, line 42
- Be direct and technical — match the user's communication style`
}
```

#### 四條看似瑣碎但影響巨大的規則

**第一條：不要用 emoji**

```
❌ "I found the bug! 🎉 Let me fix it ✨"
✅ "I found the bug. Let me fix it."
```

為什麼？因為 Claude Code 的目標用戶是**工程師**。在終端環境中，emoji 看起來不專業，而且會干擾程式碼格式。

**第二條：工具呼叫前不要用冒號**

這一條極度細節導向，但反映了 Anthropic 對輸出品質的要求：

```
❌ "Let me read the file:"
   [Read tool call]

✅ "Let me read the file."
   [Read tool call]
```

冒號暗示「後面的內容是前一句的補充說明」，但工具呼叫不是文字內容——它是一個結構化的 API 呼叫。句號則表示「這句話說完了」，和後面的工具呼叫是兩個獨立的動作。

**第三條：檔案位置格式**

```
✅ src/utils/helper.ts:42
❌ line 42 of src/utils/helper.ts
❌ src/utils/helper.ts, line 42
```

`file:line` 格式的好處：
- 終端可以直接點擊跳轉（多數終端模擬器支援）
- 和 compiler error 輸出格式一致
- 更短，符合輸出效率原則

**第四條：匹配用戶的溝通風格**

```
用戶：「這個函式怎麼用？」
✅ 「傳入一個 string，回傳 boolean。例如 isValid('test') → true」

用戶：「Please provide a comprehensive explanation of this function's behavior」
✅ 「This function validates input strings against the configured pattern...」
```

model 應該**鏡像**用戶的語言風格——簡潔對簡潔，詳細對詳細。

---

## 3.4 環境資訊組裝：computeSimpleEnvInfo()

**源碼位置**：`src/constants/prompts.ts:651-710`

```typescript
// src/constants/prompts.ts:651-710
function computeSimpleEnvInfo(model: string): string {
  const parts: string[] = []

  // 1. 基礎環境
  parts.push(`Working directory: ${getCwd()}`)
  parts.push(`Is directory a git repo: ${isGitRepo() ? 'Yes' : 'No'}`)
  parts.push(`Platform: ${process.platform}`)
  parts.push(`Shell: ${getShell()}`)
  parts.push(`OS Version: ${getOSVersion()}`)

  // 2. 模型特定資訊
  parts.push(`You are powered by the model named ${getModelDisplayName(model)}.`)
  parts.push(`The exact model ID is ${model}.`)

  // 3. 知識截止日期（依模型不同）
  const cutoff = getKnowledgeCutoff(model)
  if (cutoff) {
    parts.push(`Assistant knowledge cutoff is ${cutoff}.`)
  }

  // 4. Worktree 偵測警告
  if (isInWorktree()) {
    parts.push(
      `WARNING: You are currently in a git worktree. ` +
      `Be careful with git operations as they may affect the main repository.`
    )
  }

  // 5. Fast mode 說明
  if (isFastMode()) {
    parts.push(
      `You are in fast mode. Prefer quick, focused responses. ` +
      `Skip detailed explanations unless asked.`
    )
  }

  return parts.join('\n')
}
```

### 知識截止日期：為什麼每個模型不同？

```typescript
function getKnowledgeCutoff(model: string): string | null {
  if (model.includes('opus'))   return 'May 2025'
  if (model.includes('sonnet')) return 'April 2025'
  if (model.includes('haiku'))  return 'March 2025'
  return null
}
```

不同模型的 training data 截止日期不同。把這個資訊注入 system prompt 的目的是：**讓 model 知道自己的知識邊界**。

```
用戶：「React 19.1 有什麼新功能？」

知道自己截止到 2025 年 5 月的 model：
✅ 「根據我的知識（截止到 2025 年 5 月），React 19.1 的新功能包括...」

不知道截止日期的 model：
❌ 可能捏造不存在的功能
```

### Worktree 偵測警告

```
WARNING: You are currently in a git worktree.
Be careful with git operations as they may affect the main repository.
```

Git worktree 是一個容易踩雷的場景——它讓你在同一個 repo 的不同分支上同時工作，但某些 git 操作（如 `git clean`、`git checkout`）的影響範圍可能超出當前 worktree。

這個警告讓 model 在 worktree 環境中額外謹慎，尤其是在執行 git 操作時。

### Fast Mode 說明

```
You are in fast mode. Prefer quick, focused responses.
Skip detailed explanations unless asked.
```

Fast mode 改變了 model 的行為偏向——從「完整詳細」切換到「快速精簡」。這是一個**模式切換**，而不是一個永久設定，所以放在動態區。

---

## 3.5 記憶系統注入：loadMemoryPrompt()

```typescript
// src/utils/memory/loadMemoryPrompt.ts
async function loadMemoryPrompt(): Promise<string> {
  // 1. 載入 MEMORY.md 索引
  const memoryIndex = await loadMemoryIndex()
  if (!memoryIndex) return ''

  // 2. 載入所有引用的記憶檔案
  const memories = await loadReferencedMemories(memoryIndex)

  // 3. 組裝 prompt
  return `# Memory

The following memory files contain information from previous sessions:

${memoryIndex}

${memories.map(m => `Contents of ${m.path}:\n${m.content}`).join('\n\n')}
`
}
```

記憶系統的注入是動態區的一部分——因為用戶或 AutoDream 可能隨時更新 MEMORY.md。

注入的內容包含：
1. **索引**：一個 markdown 列表，記錄每個記憶檔案的路徑和摘要
2. **記憶檔案內容**：每個被引用的 `.md` 檔案的完整內容

```
# Memory

The following memory files contain information from previous sessions:

- [project_goals.md](project_goals.md) — 專案目標和架構決策
- [feedback_code_style.md](feedback_code_style.md) — 用戶偏好的程式碼風格

Contents of project_goals.md:
# 專案目標
- 使用 TypeScript strict mode
- 所有 API 回應都用 Zod 驗證
...

Contents of feedback_code_style.md:
# 程式碼風格偏好
- 用 const 不用 let
- 函式長度不超過 50 行
...
```

這讓 model 在每次對話開始時就知道用戶的偏好和專案上下文，不需要用戶每次重新說明。

---

## 3.6 MCP 指令注入：getMcpInstructions()

**源碼位置**：`src/constants/prompts.ts:579-604`

```typescript
// src/constants/prompts.ts:579-604
function getMcpInstructions(mcpClients: McpClient[]): string {
  if (mcpClients.length === 0) return ''

  const instructions: string[] = []

  for (const client of mcpClients) {
    // 1. 取得 MCP server 自定義的指令
    const serverInstructions = client.getInstructions()
    if (!serverInstructions) continue

    // 2. 格式化
    instructions.push(`## ${client.getName()}\n${serverInstructions}`)
  }

  if (instructions.length === 0) return ''

  return `# MCP Server Instructions

The following MCP servers have provided instructions for how to use their tools and resources:

${instructions.join('\n\n')}`
}
```

### MCP 指令的注入機制

每個 MCP server 可以提供自己的**使用說明**。這些說明被注入到 system prompt 的動態區，讓 model 知道如何正確使用這些外部工具。

```
# MCP Server Instructions

## plugin:discord:discord
The sender reads Discord, not this session. Anything you want them
to see must go through the reply tool — your transcript output
never reaches their chat.

## notion
Use notion-search to find pages by title. Use notion-fetch to
read page contents. Always check if a page exists before creating.
```

這種設計讓**每個 MCP server 自己定義自己的使用規則**。model 不需要預先知道所有可能的 MCP 工具——連線後，工具的使用說明會自動注入。

### 安全考量

注意，MCP 指令是來自外部的 prompt 內容。Claude Code 的 prompt 組裝系統會：
1. 將 MCP 指令放在明確的 section 標題下（`# MCP Server Instructions`）
2. 每個 server 的指令放在子標題下（`## server_name`）
3. model 可以辨別這些指令來自外部，而非 Anthropic 的核心規則

這種結構性隔離降低了 **prompt injection** 的風險——即使 MCP server 的指令中包含惡意內容，model 也知道它的層級低於核心行為規則。

---

## 3.7 完整的 Prompt 組裝流程

讓我們把所有片段組合起來，看看完整的 system prompt 長什麼樣：

```
┌─────────────────────────────────────────────────────┐
│                  靜態區（可快取）                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  # Role                                             │
│  You are Claude Code, Anthropic's official CLI...   │
│                                                     │
│  # System behavior                                  │
│  [核心行為規則]                                      │
│                                                     │
│  # Doing tasks                                      │
│  ## Avoid over-engineering                          │
│  - Don't add features beyond what was asked         │
│  - Three similar lines > premature abstraction      │
│  [7 條反模式禁令]                                    │
│                                                     │
│  ## Making code changes                             │
│  [程式碼修改規則]                                     │
│                                                     │
│  # Executing actions with care                      │
│  [可逆性 × 影響範圍矩陣]                              │
│  [具體例子清單]                                      │
│                                                     │
│  # Using your tools                                 │
│  - Do NOT use Bash when dedicated tool provided     │
│  [工具偏好階層]                                      │
│                                                     │
│  # Tone and style                                   │
│  [風格規範]                                          │
│                                                     │
│  # Output efficiency                                │
│  [輸出效率規則]                                      │
│  [數值長度錨定]                                      │
│                                                     │
│  # Cyber risk                                       │
│  [安全邊界指令]                                      │
│                                                     │
├────── __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ───────────┤
│                                                     │
│                  動態區（每次重組）                    │
├─────────────────────────────────────────────────────┤
│                                                     │
│  # Agent tool guidance                              │
│  [如何使用 AgentTool]                                │
│                                                     │
│  # Skill tool guidance                              │
│  [可用的 skills 列表]                                │
│                                                     │
│  # Memory                                           │
│  [MEMORY.md 內容]                                   │
│  [引用的記憶檔案]                                    │
│                                                     │
│  # Environment                                      │
│  Working directory: /Users/leo/project               │
│  Is directory a git repo: Yes                       │
│  Platform: darwin                                   │
│  Shell: zsh                                         │
│  Model: claude-opus-4-6[1m]                         │
│  Knowledge cutoff: May 2025                         │
│                                                     │
│  # MCP Server Instructions                          │
│  ## discord                                         │
│  [Discord MCP 使用說明]                              │
│  ## notion                                          │
│  [Notion MCP 使用說明]                               │
│                                                     │
│  # Output style                                     │
│  [≤25 words between tool calls]                     │
│  [≤100 words final response]                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 組裝順序的工程意義

prompt 的 section 順序不是隨意的：

| 順序 | Section | 為什麼放這裡 |
|------|---------|-------------|
| 1 | Role 定義 | 先建立身份 |
| 2 | 系統行為 | 定義基本行為框架 |
| 3 | 任務執行 | 定義「怎麼做事」 |
| 4 | 風險控制 | 定義「什麼不能做」 |
| 5 | 工具約束 | 定義「用什麼做事」 |
| 6 | 風格規範 | 定義「怎麼說話」 |
| 7 | 輸出效率 | 定義「說多少」 |
| 8 | 安全邊界 | 定義紅線 |
| --- | 動態邊界 | 快取切割點 |
| 9 | Agent/Skill | 運行時可用能力 |
| 10 | 記憶 | 用戶偏好 |
| 11 | 環境 | 當前上下文 |
| 12 | MCP | 外部工具 |
| 13 | 輸出樣式 | 最後的約束（recency bias） |

注意最後一項「輸出樣式」放在最後面——這利用了 LLM 的 **recency bias**（越靠近結尾的指令，遵守率越高）。數值長度約束放在最後，確保 model 在生成每一段輸出時都能「記住」這些限制。

---

## 3.8 五個可直接套用的設計模式

以下五個模式可以直接應用到任何 Agent 的 system prompt 設計中。

### 模式一：工具偏好階層

**原則**：當有專用工具時，禁止使用通用工具。每條禁令配一個替代方案。

```
# 模板

## Using your tools

- Do NOT use [通用工具] when [專用工具] is available.
  This is CRITICAL:
  - To [操作A], use [專用工具A] instead of [通用替代]
  - To [操作B], use [專用工具B] instead of [通用替代]
  - Reserve [通用工具] exclusively for [無專用工具的場景]
```

**應用範例**——假設你在做一個資料分析 Agent：

```
# Using your tools

- Do NOT use Python code to query databases when the SQLTool is available.
  This is CRITICAL:
  - To query data, use SQLTool instead of pandas.read_sql()
  - To aggregate data, use SQLTool with GROUP BY instead of df.groupby()
  - To join tables, use SQLTool with JOIN instead of pd.merge()
  - Reserve Python exclusively for data visualization and complex transformations
```

**為什麼有效**：
- `Do NOT` + `CRITICAL` 建立最高強制力
- 具體的 `instead of` 消除模糊空間
- `Reserve...exclusively for` 定義了通用工具的合法使用範圍

### 模式二：可逆性決策矩陣

**原則**：把所有操作按「可逆性 x 影響範圍」分類，只有「可逆 + 本地」可自動執行。

```
# 模板

## Executing actions with care

Carefully consider the reversibility and blast radius of actions.

Local, reversible actions (freely allowed):
- [列出具體操作]

Hard-to-reverse actions (require confirmation):
- [列出具體操作，包含原因]

Always prefer reversible actions. Always measure twice, cut once.
```

**應用範例**——假設你在做一個 DevOps Agent：

```
## Executing actions with care

Carefully consider the reversibility and blast radius of actions.

Local, reversible actions (freely allowed):
- Reading configuration files
- Viewing logs
- Checking service status
- Creating backups

Hard-to-reverse actions (require confirmation):
- Restarting services (affects availability)
- Modifying configuration files (may break services)
- Scaling instances up/down (cost implications)
- Modifying firewall rules (security implications)

Never-allowed without explicit user command:
- Deleting infrastructure resources
- Modifying production databases
- Changing DNS records
- Rotating secrets/certificates

Always prefer reversible actions. Always measure twice, cut once.
```

**為什麼有效**：
- 三層分類（自由/確認/禁止）比二分法更精確
- 括號內的原因幫助 model 理解「為什麼」，處理清單沒覆蓋到的邊界案例
- `Always prefer reversible actions` 作為兜底規則

### 模式三：過度行為抑制

**原則**：列出 LLM 最常犯的「多做」行為，逐一禁止。

```
# 模板

## Avoid over-engineering

- Don't [常見過度行為 1]
- Don't [常見過度行為 2]
- [正面規則：用具體的量化標準]
```

**應用範例**——假設你在做一個寫作 Agent：

```
## Avoid over-elaboration

- Don't add sections the user didn't ask for
- A request for "a paragraph" doesn't need an introduction and conclusion
- Don't add caveats and disclaimers unless the topic requires them
- Don't provide "alternative perspectives" unless asked
- Don't suggest "further reading" or "next steps" unless asked
- A 200-word request doesn't need 500 words of context
- Three clear sentences are better than a verbose paragraph
```

**為什麼有效**：
- 每條規則對應一種 LLM 的具體過度行為
- 用對比句式（「A request for X doesn't need Y」）讓規則更直觀
- 最後一條用量化比較（「三句 vs 一段」）做為錨定

### 模式四：輸出結構約束

**原則**：用數值錨定和倒金字塔結構控制輸出。

```
# 模板

## Output efficiency

[核心原則：一句話]

Focus output on:
- [高優先內容類型 1]
- [高優先內容類型 2]
- [高優先內容類型 3]

Constraints:
- Keep [輸出場景A] to ≤[N] words
- Keep [輸出場景B] to ≤[M] words
- [格式規則]
```

**應用範例**——假設你在做一個客服 Agent：

```
## Output efficiency

Answer the customer's question, then stop.

Focus output on:
- Direct answer to the customer's question
- Required next steps (if any)
- Relevant policy information (only if directly applicable)

Constraints:
- Keep initial responses to ≤50 words
- Keep follow-up responses to ≤30 words
- Never repeat information the customer already knows
- If you can link to a help article instead of explaining, link it
```

**為什麼有效**：
- 「Answer then stop」是一個明確的行為終止條件
- 數值約束消除了「多長算適當」的模糊性
- 「Never repeat」阻止了 LLM 常見的回聲行為

### 模式五：安全的上下文判斷

**原則**：不是一刀切禁止，而是要求合法上下文。

```
# 模板

## [敏感領域] policy

Assist with [合法用途列表].

Refuse requests for:
- [具體的禁止場景 1]
- [具體的禁止場景 2]

[敏感操作] require clear authorization context:
[列出合法上下文]
```

**應用範例**——假設你在做一個醫療資訊 Agent：

```
## Medical information policy

Assist with general health education, medication information lookups,
and helping users prepare questions for their doctors.

Refuse requests for:
- Specific diagnoses based on symptoms
- Medication dosage recommendations
- Advice to stop or change prescribed medications
- Emergency medical guidance (direct to 911/emergency services)

Treatment-related questions require clear context:
the user is a healthcare professional, the question is academic,
or the user has explicitly stated they will consult their doctor.
```

**為什麼有效**：
- 先列合法用途，建立正面框架
- 禁止清單具體到操作層級
- 「require clear context」給了 model 判斷的依據，而不是盲目禁止

---

## 3.9 進階主題：Prompt 工程的隱含原則

### 原則一：語言強度階層

Claude Code 的 prompt 使用了精確的語言強度階層：

```
強制力：低 ────────────────────────────────────→ 高

consider  →  prefer  →  avoid  →  Do NOT  →  CRITICAL/IMPORTANT
   ↓           ↓         ↓         ↓              ↓
  建議       偏好      不鼓勵     禁令         最高禁令
```

每個等級的使用場景：

| 等級 | 用於 | 例子 |
|------|------|------|
| `consider` | 提醒注意 | 「consider the reversibility of actions」 |
| `prefer` | 有更好的選擇 | 「prefer reversible actions」 |
| `avoid` | 不鼓勵但不禁止 | 「avoid semantic backtracking」 |
| `Do NOT` | 明確禁止 | 「Do NOT use Bash when...」 |
| `CRITICAL` | 最高優先禁令 | 「This is CRITICAL to assisting the user」 |

### 原則二：正面指令 > 負面禁令

注意 Claude Code 的每條禁令幾乎都配有正面替代：

```
❌ 只有禁令：
   "Don't use Bash for file operations"
   → model 不確定該用什麼

✅ 禁令 + 替代：
   "Don't use Bash for file operations. Use Read instead of cat."
   → model 明確知道該怎麼做
```

**心理模型**：禁令告訴 model「不要走這條路」，替代方案告訴它「走這條路」。只有禁令而沒有替代，model 會在禁令的邊界來回試探。

### 原則三：例子比規則重要

在 prompt 工程中，一個具體的例子比十句抽象的規則更有效：

```
❌ 抽象規則：
   "Format file references consistently"

✅ 具體例子：
   "Use the format file_path:line_number
    ✅ src/utils/helper.ts:42
    ❌ line 42 of src/utils/helper.ts
    ❌ src/utils/helper.ts, line 42"
```

Claude Code 的 prompt 在幾乎每條規則後面都附了例子——不是因為 model 看不懂規則，而是因為例子消除了歧義。

### 原則四：利用位置效應

LLM 對 prompt 不同位置的注意力不同：

```
注意力分布:

  ████████████  開頭（角色定義、核心規則）
  ██████        中間（具體規則、例子）
  ████████████  結尾（最後的約束、數值錨定）
         ↑
      注意力低谷
```

Claude Code 利用這個特性：
- **開頭**放角色定義和最重要的行為規則
- **中間**放具體的 section（過度工程、風險控制等）
- **結尾**放數值約束（≤25 words、≤100 words）

這就是為什麼輸出長度約束放在 prompt 最後——利用 recency bias 確保 model 時刻記住長度限制。

### 原則五：Build-Time 條件編譯

```typescript
const IS_ANT = MACRO.IS_ANT  // Build-time 宏

// 在 prompt 中
if (IS_ANT) {
  // Anthropic 內部版：更嚴格的輸出規範
  section += 'Flowing prose, avoid fragments/em dashes/notation'
  section += 'Inverted pyramid: most important info first'
}
```

不同的構建目標生成不同的 prompt。這允許：
- 內部版有更嚴格的品質要求
- 外部版更寬鬆（避免過度約束影響用戶體驗）
- A/B 測試不同的 prompt 變體

這和軟體工程中的 **feature flag** 是完全一樣的概念——只不過應用在 prompt 上。

---

## 3.10 常見錯誤與反模式

在設計 Agent system prompt 時，以下是常見的反模式：

### 反模式一：用散文代替規格

```
❌ "You should try to be helpful and provide detailed responses when
    the user asks questions. Make sure your answers are accurate and
    relevant. If you're not sure about something, let the user know."

✅ "# Output rules
    - Answer the question directly in the first sentence
    - Keep responses to ≤100 words unless asked for detail
    - If uncertain, state confidence level (high/medium/low)"
```

散文讓 model 有太多解釋空間。規格消除歧義。

### 反模式二：只有正面激勵，沒有負面約束

```
❌ "Be as helpful as possible!"
   → model 理解為「做越多越好」

✅ "Complete the requested task. Do NOT add features beyond what was asked."
   → model 理解為「做完就停」
```

LLM 天性是「多做」。只有正面激勵會放大這個傾向。

### 反模式三：規則互相矛盾

```
❌ "Be thorough and detailed" + "Keep responses short"
   → model 不知道該聽哪個

✅ "Keep responses to ≤100 words. If the task requires more detail,
    ask the user before providing a longer response."
   → 明確的優先順序
```

Claude Code 的 prompt 避免矛盾的方法是：每條規則都有**具體的適用場景**，而不是泛泛的「原則」。

### 反模式四：忽略 training data 慣性

```
❌ 假設 model 會自然傾向使用你的專用工具
   → 它會傾向使用 training data 中的 shell 命令

✅ 明確禁止 shell 命令，強制使用專用工具
   → 用紀律覆蓋慣性
```

### 反模式五：一次性 prompt，沒有動態組裝

```
❌ 一個巨大的靜態 prompt 字串
   → 包含所有可能場景的規則，大部分時候用不到

✅ 動態組裝，根據當前上下文選擇 section
   → 每次 API 呼叫只包含相關的規則
```

Claude Code 的動態組裝確保了：
- 不同模式（REPL / Standard / Fast）有不同的規則
- 不同環境（Git repo / non-repo / worktree）有不同的警告
- 不同連線的 MCP server 有各自的指令

---

## 3.11 章節總結

### 核心概念回顧

| 概念 | 要點 |
|------|------|
| **行為規格書 vs 散文 prompt** | Claude Code 的 prompt 是 Design by Contract，不是「盡量幫忙」 |
| **動態組裝管線** | `getSystemPrompt()` 根據模式/環境/工具動態組裝，不是靜態字串 |
| **快取邊界** | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 把 prompt 分為靜態區（可快取）和動態區 |
| **工具約束** | `Do NOT` + `CRITICAL` + `instead of` 三件套，用紀律覆蓋 training data 慣性 |
| **風險矩陣** | 可逆性 x 影響範圍，只有「可逆 + 本地」可自動執行 |
| **過度工程防護** | 7 條精確禁令，對治 LLM「多做」的天性 |
| **數值錨定** | ≤25 words（工具間）、≤100 words（最終回應），消除「簡短」的模糊性 |
| **安全上下文** | Dual-use authorization——不是禁止，而是要求合法上下文 |
| **Memoization** | section 函式帶 memoization，避免重複計算 |

### 帶走的五個模式

```
┌──────────────────────────────────────────────────┐
│  1. 工具偏好階層                                   │
│     → 禁止通用工具，強制專用工具，每條禁令配替代     │
│                                                  │
│  2. 可逆性決策矩陣                                 │
│     → 2x2 分類，只有「可逆+本地」可自動執行         │
│                                                  │
│  3. 過度行為抑制                                   │
│     → 列出 LLM 的具體過度行為，逐一禁止             │
│                                                  │
│  4. 輸出結構約束                                   │
│     → 數值錨定 + 倒金字塔 + 場景化約束              │
│                                                  │
│  5. 安全上下文判斷                                  │
│     → 不一刀切禁止，要求合法上下文                   │
└──────────────────────────────────────────────────┘
```

### 最後的反思

傳統的 prompt engineering 是一門「藝術」——靠直覺、靠試錯、靠「這個措辭好像更有效」。

Claude Code 的 system prompt 展示了一種不同的方法：**把 prompt 當作軟體規格書來寫**。每條規則都是可測試的（你可以驗證 model 是否遵守了「≤25 words between tool calls」），每個 section 都有明確的職責，整個結構都是模組化和可維護的。

這不是關於「寫得好」的 prompt——而是關於**工程化的行為控制**。

當你的 Agent 在生產環境中服務數十萬用戶時，「盡量幫助使用者」是不夠的。你需要的是：

- **精確的行為邊界**——model 知道什麼能做、什麼不能做
- **具體的例子清單**——不留模糊空間
- **數值化的約束**——消除主觀判斷
- **動態組裝能力**——根據上下文調整規則
- **成本工程意識**——快取邊界設計

這就是 system prompt 工程化的全部含義：**不是寫 prompt，是寫行為規格書**。

---

## 源碼參考

| 檔案 | 行號 | 角色 |
|------|------|------|
| `src/constants/prompts.ts` | 全檔 (915 行) | System prompt 主檔 |
| `src/constants/prompts.ts` | 114 | `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 常量 |
| `src/constants/prompts.ts` | 199-253 | `getSimpleDoingTasksSection()` |
| `src/constants/prompts.ts` | 255-267 | `getActionsSection()` |
| `src/constants/prompts.ts` | 269-314 | `getUsingYourToolsSection()` |
| `src/constants/prompts.ts` | 403-428 | `getOutputEfficiencySection()` |
| `src/constants/prompts.ts` | 430-442 | `getSimpleToneAndStyleSection()` |
| `src/constants/prompts.ts` | 444-577 | `getSystemPrompt()` |
| `src/constants/prompts.ts` | 528-537 | 數值長度錨定 |
| `src/constants/prompts.ts` | 579-604 | `getMcpInstructions()` |
| `src/constants/prompts.ts` | 651-710 | `computeSimpleEnvInfo()` |
| `src/constants/cyberRiskInstruction.ts` | 全檔 | 網路安全風險指令 |
| `src/stubs/macros.ts` | — | Build-time 宏定義 (`IS_ANT`) |
| `src/utils/memory/loadMemoryPrompt.ts` | — | 記憶系統 prompt 注入 |
