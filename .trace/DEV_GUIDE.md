# Paperclip 開發者上手指南

> 最後更新：2026-05-05
> 本文基於 `doc/DEVELOPING.md`、`CONTRIBUTING.md` 及 `.trace/_context/` 偵察結果整理而成。

---

## 目錄

1. [Prerequisites 與環境構建](#1-prerequisites-與環境構建)
2. [本地開發 Workflow](#2-本地開發-workflow)
3. [測試策略與執行方法](#3-測試策略與執行方法)
4. [Debugging 技巧與常見踩坑](#4-debugging-技巧與常見踩坑)
5. [Contribution Workflow](#5-contribution-workflow)
6. [依賴管理與更新策略](#6-依賴管理與更新策略)

---

## 1. Prerequisites 與環境構建

### 必要工具

| 工具 | 最低版本 | 安裝方式 |
|------|---------|---------|
| Node.js | 20+ | [nodejs.org](https://nodejs.org) 或 `nvm install 20` |
| pnpm | 9+ | `npm install -g pnpm@9` 或 `corepack enable && corepack prepare pnpm@9.15.4 --activate` |
| Git | 任意現代版本 | 系統套件管理器 |

> PostgreSQL **不需要**手動安裝。Paperclip 在本地開發時自動使用 embedded PostgreSQL。

### Step-by-Step 環境構建

```sh
# 1. Clone repo
git clone https://github.com/paperclipai/paperclip.git
cd paperclip

# 2. 安裝所有 workspace 依賴
pnpm install

# 3. 初次啟動（自動處理 DB 初始化與 migration）
pnpm dev
```

啟動後可看到：
- **API Server**: `http://localhost:3100`
- **UI (Dashboard)**: 由 API Server 以 dev middleware 模式提供，同一 origin

### `.env` 設定（可選）

本地開發通常**不需要** `.env`，但如需自訂：

```sh
cp .env.example .env
```

`.env.example` 的預設值：

```dotenv
DATABASE_URL=postgres://paperclip:paperclip@localhost:5432/paperclip
PORT=3100
SERVE_UI=false
BETTER_AUTH_SECRET=paperclip-dev-secret
```

> `DATABASE_URL` **未設定**時自動使用 embedded PostgreSQL（推薦本地開發）。
> 設定後才會連接外部 Postgres。

---

## 2. 本地開發 Workflow

### 啟動流程圖

```mermaid
flowchart TD
    A[pnpm install] --> B{.paperclip/.env 存在?}
    B -- 否（主 checkout） --> C[pnpm dev]
    B -- 是（git worktree） --> D[確認 worktree 已 init]
    D --> C
    C --> E{同一 instance 已在執行?}
    E -- 是 --> F[回報既有 process，冪等]
    E -- 否 --> G[啟動 embedded PostgreSQL]
    G --> H[自動套用 pending migrations]
    H --> I[Express API Server :3100]
    I --> J[Vite dev middleware 掛載]
    J --> K[API + UI 就緒]
    K --> L[瀏覽器開啟 http://localhost:3100]
```

### 常用開發指令

```sh
# 標準開發模式（watch + 自動重啟）
pnpm dev

# 只啟動一次（不 watch 檔案變更）
pnpm dev:once

# 查看當前 repo 的 dev runner 狀態
pnpm dev:list

# 停止 dev runner
pnpm dev:stop

# Storybook（UI 元件預覽，port 6006）
pnpm storybook
```

### 確認服務健康

```sh
# 基本健康檢查
curl http://localhost:3100/api/health
# 預期回應: {"status":"ok"}

# 列出所有 companies
curl http://localhost:3100/api/companies
# 預期回應: JSON array（初始為空陣列 []）
```

### 網路 Binding 模式

```sh
# 預設（loopback only，local_trusted 模式）
pnpm dev

# LAN / Tailscale private-network 模式（authenticated/private）
pnpm dev --bind lan

# Tailscale-only 模式
pnpm dev --bind tailnet
```

### 設定優先順序

```
環境變數（最高優先）
  ↓
PAPERCLIP_ENV_FILE_PATH (.paperclip/.env 或 ~/.paperclip/.env)
  ↓
CWD/.env
  ↓
config.json (~/.paperclip/instances/default/config.json)
  ↓
內建預設值（最低優先）
```

---

## 3. 測試策略與執行方法

### 測試分層

| 層次 | 工具 | 指令 | 執行時機 |
|------|------|------|---------|
| Unit / Integration | Vitest | `pnpm test` | **日常開發，每次改動後** |
| Unit（watch 模式） | Vitest | `pnpm test:watch` | 持續 TDD 開發 |
| E2E（瀏覽器） | Playwright | `pnpm test:e2e` | UI flow 修改後或 PR 前 |
| Release Smoke | Playwright | `pnpm test:release-smoke` | Release 前驗證 |
| OpenClaw Join Smoke | 自訂腳本 | `pnpm smoke:openclaw-join` | 涉及 agent join / webhook 邏輯時 |

### Vitest 涵蓋範圍

`vitest.config.ts` 設定的 projects：

```
packages/shared        packages/db
packages/adapter-utils packages/adapters/*（所有 adapter）
server                 ui
cli
```

測試檔案位置：各 package 的 `src/**/*.test.ts`

```sh
# 執行全部 Vitest 測試
pnpm test

# 互動式 watch 模式
pnpm test:watch
```

### E2E 測試（Playwright）

```sh
# 執行 E2E 測試（需要 dev server 已啟動或 Playwright 自行管理）
pnpm test:e2e

# Release smoke（發布前完整驗收）
pnpm test:release-smoke
```

> E2E 測試適用於：UI 頁面 flow、agent join 流程、瀏覽器行為驗證。  
> **不要**把 E2E 當成日常 unit test 使用——執行成本高。

### 最小化測試原則

> 「從最小範圍的 targeted check 開始。只有在 PR 準備好或改動範圍夠廣時，才跑全 repo 的 typecheck / build / test。」  
> — `doc/DEVELOPING.md`

---

## 4. Debugging 技巧與常見踩坑

### DB 重置（清空本地資料）

```sh
# 停止 dev server 後執行
pnpm dev:stop
rm -rf ~/.paperclip/instances/default/db
pnpm dev  # 重啟時自動重建 DB 並套用 migration
```

### 手動 DB Backup

```sh
pnpm paperclipai db:backup
# 或
pnpm db:backup
```

Backup 預設位置：`~/.paperclip/instances/default/data/backups`

### 設定互動式 wizard

```sh
# 設定資料庫相關
pnpm paperclipai configure --section database

# 設定 secrets 相關
pnpm paperclipai configure --section secrets

# 設定 storage 相關
pnpm paperclipai configure --section storage
```

### 環境健康檢查與自動修復

```sh
pnpm paperclipai doctor
pnpm paperclipai doctor --repair
```

### 端口競合

預設 port 為 `3100`。如果已被佔用：

```sh
# 查看佔用程序
lsof -i :3100

# 或改變 port
PORT=3200 pnpm dev
```

> `pnpm dev` 具備冪等性：若同一 repo 的 dev runner 已在執行，會直接回報既有 process 而不啟動新的。

### `Restart Required` 橫幅

`pnpm dev:once` 追蹤 backend 相關的檔案變更與 pending migration。  
若偵測到 boot 已過時，UI 會顯示 `Restart required` 橫幅。  
可至 `Instance Settings > Experimental` 啟用 **guarded auto-restart**（等待 running agent 完成後自動重啟）。

### 查看 Logs

```sh
# dev server 的 stdout/stderr 直接在啟動的 terminal 中顯示
pnpm dev

# 若使用 dev runner（後台），查看 runner 狀態
pnpm dev:list
```

### Secrets Migration（inline env → secret refs）

```sh
# Dry run（預覽）
pnpm secrets:migrate-inline-env

# 套用遷移
pnpm secrets:migrate-inline-env --apply
```

### 一次性全新 Bootstrap

```sh
# 自動 onboard + doctor + 啟動
pnpm paperclipai run
```

---

## 5. Contribution Workflow

### 兩種 PR 路徑

**Path 1：小型聚焦修改（最快合併）**
- 一個明確的修復/改進
- 觸碰最少數量的檔案
- 所有測試通過，CI green
- Greptile score **5/5**，所有 Greptile 評論已處理
- 使用 PR template

**Path 2：較大或有影響力的改動**
1. 先至 Discord `#dev` 討論方向
2. 獲得粗略共識後再寫 code
3. PR 中附上 Before/After 截圖或影片
4. 同樣需要 Greptile 5/5 + PR template

### Branch 模型

```sh
# 從 master 建立 feature branch
git checkout master
git pull origin master
git checkout -b PAP-<issue-id>-short-description
```

> ⚠️ 未驗證：官方文件未明確指定 branch naming convention，但 worktree 相關示例使用 `PAP-<id>-<desc>` 格式。

### PR 必要要件

每個 PR **必須**包含：

1. **PR Template** (`.github/PULL_REQUEST_TEMPLATE.md`) 完整填寫
   - Thinking Path（從整體架構到具體修改的思考路徑）
   - What Changed
   - Verification（如何驗證）
   - Risks
   - **Model Used**（使用的 AI model，或 "None — human-authored"）
   - Checklist

2. **Greptile Score 5/5**：所有自動 review 評論皆已處理

3. **CI Green**：所有 GitHub Actions checks 通過

### Thinking Path 範例

```
- Paperclip 管理 AI agent 組成的零人力公司
- 每個 agent 需要 budget policy 控制成本
- 但 monthly window 的計算在跨月時有 off-by-one 問題
- 所以這個 PR 修正 budgets.ts 中月份邊界的計算邏輯
- 確保跨月不會錯誤地重置 budget counter
```

### CI Checks

GitHub Actions 執行以下 checks：
- 依賴解析驗證（manifest 變更時）
- TypeScript typecheck
- Vitest 測試套件
- Linting
- Greptile 自動 code review

### 功能貢獻注意事項

- 先查閱 `ROADMAP.md` 確認方向
- 未協調的功能 PR 可能被關閉（roadmap ownership 考量）
- 優先考慮以 **plugin system** 實現擴展（`doc/plugins/PLUGIN_SPEC.md`）

---

## 6. 依賴管理與更新策略

### `pnpm-lock.yaml` 政策

> **重要**：不要在 PR 中提交 `pnpm-lock.yaml`。

| 動作 | 負責方 |
|------|--------|
| `pnpm-lock.yaml` 的更新 | GitHub Actions（push to master 時自動執行） |
| PR 中的依賴解析驗證 | CI（`pnpm install` 但不更新 lockfile） |
| 凍結 lockfile 驗證 | Push to master 後以 `--frozen-lockfile` 執行 |

Master push 流程：
```
pnpm install --lockfile-only --no-frozen-lockfile
  → commit back pnpm-lock.yaml if changed
  → pnpm install --frozen-lockfile (verification)
```

### 新增依賴

```sh
# 新增 workspace package 的依賴
pnpm add <package> --filter @paperclipai/server

# 新增開發依賴
pnpm add -D <package> --filter @paperclipai/ui

# 新增 root devDependency
pnpm add -D -w <package>
```

### Worktree 開發環境

多個 git worktree 開發時，**絕對不能**讓兩個 Paperclip server 共用同一個 embedded PostgreSQL 資料目錄。

```sh
# 方法 1：在現有 worktree 中初始化隔離 instance
paperclipai worktree init

# 方法 2：一步建立 git worktree + 初始化 Paperclip instance
pnpm paperclipai worktree:make paperclip-pr-432

# 從 origin/main 建立新 branch 並初始化
pnpm paperclipai worktree:make my-feature --start-point origin/main

# 不需要 seed DB 的輕量 worktree
pnpm paperclipai worktree:make experiment --no-seed
```

Worktree 初始化後，`.paperclip/.env` 自動設定：
- `PAPERCLIP_IN_WORKTREE=true`
- `PAPERCLIP_WORKTREE_NAME=<worktree-name>`
- `PAPERCLIP_WORKTREE_COLOR=<hex-color>`
- 獨立的 server port（自動選擇空閒 port）
- 獨立的 embedded PostgreSQL port

Worktree 資料位置：`~/.paperclip-worktrees/instances/<worktree-id>/`

```sh
# 查看 worktree 的環境變數
paperclipai worktree env

# 套用到 shell
eval "$(paperclipai worktree env)"

# Repair 已存在但損壞的 worktree
pnpm paperclipai worktree repair

# Reseed DB（從 default instance 重新植入資料）
pnpm paperclipai worktree reseed --from-instance default --seed-mode minimal
```

### DB Seed 模式

| 模式 | 說明 | 適用情境 |
|------|------|---------|
| `minimal`（預設） | 保留核心 app state（companies, projects, issues 等），略過 heartbeat runs 等歷史資料 | 一般 feature 開發 |
| `full` | 完整邏輯複製 | 需要還原完整歷史狀態時 |
| `--no-seed` | 空白 instance | 全新功能開發、不需要既有資料 |

> **注意**：Worktree seed 預設會自動 quarantine 複製過來的 live execution（停用 heartbeat timer、重置 running agents 為 idle），防止 worktree 意外繼續執行 source instance 的工作。

---

## 附錄：重要路徑速查

| 目的 | 路徑 |
|------|------|
| Embedded DB 資料 | `~/.paperclip/instances/default/db/` |
| DB Backups | `~/.paperclip/instances/default/data/backups/` |
| Local Storage | `~/.paperclip/instances/default/data/storage/` |
| Secrets Master Key | `~/.paperclip/instances/default/secrets/master.key` |
| Instance Config | `~/.paperclip/instances/default/config.json` |
| Worktree Instances | `~/.paperclip-worktrees/instances/<id>/` |
| Repo-local Worktree Config | `.paperclip/config.json` + `.paperclip/.env` |

---

## 附錄：有用連結

- 公式文件：<https://docs.paperclip.ing/>
- API 架構：<https://docs.paperclip.ing/start/architecture>
- Discord `#dev`：<https://discord.gg/m4HZY7xNG3>
- CLI 參考：`doc/CLI.md`
- Plugin 開發：`doc/plugins/PLUGIN_SPEC.md`
- Docker 詳細手順：`doc/DOCKER.md`
- Release 流程：`doc/RELEASING.md`
