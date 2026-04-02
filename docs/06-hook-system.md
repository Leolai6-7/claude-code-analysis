# 第六章：Hook 系統

> **源碼**: `src/utils/hooks.ts` (5,022 行), `src/utils/hooks/`, `src/types/hooks.ts`

## 27 種 Hook 事件

### 工具執行 Hooks
| 事件 | 觸發時機 |
|------|---------|
| `PreToolUse` | 工具執行前（可阻止/修改輸入） |
| `PostToolUse` | 工具成功執行後（可修改輸出） |
| `PostToolUseFailure` | 工具執行失敗後 |

### 權限 Hooks
| 事件 | 觸發時機 |
|------|---------|
| `PermissionRequest` | 權限對話框顯示時 |
| `PermissionDenied` | Auto mode classifier 拒絕時 |

### 用戶輸入
| `UserPromptSubmit` | 用戶提交 prompt 時 |

### Session 生命週期
| 事件 | 觸發時機 |
|------|---------|
| `SessionStart` | 新 session 啟動 |
| `SessionEnd` | Session 結束（timeout: 1,500ms） |
| `Stop` | Model 回應完畢前 |
| `StopFailure` | 因 API 錯誤結束時 |

### 子 Agent
| 事件 | 觸發時機 |
|------|---------|
| `SubagentStart` | 子 agent 啟動 |
| `SubagentStop` | 子 agent 結束 |

### 壓縮
| 事件 | 觸發時機 |
|------|---------|
| `PreCompact` | 壓縮前 |
| `PostCompact` | 壓縮後 |

### MCP Elicitation
| 事件 | 觸發時機 |
|------|---------|
| `Elicitation` | MCP 伺服器請求用戶輸入 |
| `ElicitationResult` | 用戶回應 MCP elicitation |

### 設定與檔案
| 事件 | 觸發時機 |
|------|---------|
| `Setup` | Repo 初始化/維護 |
| `ConfigChange` | 設定檔案變更 |
| `InstructionsLoaded` | CLAUDE.md 或 rule 載入 |
| `CwdChanged` | 工作目錄變更 |
| `FileChanged` | 監控的檔案變更 |

### 團隊協作
| 事件 | 觸發時機 |
|------|---------|
| `TeammateIdle` | Teammate 即將閒置 |
| `TaskCreated` | 任務建立 |
| `TaskCompleted` | 任務完成 |

### Worktree
| 事件 | 觸發時機 |
|------|---------|
| `WorktreeCreate` | 建立隔離 worktree |
| `WorktreeRemove` | 移除 worktree |

## 5 種 Hook 執行類型

| 類型 | 機制 | 成本 |
|------|------|------|
| `command` | 執行 shell 腳本 | 低（subprocess） |
| `prompt` | 單次 LLM 查詢（Haiku） | 中 |
| `agent` | 多輪 LLM agent（≤50 輪） | 高 |
| `http` | HTTP 請求（帶 SSRF 防護） | 低 |
| `callback` | 內聯 TypeScript（內部用） | 極低 |

## Hook 來源優先順序

```
policySettings    — 企業管理（最高）
flagSettings      — Feature flag
localSettings     — .claude/settings.local.json
projectSettings   — .claude/settings.json
userSettings      — ~/.claude/settings.json
pluginHook        — Plugin 註冊
sessionHook       — Session 臨時 hook
builtinHook       — ANT 內部 hook
```

## Hook 執行流程

```typescript
// hooks.ts:1952 — executeHooks() async generator
async function* executeHooks(event, options) {
  // 1. 信任檢查 — 未信任的 workspace 阻止所有 hook
  checkHasTrustDialogAccepted()

  // 2. 匹配 — 按事件 + matcher + 工具名稱過濾
  const matching = getMatchingHooks(event, toolName)

  // 3. 進度通知 — UI 顯示 hook 執行中
  yield progressMessage

  // 4. 並行執行 — 每個 hook 有獨立 timeout
  const results = await Promise.all(
    matching.map(hook => executeWithTimeout(hook, signal))
  )

  // 5. 結果聚合 — 合併為 AggregatedHookResult
  yield aggregatedResult
}
```

### Timeout 設定

| 場景 | 默認值 |
|------|--------|
| 一般 hook | 10 分鐘 |
| SessionEnd hook | 1,500ms |
| 自訂 | hook config 的 `timeout` 欄位（秒） |

## Hook 輸出 Schema

```typescript
SyncHookJSONOutput = {
  continue?: boolean           // 是否繼續後續 hook
  suppressOutput?: boolean     // 抑制輸出
  stopReason?: string          // 停止原因
  decision?: 'approve' | 'block'  // 批准/阻止

  hookSpecificOutput?: {
    // PreToolUse: permissionDecision, updatedInput, additionalContext
    // UserPromptSubmit: additionalContext
    // PostToolUse: updatedMCPToolOutput, additionalContext
    // SessionStart: watchPaths, initialUserMessage
    // Elicitation: action, content
    // PermissionRequest: decision (allow/deny)
  }
}

// 非同步 hook
AsyncHookJSONOutput = {
  async: true
  asyncTimeout?: number  // 毫秒
}
```

### Exit Code 語意

| Exit Code | 行為 |
|-----------|------|
| 0 | 成功 — transcript 模式顯示輸出 |
| 2 | 阻塞錯誤 — stderr 立即顯示給 model |
| 其他 | 非阻塞 — stderr 只顯示給用戶 |

## 非同步 Hook Registry

**源碼**: `src/utils/hooks/AsyncHookRegistry.ts`

```
registerPendingAsyncHook()  → 註冊背景 hook
    ↓
checkForAsyncHookResponses() → 輪詢完成狀態
    ↓
finalizePendingAsyncHooks() → Session 結束時清理
```

默認 async timeout: 15,000ms

## Ralph Loop 如何利用 Hook

Ralph Loop 利用 `Stop` hook 實現迭代：

```json
// hooks.json
{
  "Stop": [{
    "type": "command",
    "command": "bash ${CLAUDE_PLUGIN_ROOT}/hooks/stop-hook.sh"
  }]
}
```

當 model 嘗試結束回應時，Stop hook 攔截，重新注入相同的 prompt，形成自我參照的迭代循環。

## 源碼參考

| 檔案 | 行數 | 角色 |
|------|------|------|
| `src/utils/hooks.ts` | 5,022 | 主編排、executeHooks() generator |
| `src/types/hooks.ts` | 291 | 型別定義、Zod schemas |
| `src/utils/hooks/AsyncHookRegistry.ts` | 310 | 非同步 hook 管理 |
| `src/utils/hooks/hookEvents.ts` | 193 | 事件發射 |
| `src/utils/hooks/hooksConfigManager.ts` | 401 | Hook 事件 metadata |
| `src/utils/hooks/hooksSettings.ts` | 150+ | Hook 聚合、去重 |
| `src/utils/hooks/sessionHooks.ts` | 150+ | Session 臨時 hook |
| `src/utils/hooks/execPromptHook.ts` | 100+ | LLM prompt hook 執行 |
| `src/utils/hooks/execAgentHook.ts` | 340 | Multi-turn agent hook |
| `src/utils/hooks/execHttpHook.ts` | 100+ | HTTP hook + SSRF 防護 |
