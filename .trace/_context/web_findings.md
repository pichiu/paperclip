# 網路搜尋發現摘要

## 搜尋結果總覽

### 1. 專案官方資訊

| 資源 | URL | 內容 |
|------|-----|------|
| 官方網站 | https://paperclip.ing/ | 登陸頁面 |
| 官方文件 | https://docs.paperclip.ing/ | 基於 Mintlify 的公開文件 |
| GitHub | https://github.com/paperclipai/paperclip | 原始碼 |
| 架構文件 | https://docs.paperclip.ing/start/architecture | 官方架構說明 |
| Discord | https://discord.gg/m4HZY7xNG3 | 社群 |
| Twitter/X | https://x.com/papercliping | 官方帳號 |

### 2. 文章與解說

| 文章 | URL | 重點摘要 |
|------|-----|---------|
| Towards AI 解說 | https://pub.towardsai.net/paperclip-the-open-source-operating-system-for-zero-human-companies-2c16f3f22182 | 定位為「多代理程式公司的開源作業系統」 |
| MindStudio heartbeat 解說 | https://www.mindstudio.ai/blog/what-is-heartbeat-pattern-paperclip-ai-agents-24-7 | heartbeat pattern 的詳細說明 |
| MindStudio AI 團隊建置指南 | https://www.mindstudio.ai/blog/how-to-build-multi-agent-company-paperclip-claude-code | Claude Code + Paperclip 實踐指南 |
| DEV Community Deep Dive | https://dev.to/truongpx396/paperclip-deep-dive-a-build-guide-for-an-ai-company-control-plane-dda | 程式碼層級的深度解析 |
| Zeabur 部署指南 | https://zeabur.com/blogs/deploy-paperclip-ai-agent-orchestration | Docker 部署步驟 |
| Flowtivity 解說 | https://flowtivity.ai/blog/zero-human-company-paperclip-ai-agent-orchestration/ | 商業用途解說 |
| Medium 解說 | https://medium.com/@creativeaininja/paperclip-the-open-source-platform-turning-ai-agents-into-an-actual-company-7348015c5bf7 | 2026 年 3 月的解說文章 |
| jimmysong.io | https://jimmysong.io/ai/paperclip/ | 中文解說 |

### 3. 重點發現

#### 創立與成長
- 開發者：匿名（@dotta）
- 公開日期：2026 年 3 月 4 日
- 三週內 GitHub Stars 突破 30,000（2026 年 Q1 成長最快的 OSS AI 儲存庫之一）

#### heartbeat pattern 說明（MindStudio）
> "The heartbeat pattern is a scheduling and context-management mechanism that wakes an AI agent at regular intervals, injects fresh context into its working memory, and allows it to operate without any human trigger."

這是 Paperclip 的核心設計模式。以「排程驅動」方式喚醒代理程式，並透過維持 session 狀態來實現 24/7 自主運作。

#### 架構文件（官方）
https://docs.paperclip.ing/start/architecture

Paperclip 的官方架構文件說明了各主要元件之間的關係：
- Control Plane（Paperclip Server）
- Agent Adapters（Claude Code、Codex、OpenClaw 等）
- Board Dashboard（React UI）
- Storage（Embedded Postgres 或外部資料庫）

#### Awesome Paperclip（社群 plugin 集合）
- https://github.com/gsxdsm/awesome-paperclip

社群整理的 plugin 和工具精選清單，顯示 plugin 生態系統的存在。

#### GitHub Releases
- https://github.com/paperclipai/paperclip/releases/tag/v2026.318.0
- 發布版本格式：`v{year}.{dayOfYear}.{patch}`

### 4. 相關與競爭專案

| 專案 | 關係 |
|------|------|
| OpenClaw | Paperclip 使用的代理程式之一（OpenClaw ≒ 員工，Paperclip ≒ 公司） |
| Claude Code (Anthropic) | Paperclip 使用的主要代理程式之一 |
| OpenAI Codex | 支援的代理程式 |
| Cursor | 支援的代理程式（cursor_local adapter） |
| OpenCode | 支援的代理程式 |

### 5. 社群與支援

- Discord 的 #dev 頻道是主要的開發者討論場所
- 貢獻需提供 PR 範本 + Greptile 評分 5/5
- 建議透過 plugin 系統進行擴充（直接對核心提交 PR 需謹慎）

### 6. 注意事項與未解決的疑問

- `paperclipai.net` 另一個網域存在，但與官方的關係 ⚠️ 尚未確認
- 存在 `agencyenterprise/paperclip-ai` 的 fork
- ClipMart（一鍵發布公司範本的市集）目前為 COMING SOON 狀態
