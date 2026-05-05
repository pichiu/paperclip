# Stage 1 偵察レポート

## プロジェクト概要

**Paperclip** は、AI エージェントのチームをひとつの「会社」として運営するための Node.js + React 製オープンソース Control Plane。
"If OpenClaw is an _employee_, Paperclip is the _company_" というスローガンが示すとおり、エージェントを雇用し、組織図・目標・予算・ガバナンスを管理するオーケストレーション基盤である。

- **URL**: https://paperclip.ing/
- **GitHub**: https://github.com/paperclipai/paperclip
- **ライセンス**: MIT
- **初公開**: 2026年3月4日（GitHub stars 3週間で 30,000 超）

---

## 技術スタック

| カテゴリ | 技術 | バージョン | 用途 |
|---------|------|-----------|------|
| Runtime | Node.js | ≥20 | サーバー実行環境 |
| Package Manager | pnpm | 9.15.4 | monorepo 管理 |
| Backend Framework | Express.js | 4.x | HTTP API サーバー |
| Frontend Framework | React | 18.x | ダッシュボード UI |
| Build Tool (UI) | Vite | - | フロントエンドビルド |
| ORM | Drizzle ORM | ^0.38.4 | DB スキーマ・クエリ |
| Database | PostgreSQL (embedded-postgres) | 18.1.0-beta.16 | データ永続化 |
| Auth | Better Auth | - | 認証・セッション管理 |
| Language | TypeScript | ^5.7.3 | 全パッケージ共通 |
| Test Framework | Vitest | ^3.0.5 | ユニット・統合テスト |
| E2E Test | Playwright | ^1.58.2 | ブラウザテスト |
| HTTP Client (UI) | TanStack Query (React Query) | - | データフェッチング |
| Router (UI) | React Router | - | SPA ルーティング |
| Container | Docker | - | 本番デプロイ |
| CI/CD | GitHub Actions | - | PR チェック・リリース |
| Telemetry | 独自 (anonymous) | - | 使用状況収集 |

---

## Monorepo 構成

```
paperclip/
├── server/               # Express.js API サーバー (@paperclipai/server)
│   └── src/
│       ├── index.ts      # エントリポイント（DB初期化・HTTPサーバー起動）
│       ├── app.ts        # Express アプリ（ルート登録）
│       ├── routes/       # API エンドポイント群
│       ├── services/     # ビジネスロジック（heartbeat, issues, budgets 等）
│       ├── adapters/     # エージェント実行アダプター
│       ├── auth/         # 認証（Better Auth）
│       ├── middleware/   # HTTP middleware
│       ├── storage/      # ファイルストレージ抽象
│       └── realtime/     # WebSocket（ライブイベント）
├── ui/                   # React ダッシュボード (@paperclipai/ui)
│   └── src/
│       ├── main.tsx      # React エントリポイント
│       ├── App.tsx       # ルートコンポーネント
│       ├── pages/        # ページコンポーネント群
│       ├── components/   # 共通 UI コンポーネント
│       ├── api/          # API クライアント
│       ├── context/      # React Context プロバイダー群
│       └── plugins/      # プラグイン UI ブリッジ
├── cli/                  # CLI ツール (@paperclipai/cli)
│   └── src/
│       ├── index.ts      # CLI エントリポイント
│       └── commands/     # CLI サブコマンド群
├── packages/
│   ├── db/               # DB スキーマ・マイグレーション (@paperclipai/db)
│   │   └── src/
│   │       ├── schema/   # Drizzle テーブル定義 (~70 テーブル)
│   │       └── migrations/
│   ├── shared/           # 共有型・定数 (@paperclipai/shared)
│   ├── adapter-utils/    # アダプター共通ユーティリティ
│   ├── mcp-server/       # MCP サーバー実装
│   ├── adapters/         # エージェントアダプター群
│   │   ├── claude-local/   # Claude Code ローカル実行
│   │   ├── codex-local/    # OpenAI Codex ローカル実行
│   │   ├── cursor-local/   # Cursor ローカル実行
│   │   ├── gemini-local/   # Gemini ローカル実行
│   │   ├── openclaw-gateway/ # OpenClaw Webhook ゲートウェイ
│   │   ├── opencode-local/ # OpenCode ローカル実行
│   │   ├── pi-local/       # Pi ローカル実行
│   │   └── acpx-local/     # ACPX ローカル実行
│   └── plugins/
│       ├── sdk/            # プラグイン開発 SDK (@paperclipai/plugin-sdk)
│       └── examples/       # サンプルプラグイン
├── doc/                  # 内部設計ドキュメント
├── docs/                 # Mintlify 公開ドキュメント
├── evals/                # LLM 評価スクリプト (promptfoo)
├── skills/               # エージェントスキル定義
├── scripts/              # ビルド・リリーススクリプト
├── tests/                # E2E・リリーススモークテスト
├── docker/               # Docker/Compose 設定
└── releases/             # リリース管理
```

---

## アーキテクチャパターン

- **Monorepo**（pnpm workspaces）
- **Server-side Rendered + SPA Hybrid**: Express が静的ファイルを serve し、React SPA として動作
- **Embedded PostgreSQL**: ローカル開発はゼロ設定、本番は外部 Postgres に切り替え可能
- **Plugin Architecture**: out-of-process Worker による拡張プラグインシステム
- **Adapter Pattern**: 各エージェントランタイムをアダプター経由で統一インターフェースに

---

## 既存ドキュメント一覧

### `doc/` 内部設計ドキュメント

| ファイル | 内容 |
|--------|------|
| `doc/SPEC.md` | 技術仕様（Company/Agent/Org/Heartbeat/Budget モデル） |
| `doc/SPEC-implementation.md` | V1 実装契約 |
| `doc/PRODUCT.md` | プロダクト定義・設計原則 |
| `doc/DEVELOPING.md` | 開発ガイド（DB/テスト/Docker/worktree 詳細） |
| `doc/DEPLOYMENT-MODES.md` | デプロイモード定義 |
| `doc/TASKS.md` | タスク管理データモデル |
| `doc/TASKS-mcp.md` | MCP タスク仕様 |
| `doc/GOAL.md` | ゴール管理仕様 |
| `doc/CLI.md` | CLI コマンドリファレンス |
| `doc/DATABASE.md` | DB 設計ドキュメント |
| `doc/DOCKER.md` | Docker 詳細手順 |
| `doc/RELEASING.md` | リリースプロセス |
| `doc/plugins/PLUGIN_SPEC.md` | プラグインシステム仕様 |
| `doc/execution-semantics.md` | 実行セマンティクス |
| `doc/memory-landscape.md` | メモリ/コンテキスト設計 |

### `docs/` 公開ドキュメント（Mintlify）

- `docs/start/` — クイックスタート・アーキテクチャ
- `docs/adapters/` — アダプターガイド
- `docs/api/` — API リファレンス
- `docs/companies/` — 会社管理ガイド
- `docs/deploy/` — デプロイガイド
- `docs/guides/` — 開発者ガイド

---

## 既存ドキュメントと実際のコードの照合

### 一致している点
- `doc/SPEC.md` が記述するエンティティ（Company, Agent, Issue, Heartbeat Run, Budget Policy）はすべて `packages/db/src/schema/` のテーブルとして実装済み
- アダプタータイプ（claude_local, codex_local, openclaw_gateway 等）は `packages/adapters/` と一致

### 落差・注意点
- `doc/SPEC.md` に記載の「billing codes」と「request depth」は schema に実装済み（`issues.billing_code`, `issues.request_depth`）だが、UI 上での表示・編集は ⚠️ 未確認
- `doc/SPEC.md` の「Budget Delegation（cascading）」は `server/src/services/budgets.ts` で月次ウィンドウ・生涯ウィンドウのポリシーとして実装されているが、完全な cascading 委任は ⚠️ 一部未実装の可能性あり
- `ROADMAP.md` の「Memory / Knowledge」「Enforced Outcomes」「CEO Chat」「Cloud deployments」は⚪未実装

---

## 設定ファイル

### `.env.example` の主要環境変数

```
DATABASE_URL=postgres://...        # 未設定時は embedded postgres 使用
PORT=3100
SERVE_UI=false                     # 本番は true
BETTER_AUTH_SECRET=...
PAPERCLIP_TELEMETRY_DISABLED=1     # テレメトリ無効化
```

### デプロイモード

| モード | 説明 |
|-------|------|
| `local_trusted` | シングルユーザー・ログイン不要（デフォルト） |
| `authenticated` | 認証必須。`private`（LAN/Tailscale）または `public` 露出 |

---

## ファイル規模

- TypeScript ファイル: 1,149 個
- TSX ファイル: 304 個
- 総ファイル数（非 node_modules）: 2,007 個
- DB スキーマテーブル: 約 70 テーブル
- API ルートファイル: 40+
- サービスファイル: 80+

> **Stage 1.5 評価**: 2,007 ファイルで 500 超。Full trace として継続する。
> プロジェクトは API サーバー + フロントエンド + CLI + プラグインシステムを持つ複合型のため、全 trace 路径が適用される。
