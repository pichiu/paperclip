# Paperclip — プロジェクト総覧

## 一言サマリー

**Paperclip** は、AI エージェント（Claude Code、Codex、OpenClaw 等）をひとつの「会社」として組織化・運営するオープンソースの Control Plane。「エージェントは従業員、Paperclip は会社」というコンセプトで、組織図・ゴール・予算・ガバナンス・タスク管理を提供し、複数の異種エージェントを協調動作させる。

---

## 技術スタック総覧

| カテゴリ | 技術 | バージョン | 用途 |
|---------|------|-----------|------|
| Runtime | Node.js | ≥20 | サーバー実行環境 |
| Package Manager | pnpm | 9.15.4 | Monorepo 管理 |
| Backend Framework | Express.js | 4.x | HTTP API サーバー |
| Frontend Framework | React | 18.x | ダッシュボード UI |
| Build Tool (UI) | Vite | - | フロントエンドビルド |
| ORM | Drizzle ORM | ^0.38.4 | DB スキーマ・クエリ |
| Database | PostgreSQL (embedded) | 18.1.0-beta.16 | データ永続化（ゼロ設定） |
| Auth | Better Auth | - | 認証・セッション管理 |
| Language | TypeScript | ^5.7.3 | 全パッケージ共通 |
| Test Framework | Vitest | ^3.0.5 | ユニット・統合テスト |
| E2E Test | Playwright | ^1.58.2 | ブラウザテスト |
| HTTP Client (UI) | TanStack Query | - | データフェッチング |
| Router (UI) | React Router | - | SPA ルーティング |
| Container | Docker | - | 本番デプロイ |
| CI/CD | GitHub Actions | - | PR チェック・リリース |

---

## 関鍵指令速查

```bash
# 開発起動（API + UI、ウォッチモード）
pnpm dev

# 開発起動（ウォッチなし）
pnpm dev:once

# サーバーのみ起動
pnpm dev:server

# 全パッケージビルド
pnpm build

# 型チェック
pnpm typecheck

# テスト実行（Vitest のみ）
pnpm test

# テスト（ウォッチモード）
pnpm test:watch

# E2E テスト（Playwright）
pnpm test:e2e

# DB マイグレーション生成
pnpm db:generate

# DB マイグレーション適用
pnpm db:migrate

# CLI（ローカル開発）
pnpm paperclipai <command>

# 初回セットアップ
npx paperclipai onboard --yes

# Storybook
pnpm storybook
```

---

## 文書地図

| 文書 | 内容 |
|-----|------|
| [INDEX.md](./INDEX.md) | このファイル — 総覧・速查 |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | システムアーキテクチャ・コンポーネント・設計決策 |
| [DATA_MODEL.md](./DATA_MODEL.md) | データモデル・ER 図・スキーマ詳細 |
| [API_SURFACE.md](./API_SURFACE.md) | REST API・CLI コマンドリファレンス |
| [DEV_GUIDE.md](./DEV_GUIDE.md) | 開発者上手ガイド・環境設定・テスト |
| [CODEBASE_MAP.md](./CODEBASE_MAP.md) | コード地図・「何を修正するには？」速查 |
| [DISCOVERY_LOG.md](./DISCOVERY_LOG.md) | 探索記録・既知の技術債・未解決の疑問 |
| [TRACE_META.md](./TRACE_META.md) | トレースメタデータ（増分更新用） |

---

## プロジェクト専用術語表

| 用語 | 定義 |
|-----|------|
| **Company** | Paperclip の最上位オブジェクト。1 インスタンスで複数 Company を運営可能。 |
| **Agent** | エージェント。「従業員」。Claude Code / OpenClaw 等の実際のランタイムに紐付く。 |
| **Board** | ボード。人間のオペレーター。Company 全体のガバナンスを担う。 |
| **Heartbeat** | ハートビート。エージェントの定期起動サイクル。スケジュールまたはイベントでトリガー。 |
| **Issue** | タスク。チケット。会社のゴールに連鎖する作業単位。 |
| **Heartbeat Run** | 1 回のハートビート実行の記録（`heartbeat_runs` テーブル）。 |
| **Adapter** | エージェントランタイムとの接続層（例: `claude_local`、`openclaw_gateway`）。 |
| **Skill** | エージェントに注入されるマークダウン形式の知識・手順書。 |
| **Routine** | スケジュール済みの定期タスク定義（cron / webhook / API トリガー）。 |
| **Worktree** | 開発用の分離 git worktree インスタンス。独自 DB とポートを持つ。 |
| **Execution Workspace** | Issue 実行時の git worktree またはプロジェクトワークスペース。 |
| **Activity Log** | すべての変更操作の監査ログ（`activity_log` テーブル）。 |
| **Goal** | ゴール。Mission → Initiative → Project → Issue の階層。 |
| **Clipmart** | Coming Soon: 会社テンプレートのマーケットプレイス。 |
| **Board Claim** | `local_trusted` モードでの Board 所有権主張フロー。 |
| **Wake Payload** | エージェント起動時に渡されるコンテキスト束（会社・ゴール・タスク・スキル等）。 |
| **Secret Ref** | シークレット値への参照（インライン plain text の代わり）。 |
| **Invitation** | エージェントをシステムに招待するためのトークンベースの招待フロー。 |
| **Plugin** | out-of-process Worker として動作する拡張機能（npm パッケージ）。 |
