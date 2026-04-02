# 重點一：學 Anthropic 怎麼寫 System Prompt

> **源碼**: `src/constants/prompts.ts` (54KB)
>
> 這不是「寫得好」的 prompt，而是**工程化的行為控制系統**。

## 傳統寫法 vs Anthropic 寫法

```
❌ 傳統：「盡量幫助使用者，提供詳細回答」
✅ Anthropic：精確的行為規格書，每條規則對應一個可測試的行為
```

## Prompt 結構分解

### 1. 快取邊界設計

```typescript
// prompts.ts:114-115
const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

Prompt 分為兩區：
- **靜態區** (`scope: 'global'`)：不依賴用戶/session，可跨請求快取
- **動態區**：包含 MEMORY.md、MCP 指令、環境資訊等

**啟示**：把不常變的規則放前面，省 API token。

### 2. 工具約束——不是「建議」，而是「禁令」

```
# Using your tools
- Do NOT use the Bash to run commands when a relevant dedicated tool is provided.
  This is CRITICAL to assisting the user:
  - To read files use Read instead of cat, head, tail, or sed
  - To edit files use Edit instead of sed or awk
  - To create files use Write instead of cat with heredoc or echo redirection
  - To search for files use Glob instead of find or ls
  - To search the content of files, use Grep instead of grep or rg
  - Reserve using the Bash exclusively for system commands and terminal operations
```

**設計要點**：
- 用「Do NOT」而非「try to avoid」——絕對禁令
- 每條都給出**替代方案**——不只禁止，告訴 model 該做什麼
- 用「This is CRITICAL」做強調——不是所有規則都等權重

### 3. 風險控制——基於可逆性的決策框架

```
# Executing actions with care

Carefully consider the reversibility and blast radius of actions.

Local, reversible actions (freely allowed):
- editing files
- running tests

Hard-to-reverse actions (require confirmation):
- Destructive operations: deleting files/branches, dropping tables, rm -rf
- Force-push, git reset --hard, amending published commits
- Actions visible to others: pushing code, creating/closing PRs, sending messages
- Uploading content to third-party tools
```

**設計要點**：
- 不是「小心一點」，而是**具體的決策樹**
- 兩個維度：**可逆性** × **影響範圍**
- 給出明確的例子清單，不留模糊空間

### 4. 輸出規範——先結論後解釋

```
# Output efficiency

Go straight to the point. Try the simplest approach first without going in circles.

Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan

If you can say it in one sentence, don't use three.
```

**ANT（內部）版更精確**：
```
- Brief statement before your first tool call
- Short update only at key moments (load-bearing finds, direction changes, progress)
- Flowing prose, avoid fragments/em dashes/notation
- Avoid semantic backtracking — linear meaning building
- Only tables for short enumerable facts or quantitative data
```

### 5. 過度工程防護——最容易被忽略的規則

```
# Avoid over-engineering

- Don't add features, refactor code, or make "improvements" beyond what was asked
- A bug fix doesn't need surrounding code cleaned up
- A simple feature doesn't need extra configurability
- Don't add docstrings, comments, or type annotations to code you didn't change
- Don't add error handling for scenarios that can't happen
- Don't create helpers for one-time operations
- Three similar lines of code is better than a premature abstraction
```

**這是對 LLM 最大弱點的精確對策**：LLM 天生傾向「多做一點」，這些規則明確阻止了每一種常見的過度工程行為。

### 6. 安全邊界——Cyber Risk 指令

```
IMPORTANT: Assist with authorized security testing, defensive security,
CTF challenges, and educational contexts.

Refuse requests for:
- Destructive techniques
- DoS attacks
- Mass targeting
- Supply chain compromise
- Detection evasion for malicious purposes

Dual-use security tools require clear authorization context:
pentesting engagements, CTF competitions, security research, or defensive use cases.
```

**設計要點**：不是一刀切禁止安全相關操作，而是**要求上下文**。

### 7. 動態 System Prompt 組裝

```typescript
async function getSystemPrompt(tools, model, dirs, mcpClients): Promise<string[]> {
  // 1. Simple mode 快速路徑
  if (CLAUDE_CODE_SIMPLE) return [`CWD: ${getCwd()}\nDate: ${date}`]

  // 2. 靜態區（可快取）
  const staticParts = [
    getSimpleIntroSection(),        // 角色定義 + 安全指令
    getSimpleSystemSection(),       // 系統行為規則
    getSimpleDoingTasksSection(),   // 任務執行規則
    actionsSection,                 // 風險控制框架
    usingToolsSection,              // 工具使用約束
    toneAndStyleSection,            // 風格規範
    outputEfficiencySection,        // 輸出效率規則
  ]

  // 3. 動態邊界
  staticParts.push(SYSTEM_PROMPT_DYNAMIC_BOUNDARY)

  // 4. 動態區（每次重組）
  const dynamicParts = [
    agentToolSection,               // Agent 工具指導
    skillToolSection,               // Skill 工具指導
    memoryPrompt,                   // MEMORY.md 內容
    environmentSection,             // 運行環境資訊
    mcpInstructions,                // MCP 伺服器指令
    outputStyleSection,             // 輸出樣式
  ]

  return [...staticParts, ...dynamicParts]
}
```

## 可直接套用的模式

### 模式 1：工具偏好階層
```
當有專用工具時，禁止使用通用工具。
每條禁令配一個替代方案。
```

### 模式 2：可逆性決策矩陣
```
把操作分為：可逆+本地、不可逆+本地、可逆+共享、不可逆+共享。
只有「可逆+本地」可以自動執行，其他都要確認。
```

### 模式 3：過度行為抑制
```
列出 AI 最常犯的「多做」行為，逐一禁止：
- 不要加文件（沒被要求時）
- 不要重構（沒被要求時）
- 不要抽象化（只用一次的邏輯）
- 不要加錯誤處理（不可能發生的場景）
```

### 模式 4：輸出結構約束
```
「先結論後解釋」不是建議，是規格：
- 一句話能說完就不用三句
- 只在里程碑時報進度
- 只有需要用戶決策時才說話
```

### 模式 5：安全的上下文判斷
```
不是禁止危險操作，而是要求上下文：
「需要明確的授權上下文：滲透測試、CTF、安全研究、防禦用途」
```

## 數值錨定（ANT 內部版）

```
- Keep text between tool calls to ≤25 words
- Keep final responses to ≤100 words unless task requires more detail
```

**啟示**：用具體數字約束，比「簡短」有效 10 倍。

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/constants/prompts.ts` | System prompt 主檔 (54KB) |
| `src/constants/cyberRiskInstruction.ts` | 安全邊界指令 |
| `src/stubs/macros.ts` | Build-time 宏定義 |
