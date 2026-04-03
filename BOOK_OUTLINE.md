# 教科書大綱：從 Claude Code 源碼學 Agent 設計

> 從粗到細，五個顆粒度層次

## Part I：概念層（給產品經理和技術主管看的）
- Ch01: 什麼是 Coding Agent？從 Copilot 到 Claude Code 的演進
- Ch02: Agent 設計的核心挑戰：安全、記憶、協作、成本

## Part II：架構層（給架構師看的）
- Ch03: 整體架構藍圖——512K 行怎麼組織
- Ch04: 請求生命週期——一個問題從輸入到回答的完整旅程
- Ch05: 狀態管理——Reactive Store 與 React Ink TUI

## Part III：系統設計層（給資深工程師看的）
- Ch06: System Prompt 工程化——不是寫 prompt，是寫行為規格書
- Ch07: Tool 系統——43 個工具的統一接口與並發控制
- Ch08: Permission 模型——如何在自主性與安全之間取得平衡
- Ch09: Hook 系統——27 種事件驅動的可擴展架構
- Ch10: Plugin 三層架構——Intent → Materialization → Active

## Part IV：進階設計模式（給做 Agent 產品的人看的）
- Ch11: 多 Agent 協作（Swarm）——Coordinator、Mailbox、原子認領
- Ch12: 三層上下文壓縮——MicroCompact → AutoCompact → Full Compact
- Ch13: 記憶系統——即時抽取 + AutoDream 定期整理
- Ch14: MCP 整合——開放協議如何統一工具生態

## Part V：實戰層（給想自己做的人看的）
- Ch15: 從零開始：用這些模式建造你自己的 Agent
- Ch16: 反模式集——Claude Code 源碼裡學到的「不要這樣做」
