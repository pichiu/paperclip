# Entry Points

## サーバー起動フロー

### メインエントリポイント

**`server/src/index.ts:startServer()`**

1. `loadConfig()` — 設定を読み込む（ENV vars → config file → default）
2. `initTelemetry()` — テレメトリ初期化
3. **Embedded PostgreSQL / 外部 Postgres 接続**
   - `DATABASE_URL` が未設定 → `EmbeddedPostgres` を起動（`embedded-postgres` パッケージ）
   - データディレクトリ: `~/.paperclip/instances/default/db`
4. `createDb(databaseUrl)` — Drizzle ORM インスタンス生成
5. **DB マイグレーション** (`applyPendingMigrations`)
6. `createStorageServiceFromConfig()` — ストレージサービス初期化（local disk or S3）
7. `createPluginWorkerManager()` — プラグインワーカー管理者初期化
8. `createApp(...)` — Express アプリ生成（全ルート登録）
9. `createServer(app)` — Node.js HTTP サーバー生成
10. `setupLiveEventsWebSocketServer(server)` — WebSocket サーバー初期化（SSE/ライブイベント）
11. `server.listen(port)` — ポートバインド（デフォルト 3100）
12. **サービス起動**:
    - `heartbeatService(db)` — ハートビートスケジューラ起動
    - `routineService(db)` — スケジュール済みルーティン起動
    - `feedbackService(db)` — フィードバックサービス起動
    - `reconcilePersistedRuntimeServicesOnStartup()` — 起動時にランタイムサービスを復旧

### CLI エントリポイント

**`cli/src/index.ts`**

```
paperclipai onboard    # 初回セットアップ
paperclipai run        # サーバー起動（onboard + doctor + start）
paperclipai configure  # 設定対話
paperclipai doctor     # ヘルスチェック・修復
paperclipai issue      # タスク管理（list/create/update）
paperclipai worktree   # git worktree 管理
```

### フロントエンドエントリポイント

**`ui/src/main.tsx`**

```tsx
initPluginBridge(React, ReactDOM);  // プラグイン UI ブリッジ初期化
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

## Express App 初期化（`server/src/app.ts`）

`createApp(db, config, options)` が呼ばれ、以下の順で設定される：

### Middleware スタック（登録順）

1. `httpLogger` — リクエストロギング
2. `privateHostnameGuard` — プライベートホスト名ガード（`authenticated` モード時）
3. `boardMutationGuard` — Board 操作の認可チェック
4. `actorMiddleware` — 認証・アクター識別（Board user / Agent JWT）
5. 各ルート登録
6. `errorHandler` — 集中エラーハンドリング

### ルート登録（`/api` プレフィックス）

| ルートファイル | パスパターン | 主な責務 |
|-------------|------------|---------|
| `routes/health.ts` | `/api/health` | ヘルスチェック |
| `routes/auth.ts` | `/api/auth/*` | Better Auth（ログイン・セッション） |
| `routes/companies.ts` | `/api/companies` | 会社 CRUD |
| `routes/agents.ts` | `/api/agents`, `/api/companies/:id/agents` | エージェント管理 |
| `routes/issues.ts` | `/api/issues`, `/api/companies/:id/issues` | タスク管理（最大のルートファイル） |
| `routes/heartbeat.ts` (via service) | (内部) | ハートビート実行 |
| `routes/routines.ts` | `/api/routines` | スケジュール済みルーティン |
| `routes/goals.ts` | `/api/goals` | ゴール管理 |
| `routes/projects.ts` | `/api/projects` | プロジェクト管理 |
| `routes/approvals.ts` | `/api/approvals` | 承認ワークフロー |
| `routes/costs.ts` | `/api/costs` | コスト・予算 |
| `routes/secrets.ts` | `/api/secrets` | シークレット管理 |
| `routes/plugins.ts` | `/api/plugins` | プラグイン管理 |
| `routes/adapters.ts` | `/api/adapters` | アダプター設定 |
| `routes/access.ts` | `/api/access`, `/api/invites` | アクセス管理・招待 |
| `routes/activity.ts` | `/api/activity` | アクティビティログ |
| `routes/dashboard.ts` | `/api/dashboard` | ダッシュボード集計 |
| `routes/environments.ts` | `/api/environments` | 実行環境管理 |
| `routes/execution-workspaces.ts` | `/api/execution-workspaces` | Execution Workspace |
| `routes/assets.ts` | `/api/assets` | アセット（画像等） |
| `routes/instance-settings.ts` | `/api/instance-settings` | インスタンス設定 |
| `routes/llms.ts` | `/api/llms` | LLM プロバイダー情報 |
| `routes/plugin-ui-static.ts` | `/plugins/:pluginId/ui` | プラグイン UI 静的配信 |

### UI 配信モード

- `none` — API のみ
- `static` — ビルド済み React ファイルを serve
- `vite-dev` — 開発時のみ、Vite dev middleware を介して serve

## デプロイモード

| モード | 起動方法 | 認証 |
|-------|---------|------|
| `local_trusted` | デフォルト | 認証なし（ローカルループバック）|
| `authenticated/private` | `--bind lan` / `--bind tailnet` | Better Auth ログイン必須 |
| `authenticated/public` | 明示設定 | Better Auth ログイン必須 |
