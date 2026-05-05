# Paperclip システムアーキテクチャ

> **概要**: Paperclip は AI エージェントチームを「会社」として運営する Node.js + React 製 Control Plane。
> エージェントの雇用・組織図・目標・予算・ガバナンスを管理するオーケストレーション基盤。

---

## 1. 高層アーキテクチャ

```mermaid
graph LR
    subgraph Client["クライアント層"]
        UI["React SPA\n(ui/)"]
        CLI["CLI\n(cli/)"]
        EXT["外部 Webhook\n/ API クライアント"]
    end

    subgraph Server["サーバー層 (server/)"]
        EXPRESS["Express.js\nHTTP Server\n:3100"]
        WS["WebSocket\n(realtime/)"]
        AUTH["Better Auth\n(auth/)"]
        MW["Middleware Stack\n(httpLogger / actorMiddleware\n/ boardMutationGuard)"]
        ROUTES["API Routes\n(/api/*)"]

        subgraph Services["Service Layer (services/)"]
            HB["Heartbeat Engine\nheartbeat.ts (~9000行)"]
            BUDGETS["Budget Service\nbudgets.ts"]
            ISSUES["Issue Service\nissues.ts"]
            PLUGINS["Plugin Worker Manager\nplugin-worker-manager.ts"]
            ROUTINES["Routine Service\nroutines.ts"]
            RECOVERY["Recovery System\nrecovery/"]
            APPROVALS["Approval Service\napprovals.ts"]
            ACTIVITY["Activity Log\nactivity-log.ts"]
        end

        subgraph Adapters["Adapter Layer (adapters/)"]
            REG["Adapter Registry\nregistry.ts"]
            CL["claude_local"]
            CDX["codex_local"]
            GEM["gemini_local"]
            OCG["openclaw_gateway"]
            OTH["その他アダプター..."]
        end
    end

    subgraph Data["データ層"]
        PG["PostgreSQL\n(embedded or external)"]
        DRIZZLE["Drizzle ORM\n(packages/db/ ~70テーブル)"]
        STORAGE["Storage Service\n(local disk / S3)"]
    end

    subgraph PluginProc["Plugin Worker プロセス群"]
        PW1["Plugin Worker 1\n(out-of-process)"]
        PW2["Plugin Worker 2"]
    end

    subgraph AgentRT["エージェントランタイム"]
        CLAUDE["Claude Code CLI"]
        CODEX["Codex CLI"]
        GEMINI_RT["Gemini CLI"]
        OC_GW["OpenClaw\n(HTTP Webhook)"]
    end

    UI -- "REST API / TanStack Query" --> EXPRESS
    UI -- "WebSocket (live events)" --> WS
    CLI -- "HTTP API" --> EXPRESS
    EXT -- "Webhook / REST" --> EXPRESS

    EXPRESS --> MW --> ROUTES --> Services
    EXPRESS --> AUTH

    HB --> REG
    REG --> CL --> CLAUDE
    REG --> CDX --> CODEX
    REG --> GEM --> GEMINI_RT
    REG --> OCG --> OC_GW

    Services --> DRIZZLE --> PG
    Services --> STORAGE

    PLUGINS -- "IPC (child_process)" --> PW1
    PLUGINS -- "IPC (child_process)" --> PW2

    HB -- "publishLiveEvent" --> WS
```

---

## 2. コンポーネント一覧

| コンポーネント | 職責 | 関鍵ファイル / ディレクトリ | 上流依存 | 下流依存 |
|-------------|------|--------------------------|---------|---------|
| **React SPA** | ダッシュボード UI・リアルタイム表示 | `ui/src/` | — | Express API, WebSocket |
| **CLI** | セットアップ・ヘルスチェック・タスク管理 | `cli/src/` | — | Express API |
| **Express App** | HTTP ルーティング・Middleware 管理 | `server/src/app.ts`, `server/src/routes/` | React SPA, CLI | Service Layer, Auth |
| **Better Auth** | 認証・セッション管理 | `server/src/auth/` | Express | DB (sessions テーブル) |
| **Heartbeat Engine** | エージェント起動・ライフサイクル管理（中核） | `server/src/services/heartbeat.ts` | Adapter Registry, Budget Service | DB, WebSocket, Storage |
| **Adapter Registry** | 異種エージェントランタイムの統一インターフェース | `server/src/adapters/registry.ts` | Heartbeat Engine | 各 Adapter パッケージ |
| **Adapter パッケージ群** | 各エージェントランタイム実行 | `packages/adapters/*/` | Adapter Registry | エージェント CLI / Webhook |
| **Budget Service** | 階層的予算ポリシー適用 | `server/src/services/budgets.ts` | Heartbeat Engine | DB (cost_events, budget_policies) |
| **Issue Service** | タスク管理・楽観ロック checkout | `server/src/services/issues.ts` | Routes, Heartbeat Engine | DB (issues テーブル) |
| **Plugin Worker Manager** | out-of-process プラグイン管理 | `server/src/services/plugin-worker-manager.ts` | App 初期化 | Plugin Worker プロセス (IPC) |
| **Routine Service** | cron / webhook トリガーによる定期 Issue 生成 | `server/src/services/routines.ts` | App 初期化 | Issue Service, Heartbeat Engine |
| **Recovery System** | 起動時の孤立実行検出・復旧 | `server/src/services/recovery/` | App 初期化 (startup) | DB, Heartbeat Engine |
| **Approval Service** | ガバナンス承認フロー管理 | `server/src/services/approvals.ts` | Routes | DB (approvals テーブル) |
| **Activity Log** | 全操作の監査ログ・プラグインイベントバス橋渡し | `server/src/services/activity-log.ts` | 全 Services | DB, Plugin Event Bus |
| **WebSocket (realtime)** | UI へのリアルタイムイベント Push | `server/src/realtime/` | Heartbeat Engine, Services | React SPA (LiveUpdatesProvider) |
| **DB スキーマ** | Drizzle テーブル定義・マイグレーション | `packages/db/src/schema/` (~70テーブル) | — | ORM 経由で全 Services |
| **Plugin SDK** | プラグイン開発者向け API 定義 | `packages/plugins/sdk/src/` | — | Plugin Worker プロセス |
| **Shared パッケージ** | 型定義・定数・ユーティリティ | `packages/shared/src/` | — | server, ui, cli, adapters |
| **MCP Server** | エージェント向け MCP ツール提供 | `packages/mcp-server/` | Plugin Worker | エージェントランタイム |

---

## 3. 分層設計 / Module Boundary

```
┌─────────────────────────────────────────────────────┐
│ Presentation Layer                                   │
│  React SPA (ui/)          CLI (cli/)                │
│  TanStack Query / Router  Commander.js              │
└───────────────────────┬─────────────────────────────┘
                        │ HTTP / WebSocket
┌───────────────────────▼─────────────────────────────┐
│ API Gateway Layer (server/src/app.ts + routes/)      │
│  Express Middleware Stack                            │
│  Route Handlers (薄いラッパー、Service 呼び出しのみ)   │
│  Authentication (Better Auth)                        │
└───────────────────────┬─────────────────────────────┘
                        │ 関数呼び出し
┌───────────────────────▼─────────────────────────────┐
│ Service Layer (server/src/services/)                 │
│  Heartbeat Engine  Budget Service  Issue Service     │
│  Plugin Manager    Routine Service Recovery System   │
│  Approval Service  Activity Log    Cost Service      │
│  【ルール】: 他 Service を直接インポート可、Route は不可│
└──────────┬──────────────────────┬────────────────────┘
           │ Adapter 呼び出し       │ ORM クエリ
┌──────────▼──────────┐  ┌────────▼───────────────────┐
│ Adapter Layer        │  │ Data Access Layer           │
│ (server/src/adapters/│  │ Drizzle ORM                │
│  + packages/adapters)│  │ packages/db/src/schema/    │
│  統一インターフェース  │  │ ~70テーブル                 │
└──────────┬──────────┘  └────────┬───────────────────┘
           │ child_process / HTTP  │ SQL
┌──────────▼──────────┐  ┌────────▼───────────────────┐
│ Agent Runtime Layer  │  │ Storage Layer              │
│ Claude Code CLI      │  │ PostgreSQL (embedded / ext)│
│ Codex / Gemini CLI   │  │ File Storage (local / S3)  │
│ OpenClaw Webhook     │  └────────────────────────────┘
└─────────────────────┘
```

**Module Boundary ルール**:
- Route Handler は Service のみ呼び出す（DB に直接アクセスしない）
- Service 層は Adapter Registry 経由でのみエージェントランタイムを起動する
- Plugin Worker は IPC (child_process) 経由でのみ Host Services にアクセスし、DB に直接触れない
- 共有型は `packages/shared/` に集約し、循環依存を防ぐ

---

## 4. 通信パターン

| パターン | 使用箇所 | 実装 |
|---------|---------|------|
| **Sync HTTP (REST)** | UI ↔ API, CLI ↔ API | Express.js + TanStack Query |
| **WebSocket (Push)** | サーバー → UI のリアルタイムイベント | `server/src/realtime/` + `LiveUpdatesProvider` |
| **DB-backed Queue (Async)** | Heartbeat 実行キュー | `agentWakeupRequests` → `heartbeatRuns` テーブル |
| **IPC (child_process)** | Plugin Worker ↔ Host Services | `plugin-worker-manager.ts` + Plugin SDK |
| **HTTP Webhook (Fire-and-forget)** | `openclaw_gateway` アダプター | OpenClaw エージェントへの非同期通知 |
| **Pub/Sub (Domain Events)** | Plugin イベントシステム | `plugin-event-bus.ts` → Plugin Worker |
| **Optimistic Lock (DB)** | Issue checkout の排他制御 | `UPDATE WHERE status = ANY(expectedStatuses)` |
| **JWT** | エージェント ↔ API 認証 | `createLocalAgentJwt()` でエージェントごとに発行 |

---

## 5. 関鍵設計決策と Trade-off

### 5.1 DB-backed Queue（永続キュー）
**決策**: エージェント起動リクエストをインメモリキューではなく `agentWakeupRequests` テーブルに書き込む。

**根拠**: VM / コンテナが ephemeral な環境でサーバーが予期せず再起動しても、未処理のリクエストが失われない。

**Trade-off**: DB I/O が増加するが、信頼性を優先。

### 5.2 Embedded PostgreSQL
**決策**: `DATABASE_URL` 未設定時は `embedded-postgres` パッケージで PostgreSQL をプロセス内起動。

**根拠**: ゼロ設定でのローカル起動を実現（`paperclipai run` だけで動作）。

**Trade-off**: 本番環境では外部 Postgres への切り替えが必要。マルチインスタンス構成不可。

### 5.3 out-of-process Plugin Worker
**決策**: プラグインを親プロセスと同一プロセスではなく、子プロセスとして起動。

**根拠**: プラグインのクラッシュがサーバー本体に影響しない。能力ゲート（Capability Gate）によるサンドボックス化が可能。

**Trade-off**: IPC オーバーヘッドが発生。プラグインのデバッグが若干複雑。

### 5.4 Adapter Pattern for Agent Runtimes
**決策**: 各エージェントランタイムを `ServerAdapterModule` インターフェース経由で統一。

**根拠**: Claude Code、Codex、Gemini 等を同一コアロジックで扱える。新規エージェント追加時の変更範囲を Adapter パッケージに限定。

**Trade-off**: ⚠️ 未驗證 — 各ランタイムの高度な機能（ストリーミング応答の差異等）がインターフェースに収まりきらない可能性。

### 5.5 9000 行モノリシック Heartbeat Service
**決策**: `heartbeat.ts` がエージェントライフサイクルの全フェーズを一ファイルで管理。

**根拠**: コンテキスト切り替えコスト削減。ハートビートの全ステップをトレースしやすい。

**Trade-off**: ファイルサイズが巨大で変更時の競合リスクが高い。将来的な分割リファクタリングが課題。

### 5.6 Deployment Mode（local_trusted / authenticated）
**決策**: デフォルトは認証なしの `local_trusted` モード。

**根拠**: ローカル開発者の摩擦を最小化。シングルユーザー前提ならログイン不要。

**Trade-off**: LAN / インターネット露出時は明示的に `authenticated` モードへの切り替えが必要。

---

## 6. Heartbeat 実行シーケンス図

```mermaid
sequenceDiagram
    participant Timer as タイマー / イベント
    participant HB as Heartbeat Engine
    participant DB as PostgreSQL
    participant Budget as Budget Service
    participant Adapter as Adapter Registry
    participant Agent as エージェントランタイム<br/>(例: Claude Code CLI)
    participant WS as WebSocket (realtime)
    participant UI as React SPA

    Timer->>HB: drainWakeupQueue() 呼び出し
    HB->>DB: agentWakeupRequests を SELECT
    HB->>DB: heartbeatRuns INSERT (status: queued)

    HB->>DB: issues.checkout() — 楽観ロック<br/>UPDATE WHERE status = ANY(expected)
    DB-->>HB: checkout 成功 / 失敗

    HB->>Budget: getInvocationBlock(agentId, companyId)
    Budget->>DB: 予算ポリシー・使用量を確認
    Budget-->>HB: blocked: false (または blocked: true + reason)

    alt ブロックなし
        HB->>DB: agent_task_sessions から sessionId 取得
        HB->>HB: buildPaperclipWakePayload()<br/>(company, issue, skills, secrets, workspace 等を組み立て)
        HB->>DB: heartbeatRuns UPDATE (status: running)
        HB->>WS: publishLiveEvent("run.started")
        WS-->>UI: リアルタイム更新通知

        HB->>Adapter: execute(AdapterExecutionContext)
        Adapter->>Agent: CLI 起動 / Webhook 送信

        loop エージェント実行中
            Agent-->>Adapter: stdout / 進捗イベント
            Adapter-->>HB: ストリーミング結果
            HB->>WS: publishLiveEvent("run.progress")
            WS-->>UI: ログ・進捗表示更新
        end

        Agent-->>Adapter: 完了 (AdapterExecutionResult)
        Adapter-->>HB: result

        HB->>DB: cost_events INSERT (トークン使用量記録)
        HB->>DB: agent_task_sessions UPDATE (sessionId 保存)
        HB->>DB: heartbeatRuns UPDATE (status: completed)
        HB->>DB: issues UPDATE (status 更新)
        HB->>WS: publishLiveEvent("run.completed")
        WS-->>UI: 完了通知・結果表示

    else 予算ブロック
        HB->>DB: heartbeatRuns UPDATE (status: blocked)
        HB->>WS: publishLiveEvent("run.blocked")
        WS-->>UI: ブロック通知表示
    end
```

---

## 補足: 主要 DB テーブル関係（概略）

```mermaid
graph TD
    companies["companies"] --> agents["agents"]
    companies --> projects["projects"]
    projects --> issues["issues"]
    agents --> issues
    issues --> heartbeatRuns["heartbeat_runs"]
    heartbeatRuns --> heartbeatRunEvents["heartbeat_run_events"]
    agents --> agentWakeupRequests["agent_wakeup_requests"]
    issues --> issueTreeHolds["issue_tree_holds"]
    companies --> budgetPolicies["budget_policies"]
    agents --> agentTaskSessions["agent_task_sessions"]
    heartbeatRuns --> costEvents["cost_events"]
    companies --> goals["goals"]
    goals --> issues
```

---

*生成日: 2026-05-05 — ソース: `.trace/_context/` コンテキストファイル群 + コードベース直接読取*
