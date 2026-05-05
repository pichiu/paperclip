# Data Flow — 代表的ユースケース：Issue の Checkout とエージェント実行

## ユースケース概要

「エージェントが Issue (タスク) を checkout し、ハートビートを通じて実際に作業を実行するまで」の完全なデータフロー。

---

## フロー全体図

```
HTTP Client (Board UI / Agent)
    │
    ▼
actorMiddleware              # 認証・アクター識別
    │
    ▼
POST /api/issues/:id/checkout
    │
    ▼
issueService.checkout()      # 楽観ロック付きの Issue チェックアウト
    │
    ▼ (assignee が agent の場合)
heartbeatService.wakeup()    # エージェントウェイクアップキュー
    │
    ▼
enqueueWakeup()              # agentWakeupRequests テーブルに挿入
    │
    ▼
heartbeat timer loop         # 定期的なキュードレイン
    │
    ▼
budgetService.getInvocationBlock()   # 予算チェック
    │
    ▼
secretService.inject()       # シークレット注入
    │
    ▼
getServerAdapter()           # アダプター選択 (claude_local等)
    │
    ▼
adapter.execute()            # 実際のエージェント実行
    │
    ▼
heartbeatRuns テーブル更新 (status: running → completed/failed)
    │
    ▼
costService.record()         # トークンコスト記録
    │
    ▼
activity logActivity()       # 監査ログ記録
    │
    ▼
publishLiveEvent()           # WebSocket で Board UI にプッシュ
```

---

## Step 1: 認証・アクター識別 (`server/src/middleware/auth.ts`)

`actorMiddleware` が実行され、リクエストを発したアクターを識別：

| ソース | アクター型 |
|-------|----------|
| `local_trusted` モード | 常に `board` ユーザー（認証不要） |
| `Authorization: Bearer <jwt>` | エージェントの短命 JWT (`agent` 型) |
| `Authorization: Bearer <api-key>` | Agent API Key (`agent` 型) または Board API Key |
| Cookie セッション | Better Auth セッション (`user` 型) |

`req.actor` オブジェクトにアクター情報をセット。

---

## Step 2: Issue チェックアウト (`server/src/routes/issues.ts:2911`)

```
POST /api/issues/:id/checkout
Body: { agentId, expectedStatuses }
```

### バリデーション
- `validate(checkoutIssueSchema)` — Zod スキーマによる入力バリデーション
- `assertCompanyAccess(req, companyId)` — アクターが対象会社にアクセス可能か確認
- プロジェクトが pause 中でないか確認

### 楽観ロック付き checkout (`issueService.checkout()`)

`server/src/services/issues.ts` にて：

```sql
UPDATE issues
SET 
  status = 'in_progress',
  checkout_run_id = :runId,
  execution_locked_at = NOW()
WHERE id = :issueId
  AND status = ANY(:expectedStatuses)  -- 楽観ロック
RETURNING *
```

`expectedStatuses` が合わない（他のエージェントが先に取得した）場合は `409 Conflict` を返す。

### アクティビティログ
`logActivity(db, { action: "issue.checked_out", ... })` → `activity_log` テーブル挿入

---

## Step 3: エージェントウェイクアップ (`enqueueWakeup`)

`server/src/services/heartbeat.ts:7834`

1. **予算ブロックチェック**: `budgets.getInvocationBlock()` 
   - エージェント/プロジェクト/会社レベルの月次/生涯予算ポリシーを確認
   - ブロックの場合 → `agentWakeupRequests` に `status: skipped` で記録、`409` エラー
2. **エージェント状態チェック**: `agent.status` が `paused/terminated/pending_approval` なら同様にスキップ
3. **DB トランザクション内で**:
   - `heartbeatRuns` テーブルに `status: queued` で新規ラン挿入
   - `agentWakeupRequests` テーブルに記録
   - `issues` の `execution_run_id` を更新（実行ロック）

---

## Step 4: ハートビートスケジューラーのキュードレイン

`server/src/services/heartbeat.ts` の内部ループが定期的（またはウェイクアップリクエスト受信時）に：

1. `heartbeatRuns` から `status: queued` のランを取得
2. `budgets.getInvocationBlock()` で再度予算確認
3. `secretService.inject()` でエージェント設定の secret refs を実値に展開
4. `companySkillService.getSkillsForAgent()` でエージェントのスキルを取得
5. `realizeExecutionWorkspace()` でワークスペース解決（git worktree, CWD 等）
6. `buildPaperclipWakePayload()` でエージェントへ渡す完全なコンテキストペイロードを構築
7. `getServerAdapter(adapterType).execute(context)` で実際のアダプターを呼び出す

---

## Step 5: アダプター実行 (`packages/adapters/claude-local/src/server/execute.ts`)

`execute(context: AdapterExecutionContext)` が呼ばれる：

1. **コマンド解決**: `claude` CLI のパス解決
2. **セッション状態**: 既存セッション ID から継続実行、または新規セッション
3. **プロンプトバンドル構築**: `buildPaperclipWakePayload` で生成されたペイロードを `--prompt` に渡す
4. **環境変数注入**: シークレット、APIキー、ワークスペースパス等
5. **子プロセス起動**: `child_process.spawn("claude", args, { cwd, env })`
6. **stdout/stderr ストリーム処理**:
   - ログを `heartbeatRunEvents` テーブルにリアルタイム挿入
   - 同時にログファイルにも書き出す（`run-log-store.ts`）
7. **完了後**: `AdapterExecutionResult` を返す（exitCode, usageSummary 等）

---

## Step 6: ラン完了処理

1. `heartbeatRuns.status` を `running` → `completed` / `failed` に更新
2. `costService.record()` — トークン使用量を `cost_events` テーブルに記録
3. `issues.executionRunId` をクリア（実行ロック解除）
4. `publishLiveEvent("heartbeat_run_updated", ...)` — WebSocket 経由で UI に通知
5. `budgets.recordCost()` — 予算ポリシーの累積コスト更新、ハードストップ確認

---

## 補足：エージェントから Paperclip API を呼ぶ逆方向のフロー

実行中のエージェント（例: Claude Code）は、短命 JWT（`createLocalAgentJwt`）を使って Paperclip API を呼び戻す：

```
GET  /api/issues/:id/heartbeat-context  # 現在のタスクコンテキスト取得
POST /api/issues/:id/comments           # コメント投稿
PATCH /api/issues/:id                   # ステータス更新
POST /api/issues/:id/children           # 子タスク作成
POST /api/issues/:id/checkout           # タスクをエージェントがチェックアウト
POST /api/issues/:id/release            # タスクを解放
```

この JWT は `server/src/agent-auth-jwt.ts` で生成・検証されており、有効期限は短い（ランのライフタイムのみ）。

---

## データ変換サマリー

| 層 | 変換内容 |
|---|---------|
| HTTP Routing | パスパラメータ展開 + Zod バリデーション |
| Middleware | アクター識別 → `req.actor` |
| Service層 | ビジネスルール（状態遷移・楽観ロック）、Drizzle ORM によるDB操作 |
| Adapter層 | Paperclip コンテキスト → エージェント CLI コマンド |
| エージェント実行 | Claude Code がプロンプトを受け取り、Paperclip Skill経由でAPI操作 |
| 完了後 | 使用量・コスト・ログ → DB への永続化 + WebSocket プッシュ |
