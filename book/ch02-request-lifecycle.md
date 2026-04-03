# Chapter 2: 一個問題的完整旅程

> 從用戶輸入到回答的請求生命週期——追蹤一個問題如何穿越 512K 行程式碼，經歷 System Prompt 組裝、API 通訊、Streaming 解析、工具執行、錯誤處理，最終變成一個回答。

---

## 2.1 全局視角：請求生命週期總覽

在深入每個環節之前，先看完整的流程圖。這張圖是本章的地圖，後續每一節都會展開其中一個方框。

```
                         一個問題的完整旅程
                         ==================

[用戶在終端輸入問題]
        |
        v
+------------------+
| cli.tsx 入口分派  |  <-- Fast-path: --version 零 import 直接返回
+------------------+       Full-path: dynamic import init.ts
        |
        v
+------------------+
| init.ts 完整初始化 |  <-- 設定、TLS、Git 偵測、OAuth、API 預連線
+------------------+
        |
        v
+------------------+
| UserPromptSubmit |  <-- Hook 可修改/阻止輸入
| Hook             |
+------------------+
        |
        v
+----------------------------+
| System Prompt 組裝          |
| constants/prompts.ts        |
|                            |
| [靜態區] ← 可跨請求快取     |
|   intro + rules + tools    |
|   + tone + output          |
| ---- 快取邊界 ----          |
| [動態區] ← 每次重組         |
|   memory + env + MCP       |
|   + skills                 |
+----------------------------+
        |
        v
+----------------------------+
| Messages 組裝               |
|   歷史訊息（可能已壓縮）      |
|   + 新用戶訊息              |
|   + System reminders       |
+----------------------------+
        |
        v
+----------------------------+
| API 請求建構                 |
|   Provider 路由:             |
|   1P / Bedrock / Vertex    |
|   / Azure                  |
+----------------------------+
        |
        v
+----------------------------+       +------------------+
| Streaming 回應解析          | ----> | Text Block       |
|                            |       | -> 直接渲染到終端  |
| content_block_start        |       +------------------+
| content_block_delta        |
| content_block_stop         |       +------------------+
| message_stop               | ----> | Thinking Block   |
|                            |       | -> Extended       |
+----------------------------+       |    thinking 顯示  |
        |                            +------------------+
        |
        v (Tool Use Block)
+----------------------------+
| 工具執行生命週期              |
|                            |
| 1. Tool Lookup             |
| 2. Zod Validation          |
| 3. PreToolUse Hooks        |
| 4. Permission Check        |
| 5. Execute                 |
| 6. PostToolUse Hooks       |
| 7. Result Processing       |
+----------------------------+
        |
        v
+----------------------------+
| Result 加入 Messages        |
| -> 下一輪 API 呼叫          | ----+
+----------------------------+      |
        ^                          |
        |    (Model 需要更多工具)    |
        +--------------------------+
        |
        v (Model 結束回應)
+----------------------------+
| Stop Hook                  |
|   可注入額外提示讓 Model 繼續 |
+----------------------------+
        |
        v
+----------------------------+
| Session Memory 抽取（背景）  |
| -> 非阻塞，fork 子 agent    |
+----------------------------+
        |
        v
[等待下一個用戶輸入]
```

這個流程有幾個關鍵特徵：

1. **Fast-path 分派**：`--version` 等簡單命令不走完整初始化路徑
2. **快取邊界**：System prompt 分靜態/動態兩區，靜態區可跨請求快取省錢
3. **Agentic Loop**：工具執行結果回饋到下一輪 API 呼叫，形成閉環
4. **Hook 切面**：UserPromptSubmit、PreToolUse、PostToolUse、Stop 四個關鍵切點

接下來，我們逐一展開。

---

## 2.2 入口點：從 cli.tsx 到 init.ts

### 2.2.1 cli.tsx — Fast-path 分派

Claude Code 的第一行程式碼執行的是 `src/entrypoints/cli.tsx`。這個檔案的設計哲學是：**能不載入的模組就不載入**。

```typescript
// file: src/entrypoints/cli.tsx
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // Fast-path 1: --version 零模組載入
  if (args[0] === '--version') {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;  // 直接返回，不載入任何模組
  }

  // Fast-path 2: --dump-system-prompt（內部診斷用）
  // Fast-path 3: --claude-in-chrome-mcp（Chrome 整合）
  // Fast-path 4: --chrome-native-host（Native Messaging）

  // 完整啟動路徑——動態 import
  const { init } = await import('../entrypoints/init.js');
  await init();
  // ... 渲染 React Ink App
}
```

為什麼要這樣設計？看這張時間對比：

```
--version fast-path:
  process.argv 解析 → console.log → 退出
  耗時: ~5ms

完整啟動路徑:
  dynamic import init.js → enableConfigs → TLS → Git 偵測 → OAuth
  → Telemetry → API 預連線 → React Ink 渲染
  耗時: ~800-1500ms
```

`--version` 只需要讀一個 build-time 注入的常量 `MACRO.VERSION`。如果為了這個操作載入整個 init 模組（包含 OAuth、Telemetry、MCP 等），就是浪費 1 秒在用戶不需要的事情上。

這個模式叫 **Fast-path Dispatch**：在入口點的最頂層，用最少的程式碼處理最常見的簡單請求，只有需要完整功能時才載入重型模組。

### 2.2.2 init.ts — 完整初始化序列

當用戶真的要啟動互動式對話時，`init()` 被呼叫。這個函式用 `memoize` 包裝，確保只執行一次：

```typescript
// file: src/entrypoints/init.ts
export const init = memoize(async () => {
  // 階段 1：基礎設定
  enableConfigs()                            // 啟用設定系統
  applySafeConfigEnvironmentVariables()      // 安全環境變數
  applyExtraCACertsFromConfig()              // TLS CA 憑證

  // 階段 2：網路設定
  configureGlobalAgents()                    // HTTP/HTTPS 代理
  configureGlobalMTLS()                      // 雙向 TLS

  // 階段 3：平台適配
  setShellIfWindows()                        // Windows shell 設定

  // 階段 4：環境偵測
  detectCurrentRepository()                  // 偵測 Git repo、分支、遠端

  // 階段 5：生命週期管理
  setupGracefulShutdown()                    // SIGINT/SIGTERM 處理

  // 階段 6：可觀測性
  initializeTelemetry()                      // OpenTelemetry spans + metrics

  // 階段 7：認證
  populateOAuthAccountInfoIfNeeded()         // OAuth 帳號資訊
  initializePolicyLimitsLoadingPromise()     // 企業政策限制

  // 階段 8：效能優化
  preconnectAnthropicApi()                   // TCP 預連線，減少首次請求延遲
});
```

初始化序列的順序是精心設計的：

```
依賴關係圖：

enableConfigs ─────────────────────────────────┐
    |                                          |
    v                                          v
applySafeConfigEnv ──> configureGlobalAgents   policyLimits
    |                       |
    v                       v
applyExtraCACerts ──> configureGlobalMTLS
                            |
                            v
                    preconnectAnthropicApi  <-- 必須在 TLS/proxy 之後
```

注意 `preconnectAnthropicApi()` 放在最後。這不是隨意的——它需要 TLS 憑證、代理設定、mTLS 都就緒後才能建立連線。但它又需要盡早執行，因為 TCP 握手 + TLS 協商需要時間，提前做可以在用戶打字的時候完成。

### 2.2.3 React Ink 應用啟動

init 完成後，CLI 啟動 React Ink 應用：

```
App (React Ink root)
├── AppState Provider (Zustand-like store)
├── MCP Connection Manager
├── Plugin Manager
├── Screen Router
│   ├── Onboarding Screen (首次使用)
│   └── Main Chat Screen
│       ├── Message List
│       │   ├── UserMessage
│       │   ├── AssistantMessage
│       │   ├── ToolUseMessage
│       │   └── ToolResultMessage
│       ├── PromptInput (用戶輸入框)
│       ├── StatusLine (token 用量、模型資訊)
│       └── Permission Dialog (條件渲染)
└── Spinner / Progress Indicators
```

此時用戶看到提示符，可以開始輸入問題。

---

## 2.3 System Prompt 組裝

用戶輸入問題後，在發送 API 請求之前，Claude Code 需要組裝 System Prompt。這是整個系統最重要的設計之一。

### 2.3.1 快取邊界設計

```typescript
// file: src/constants/prompts.ts:114-115
const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

System prompt 被一個邊界標記分成兩區：

```
+========================================================+
|                    靜態區（可快取）                       |
|                                                        |
|  scope: 'global' — 不依賴用戶/session，跨請求不變        |
|                                                        |
|  1. Intro Section          角色定義 + 安全指令           |
|  2. System Rules           系統行為規則                  |
|  3. Doing Tasks            任務執行規則                  |
|  4. Actions Section        風險控制框架                  |
|  5. Using Tools            工具使用約束                  |
|  6. Tone & Style           風格規範                     |
|  7. Output Efficiency      輸出效率規則                  |
|                                                        |
+= __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__ ==================+
|                                                        |
|                    動態區（每次重組）                     |
|                                                        |
|  8. Agent Tool Section     Agent 工具指導               |
|  9. Skill Tool Section     Skill 工具指導               |
| 10. Memory Prompt          MEMORY.md 內容               |
| 11. Environment Section    CWD、日期、Git、平台資訊       |
| 12. MCP Instructions       MCP 伺服器使用說明            |
| 13. Output Style           輸出樣式設定                  |
|                                                        |
+========================================================+
```

**為什麼快取邊界重要？**

Anthropic API 支援 prompt caching。當連續的 API 請求前綴相同時，相同的 token 不需要重新處理，可以顯著降低成本和延遲。

```
請求 1: [靜態區 ~3000 tokens] + [動態區 ~500 tokens] + [對話歷史]
請求 2: [靜態區 ~3000 tokens] + [動態區 ~500 tokens] + [對話歷史 + 新訊息]
         ^^^^^^^^^^^^^^^^^^^^^^^^
         這部分完全相同，API 直接用快取

成本差異（以 Claude Sonnet 為例）：
- 未快取 input token:  $3 / M tokens
- 已快取 input token:  $0.30 / M tokens  <-- 便宜 10 倍
```

靜態區的 ~3000 tokens 在整個 session 中保持不變。如果一個 session 有 50 輪對話，這就是 50 次 * 3000 tokens * ($3 - $0.30) / M = $0.405 的節省。對於大規模部署，這個數字乘以用戶數就是顯著的成本差異。

**設計原則**：把不會變的東西放在前面（靜態區），會變的東西放在後面（動態區）。邊界標記 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 確保快取前綴的一致性。

### 2.3.2 靜態區：行為規格書

靜態區不是「溫馨的建議」，而是精確的**行為規格書**。每一節都對應可測試的行為：

**Intro Section — 角色定義 + 安全指令**

定義 Claude Code 是誰、不能做什麼。包含 cyber risk 指令（拒絕破壞性攻擊，允許授權安全測試）。

**Using Tools Section — 工具使用約束**

```
# Using your tools

- Do NOT use the Bash to run commands when a relevant dedicated tool
  is provided. This is CRITICAL to assisting the user:
  - To read files use Read instead of cat, head, tail, or sed
  - To edit files use Edit instead of sed or awk
  - To create files use Write instead of cat with heredoc or echo
  - To search for files use Glob instead of find or ls
  - To search the content of files, use Grep instead of grep or rg
  - Reserve using the Bash exclusively for system commands
```

注意設計要點：
- 用 `Do NOT`（絕對禁令），不用 `try to avoid`（軟性建議）
- 每條禁令配一個**替代方案**——告訴 model 該做什麼
- `This is CRITICAL` 做權重強調——不是所有規則都等重要

**Actions Section — 風險控制框架**

```
# Executing actions with care

Carefully consider the reversibility and blast radius of actions.

Local, reversible actions (freely allowed):
- editing files
- running tests

Hard-to-reverse actions (require confirmation):
- Destructive operations: deleting files/branches, dropping tables
- Force-push, git reset --hard, amending published commits
- Actions visible to others: pushing code, creating/closing PRs
```

這不是模糊的「小心一點」，而是具體的決策矩陣：**可逆性** x **影響範圍**。

**Output Efficiency Section — 輸出效率**

```
# Output efficiency

Go straight to the point. Try the simplest approach first.

Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan

If you can say it in one sentence, don't use three.
```

Anthropic 內部版本更精確，使用數值錨定：

```
- Keep text between tool calls to <= 25 words
- Keep final responses to <= 100 words unless task requires more
```

### 2.3.3 動態區：每次重組的上下文

動態區的內容取決於當前 session 的狀態：

```typescript
// file: src/constants/prompts.ts (組裝邏輯)
async function getSystemPrompt(tools, model, dirs, mcpClients): Promise<string[]> {
  // 1. Simple mode 快速路徑
  if (CLAUDE_CODE_SIMPLE) return [`CWD: ${getCwd()}\nDate: ${date}`]

  // 2. 靜態區（可快取）
  const staticParts = [
    getSimpleIntroSection(),        // 角色 + 安全
    getSimpleSystemSection(),       // 系統規則
    getSimpleDoingTasksSection(),   // 任務執行
    actionsSection,                 // 風險控制
    usingToolsSection,              // 工具約束
    toneAndStyleSection,            // 風格
    outputEfficiencySection,        // 輸出效率
  ]

  // 3. 動態邊界
  staticParts.push(SYSTEM_PROMPT_DYNAMIC_BOUNDARY)

  // 4. 動態區（每次重組）
  const dynamicParts = [
    agentToolSection,               // Agent 工具指導
    skillToolSection,               // Skill 工具指導
    memoryPrompt,                   // MEMORY.md 內容
    environmentSection,             // 運行環境
    mcpInstructions,                // MCP 伺服器指令
    outputStyleSection,             // 輸出樣式
  ]

  return [...staticParts, ...dynamicParts]
}
```

**Memory Prompt** — 載入用戶的 `~/.claude/projects/<path>/memory/MEMORY.md`，讓 model 知道用戶的偏好和過去的決策。

**Environment Section** — 包含 CWD、日期、平台、Git 分支等運行時資訊：

```
Working directory: /Users/leo/my-project
Is directory a git repo: Yes
Platform: darwin
Shell: zsh
OS Version: Darwin 25.2.0
```

**MCP Instructions** — 每個連接的 MCP 伺服器可以提供自己的使用說明，這些說明被注入到 system prompt 中。

**Skill Tool Section** — 列出所有可用 skill 的名稱和描述，讓 model 知道什麼時候該呼叫 `Skill` 工具。

### 2.3.4 完整 System Prompt 的 Token 分佈

```
典型的 System Prompt token 分佈：

+------------------------------------------+
| Intro + System Rules        ~800 tokens  |  靜態
| Doing Tasks + Actions       ~600 tokens  |  靜態
| Using Tools                 ~500 tokens  |  靜態
| Tone + Output Efficiency    ~300 tokens  |  靜態
+------ 快取邊界 ---------------------------+
| Agent/Skill Tool Section    ~400 tokens  |  動態
| Memory Prompt               ~200 tokens  |  動態（視 MEMORY.md 大小）
| Environment                 ~100 tokens  |  動態
| MCP Instructions            ~300 tokens  |  動態（視連接的 MCP 數量）
+------------------------------------------+
  Total: ~3,200 tokens（典型值）

  快取命中率: ~2,200 / 3,200 = ~69%
```

---

## 2.4 Messages 組裝與 API 請求建構

### 2.4.1 Messages 組裝

System prompt 就緒後，需要組裝完整的 messages 陣列：

```typescript
// file: src/utils/messages/ (概念模型)

// 最終發送給 API 的結構
{
  model: "claude-sonnet-4-20250514",
  max_tokens: 16384,
  system: systemPrompt,       // 上一節組裝的 system prompt
  messages: [
    // 歷史訊息（可能已經過壓縮）
    { role: "user", content: "之前的問題..." },
    { role: "assistant", content: "之前的回答..." },
    // 工具呼叫歷史
    { role: "assistant", content: [
      { type: "tool_use", id: "toolu_xxx", name: "Read", input: {...} }
    ]},
    { role: "user", content: [
      { type: "tool_result", tool_use_id: "toolu_xxx", content: "檔案內容..." }
    ]},
    // 新用戶訊息
    { role: "user", content: "現在的問題" },
  ],
  // System reminders（注入在 messages 中間）
  // 用於提醒 model 關鍵資訊（如 CLAUDE.md 內容、MCP 指令）
}
```

**System Reminders** 是一個巧妙的設計。某些重要資訊（如 CLAUDE.md 規則、當前日期、MCP 指令）會被注入為 `system` 角色的訊息，穿插在對話歷史中。這確保即使對話很長，model 也不會「忘記」這些規則。

### 2.4.2 Provider 路由

Claude Code 支援多個 API 提供商。請求建構時需要根據設定選擇正確的路由：

```
API Provider 路由
=================

+------------------+     +-------------------+
| 用戶設定          | --> | Provider Router   |
| - API key 類型   |     |                   |
| - 環境變數       |     | 判斷使用哪個 API   |
| - 組織設定       |     +-------------------+
+------------------+           |
                               |
              +----------------+----------------+
              |                |                |
              v                v                v
    +------------------+  +---------+  +------------------+
    | 1P Anthropic API |  | Bedrock |  | Vertex AI        |
    | api.anthropic.com|  | AWS SDK |  | Google Cloud     |
    +------------------+  +---------+  +------------------+
              |                                |
              v                                v
    +------------------+              +------------------+
    | Azure OpenAI     |              | 其他相容 API      |
    | (如果設定)        |              | (透過 base URL)   |
    +------------------+              +------------------+
```

每個 provider 的請求格式略有不同：

```typescript
// 1P Anthropic API (直連)
// - 使用 Anthropic SDK
// - 支援 prompt caching（cache_control）
// - 支援 extended thinking
// - 支援 streaming

// AWS Bedrock
// - 使用 AWS SDK
// - 模型 ID 格式不同 (anthropic.claude-xxx)
// - 認證走 AWS IAM

// Google Vertex AI
// - 使用 Google Cloud SDK
// - 端點格式不同
// - 認證走 Google OAuth

// Azure
// - 使用 Azure OpenAI SDK
// - 部署名稱映射
```

Provider 路由是透明的——下游的 streaming 處理、工具執行等邏輯不需要知道請求發給了哪個 provider。

### 2.4.3 Token Usage 追蹤

每次 API 呼叫後，系統追蹤 token 使用量：

```typescript
// file: src/services/api/ (概念模型)

interface TokenUsage {
  input_tokens: number       // 輸入 token（含 system prompt + messages）
  output_tokens: number      // 輸出 token（model 的回應）
  cache_creation_input_tokens: number  // 新快取的 token
  cache_read_input_tokens: number      // 從快取讀取的 token
}

// 累積追蹤
function updateUsage(currentUsage: TokenUsage, newUsage: TokenUsage): void {
  // 累加每個欄位
}

function accumulateUsage(sessionUsage: TokenUsage, turnUsage: TokenUsage): void {
  // 累加到 session 層級
  // 用於 /cost 命令顯示、StatusLine 渲染
}
```

Token usage 的追蹤有兩個層級：

```
Turn-level usage:
  這一輪 API 呼叫用了多少 token

Session-level usage:
  整個 session 累計用了多少 token
  → 用於 StatusLine 顯示（底部的 token 計數器）
  → 用於 /cost 命令計算費用
  → 用於 AutoCompact 閾值判斷
```

---

## 2.5 Streaming 回應處理

### 2.5.1 Streaming 事件流

Claude API 回傳的是一個 Server-Sent Events (SSE) 串流。Claude Code 將其抽象為 async iterable：

```typescript
// file: src/services/api/ (streaming 處理)

for await (const event of apiStream) {
  switch (event.type) {
    case 'message_start':
      // 回應開始，包含 message 元資料
      break;

    case 'content_block_start':
      // 開始新的 content block
      // block.type 可能是: 'text' | 'tool_use' | 'thinking'
      handleBlockStart(event.content_block);
      break;

    case 'content_block_delta':
      // 增量內容更新
      // delta.type 可能是:
      //   'text_delta'        → 文字內容
      //   'input_json_delta'  → 工具輸入 JSON 片段
      //   'thinking_delta'    → 思考內容
      handleDelta(event.delta);
      break;

    case 'content_block_stop':
      // Block 完成
      // 如果是 tool_use block → 觸發工具執行
      handleBlockStop(event.index);
      break;

    case 'message_delta':
      // message-level 更新（stop_reason, usage）
      break;

    case 'message_stop':
      // 整個回應結束
      // → 觸發 Stop hook
      handleMessageStop();
      break;
  }
}
```

### 2.5.2 三種 Content Block 的處理

```
content_block_start 事件到達

  block.type == 'text'
  ├── 後續收到 text_delta 事件
  ├── 每個 delta 立即推送到 React Ink 渲染
  ├── 用戶看到文字逐字出現
  └── content_block_stop → 文字 block 完成

  block.type == 'thinking'
  ├── Extended thinking 內容
  ├── thinking_delta 事件逐步到達
  ├── 渲染到 UI 的「思考」區域
  └── content_block_stop → 思考 block 完成

  block.type == 'tool_use'
  ├── 包含 tool name 和 id
  ├── 後續收到 input_json_delta 事件
  ├── JSON 片段需要累積（見下節）
  └── content_block_stop → 觸發工具執行
```

### 2.5.3 Tool Use Block 的 JSON 累積

工具輸入的 JSON 是逐片段到達的，需要累積後才能解析：

```typescript
// Tool Use Block JSON 累積過程

// Step 1: content_block_start
// { type: "tool_use", id: "toolu_abc123", name: "Read" }
// 此時還沒有 input

// Step 2: 一系列 input_json_delta
// delta: '{"fil'
// delta: 'e_pa'
// delta: 'th": '
// delta: '"/src'
// delta: '/main'
// delta: '.ts"}'

// 累積器
let jsonAccumulator = '';

function handleInputJsonDelta(delta: string): void {
  jsonAccumulator += delta;  // 逐步拼接
  // 可選：嘗試部分解析用於 UI 預覽
}

// Step 3: content_block_stop
// 此時 jsonAccumulator = '{"file_path": "/src/main.ts"}'
const toolInput = JSON.parse(jsonAccumulator);
// → { file_path: "/src/main.ts" }
// 現在可以觸發工具執行了
```

這個設計的好處是：在 JSON 還在累積的過程中，UI 可以顯示「正在準備工具呼叫...」，給用戶即時回饋。

### 2.5.4 多 Block 的處理

一次 API 回應可能包含多個 content block：

```
典型的多 block 回應：

Block 0 (text):     "讓我讀取這個檔案來了解結構。"
Block 1 (tool_use): Read { file_path: "/src/main.ts" }
Block 2 (tool_use): Read { file_path: "/src/utils.ts" }

處理順序：
1. Block 0 text 逐字渲染到終端
2. Block 1 tool_use JSON 累積
3. Block 2 tool_use JSON 累積
4. message_stop 到達
5. 兩個 tool_use block 進入工具執行流程
   → 先分區（concurrent vs serial）
   → Read 是 concurrent-safe，兩個同時執行
```

---

## 2.6 工具執行生命週期

這是請求生命週期中最複雜的部分。一個工具呼叫要經過 7 個階段。

### 2.6.1 完整執行流程

```typescript
// file: src/services/tools/toolExecution.ts (~60KB)

async function runToolUse(toolUseBlock: ToolUseBlock): Promise<ToolResult> {
  // ====== 階段 1: 工具查找 ======
  const tool = findTool(toolUseBlock.name, tools);
  // 支援 alias 匹配（工具重命名後的向後相容）
  // 找不到 → 回傳錯誤訊息給 model

  // ====== 階段 2: 輸入驗證 ======
  // 2a. Zod schema 解析
  const parseResult = tool.inputSchema.safeParse(toolUseBlock.input);
  if (!parseResult.success) {
    return { error: formatZodError(parseResult.error) };
  }
  // 2b. 工具特定驗證
  const validation = tool.validateInput(parseResult.data);
  if (!validation.valid) {
    return { error: validation.message };
  }

  // ====== 階段 3: PreToolUse Hooks ======
  const hookResult = await executeHooks('PreToolUse', {
    toolName: tool.name,
    input: parseResult.data,
  });
  // Hook 可以：
  //   - 阻止執行 (decision: 'block')
  //   - 修改輸入 (hookSpecificOutput.updatedInput)
  //   - 自動批准 (hookSpecificOutput.permissionDecision: 'allow')
  //   - 注入額外上下文 (hookSpecificOutput.additionalContext)
  if (hookResult.decision === 'block') {
    return { error: hookResult.stopReason };
  }

  // ====== 階段 4: 權限檢查 ======
  // 4a. 工具特定權限
  const permResult = tool.checkPermissions(input, context);

  // 4b. 規則匹配 (deny > allow > ask)
  const ruleResult = checkRuleBasedPermissions(tool.name, input);

  // 4c. Hook 覆蓋
  const hookDecision = resolveHookPermissionDecision(hookResult);

  // 4d. Permission mode 評估
  //     bypass → 自動允許
  //     auto   → classifier 判斷
  //     default → 顯示提示

  // 4e. 互動式提示（如需要）
  const userDecision = await canUseTool(tool, input);
  // 用戶選擇：允許一次 / 永遠允許 / 拒絕
  if (userDecision === 'denied') {
    return { error: "permission denied by user" };
  }

  // ====== 階段 5: 工具執行 ======
  const result = await tool.call(
    input,           // 驗證過的輸入（可能被 hook 修改）
    context,         // 執行上下文（CWD, session 資訊等）
    canUseTool,      // 權限回呼函式
    parentMessage,   // 觸發此工具呼叫的 assistant message
    onProgress       // 進度回呼（streaming progress）
  );

  // ====== 階段 6: PostToolUse Hooks ======
  const postHookResult = await executeHooks('PostToolUse', {
    toolName: tool.name,
    input: input,
    output: result,
  });
  // Hook 可以：
  //   - 修改輸出 (hookSpecificOutput.updatedMCPToolOutput)
  //   - 注入額外上下文
  //   - 阻止結果

  // ====== 階段 7: 結果處理 ======
  const processed = mapToolResultToToolResultBlockParam(result);
  return processToolResultBlock(processed);
  // 包含：大結果持久化、內容截斷、MCP metadata 處理
}
```

### 2.6.2 權限檢查的優先順序

權限檢查是多層的，理解優先順序很重要：

```
權限檢查優先順序（從高到低）：

1. Policy Settings deny 規則     → 企業管理，無法覆蓋
                    |
2. Tool-specific checkPermissions → 工具自己的邏輯
                    |              （如 FileWrite 檢查路徑）
                    |
3. Rule-based deny 規則          → 比 allow 優先
                    |
4. PreToolUse Hook decision      → Hook 可以 allow/block
                    |
5. Rule-based allow 規則         → 自動允許
                    |
6. Permission Mode 評估          → bypass/auto/default
                    |
7. 互動式提示                    → 最後的防線
```

規則匹配的格式支援多種粒度：

```
ToolName              → 工具級別（匹配該工具所有呼叫）
ToolName(content)     → 內容級別
Bash(git add)         → 精確匹配
Bash(npm:*)           → 前綴匹配
Bash(python *)        → 萬用字元匹配
mcp__server1          → 匹配整個 MCP 伺服器的所有工具
```

### 2.6.3 並發控制：partitionToolCalls

當 model 一次回傳多個 tool_use block 時，系統需要決定哪些可以並行、哪些必須串行：

```typescript
// file: src/services/tools/toolOrchestration.ts

function partitionToolCalls(
  toolCalls: ToolUseBlock[]
): { concurrent: ToolUseBlock[], serial: ToolUseBlock[] } {
  const concurrent = [];  // isConcurrencySafe() === true
  const serial = [];      // isConcurrencySafe() === false

  for (const call of toolCalls) {
    const tool = findTool(call.name);
    if (tool.isConcurrencySafe()) {
      concurrent.push(call);
    } else {
      serial.push(call);
    }
  }

  return { concurrent, serial };
}
```

執行策略：

```
Model 回傳 4 個 tool_use blocks:
  [Grep("error"), Glob("*.ts"), Bash("npm test"), FileEdit("fix")]

partitionToolCalls 分區:
  concurrent: [Grep("error"), Glob("*.ts")]    ← 讀取類，可並行
  serial:     [Bash("npm test"), FileEdit("fix")] ← 寫入類，必須串行

執行順序:
  Step 1: Grep + Glob  → Promise.all() 並行執行
  Step 2: Bash         → 等 Step 1 完成後串行執行
  Step 3: FileEdit     → 等 Step 2 完成後串行執行

時間線:
  t=0s   [Grep ████████]
         [Glob ███]
  t=1.2s                [Bash █████████████]
  t=3.5s                                    [FileEdit ██]
  t=4.0s  完成

比起全部串行：
  t=0s   [Grep ████████]
  t=1.2s                [Glob ███]
  t=1.5s                          [Bash █████████████]
  t=3.8s                                              [FileEdit ██]
  t=4.3s  完成

節省: ~0.3s（此例中不多，但當有 5-6 個 Grep 並行時差距明顯）
```

並發安全的工具（可並行）：

| 工具 | 原因 |
|------|------|
| GlobTool | 只讀檔案系統 |
| GrepTool | 只讀檔案系統 |
| FileReadTool | 只讀檔案系統 |
| WebSearchTool | 無副作用 |
| WebFetchTool | 無副作用 |
| TaskGetTool | 只讀狀態 |
| TaskListTool | 只讀狀態 |
| ToolSearchTool | 只讀 |
| LSPTool | 只讀查詢 |
| BriefTool | 只輸出 |
| ListMcpResourcesTool | 只讀 |
| ReadMcpResourceTool | 只讀 |

非並發安全的工具（必須串行）：

| 工具 | 原因 |
|------|------|
| BashTool | 可能修改檔案系統 |
| FileWriteTool | 寫入檔案 |
| FileEditTool | 修改檔案 |
| AgentTool | 生成子 agent（有副作用） |
| SkillTool | 可能執行任何操作 |
| NotebookEditTool | 修改 notebook |
| TaskCreateTool | 修改狀態 |

**Context Modifiers 佇列化**：當並行工具執行時，它們產生的 context modifier（如修改 CWD、更新狀態）會被佇列化，批次結束後依序套用，避免 race condition。

### 2.6.4 大結果持久化到磁碟

當工具輸出超過 `maxResultSizeChars` 閾值時，完整結果不會放進 messages——那會浪費 context window：

```
大結果持久化流程
================

工具執行完成
    |
    v
結果大小 > maxResultSizeChars?
    |
    +-- No → 正常回傳完整結果
    |
    +-- Yes → 持久化流程
              |
              v
        1. 將完整結果寫入磁碟暫存檔
           路徑: ~/.claude/tmp/tool-result-<id>.txt
              |
              v
        2. 回傳給 model 的只有：
           "Result too large. Preview (first 200 lines):
            ...
            Full result saved to: /path/to/tool-result-<id>.txt
            Use FileReadTool to read specific sections."
              |
              v
        3. Model 如果需要完整內容，
           可以用 FileReadTool 讀取特定段落
```

這個設計避免了一個常見問題：用戶跑 `grep -r` 搜尋，結果有幾萬行，如果全部塞進 messages 會立刻觸發 context overflow。持久化到磁碟後，model 可以選擇性地讀取需要的部分。

---

## 2.7 錯誤處理

生產系統的關鍵不是「正常路徑走得通」，而是「異常路徑處理得好」。Claude Code 的錯誤處理覆蓋了所有常見的失敗場景。

### 2.7.1 Rate Limit (429) 與 Overload (529) — 指數退避

```typescript
// file: src/services/api/ (重試邏輯)

const BASE_DELAY_MS = 500;  // 基礎延遲 500ms

async function retryWithExponentialBackoff(
  fn: () => Promise<Response>,
  maxRetries: number
): Promise<Response> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (error.status === 429 || error.status === 529) {
        // 指數退避: 500ms, 1000ms, 2000ms, 4000ms, ...
        const delay = BASE_DELAY_MS * Math.pow(2, attempt);

        // 如果 API 回傳 retry-after header，優先使用
        const retryAfter = error.headers?.['retry-after'];
        const waitTime = retryAfter
          ? parseInt(retryAfter) * 1000
          : delay;

        await sleep(waitTime);
        continue;
      }
      throw error;  // 其他錯誤不重試
    }
  }
  throw new Error('Max retries exceeded');
}
```

退避時間表：

```
嘗試   延遲          累積等待
 0     500ms         0.5s
 1     1,000ms       1.5s
 2     2,000ms       3.5s
 3     4,000ms       7.5s
 4     8,000ms       15.5s
 5     16,000ms      31.5s
 ...
```

### 2.7.2 Token Overflow — 自動調整

當 API 回傳 token limit 錯誤時，系統不是簡單地報錯，而是嘗試自動解決：

```
Token Overflow 處理流程
========================

API 回傳 "context window exceeded" 或 "max_tokens exceeded"
    |
    v
觸發 AutoCompact
    |
    ├── Layer 1: MicroCompact（零成本清理舊結果）
    |
    ├── Layer 2: Session Memory Compact（輕量壓縮）
    |
    ├── Layer 3: Full Conversation Compact（完整重組）
    |
    v
壓縮後重試 API 呼叫
    |
    ├── 成功 → 繼續正常流程
    |
    └── 仍然失敗 → 斷路器檢查
        |
        ├── consecutiveFailures < 3 → 再試
        |
        └── consecutiveFailures >= 3 → 放棄，通知用戶
            "Context window full. Please start a new session
             or use /compact to manually compress."
```

AutoCompact 的觸發閾值計算：

```typescript
// file: src/services/compact/autoCompact.ts:72-90
function getAutoCompactThreshold(model: string): number {
  const contextWindow = getContextWindowForModel(model);
  // 例如 Claude Sonnet: 200,000 tokens

  const reservedForSummary = Math.min(maxOutputTokens, 20_000);
  // 保留 20K tokens 給壓縮摘要

  const effectiveWindow = contextWindow - reservedForSummary;
  // 200,000 - 20,000 = 180,000

  return effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS;
  // 180,000 - 13,000 = 167,000 ← 觸發點
}
```

### 2.7.3 Auth 錯誤重新整理

```
API 回傳 401 (Unauthorized) 或 403 (Forbidden)
    |
    v
檢查錯誤類型
    |
    ├── OAuth token 過期
    |   └── 嘗試 refresh token
    |       ├── 成功 → 用新 token 重試
    |       └── 失敗 → 提示用戶重新認證
    |
    ├── API key 無效
    |   └── 提示用戶更新 API key
    |
    └── 權限不足（如模型存取限制）
        └── 提示用戶檢查帳號設定
```

### 2.7.4 Circuit Breaker — 斷路器模式

斷路器防止系統在持續失敗時反覆重試，浪費資源：

```typescript
// file: src/services/compact/autoCompact.ts:257-265

type AutoCompactTrackingState = {
  compacted: boolean;
  turnCounter: number;
  consecutiveFailures?: number;  // 斷路器計數器
};

const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3;

// 每次嘗試壓縮時
if (tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  // 斷路器跳脫——停止所有後續嘗試
  log("circuit breaker tripped after 3 consecutive failures");
  return { wasCompacted: false };
}

// 失敗時
tracking.consecutiveFailures = (tracking.consecutiveFailures ?? 0) + 1;

// 成功時重置
tracking.consecutiveFailures = 0;
```

斷路器模式的狀態轉換：

```
        成功
  +----→ [CLOSED] ←----+
  |      (正常運作)      |
  |          |          |
  |       失敗 (1/3)    | 成功（重置計數）
  |          |          |
  |          v          |
  |      [HALF-OPEN]   |
  |      (嘗試中)       |
  |          |          |
  |       失敗 (2/3)    |
  |          |          |
  |          v          |
  |      [HALF-OPEN]   |
  |      (再次嘗試)     |
  |          |          |
  |       失敗 (3/3)    |
  |          |          |
  |          v          |
  |      [OPEN]         |
  |      (停止重試)      |
  |          |          |
  |       (新 session   |
  |        重置)        |
  +----------+----------+
```

### 2.7.5 Persistent Retry — 無人值守模式

在 headless/unattended 模式下（如 CI/CD 或 cron 任務），錯誤處理更加積極：

```
無人值守模式的重試策略
======================

初始失敗
    |
    v
指數退避重試
    |
    初始延遲: 500ms (BASE_DELAY_MS)
    最大延遲: 5 分鐘 (MAX_BACKOFF_MS = 300,000)
    |
    計算公式: min(BASE_DELAY_MS * 2^attempt, MAX_BACKOFF_MS)
    |
    退避序列:
    500ms → 1s → 2s → 4s → 8s → 16s → 32s → 64s → 128s → 256s → 300s → 300s → ...
    |
    v
重置條件
    |
    最大累積重試時間: 6 小時 (MAX_RETRY_DURATION = 21,600,000ms)
    超過 6 小時 → 放棄，退出進程
    |
    v
成功恢復
    |
    重試成功 → 重置所有計數器 → 繼續正常執行
```

為什麼是 5 分鐘最大退避？因為 API rate limit 通常在幾分鐘內恢復。如果 5 分鐘還不行，大概率是更嚴重的問題（服務中斷、帳號問題）。

為什麼是 6 小時上限？這是為了防止 cron 任務永遠卡住。6 小時足夠等過大多數服務中斷，但不會無限消耗資源。

### 2.7.6 工具層級的錯誤處理

工具執行的錯誤不會導致整個 session 崩潰——它們被結構化地回傳給 model：

```
工具錯誤處理策略
================

[Input validation failure]
    → 回傳結構化錯誤訊息給 model
    → model 自動修正輸入重試
    → 例: "Invalid file_path: expected string, got number"

[Permission denied]
    → 回傳 "Permission denied for Bash(rm -rf /)"
    → model 知道這個操作被拒絕，嘗試替代方案

[Execution error]
    → 回傳 stderr 內容給 model
    → model 分析錯誤，決定下一步
    → 例: Bash("npm test") → stderr: "Test suite failed: 3 failures"

[Timeout]
    → 回傳 "Tool execution timed out after 120s"
    → model 決定是否重試或改用其他方法

[Tool not found]
    → 回傳 "Unknown tool: XxxTool. Available tools: ..."
    → model 使用正確的工具名稱重試
```

關鍵設計原則：**工具錯誤不是系統錯誤**。它們是 model 學習的訊號。model 看到 `npm test` 失敗的 stderr 後，可以分析具體哪個測試失敗，修改程式碼，再跑一次。這就是 Agentic Loop 的核心。

---

## 2.8 完整的 Agentic Loop

現在我們可以把所有環節串在一起，看一個真實的互動是怎麼運作的。

### 2.8.1 一個完整範例

用戶輸入：「幫我修復 src/utils.ts 裡的 TypeScript 編譯錯誤」

```
Round 1: 理解問題
=================

用戶輸入: "幫我修復 src/utils.ts 裡的 TypeScript 編譯錯誤"
    |
    v
UserPromptSubmit Hook → (無修改)
    |
    v
System Prompt 組裝 (靜態區快取命中 + 動態區重組)
    |
    v
Messages 組裝: [system_prompt, user_message]
    |
    v
API 請求 → Claude 回應:
  Text: "讓我先看看這個檔案的內容和錯誤。"
  ToolUse: Read({ file_path: "/src/utils.ts" })
  ToolUse: Bash({ command: "npx tsc --noEmit src/utils.ts 2>&1" })
    |
    v
partitionToolCalls:
  concurrent: [Read]   ← 讀取，可並行
  serial: [Bash]       ← 執行，必須串行
    |
    v
Step 1: Read 執行
  → PreToolUse Hook → 無阻止
  → Permission: FileRead 預設允許
  → 執行: 讀取檔案內容
  → PostToolUse Hook → 無修改
  → 結果: 200 行 TypeScript 程式碼

Step 2: Bash 執行
  → PreToolUse Hook → 無阻止
  → Permission: Bash(npx tsc*) → 匹配 allow 規則
  → 執行: 跑 TypeScript 編譯器
  → PostToolUse Hook → 無修改
  → 結果: "src/utils.ts(42,15): error TS2345: ..."
    |
    v
兩個 ToolResult 加入 Messages


Round 2: 修復錯誤
=================

Messages: [system, user, assistant(text+tools), user(results)]
    |
    v
API 請求 → Claude 回應:
  Text: "找到了。第 42 行的型別不匹配，需要..."
  ToolUse: Edit({
    file_path: "/src/utils.ts",
    old_string: "...",
    new_string: "..."
  })
    |
    v
Edit 執行
  → PreToolUse Hook → 無阻止
  → Permission: acceptEdits 模式 → 自動允許
  → 執行: 修改檔案
  → PostToolUse Hook → 無修改
  → 結果: "Successfully edited"
    |
    v
ToolResult 加入 Messages


Round 3: 驗證修復
=================

Messages: [system, user, assistant, results, assistant(edit), result]
    |
    v
API 請求 → Claude 回應:
  Text: ""
  ToolUse: Bash({ command: "npx tsc --noEmit src/utils.ts 2>&1" })
    |
    v
Bash 執行 → 結果: (無輸出，編譯成功)
    |
    v
ToolResult 加入 Messages


Round 4: 回報結果
=================

Messages: [... 之前所有 ...]
    |
    v
API 請求 → Claude 回應:
  Text: "修復完成。第 42 行的型別錯誤是因為..."
  (無 ToolUse → 回應結束)
    |
    v
Stop Hook → (無額外指令)
    |
    v
Session Memory 抽取 (背景)
  → fork 子 agent
  → 記錄: 修復了 utils.ts 的型別錯誤
    |
    v
[等待下一個用戶輸入]
```

### 2.8.2 Loop 的終止條件

Agentic loop 什麼時候停止？

```
Loop 終止條件
=============

1. Model 回應不含 tool_use block
   → 自然結束，Model 認為任務完成

2. Stop Hook 不注入新提示
   → Hook 沒有要求 Model 繼續

3. 用戶按 Ctrl+C / ESC
   → AbortSignal 觸發，中斷當前操作

4. Token 超限 + AutoCompact 失敗
   → 斷路器跳脫，強制結束

5. 連續失敗計數達到上限
   → API 持續錯誤，放棄重試

6. 政策限制（policyLimits）
   → 企業管理限制 token 用量或工具呼叫次數
```

### 2.8.3 每輪的 Token 成本分析

```
典型的 4 輪互動 Token 分析
===========================

Round 1 (讀取 + 編譯)
  Input:  system(3,200) + user(50) = 3,250 tokens
  Output: text(30) + tool_use(50) = 80 tokens
  Cost:   input $0.0098 + output $0.0012

Round 2 (修復)
  Input:  3,250 + 80(R1 output) + 3,000(tool results) + 50(空) = 6,380 tokens
          其中 3,200 快取命中 → 有效 input = 3,180 new + 3,200 cached
  Output: text(50) + tool_use(40) = 90 tokens
  Cost:   new_input $0.0095 + cached $0.00096 + output $0.0014

Round 3 (驗證)
  Input:  6,380 + 90 + 500 = 6,970 tokens
  Output: tool_use(30) tokens
  Cost:   (同上模式，快取比例更高)

Round 4 (回報)
  Input:  6,970 + 30 + 100 = 7,100 tokens
  Output: text(80) tokens
  Cost:   ...

Total: ~24,000 input tokens + ~280 output tokens
       ≈ $0.04-0.07 (Claude Sonnet 定價)

快取節省: ~3,200 tokens × 3 次 × ($3 - $0.30)/M ≈ $0.026
         → 快取邊界設計在此例中節省了 ~35% 的 input 成本
```

---

## 2.9 Token Usage 追蹤的完整流程

Token 追蹤貫穿整個請求生命週期，理解它的機制對成本優化至關重要。

```typescript
// Token 追蹤的兩個層級

// Turn-level: 單輪 API 呼叫
interface TurnUsage {
  input_tokens: number;
  output_tokens: number;
  cache_creation_input_tokens: number;
  cache_read_input_tokens: number;
}

// Session-level: 整個 session 累積
interface SessionUsage {
  totalInputTokens: number;
  totalOutputTokens: number;
  totalCacheCreationTokens: number;
  totalCacheReadTokens: number;
  turnCount: number;
}
```

追蹤流程：

```
API 回應的 message_delta 事件
    |
    v
提取 usage 欄位
    |
    v
updateUsage(currentTurnUsage, delta)  ← 更新當前 turn
    |
    v
Turn 結束後
    |
    v
accumulateUsage(sessionUsage, turnUsage)  ← 累加到 session
    |
    v
兩個消費者:
    |
    ├── StatusLine 元件
    |   → 底部顯示 "Tokens: 24.3K in / 0.3K out"
    |
    ├── /cost 命令
    |   → 計算費用: "$0.07 this session"
    |
    └── AutoCompact 觸發判斷
        → if (sessionUsage.totalInputTokens > threshold)
          → 觸發壓縮
```

---

## 2.10 設計模式

本章涵蓋的請求生命週期中，有幾個可以直接帶走的設計模式：

### 模式 1：Fast-path Dispatch

```
問題：入口點載入重型模組太慢
解法：在最頂層用最少的程式碼處理簡單請求

                  ┌─ simple cmd? ─→ 快速回傳（~5ms）
process.argv ─────┤
                  └─ full cmd? ──→ dynamic import → 完整初始化（~1s）
```

**適用場景**：任何有「輕量操作」和「重量操作」混合的 CLI 工具。`--version`、`--help` 不需要載入資料庫連線、OAuth flow 等重型依賴。

### 模式 2：Static/Dynamic Cache Boundary

```
問題：每次 API 請求都發送完整的 system prompt 很貴
解法：把不變的部分放前面（快取命中），變化的部分放後面

[靜態：行為規則，3000 tokens] ← 跨請求快取
-------- 邊界標記 --------
[動態：環境/記憶，500 tokens] ← 每次重組
```

**適用場景**：任何使用 LLM API 且 system prompt 較長的應用。成本節省與 session 長度成正比。

### 模式 3：Agentic Loop with Tool Result Feedback

```
問題：AI 需要多步驟完成任務，但 API 是 request-response 模式
解法：把工具結果作為新的 user message 送回 API

      +-→ API → Tool → Result ─+
      |                        |
      +── 加入 Messages ←──────+
      |
      +─→ Model 不再需要工具 → 結束
```

**關鍵**：工具錯誤不是系統錯誤，而是 model 的學習訊號。讓 model 看到 stderr，它會自己修正。

### 模式 4：Concurrent/Serial Tool Partitioning

```
問題：多個工具呼叫如何安全並行？
解法：根據工具的副作用特性分區

isConcurrencySafe() == true  → Promise.all() 並行
isConcurrencySafe() == false → 逐一串行

判斷標準：
  讀取 → 安全並行
  寫入 → 必須串行
  有狀態副作用 → 必須串行
```

### 模式 5：Circuit Breaker for Retry Operations

```
問題：持續失敗的重試會浪費資源並陷入死循環
解法：N 次連續失敗後停止重試

CLOSED ──失敗──→ count++
  ^               |
  |          count >= 3
  |               |
  +──成功──── OPEN（停止重試）
  |               |
  +── 新 session → 重置
```

**N=3 是經驗值**：足夠排除偶發失敗，又不會在持續失敗時浪費太多資源。

### 模式 6：Exponential Backoff with Cap

```
問題：Rate limit 後如何合理重試？
解法：指數退避 + 上限封頂

delay = min(BASE_DELAY * 2^attempt, MAX_DELAY)

BASE_DELAY = 500ms     → 快速首次重試
MAX_DELAY  = 300,000ms → 5 分鐘封頂（防止等太久）
TOTAL_CAP  = 6 hours   → 無人模式的最終安全閥
```

### 模式 7：Hook-based Aspect-Oriented Middleware

```
問題：如何在不修改核心邏輯的情況下擴展行為？
解法：在生命週期的關鍵切點注入 Hook

UserPromptSubmit → 修改/攔截用戶輸入
PreToolUse       → 修改/攔截工具呼叫
PostToolUse      → 修改/攔截工具結果
Stop             → 注入繼續指令

每個切點支援：
  - command (shell 腳本)
  - prompt (LLM 單次查詢)
  - agent (LLM 多輪 agent)
  - http (HTTP 請求)
  - callback (內聯函式)
```

---

## 2.11 生命週期中的資料流圖

用另一個角度看整個請求生命週期——追蹤資料的流動方向：

```
                        資料流圖
                        ========

用戶輸入 (string)
    │
    ├──→ UserPromptSubmit Hook ──→ 可能修改的 string
    │
    v
System Prompt 組裝
    │
    ├──→ MEMORY.md (磁碟 → 記憶體)
    ├──→ .claude/settings.json (磁碟 → 設定)
    ├──→ MCP server instructions (IPC → 指令)
    ├──→ environment info (OS API → 環境資訊)
    │
    v
Messages Array
    │
    ├──→ 歷史 messages (記憶體，可能已壓縮)
    ├──→ 新 user message
    ├──→ system reminders (注入)
    │
    v
HTTP Request Body (JSON)
    │
    ├──→ Provider Router ──→ 選擇 API endpoint
    │
    v
SSE Stream (網路 → 記憶體)
    │
    ├──→ text_delta ──→ React Ink render ──→ 終端輸出 (stdout)
    ├──→ thinking_delta ──→ UI 思考區域 ──→ 終端輸出
    ├──→ input_json_delta ──→ JSON accumulator ──→ tool input
    ├──→ usage ──→ updateUsage ──→ SessionUsage
    │
    v
Tool Use Blocks
    │
    ├──→ partition ──→ concurrent / serial queues
    │
    v
Tool Execution
    │
    ├──→ 檔案系統 (FileRead/Write/Edit)
    ├──→ subprocess (Bash)
    ├──→ 網路 (WebFetch/Search)
    ├──→ 子 agent (Agent/SendMessage)
    ├──→ MCP IPC (MCPTool)
    │
    v
Tool Results
    │
    ├──→ 小結果 → 直接加入 messages
    ├──→ 大結果 → 磁碟暫存 + 預覽加入 messages
    │
    v
回到 Messages Array → 下一輪 API 呼叫
    │
    └──→ (loop 結束後)
         │
         ├──→ Stop Hook
         ├──→ Session Memory 抽取 (背景子 agent → 磁碟)
         └──→ Token Usage 累積 → StatusLine 渲染
```

---

## 2.12 與其他章節的連接

請求生命週期是全書的骨幹。後續章節都是展開其中某個環節的深度分析：

```
本章概覽                         深度章節
========                        ========

System Prompt 組裝    ──────→   Ch03: System Prompt 工程化
  (2.3 節)                      逐行拆解 prompts.ts 的每個 section

工具執行生命週期      ──────→   (Tool 系統深度分析)
  (2.6 節)                      43 個工具的統一接口設計

Permission Check     ──────→   (Permission 模型深度分析)
  (2.6.2 節)                    6 種模式、8 層優先順序

Hook 切點            ──────→   (Hook 系統深度分析)
  (多處提及)                     27 種事件、5 種執行類型

Token Overflow       ──────→   Ch05: 三層上下文壓縮
  (2.7.2 節)                    MicroCompact → AutoCompact → Full

Session Memory       ──────→   Ch06: AutoDream 記憶系統
  (2.8.1 末)                    即時抽取 + 定期整理

Agentic Loop         ──────→   Ch04: 多 Agent 協作
  (2.8 節)                      從單 agent loop 到 multi-agent swarm
```

---

## 章節總結

| 要點 | 內容 |
|------|------|
| **入口分派** | cli.tsx fast-path 設計讓 `--version` 在 ~5ms 內回應，完整初始化走 dynamic import |
| **初始化序列** | init.ts 按依賴關係排序 12 步初始化，preconnect 放最後但最早產生效益 |
| **System Prompt** | 分靜態/動態兩區，快取邊界 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 節省 ~35% input 成本 |
| **Streaming** | SSE 事件流：`content_block_start` → `delta` → `stop` → `message_stop`，tool JSON 需累積 |
| **Provider 路由** | 支援 1P API / Bedrock / Vertex / Azure，下游邏輯透明 |
| **工具執行** | 7 階段生命週期：查找 → 驗證 → PreHook → 權限 → 執行 → PostHook → 結果處理 |
| **並發控制** | `partitionToolCalls` 分讀取（並行）和寫入（串行），context modifier 佇列化 |
| **大結果** | 超過閾值的工具輸出持久化到磁碟，只回傳預覽 |
| **Rate Limit** | 指數退避：BASE=500ms，2^n 增長，5 分鐘封頂 |
| **斷路器** | 連續失敗 3 次停止重試，新 session 重置 |
| **Persistent Retry** | 無人模式：5 分鐘最大退避，6 小時總上限 |
| **Agentic Loop** | 工具結果回饋到下一輪 API 呼叫，model 自主決定何時停止 |
| **Token 追蹤** | `updateUsage` 更新 turn 層級，`accumulateUsage` 累加 session 層級 |

**一句話總結**：Claude Code 的請求生命週期是一個**帶有 Hook 切面的 Agentic Loop**——用戶輸入觸發 System Prompt 組裝和 API 呼叫，Streaming 回應被解析為文字和工具呼叫，工具執行結果回饋到下一輪 API 呼叫，直到 model 決定停止。每個環節都有 Hook 可擴展、有錯誤處理保護、有成本優化機制。
