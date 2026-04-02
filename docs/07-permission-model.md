# 第七章：Permission 模型

> **源碼**: `src/utils/permissions/`, `src/components/permissions/`

## Permission Mode

| 模式 | 行為 |
|------|------|
| `default` | 受限操作時提示用戶 |
| `plan` | 暫停執行供用戶審查 |
| `acceptEdits` | 自動接受檔案編輯 |
| `bypassPermissions` | 跳過所有權限檢查（危險） |
| `dontAsk` | 自動拒絕（不提示） |
| `auto` | 使用 transcript classifier 自動判斷（ANT only） |

## 規則行為

每條規則有三種行為之一：

| 行為 | 結果 |
|------|------|
| `allow` | 自動允許 |
| `deny` | 自動拒絕 |
| `ask` | 強制提示用戶 |

### 規則格式

```
ToolName              → 工具級別（匹配所有該工具呼叫）
ToolName(content)     → 內容級別
Bash(git add)         → 精確匹配
Bash(npm:*)           → 前綴匹配
Bash(python *)        → 萬用字元匹配
mcp__server1          → 匹配整個 MCP 伺服器的所有工具
```

## 設定階層

| 優先順序 | 來源 | 可編輯 |
|---------|------|--------|
| 1 | `policySettings`（企業管理） | ❌ |
| 2 | `flagSettings`（Feature flag） | ❌ |
| 3 | `localSettings`（.claude/settings.local.json） | ✅ |
| 4 | `projectSettings`（.claude/settings.json） | ✅ |
| 5 | `userSettings`（~/.claude/settings.json） | ✅ |
| 6 | `cliArg`（CLI 參數） | ❌ |
| 7 | `command`（Session 命令） | ❌ |
| 8 | `session`（Session 臨時） | ❌ |

### Managed Rules Override

當 `allowManagedPermissionRulesOnly` 啟用時：
- 只有 policy rules 生效
- 用戶無法新增/修改規則
- "Always allow" 選項在 UI 隱藏

## 權限檢查流程

```
Tool 呼叫
    │
    ├── 1. Tool-specific checkPermissions()
    │   └── 回傳 allow/deny/ask + 可選 updatedInput
    │
    ├── 2. Rule-based matching
    │   ├── getDenyRules() → 是否被 deny？
    │   ├── getAllowRules() → 是否被 allow？
    │   └── getAskRules() → 是否需要 ask？
    │
    ├── 3. Hook-based override
    │   └── PreToolUse hook 可強制 allow/deny
    │
    ├── 4. Permission mode evaluation
    │   ├── bypass → 自動允許
    │   ├── auto → classifier 判斷
    │   └── default → 顯示提示
    │
    └── 5. Interactive prompt（如需要）
        ├── 用戶選擇：一次/總是/拒絕
        └── 更新 session 或持久設定
```

## 危險模式防護

**源碼**: `src/utils/permissions/dangerousPatterns.ts`

進入 auto mode 時，自動移除危險的 allow 規則：

**Bash 危險模式**：
- `Bash` 或 `Bash(*)`（工具級別全允許）
- `*`（萬用字元全允許）
- 腳本直譯器：`python:*`, `node:*`, `ruby:*` 等
- 變體：`python*`, `python *`, `python -*`

**PowerShell 危險模式**：
- 同 Bash + PS 特定：`pwsh`, `powershell`, `iex`, `invoke-expression`

## 拒絕追蹤（Denial Tracking）

**源碼**: `src/utils/permissions/denialTracking.ts`

防止 classifier 陷入拒絕循環：

```typescript
maxConsecutive = 3   // 連續拒絕上限
maxTotal = 20        // Session 總拒絕上限

// 超過後回退到提示模式
shouldFallbackToPrompting(state) → true
```

## Permission Decision 原因

| 原因類型 | 觸發場景 |
|---------|---------|
| `rule` | 權限規則匹配 |
| `mode` | 當前 permission mode |
| `hook` | Hook 拒絕 |
| `classifier` | Auto mode classifier |
| `workingDir` | 工作目錄限制 |
| `safetyCheck` | 安全限制（如 .claude/ 路徑） |
| `sandboxOverride` | 沙箱外執行 |
| `asyncAgent` | Headless agent 無提示拒絕 |

## Bypass Killswitch

**源碼**: `src/utils/permissions/bypassPermissionsKillswitch.ts`

企業可以遠端禁用 bypass mode，即使用戶設定了也不會生效。

## 源碼參考

| 檔案 | 角色 |
|------|------|
| `src/types/permissions.ts` | 型別定義 |
| `src/utils/permissions/permissions.ts` | 核心權限邏輯 |
| `src/utils/permissions/permissionsLoader.ts` | 設定載入 |
| `src/utils/permissions/permissionRuleParser.ts` | 規則解析 |
| `src/utils/permissions/shellRuleMatching.ts` | Shell 規則匹配 |
| `src/utils/permissions/PermissionMode.ts` | 模式設定 |
| `src/utils/permissions/PermissionUpdate.ts` | 規則更新 |
| `src/utils/permissions/dangerousPatterns.ts` | 危險模式偵測 |
| `src/utils/permissions/denialTracking.ts` | 拒絕追蹤 |
| `src/utils/permissions/bypassPermissionsKillswitch.ts` | Bypass killswitch |
| `src/components/permissions/PermissionPrompt.tsx` | 提示 UI |
| `src/components/permissions/PermissionRequest.tsx` | 請求協調器 |
