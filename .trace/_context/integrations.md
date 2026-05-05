# 外部整合（Integrations）

## 1. エージェントランタイム（最大の外部依存）

### Claude Code (Anthropic)

- **アダプター**: `packages/adapters/claude-local/`
- **接続方式**: ローカル `claude` CLI を子プロセスとして起動
- **認証**: `ANTHROPIC_API_KEY` または Claude Code セッション
- **セッション継続**: `claude --resume <sessionId>` で前のセッションから再開
- **モデル**: claude-opus-4-7, claude-sonnet-4-6, claude-haiku-4-6 等

**障害処理**:
- `isClaudeTransientUpstreamError()` で一時的エラーを検出 → bounded retry（`BOUNDED_TRANSIENT_HEARTBEAT_RETRY_DELAYS_MS`）
- `isClaudeMaxTurnsResult()` → max_turns_continuation で次のハートビートに継続
- `detectClaudeLoginRequired()` → ログイン必要状態を検出して Board に通知

### OpenClaw Gateway

- **アダプター**: `packages/adapters/openclaw-gateway/`
- **接続方式**: WebSocket over HTTPS（`wss://...`）
- **認証**: Device Identity (ECDH keypair) — `deviceId` + 公開鍵を OpenClaw に登録
- **プロトコル**: JSON-RPC スタイルのリクエスト/レスポンスフレーム
- **起動方式**: Fire-and-forget（Webhook）ではなく、常時接続 WebSocket でコールバックを待つ

**障害処理**:
- WebSocket 切断時に再接続リトライ
- タイムアウト設定あり（`request timeout`）
- `pendingRequest` Map でリクエストの応答を追跡

### Codex / Gemini / OpenCode / Pi / Cursor

同様のパターン（CLI 子プロセス起動）。各アダプターが CLI の stdout/stderr をパースしてトークン使用量・実行結果を取得。

---

## 2. PostgreSQL（組み込み or 外部）

- **パッケージ**: `embedded-postgres@18.1.0-beta.16`（パッチ済み）
- **接続**: `DATABASE_URL` 環境変数で外部 Postgres に切り替え可能
- **ORM**: Drizzle ORM
- **マイグレーション**: `packages/db/src/migrations/` に SQL ファイル

**接続ライフサイクル**:
1. `server/src/index.ts` の `startServer()` でインスタンス起動（または外部接続）
2. `applyPendingMigrations()` で未適用マイグレーションを適用
3. `createDb(url)` で Drizzle ORM インスタンス生成

---

## 3. Better Auth（認証）

- **パッケージ**: `better-auth`
- **モード**: `authenticated` デプロイモードでのみ使用
- **提供機能**: セッション管理、ユーザー登録/ログイン、Cookie ベース認証
- **設定**: `BETTER_AUTH_SECRET` 環境変数（HMAC シークレット）
- **エンドポイント**: `/api/auth/*` 以下すべて

---

## 4. Object Storage（ファイル添付・ワーク成果物）

**`server/src/storage/`**:

| プロバイダー | 設定 | 用途 |
|----------|-----|-----|
| `local_disk` | `PAPERCLIP_STORAGE_DIR` | ローカル開発・自己ホスト |
| `s3` | `STORAGE_S3_BUCKET` 等 | 本番 S3 互換ストレージ |

`StorageService` インターフェース:
- `put(key, buffer, contentType)` — ファイルアップロード
- `get(key)` — ファイル取得
- `delete(key)` — ファイル削除
- `getSignedUrl(key, expiry)` — 署名付き URL 生成

---

## 5. Telemetry（匿名使用統計）

- **`server/src/telemetry.ts`**
- **送信内容**: 匿名の使用パターン（プロジェクト名はハッシュ化）
- **除外対象**: 個人情報・Issue 内容・プロンプト・ファイルパス・シークレット
- **無効化**:
  - `PAPERCLIP_TELEMETRY_DISABLED=1`
  - `DO_NOT_TRACK=1`
  - `CI=true` (自動無効)
  - config file で `telemetry.enabled: false`

---

## 6. GitHub Integration

`server/src/services/github-fetch.ts`:
- GitHub API 呼び出し（PR 情報取得等）
- ⚠️ 主な用途は未確認（内部ドキュメントには PR レビューツールは Paperclip の役割外と明記）

---

## 7. MCP Server

`packages/mcp-server/`:
- Paperclip のデータを MCP (Model Context Protocol) サーバーとして公開
- これにより、Claude Code 等の MCP 対応エージェントが Paperclip のデータを直接ツールとして利用可能

---

## 8. Feedback / Trace Share

`server/src/services/feedback-share-client.ts`:
- フィードバックの匿名共有機能
- フラッシュ間隔: 5,000ms (`FEEDBACK_EXPORT_FLUSH_INTERVAL_MS`)

---

## 9. エラーハンドリングパターン

| シナリオ | 処理方法 |
|---------|---------|
| Claude 一時エラー | bounded retry（指数バックオフ）|
| 予算超過 | エージェント停止、Board に通知 |
| エージェント応答なし | Watchdog が検出、recovery issue 自動作成 |
| DB 接続失敗 | サーバー起動拒否（fail-fast） |
| プラグインクラッシュ | ワーカープロセス再起動（他に影響なし） |
| ストレージエラー | エラーレスポンス返却（キャッチ後ログ） |
| Webhook タイムアウト | タイムアウト設定後エラー記録 |
