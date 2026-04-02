# 第三章：請求/回應生命週期

## 完整流程圖

```
[用戶輸入]
    │
    ├─── UserPromptSubmit Hook ──→ 可修改/阻止輸入
    │
    ▼
[System Prompt 組裝]
    │   constants/prompts.js
    │   - Base prompt + tool descriptions
    │   - CLAUDE.md files
    │   - Plugin rules + skill descriptions
    │   - MCP instructions + memory
    │
    ▼
[Messages 組裝]
    │   utils/messages/
    │   - 歷史訊息（可能已壓縮）
    │   - 新用戶訊息
    │   - System reminders
    │
    ▼
[Anthropic Messages API 呼叫]
    │   services/api/
    │   - Streaming response
    │   - model selection (Opus/Sonnet/Haiku)
    │
    ▼
[Response Streaming 解析]
    │
    ├── Text Block ──→ 直接渲染到終端
    │
    ├── Thinking Block ──→ Extended thinking 顯示
    │
    └── Tool Use Block ──→ 進入工具執行流程
            │
            ▼
        [Tool Lookup]
            │   在 tools[] 中找到對應工具
            │
            ▼
        [Input Validation]
            │   Zod schema 驗證
            │   Tool-specific validateInput()
            │
            ▼
        [PreToolUse Hooks]
            │   可阻止、修改輸入、自動批准
            │
            ▼
        [Permission Check]
            │   1. Tool-specific checkPermissions()
            │   2. Rule-based matching (allow/deny/ask)
            │   3. Permission mode evaluation
            │   4. Interactive prompt (if needed)
            │
            ▼
        [Tool Execution]
            │   tool.call(input, context, ...)
            │   - Streaming progress
            │   - Abort signal 支援
            │
            ▼
        [PostToolUse Hooks]
            │   可修改輸出、阻止結果
            │
            ▼
        [Result Processing]
            │   - 大結果持久化到磁碟
            │   - 內容截斷
            │   - MCP metadata 處理
            │
            ▼
        [Result 加入 Messages]
            │
            ▼
        [下一輪 API 呼叫]
            │   （如果 model 還需要使用更多工具）
            │
            ▼
[Stop Hook]
    │   model 結束回應前觸發
    │   - 可注入額外提示讓 model 繼續
    │   - Ralph Loop 利用此 hook 實現迭代
    │
    ▼
[Session Memory 抽取]（背景）
    │   如果達到 token 閾值
    │   Fork 子 agent 更新 session-memory.md
    │
    ▼
[等待下一個用戶輸入]
```

## 工具並發控制

```typescript
// src/services/tools/toolOrchestration.ts

// 當 model 一次回傳多個 tool_use block:
partitionToolCalls(toolCalls)
  ├── Concurrent-safe tools → runToolsConcurrently()
  │   (Glob, Grep, FileRead, WebSearch, TaskGet...)
  │
  └── Non-concurrent tools → runToolsSerially()
      (Bash, FileWrite, FileEdit, Agent...)
```

**並發安全判斷**：每個工具實作 `isConcurrencySafe()` 方法。讀取類工具可並行，寫入類工具必須串行。

## 對話壓縮 (Compact)

當 context window 接近上限時觸發自動壓縮：

```
[Auto-Compact 觸發]
    │
    ├── PreCompact Hook
    │
    ▼
[壓縮服務]
    │   services/compact/
    │   - 將歷史訊息摘要為更短版本
    │   - 保留關鍵資訊（檔案路徑、決策、錯誤）
    │   - 丟棄冗餘的工具輸出
    │
    ├── PostCompact Hook
    │
    ▼
[壓縮後的 Messages]
    │   替換原始歷史
    │   下一輪 API 呼叫使用壓縮版
```

## Streaming 架構

```typescript
// API response 是一個 async iterable
for await (const event of apiStream) {
  switch (event.type) {
    case 'content_block_start':
      // 開始新的 text/tool_use block
    case 'content_block_delta':
      // 增量內容更新 → React re-render
    case 'content_block_stop':
      // Block 完成 → 觸發工具執行
    case 'message_stop':
      // 回應結束 → 觸發 Stop hook
  }
}
```

## 錯誤處理流程

```
[API Error]
    ├── Rate limit → 等待 + 重試
    ├── Token limit → 觸發 auto-compact → 重試
    ├── Network error → 重試 (指數退避)
    └── Auth error → 重新認證提示

[Tool Error]
    ├── Input validation failure → 回傳錯誤訊息給 model
    ├── Permission denied → 回傳 "permission denied" 給 model
    ├── Execution error → 回傳 stderr 給 model
    └── Timeout → 回傳 timeout 訊息給 model

[Hook Error]
    ├── Exit code 0 → 正常輸出
    ├── Exit code 2 → 阻塞錯誤，顯示給 model
    └── Other codes → 非阻塞錯誤，只顯示給用戶
```

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/services/tools/toolExecution.ts` | 工具執行主流程 (60KB) |
| `src/services/tools/toolOrchestration.ts` | 並發控制 |
| `src/services/tools/toolHooks.ts` | PreToolUse/PostToolUse hook 觸發 |
| `src/services/tools/StreamingToolExecutor.ts` | Streaming 工具執行 |
| `src/services/compact/` | 對話壓縮服務 |
| `src/services/api/` | API 通訊與 streaming |
| `src/utils/messages/` | 訊息組裝與處理 |
