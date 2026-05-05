# Extension Points（拡張ポイント）

## 1. Adapter System（最も重要な拡張ポイント）

### 概要

新しいエージェントランタイムをサポートするには、**Adapter パッケージ**を作成して `packages/adapters/` に追加する。

### Adapter インターフェース (`packages/adapter-utils/src/types.ts`)

```typescript
interface ServerAdapterModule {
  execute(context: AdapterExecutionContext): Promise<AdapterExecutionResult>;
  testEnvironment(context): Promise<AdapterEnvironmentTestResult>;
  sessionCodec?: AdapterSessionCodec;  // セッション状態のシリアライズ/デシリアライズ
  listSkills?(context): Promise<AdapterSkillSnapshot>;
  syncSkills?(context): Promise<void>;
  listModels?(): Promise<AdapterModel[]>;
  getQuotaWindows?(context): Promise<QuotaWindow[]>;
}
```

`execute()` が核心：エージェントランタイムを起動し、`AdapterExecutionResult` を返す。

### 既存アダプター一覧

| アダプター | パッケージ | 起動方式 |
|----------|---------|---------|
| `claude_local` | `@paperclipai/adapter-claude-local` | ローカル Claude Code CLI を子プロセスで起動 |
| `codex_local` | `@paperclipai/adapter-codex-local` | ローカル Codex CLI を子プロセスで起動 |
| `cursor_local` | `@paperclipai/adapter-cursor-local` | Cursor API/CLI ブリッジ |
| `gemini_local` | `@paperclipai/adapter-gemini-local` | ローカル Gemini CLI を起動 |
| `openclaw_gateway` | `@paperclipai/adapter-openclaw-gateway` | HTTP Webhook（Fire-and-forget） |
| `opencode_local` | `@paperclipai/adapter-opencode-local` | ローカル OpenCode CLI を起動 |
| `pi_local` | `@paperclipai/adapter-pi-local` | ローカル Pi CLI を起動 |
| `acpx_local` | `@paperclipai/adapter-acpx-local` | ACPX ローカル実行 |

### Plugin-based External Adapter

`doc/adapter-plugin.md` に記載の通り、サーバー側に直接実装せずに**プラグイン経由でアダプターを提供**することも可能。これにより外部ベンダーが Paperclip Core を fork せずに独自アダプターを配布できる。

---

## 2. Plugin System

### プラグインの作成方法

`packages/plugins/sdk/src/index.ts` の `definePlugin()` を使用：

```typescript
import { definePlugin, runWorker } from "@paperclipai/plugin-sdk";

const plugin = definePlugin({
  async setup(ctx) {
    // イベントリスナー登録
    ctx.events.on("issue.created", async (event) => {
      // Issue が作成されたときの処理
    });
    
    // ジョブ登録（スケジューラーから呼ばれる）
    ctx.jobs.register("sync-data", async (job) => {
      // バックグラウンドジョブ処理
    });
    
    // ツール登録（エージェントから呼べる MCP ツール）
    ctx.tools.register("my-tool", async (params) => {
      // ツール実装
    });
    
    // Launcher 登録（UI にメニューアイテムを追加）
    ctx.launchers.register("my-launcher", ...);
  },
  
  async onHealth() {
    return { status: "ok" };
  },
});

export default plugin;
runWorker(plugin, import.meta.url);
```

### プラグインが利用できる Host Services

| サービス | 説明 |
|---------|------|
| `ctx.events` | Paperclip のドメインイベント購読 |
| `ctx.jobs` | ジョブの登録・スケジューリング |
| `ctx.tools` | MCP ツールの登録 |
| `ctx.state` | プラグイン状態ストア（key-value） |
| `ctx.db` | 制限付き DB アクセス（プラグインスキーマのみ） |
| `ctx.data` | データプロバイダー登録 |
| `ctx.launchers` | UI Launcher の登録 |
| `ctx.logger` | 構造化ログ |
| `ctx.secrets` | シークレット読み取り |

### プラグインのマニフェスト (`PaperclipPluginManifestV1`)

`plugin.json` または `package.json` の `paperclip` フィールド：

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "capabilities": ["events", "jobs", "tools"],
  "worker": "./dist/worker.js",
  "ui": "./dist/ui.js",
  "apiRoutes": [...],
  "webhooks": [...],
  "jobs": [...],
  "settings": { "schema": {...} }
}
```

`capabilities` は宣言した能力のみが使用可能（CapabilityDeniedError が投げられる）。

---

## 3. Skill System（エージェントスキル）

### スキルの仕組み

スキルはマークダウンドキュメントで、エージェントの起動時にプロンプトとして注入される。これにより、エージェントにコードを変更せずに新しい知識・手順を教えることができる。

**Paperclip Skill** (`skills/paperclip/SKILL.md`): Paperclip API の操作方法を教えるスキル。全エージェントに注入推奨。

### スキルの登録

1. **会社スキル** (`company_skills` テーブル): 特定の会社のエージェントに使用可能なスキルのリスト
2. **エージェントスキル**: 個別エージェントに関連付けられたスキル
3. **アダプタースキル**: アダプターが管理するスキル（Claude Code の場合はスキルファイル同期）

### スキルの発見・管理

- `GET /api/skills/index` — 利用可能スキル一覧
- `GET /api/skills/paperclip` — Paperclip ハートビートスキルのマークダウン
- アダプターの `listSkills()` / `syncSkills()` によりアダプター側のスキルと同期

---

## 4. Routine System（スケジュール済みルーティン）

ルーティンは「定期実行されるタスク定義」で、スケジュールに応じて自動的に Issue を作成してエージェントを起動する。

### Trigger 種別 (`routines.ts`)

| Trigger | 説明 |
|--------|------|
| `cron` | cron 式によるスケジュール |
| `webhook` | HTTP Webhook トリガー |
| `api` | Paperclip API からの手動トリガー |

### Routine 変数

ルーティンテンプレートに変数を埋め込める：
- `{{BRANCH}}` — 現在の git ブランチ
- カスタム変数定義

---

## 5. Execution Workspace Policy（実行ワークスペース）

プロジェクトに「実行ワークスペースポリシー」を設定することで、エージェントの作業場所を制御：

- **`none`**: ワークスペースなし（デフォルト作業ディレクトリ）
- **`shared`**: 全エージェントが共有ワークスペースを使用
- **`isolated`**: Issue ごとに git worktree を自動作成
- **`operator`**: オペレーター指定ブランチ

`workspaceStrategy.provisionCommand` を設定すると、ワークスペース作成後にカスタムコマンドを実行可能（例: `pnpm install`）。

---

## 6. Governance Hooks（ガバナンスフック）

### Board Approval Gates

`server/src/services/approvals.ts` の承認システムはフックポイントとして機能：

- エージェント採用時に Board 承認を必須化
- `agent.hire_hook.ts` (`server/src/services/hire-hook.ts`) で採用時のコールバックを定義

### Agent Execution Policy

各 Issue に `executionPolicy` を設定することで、実行フローをカスタマイズ：
- `stages`: 承認が必要なステージを追加
- `monitor`: 外部サービスのポーリングステージを定義
