# 從 Claude Code 源碼學 Agent 設計

> 基於 Claude Code v2.1.90（512,685 行 TypeScript）的逐行拆解
> 從概念到源碼，教你 Anthropic 如何設計生產級 AI Agent

---

## 這本書在講什麼？

Claude Code 是 Anthropic 官方的 CLI Agent——不是 demo，不是論文，是每天被數十萬開發者使用的生產系統。

這本書把它的 512K 行源碼拆開，提取出四個最值得學習的 Agent 設計模式，從概念到逐行源碼帶你走一遍。

## 目錄

### Part I：基礎

| 章節 | 內容 | 行數 |
|------|------|------|
| [Ch01: 什麼是 Coding Agent](book/ch01-what-is-coding-agent.md) | 三代演進、512K 行怎麼組織、Agent 設計的五大核心問題 | 345 |
| [Ch02: 一個問題的完整旅程](book/ch02-request-lifecycle.md) | 從用戶輸入到回答——入口點、API 通訊、工具執行、錯誤處理的完整生命週期 | 1,698 |

### Part II：四大設計模式（逐行級深度）

| 章節 | 核心問題 | 行數 |
|------|---------|------|
| [Ch03: System Prompt 工程化](book/ch03-system-prompt-engineering.md) | 如何把 prompt 從「寫得好」變成「行為規格書」——工具約束、風險控制、過度工程防護 | 1,602 |
| [Ch04: 多 Agent 協作架構](book/ch04-multi-agent-swarm.md) | 如何讓多個 Agent 安全並行——Coordinator、Permission Queue、原子認領、Team Memory | 2,225 |
| [Ch05: 三層上下文壓縮](book/ch05-context-compression.md) | 如何管理有限的 context window——MicroCompact（零成本）→ AutoCompact → Full Compact | 1,583 |
| [Ch06: AutoDream 記憶系統](book/ch06-autodream-memory.md) | 如何讓 AI 有長期記憶——即時抽取 + 定期整理的雙層架構 | 1,649 |

**總計：9,102 行教學文件**

## 四個可直接帶走的設計模式

### 1. System Prompt 工程化
傳統寫法：「盡量幫助使用者」。Anthropic 的寫法：工具禁令（Do NOT）、可逆性決策矩陣、過度工程抑制清單、數值錨定（≤25 字）。可以直接套用到任何 AI 產品。

### 2. 多 Agent 協作（Swarm）
不需要 Redis/RabbitMQ——用檔案鎖 + JSON 做 IPC、兩目錄佇列解耦請求/回應、CAS 原子認領解決競態、Leader 審批模式保持人類控制。

### 3. 三層上下文壓縮
MicroCompact 不呼叫 API 就能釋放空間（最聰明的優化）。AutoCompact 用斷路器防死循環。Full Compact 壓縮後精選重新注入 50K token 的檔案和 skill。

### 4. AutoDream 記憶整理
觸發條件從便宜到貴排列（stat → scan → lock）。四階段整理（Orient → Gather → Consolidate → Prune）。記憶分四類（user/feedback/project/reference）。MEMORY.md 200 行硬限制。

## 源碼來源

- **NPM 包**: `@anthropic-ai/claude-code@2.1.90`
- **反編譯源碼**: [Leolai6-7/claude-code-source-code](https://github.com/Leolai6-7/claude-code-source-code)

## 適合誰讀？

| 讀者 | 建議路徑 | 時間 |
|------|---------|------|
| 產品經理 / 技術主管 | Ch01 | 15 分鐘 |
| 想了解 Agent 架構的工程師 | Ch01 → Ch02 | 30 分鐘 |
| 做 Agent 產品的工程師 | Ch01 → Ch02 → 挑一章深入 | 1-2 小時 |
| 想完整學習的人 | 全部 6 章 | 半天 |

## 補充資料

`docs/` 目錄包含 13 個模組級分析文件，覆蓋 Tool 系統、Plugin 機制、Hook 系統、Permission 模型、MCP 整合等，可作為進一步參考。

## License

教學文件以 MIT License 發布。源碼分析基於 Anthropic 的公開 NPM 包。
