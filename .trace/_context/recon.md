# Stage 1 偵察報告

## 專案概要

**Paperclip** 是一個以 Node.js + React 開發的開源 Control Plane，用於將 AI 代理程式團隊作為一家「公司」來運營。
正如其口號「If OpenClaw is an _employee_, Paperclip is the _company_」所示，Paperclip 是一個用於雇用代理程式、管理組織架構、目標、budget 和 governance 的協調基礎設施。

- **URL**: https://paperclip.ing/
- **GitHub**: https://github.com/paperclipai/paperclip
- **授權條款**: MIT
- **首次發布**: 2026 年 3 月 4 日（GitHub stars 三週內突破 30,000）

---

## 技術堆疊

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Node.js | ≥20 | 伺服器執行環境 |
| Package Manager | pnpm | 9.15.4 | monorepo 管理 |
| Backend Framework | Express.js | 4.x | HTTP API 伺服器 |
| Frontend Framework | React | 18.x | 儀表板 UI |
| Build Tool (UI) | Vite | - | 前端建置 |
| ORM | Drizzle ORM | ^0.38.4 | DB schema 與查詢 |
| Database | PostgreSQL (embedded-postgres) | 18.1.0-beta.16 | 資料持久化 |
| Auth | Better Auth | - | 驗證與 session 管理 |
| Language | TypeScript | ^5.7.3 | 所有套件共用 |
| Test Framework | Vitest | ^3.0.5 | 單元與整合測試 |
| E2E Test | Playwright | ^1.58.2 | 瀏覽器測試 |
| HTTP Client (UI) | TanStack Query (React Query) | - | 資料擷取 |
| Router (UI) | React Router | - | SPA 路由 |
| Container | Docker | - | 正式環境部署 |
| CI/CD | GitHub Actions | - | PR 檢查與發布 |
| Telemetry | 自製（匿名） | - | 使用狀況收集 |

---

## Monorepo 結構

```
paperclip/
├── server/               # Express.js API 伺服器 (@paperclipai/server)
│   └── src/
│       ├── index.ts      # 進入點（DB 初始化・HTTP 伺服器啟動）
│       ├── app.ts        # Express 應用程式（路由註冊）
│       ├── routes/       # API 端點群
│       ├── services/     # 商業邏輯（heartbeat、issues、budgets 等）
│       ├── adapters/     # 代理程式執行 adapter
│       ├── auth/         # 驗證（Better Auth）
│       ├── middleware/   # HTTP middleware
│       ├── storage/      # 檔案儲存抽象層
│       └── realtime/     # WebSocket（即時事件）
├── ui/                   # React 儀表板 (@paperclipai/ui)
│   └── src/
│       ├── main.tsx      # React 進入點
│       ├── App.tsx       # 根元件
│       ├── pages/        # 頁面元件群
│       ├── components/   # 共用 UI 元件
│       ├── api/          # API client
│       ├── context/      # React Context provider 群
│       └── plugins/      # plugin UI bridge
├── cli/                  # CLI 工具 (@paperclipai/cli)
│   └── src/
│       ├── index.ts      # CLI 進入點
│       └── commands/     # CLI 子命令群
├── packages/
│   ├── db/               # DB schema 與 migration (@paperclipai/db)
│   │   └── src/
│   │       ├── schema/   # Drizzle 資料表定義（約 70 張資料表）
│   │       └── migrations/
│   ├── shared/           # 共用型別與常數 (@paperclipai/shared)
│   ├── adapter-utils/    # adapter 共用工具
│   ├── mcp-server/       # MCP server 實作
│   ├── adapters/         # 代理程式 adapter 群
│   │   ├── claude-local/   # Claude Code 本機執行
│   │   ├── codex-local/    # OpenAI Codex 本機執行
│   │   ├── cursor-local/   # Cursor 本機執行
│   │   ├── gemini-local/   # Gemini 本機執行
│   │   ├── openclaw-gateway/ # OpenClaw Webhook gateway
│   │   ├── opencode-local/ # OpenCode 本機執行
│   │   ├── pi-local/       # Pi 本機執行
│   │   └── acpx-local/     # ACPX 本機執行
│   └── plugins/
│       ├── sdk/            # plugin 開發 SDK (@paperclipai/plugin-sdk)
│       └── examples/       # 範例 plugin
├── doc/                  # 內部設計文件
├── docs/                 # Mintlify 公開文件
├── evals/                # LLM 評估腳本 (promptfoo)
├── skills/               # 代理程式 skill 定義
├── scripts/              # 建置與發布腳本
├── tests/                # E2E 與發布 smoke 測試
├── docker/               # Docker/Compose 設定
└── releases/             # 發布管理
```

---

## 架構模式

- **Monorepo**（pnpm workspaces）
- **Server-side Rendered + SPA Hybrid**：Express 提供靜態檔案，以 React SPA 方式運作
- **Embedded PostgreSQL**：本機開發零設定，正式環境可切換為外部 Postgres
- **Plugin Architecture**：透過 out-of-process Worker 的可擴展 plugin 系統
- **Adapter Pattern**：各代理程式 runtime 透過 adapter 統一為一致介面

---

## 現有文件清單

### `doc/` 內部設計文件

| 檔案 | 內容 |
|------|------|
| `doc/SPEC.md` | 技術規格（Company/Agent/Org/Heartbeat/Budget 模型） |
| `doc/SPEC-implementation.md` | V1 實作契約 |
| `doc/PRODUCT.md` | 產品定義與設計原則 |
| `doc/DEVELOPING.md` | 開發指南（DB/測試/Docker/worktree 詳細說明） |
| `doc/DEPLOYMENT-MODES.md` | 部署模式定義 |
| `doc/TASKS.md` | 任務管理資料模型 |
| `doc/TASKS-mcp.md` | MCP 任務規格 |
| `doc/GOAL.md` | goal 管理規格 |
| `doc/CLI.md` | CLI 命令參考 |
| `doc/DATABASE.md` | DB 設計文件 |
| `doc/DOCKER.md` | Docker 詳細操作說明 |
| `doc/RELEASING.md` | 發布流程 |
| `doc/plugins/PLUGIN_SPEC.md` | plugin 系統規格 |
| `doc/execution-semantics.md` | 執行語義 |
| `doc/memory-landscape.md` | 記憶體/context 設計 |

### `docs/` 公開文件（Mintlify）

- `docs/start/` — 快速入門・架構說明
- `docs/adapters/` — adapter 指南
- `docs/api/` — API 參考
- `docs/companies/` — 公司管理指南
- `docs/deploy/` — 部署指南
- `docs/guides/` — 開發者指南

---

## 現有文件與實際程式碼的對照

### 吻合之處
- `doc/SPEC.md` 所描述的實體（Company、Agent、Issue、Heartbeat Run、Budget Policy）均已在 `packages/db/src/schema/` 中以資料表形式實作
- Adapter 類型（claude_local、codex_local、openclaw_gateway 等）與 `packages/adapters/` 一致

### 落差與注意事項
- `doc/SPEC.md` 中記載的「billing codes」與「request depth」已在 schema 中實作（`issues.billing_code`、`issues.request_depth`），但在 UI 上的顯示與編輯 ⚠️ 尚未確認
- `doc/SPEC.md` 的「Budget Delegation（cascading）」已在 `server/src/services/budgets.ts` 中以月結視窗・終身視窗的 policy 方式實作，但完整的 cascading 委派 ⚠️ 可能部分尚未實作
- `ROADMAP.md` 中的「Memory / Knowledge」「Enforced Outcomes」「CEO Chat」「Cloud deployments」⚪ 尚未實作

---

## 設定檔

### `.env.example` 的主要環境變數

```
DATABASE_URL=postgres://...        # 未設定時使用 embedded postgres
PORT=3100
SERVE_UI=false                     # 正式環境設為 true
BETTER_AUTH_SECRET=...
PAPERCLIP_TELEMETRY_DISABLED=1     # 停用 telemetry
```

### 部署模式

| 模式 | 說明 |
|------|------|
| `local_trusted` | 單一使用者・無需登入（預設） |
| `authenticated` | 必須驗證。`private`（LAN/Tailscale）或 `public` 公開存取 |

---

## 檔案規模

- TypeScript 檔案：1,149 個
- TSX 檔案：304 個
- 總檔案數（不含 node_modules）：2,007 個
- DB schema 資料表：約 70 張
- API 路由檔案：40 個以上
- Service 檔案：80 個以上

> **Stage 1.5 評估**：2,007 個檔案，超過 500。作為 Full trace 繼續進行。
> 本專案為 API 伺服器 + 前端 + CLI + plugin 系統的複合型專案，因此適用所有 trace 路徑。
