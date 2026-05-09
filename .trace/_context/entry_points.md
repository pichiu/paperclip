# 進入點（Entry Points）

## 伺服器啟動流程

### 主要進入點

**`server/src/index.ts:startServer()`**

1. `loadConfig()` — 讀取設定（環境變數 → 設定檔 → 預設值）
2. `initTelemetry()` — 初始化 telemetry
3. **Embedded PostgreSQL / 外部 Postgres 連線**
   - `DATABASE_URL` 未設定 → 啟動 `EmbeddedPostgres`（`embedded-postgres` 套件）
   - 資料目錄：`~/.paperclip/instances/default/db`
4. `createDb(databaseUrl)` — 建立 Drizzle ORM 實例
5. **DB migration**（`applyPendingMigrations`）
6. `createStorageServiceFromConfig()` — 初始化儲存服務（本機磁碟或 S3）
7. `createPluginWorkerManager()` — 初始化 plugin worker 管理員
8. `createApp(...)` — 建立 Express 應用程式（註冊所有路由）
9. `createServer(app)` — 建立 Node.js HTTP 伺服器
10. `setupLiveEventsWebSocketServer(server)` — 初始化 WebSocket 伺服器（SSE/即時事件）
11. `server.listen(port)` — 綁定埠號（預設 3100）
12. **啟動各服務**：
    - `heartbeatService(db)` — 啟動 heartbeat 排程器
    - `routineService(db)` — 啟動已排程的 routine
    - `feedbackService(db)` — 啟動 feedback 服務
    - `reconcilePersistedRuntimeServicesOnStartup()` — 啟動時復原 runtime 服務

### CLI 進入點

**`cli/src/index.ts`**

```
paperclipai onboard    # 初次設定
paperclipai run        # 啟動伺服器（onboard + doctor + start）
paperclipai configure  # 互動式設定
paperclipai doctor     # 健康檢查與修復
paperclipai issue      # 任務管理（list/create/update）
paperclipai worktree   # git worktree 管理
```

### 前端進入點

**`ui/src/main.tsx`**

```tsx
initPluginBridge(React, ReactDOM);  // 初始化 plugin UI bridge
createRoot(container).render(
  <BrowserRouter>
    <QueryClientProvider client={queryClient}>
      <ThemeProvider>
        <CompanyProvider>
          <LiveUpdatesProvider>  // WebSocket SSE
            <App />
          </LiveUpdatesProvider>
        </CompanyProvider>
      </ThemeProvider>
    </QueryClientProvider>
  </BrowserRouter>
)
```

## Express App 初始化（`server/src/app.ts`）

呼叫 `createApp(db, config, options)` 後，依以下順序進行設定：

### Middleware 堆疊（依註冊順序）

1. `httpLogger` — 請求記錄
2. `privateHostnameGuard` — 私有主機名稱守衛（`authenticated` 模式時）
3. `boardMutationGuard` — Board 操作授權檢查
4. `actorMiddleware` — 驗證與 actor 識別（Board 使用者 / Agent JWT）
5. 各路由註冊
6. `errorHandler` — 集中式錯誤處理

### 路由註冊（`/api` 前綴）

| 路由檔案 | 路徑模式 | 主要職責 |
|----------|----------|----------|
| `routes/health.ts` | `/api/health` | 健康檢查 |
| `routes/auth.ts` | `/api/auth/*` | Better Auth（登入・session） |
| `routes/companies.ts` | `/api/companies` | 公司 CRUD |
| `routes/agents.ts` | `/api/agents`、`/api/companies/:id/agents` | 代理程式管理 |
| `routes/issues.ts` | `/api/issues`、`/api/companies/:id/issues` | 任務管理（最大路由檔案） |
| `routes/heartbeat.ts`（透過 service） | （內部） | heartbeat 執行 |
| `routes/routines.ts` | `/api/routines` | 已排程 routine |
| `routes/goals.ts` | `/api/goals` | goal 管理 |
| `routes/projects.ts` | `/api/projects` | 專案管理 |
| `routes/approvals.ts` | `/api/approvals` | 審核工作流程 |
| `routes/costs.ts` | `/api/costs` | 成本與 budget |
| `routes/secrets.ts` | `/api/secrets` | secret 管理 |
| `routes/plugins.ts` | `/api/plugins` | plugin 管理 |
| `routes/adapters.ts` | `/api/adapters` | adapter 設定 |
| `routes/access.ts` | `/api/access`、`/api/invites` | 存取管理與邀請 |
| `routes/activity.ts` | `/api/activity` | 活動日誌 |
| `routes/dashboard.ts` | `/api/dashboard` | 儀表板彙總 |
| `routes/environments.ts` | `/api/environments` | 執行環境管理 |
| `routes/execution-workspaces.ts` | `/api/execution-workspaces` | Execution Workspace |
| `routes/assets.ts` | `/api/assets` | 資產（圖片等） |
| `routes/instance-settings.ts` | `/api/instance-settings` | 執行個體設定 |
| `routes/llms.ts` | `/api/llms` | LLM provider 資訊 |
| `routes/plugin-ui-static.ts` | `/plugins/:pluginId/ui` | plugin UI 靜態檔案提供 |

### UI 提供模式

- `none` — 僅提供 API
- `static` — 提供已建置的 React 檔案
- `vite-dev` — 僅開發時，透過 Vite dev middleware 提供

## 部署模式

| 模式 | 啟動方式 | 驗證 |
|------|----------|------|
| `local_trusted` | 預設 | 無需驗證（本機 loopback） |
| `authenticated/private` | `--bind lan` / `--bind tailnet` | 必須 Better Auth 登入 |
| `authenticated/public` | 明確設定 | 必須 Better Auth 登入 |
