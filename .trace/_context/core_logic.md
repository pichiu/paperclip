# 核心領域ロジック

## Paperclip の「心臓」

Paperclip の最もコアな非 trivial ロジックは **Heartbeat Execution Engine**（`server/src/services/heartbeat.ts`）である。このファイルは 9,000 行超のモノリシックなサービスで、エージェントのライフサイクル全体を管理する。

---

## 1. Heartbeat Execution Engine

### 核心パターン: DB-backed Queue + Atomic Execution Lock

```
agentWakeupRequests テーブル
    │  (INSERT when wakeup requested)
    ▼
heartbeatRuns テーブル (status: queued)
    │  (timer/event driven drain)
    ▼
execute() with atomic lock on issues
    │
    ▼
heartbeatRuns (status: running → completed/failed)
```

**なぜ DB-backed か？**: VM が ephemeral な環境でもキューが失われないように、インメモリキューではなくDBにキューを永続化している。

### 楽観ロック（Issue チェックアウト）

`server/src/services/issues.ts` の `checkout()` メソッドは、`expectedStatuses` を使った楽観ロックで二重チェックアウトを防ぐ。アトミックな `UPDATE WHERE status = ANY(expectedStatuses)` によって、複数エージェントが同じタスクを同時に取得することを防ぐ。

### セッション継続性（`agent_task_sessions` テーブル）

エージェントのハートビートをまたいでセッションを継続するための仕組み：
- `taskKey`: エージェント + タスクの組み合わせを一意識別するキー
- `sessionId`: Claude Code のセッション ID 等を保持
- 次のハートビートで `sessionId` を再利用することで、前回のコンテキストを引き継ぐ

### Context Snapshot の構築 (`buildPaperclipWakePayload`)

`server/src/services/heartbeat.ts:1789` でエージェントへの「覚醒ペイロード」を構築：

```typescript
{
  // 会社・プロジェクト・ゴール情報
  company: { id, name, goal },
  project: { id, name, description },
  
  // 現在のタスク
  issue: {
    id, identifier, title, description, status,
    parentIssue,      // 親タスクへの連鎖（goal alignment の根拠）
    goalContext,      // ゴール系統
  },
  
  // 組織コンテキスト
  agentConfig,       // エージェントの role/title/capabilities
  skills,            // 注入されたスキルマークダウン（Paperclip Skill等）
  
  // 実行環境
  workspace,         // CWD, git worktree パス等
  secrets,           // 展開済みシークレット
  
  // 前のランのサマリー（継続時）
  previousRunSummary,
}
```

これがエージェントへのプロンプトの「本文」となる。

---

## 2. Budget & Cost System

### 予算ポリシーの階層

`server/src/services/budgets.ts`:

```
Company レベル予算ポリシー
    ├── Project レベル予算ポリシー
    │       └── Agent レベル予算ポリシー
    └── Agent レベル予算ポリシー（会社直属）
```

各ポリシーは：
- `windowKind`: `monthly` または `lifetime`
- `hardStopAmount`: これを超えたらエージェントを強制停止
- `warningAmount`: 警告閾値

`getInvocationBlock()` はエージェントが実行を開始する前に全レベルのポリシーを確認し、ブロックすべき場合は `{ blocked: true, reason, scopeType, scopeId }` を返す。

### コストイベント記録

`cost_events` テーブルに以下の粒度で記録：
- company / agent / project / goal / issue / provider / model / billing_type

これにより、細粒度のコスト分析が可能。

---

## 3. Issue Tree Control System

`server/src/services/issue-tree-control.ts` が管理するツリーレベルの実行制御：

### Issue Tree Hold（ポーズ/停止）

Issue ツリー（親子関係）全体を一時停止する機能：
- `issue_tree_holds` テーブル: ホールドの定義
- `issue_tree_hold_members` テーブル: どの Issue がホールドに含まれるか

ホールド状態は、エージェントが新しいタスクを checkout する前にチェックされ、ツリーがホールド中なら checkout が拒否される。

### Issue Execution Policy

`server/src/services/issue-execution-policy.ts`:

各 Issue に `executionPolicy` JSONB フィールドがあり：
- `mode`: `default` | `manual_only` | `blocked`
- `stages`: 実行ステージの定義（例: 外部サービスモニタリング）
- `monitor`: スケジュールされた外部サービスチェック設定

---

## 4. Plugin System

### アーキテクチャ

プラグインは**out-of-process Worker**として動作（`server/src/services/plugin-worker-manager.ts`）：

```
Paperclip Server
    │
    ├── PluginWorkerManager
    │       └── Worker Process (Node.js child_process)
    │               ├── Plugin SDK (@paperclipai/plugin-sdk)
    │               └── Plugin Code (npm package)
    │
    └── Host Services (IPC via plugin-host-services.ts)
            ├── DB アクセス (company-scoped)
            ├── Storage アクセス
            ├── Job Scheduling
            └── Tool Registration
```

### プラグインライフサイクル (`plugin-lifecycle.ts`)

状態機械：

```
installed → ready → disabled
    │           │       │
    │           ├── error
    │           └── upgrade_pending
    └── uninstalled
```

- `installed` → `ready`: プラグインのワーカープロセスが起動
- `ready` → `disabled`/`error`: プロセスを graceful shutdown

### Capability Gate

`plugin-capability-validator.ts` がプラグインのマニフェストで宣言された `capabilities` を検証し、ホスト IPC 呼び出しが宣言外の能力を使おうとした場合はブロックする。

---

## 5. Governance & Approval System

`server/src/services/approvals.ts`:

承認が必要なアクション：
- 新規エージェント採用（`agent.hire`）
- CEO の初期戦略提案
- 設定変更（revision 管理）

承認フロー：
1. エージェントまたはシステムが承認リクエストを作成 (`approvals` テーブル)
2. Board ユーザーが `/api/approvals/:id/approve` または `/api/approvals/:id/reject` を呼ぶ
3. 承認後に対応するアクションが実行される（例: エージェントの `status` を `active` に変更）

`agent_config_revisions` テーブルでエージェント設定の変更履歴を管理し、ロールバック可能。

---

## 6. Recovery System

`server/src/services/recovery/` ディレクトリ:

孤立した実行（サーバー再起動で `running` のまま残ったラン）を検出・復旧：
- `reconcilePersistedRuntimeServicesOnStartup()`: 起動時に実行中のランをチェック
- ストランドしたランを `failed` に遷移させ、Recovery Issue を自動生成
- Watchdog がラン進捗を監視し、一定時間応答がないランを検出

---

## 重要な abstraction パターン

| パターン | 実装場所 | 用途 |
|--------|--------|------|
| **Adapter Pattern** | `server/src/adapters/registry.ts` | 異なるエージェントランタイムを統一インターフェースに |
| **Service Layer** | `server/src/services/` | ビジネスロジックとルートハンドラーの分離 |
| **Activity Log** | `server/src/services/activity-log.ts` | 全変更操作の監査ログ |
| **Live Events** | `server/src/realtime/` | WebSocket で UI にリアルタイム更新をプッシュ |
| **Optimistic Lock** | `issueService.checkout()` | 二重チェックアウト防止 |
| **State Machine** | `plugin-lifecycle.ts` | プラグインライフサイクル管理 |
