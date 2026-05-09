# Paperclip — 專案總覽

## 一句話摘要

**Paperclip** 是一個開源的 Control Plane，用於將 AI agent（Claude Code、Codex、OpenClaw 等）組織化並以「公司」形式運營。以「agent 是員工、Paperclip 是公司」為核心概念，提供組織架構、目標、budget、governance 與任務管理，讓多種異質 agent 協同運作。

---

## 技術堆疊總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Node.js | ≥20 | 伺服器執行環境 |
| Package Manager | pnpm | 9.15.4 | Monorepo 管理 |
| Backend Framework | Express.js | 4.x | HTTP API 伺服器 |
| Frontend Framework | React | 18.x | 儀表板 UI |
| Build Tool (UI) | Vite | - | 前端建置 |
| ORM | Drizzle ORM | ^0.38.4 | DB schema 與查詢 |
| Database | PostgreSQL (embedded) | 18.1.0-beta.16 | 資料持久化（零設定） |
| Auth | Better Auth | - | 認證與 session 管理 |
| Language | TypeScript | ^5.7.3 | 全套件共用 |
| Test Framework | Vitest | ^3.0.5 | 單元與整合測試 |
| E2E Test | Playwright | ^1.58.2 | 瀏覽器測試 |
| HTTP Client (UI) | TanStack Query | - | 資料擷取 |
| Router (UI) | React Router | - | SPA 路由 |
| Container | Docker | - | 正式部署 |
| CI/CD | GitHub Actions | - | PR 檢查與發布 |

---

## 常用指令速查

```bash
# 啟動開發環境（API + UI，watch 模式）
pnpm dev

# 啟動開發環境（不啟用 watch）
pnpm dev:once

# 僅啟動伺服器
pnpm dev:server

# 建置全部套件
pnpm build

# 型別檢查
pnpm typecheck

# 執行測試（僅 Vitest）
pnpm test

# 測試（watch 模式）
pnpm test:watch

# E2E 測試（Playwright）
pnpm test:e2e

# 產生 DB migration
pnpm db:generate

# 套用 DB migration
pnpm db:migrate

# CLI（本地開發）
pnpm paperclipai <command>

# 初始設定
npx paperclipai onboard --yes

# Storybook
pnpm storybook
```

---

## 文件地圖

| 文件 | 內容 |
|------|------|
| [INDEX.md](./INDEX.md) | 此文件 — 總覽與速查 |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 系統架構、元件與設計決策 |
| [DATA_MODEL.md](./DATA_MODEL.md) | 資料模型、ER 圖與 schema 詳細說明 |
| [API_SURFACE.md](./API_SURFACE.md) | REST API 與 CLI 指令參考 |
| [DEV_GUIDE.md](./DEV_GUIDE.md) | 開發者上手指南、環境設定與測試 |
| [CODEBASE_MAP.md](./CODEBASE_MAP.md) | 程式碼地圖與「如何修改 X？」速查 |
| [DISCOVERY_LOG.md](./DISCOVERY_LOG.md) | 探索記錄、已知技術債與待解問題 |
| [TRACE_META.md](./TRACE_META.md) | Trace metadata（供增量更新使用） |

---

## 專案專用術語表

| 術語 | 定義 |
|------|------|
| **Company** | Paperclip 的最高層級物件。單一 instance 可運營多個 Company。 |
| **Agent** | Agent，即「員工」。對應 Claude Code / OpenClaw 等實際 runtime。 |
| **Board** | Board，即人類操作者。負責整個 Company 的 governance。 |
| **Heartbeat** | Heartbeat，即 agent 的定期觸發週期。可由排程或事件觸發。 |
| **Issue** | 任務、工單。連結至公司目標的工作單位。 |
| **Heartbeat Run** | 一次 heartbeat 執行的記錄（`heartbeat_runs` 資料表）。 |
| **Adapter** | 與 agent runtime 之間的連接層（例如：`claude_local`、`openclaw_gateway`）。 |
| **Skill** | 注入至 agent 的 Markdown 格式知識與操作說明。 |
| **Routine** | 已排程的定期任務定義（cron / webhook / API 觸發）。 |
| **Worktree** | 開發用的獨立 git worktree 實例，擁有專屬 DB 與連接埠。 |
| **Execution Workspace** | Issue 執行時所使用的 git worktree 或專案工作空間。 |
| **Activity Log** | 所有變更操作的稽核日誌（`activity_log` 資料表）。 |
| **Goal** | 目標。階層為 Mission → Initiative → Project → Issue。 |
| **Clipmart** | Coming Soon：公司範本的 marketplace。 |
| **Board Claim** | 在 `local_trusted` 模式下的 Board 所有權申請流程。 |
| **Wake Payload** | Agent 啟動時傳入的上下文資料包（公司、目標、任務、skill 等）。 |
| **Secret Ref** | 指向 secret 值的參照（取代內嵌純文字）。 |
| **Invitation** | 透過 token 邀請 agent 加入系統的流程。 |
| **Plugin** | 以 out-of-process Worker 形式運作的擴充功能（npm 套件）。 |
