# DISCOVERY_LOG.md — 探索中の発見・問題・技術債記録

> 生成日: 2026-05-05
> 対象コミット: main ブランチ（探索時点）
> ⚠️ 未驗證 = このトレースセッション内で確認できなかった情報

---

## 1. Web Search 発見摘要

### プロジェクトの位置づけと成長速度

Paperclip は 2026年3月4日に公開され、3週間で GitHub Stars 30,000 超を達成した。外部メディアは "Open-Source Operating System for Multi-Agent Companies" と評している（Towards AI）。マーケティング上の核心フレーズは「If OpenClaw is an _employee_, Paperclip is the _company_」。

### ハートビートパターンの外部説明（MindStudio 記事より）

> "The heartbeat pattern is a scheduling and context-management mechanism that wakes an AI agent at regular intervals, injects fresh context into its working memory, and allows it to operate without any human trigger."

コードで確認済み：`server/src/services/heartbeat.ts`（9,000行超）がこのパターンを実装。`agentWakeupRequests` → `heartbeatRuns` → adapter 実行の DB-backed キューとして動作。

### バージョン形式の慣習

リリースは `v{year}.{dayOfYear}.{patch}` 形式（例: `v2026.318.0`）。日付ベースのバージョンニングで semver は不採用。

### コミュニティエコシステムの現状

- `https://github.com/gsxdsm/awesome-paperclip` にコミュニティプラグイン集あり
- ClipHub（ワンクリック会社テンプレート配布）は⚪ COMING SOON 状態
- `doc/CLIPHUB.md` に設計仕様あり。`docs/specs/cliphub-plan.md` は部分的に superseded

### ⚠️ 未驗證 の外部情報

- `paperclipai.net` という別ドメインが存在するが公式との関係不明
- `agencyenterprise/paperclip-ai` というフォークが存在するが活動状況・目的不明
- Discord `#dev` チャンネルへの PR 提出前の相談が推奨されているが、実際の応答速度・受け入れ基準は確認不能

---

## 2. 既存ドキュメントとコードの乖離

### 2.1 `billing_code` / `request_depth` — 実装済みだが UI 欠如

**ドキュメント**: `doc/SPEC.md` に `billing_code`（コスト帰属用コード）と `request_depth`（エージェント委任の深さ）が言及。

**コード実態**:
- `packages/db/src/schema/issues.ts:48-49` — `requestDepth`, `billingCode` カラムが DB スキーマに存在
- `packages/db/src/schema/cost_events.ts:19` — `billing_code` カラムあり
- `packages/db/src/schema/finance_events.ts:21` — `billing_code` カラムあり

**乖離**: UI（`ui/src/`）には `billing_code` / `request_depth` の表示・編集コンポーネントが見当たらない。⚠️ 未驗證 — データは蓄積されているが Board 画面で参照・利用できない可能性。

### 2.2 Budget Delegation（カスケード委任）— 設計は明記、実装は部分的

**ドキュメント**: `doc/SPEC.md` §1（Board Governance）に「CEO がエージェントに予算を委任でき、マネージャーエージェントもレポートに委任可能」と記載。

**コード実態**:
- `server/src/services/budgets.ts` は Company / Project / Agent の3階層ポリシーを実装
- `getInvocationBlock()` で全階層を確認する仕組みあり

**乖離**: SPEC の「cascading budget delegation」（CEO→部下への連鎖委任）が完全実装されているかは ⚠️ 未驗證。`doc/SPEC.md` 自体に「How this cascading budget delegation works in practice is TBD」と明記されており、仕様レベルで未確定。

### 2.3 `doc/SPEC.md` の V1 チェックリスト — 一部 DRAFT 状態のまま残存

`doc/SPEC.md` §9（Frontend/UI）と §10（V1 Scope）は `[DRAFT]` マークが付いたまま。チェックリスト形式の V1 Must Have 項目も `[ ]` 未チェックのものが多く残る。これは仕様書としての信頼性に疑問を生じさせる — 実装は完成しているが文書がメンテされていない可能性。

### 2.4 `doc/plans/` の設計計画ドキュメント — 一部 superseded

`doc/AGENTCOMPANIES_SPEC_INVENTORY.md` で `docs/specs/cliphub-plan.md` が "Earlier blueprint bundle plan; partially superseded" と明記。`doc/plans/` 内の各計画ドキュメント（例: `2026-02-16-module-system.md`）が現在の実装とどの程度整合しているか、個別確認が必要。

### 2.5 `server/src/services/github-fetch.ts` の用途が不明確

**コード実態**:
- `server/src/services/company-portability.ts` と `server/src/services/company-skills.ts` が `ghFetch`, `gitHubApiBase`, `resolveRawGitHubUrl` をインポートしている

**乖離**: `doc/` 内のドキュメントには GitHub 連携の詳細説明なし。Company Import/Export で GitHub 上のスキルや会社テンプレートを直接取得する機能と推測されるが、⚠️ 未驗證。

---

## 3. TODO / FIXME / HACK 彙整

コードベース全体（`server/src/`, `packages/`, `ui/src/`, `cli/src/`）を検索した結果、明示的な `TODO/FIXME/HACK` は少ないが、以下の重要なものを確認:

| 場所 | 種別 | 内容 |
|------|------|------|
| `ui/src/adapters/runtime-json-fields.tsx:5` | TODO | `// TODO(issue-worktree-support): re-enable this UI once the workflow is ready to ship.` — Issue-scoped worktree の UI が無効化されたまま |
| `ui/src/pages/AgentDetail.tsx:882` | TODO | `// } else if (activeView === "skills") { // TODO: bring back later` — AgentDetail ページのスキルビューが無効化 |
| `cli/src/commands/client/company.ts:383` | TODO | `// TODO: replace this temporary claude_local fallback with adapter selection in the import TUI.` — Company import TUI がアダプター選択なしで `claude_local` にフォールバック |
| `server/src/services/plugin-loader.ts:170` | 未実装フラグ | `Registry support is not yet implemented; this field is reserved.` — プラグインリモートレジストリ機能が予約済みだが未実装 |
| `server/src/services/plugin-loader.ts:1038` | 未実装フラグ | `"plugin-loader: remote registry discovery is not yet implemented"` — 実行時にもエラーを出す |
| `ui/src/pages/RoutineDetail.tsx:1010` | UI フラグ | `{kind === "webhook" ? " — COMING SOON" : ""}` — Routine の Webhook トリガーが UI で COMING SOON 表示 |
| `ui/src/adapters/metadata.ts:26,40` | 設計フラグ | `comingSoon` フラグによりアダプターを無効化する仕組みあり — どのアダプターが COMING SOON かは要確認 |

---

## 4. 未解答の疑問

### アーキテクチャ上の疑問

**Q1. Plugin Remote Registry の設計意図**
`plugin-loader.ts` に `"registry"` タイプが type union に含まれ `Registry support is not yet implemented` コメントがある。ClipHub との統合を想定しているのか、独立したプラグインレジストリなのか不明。

**Q2. GitHub Integration の全体像**
`github-fetch.ts` が Company Portability と Company Skills から使われているが、認証フロー（GitHub トークンの管理場所）が不明。エージェントが GitHub PR を操作する場合と、Board が GitHub からスキルをインポートする場合で認証経路が異なる可能性。

**Q3. `embedded-postgres@18.1.0-beta.16` の「パッチ済み」の内容**
`recon.md` に「パッチ済み」と記載があるが、どのような変更が加えられているかは確認していない。本番利用では外部 Postgres を推奨しているが、embedded の beta 版を本番で使うリスクは ⚠️ 未驗證。

**Q4. `feedbackService` の動作モード**
`server/src/index.ts` の起動フローに `feedbackService(db)` があるが、`integrations.md` の記述では `server/src/services/feedback-share-client.ts` のフラッシュ間隔が 5000ms とある。フィードバックデータの送信先・プライバシー境界が `telemetry` と同様に適切に制御されているか ⚠️ 未驗證。

**Q5. `MAXIMIZER MODE` の実装計画**
`ROADMAP.md` に「⚪ MAXIMIZER MODE」とある。「more aggressive delegation, deeper follow-through」の具体的な実装イメージ（例: 並列実行数の制限緩和、承認ゲートのスキップ、自律エスカレーション閾値の変更）が不明。

**Q6. `acpx_local` アダプターの "ACPX" 正体**
`packages/adapters/acpx-local/` に `acpx_local` アダプターが存在するが、`recon.md` や外部記事に説明がない。⚠️ 未驗證。

**Q7. `pi_local` アダプターの "Pi" エージェントの正体**
同様に `packages/adapters/pi-local/` の Pi エージェントが何を指すか不明。Claude Pi モデルか、独立したエージェントランタイムか ⚠️ 未驗證。

---

## 5. 既知の技術債

### 5.1 `heartbeat.ts` の肥大化（9,000行超のモノリシックサービス）

**場所**: `server/src/services/heartbeat.ts`

**問題**: エージェントのライフサイクル全体（スケジューリング、コンテキスト構築、アダプター呼び出し、コスト記録、復旧、ウォッチドッグ）が単一ファイルに集中している。9,000行超は明らかに SRP（単一責任原則）違反。テストしにくく、変更リスクが高い。

**影響範囲**: `data_flow.md` で示した Issue Checkout → Agent Execution の全フローがこのファイルに依存。

### 5.2 Plugin Registry（リモート）が未実装のまま型定義に混在

**場所**: `server/src/services/plugin-loader.ts:110,170,1038`

```typescript
| "registry";  // future: remote plugin registry URL
```

type union に将来用途の値が入っており、実行時にエラーを投げる。dead code として残っており、将来の実装者に混乱を与える。ClipHub の設計（`doc/CLIPHUB.md`）との接続点が設計済みだが未実装。

### 5.3 Issue Worktree UI が無効化されたまま

**場所**: `ui/src/adapters/runtime-json-fields.tsx:5`

`issue-worktree-support` というラベル付きで UI が無効化されている。`doc/plans/2026-03-10-workspace-strategy-and-git-worktrees.md` に設計計画があるが、実装が完了しているか不明。バックエンド側では `workspace_strategy = 'isolated'` が動作する可能性があるが、UI からの設定操作が欠けている。

### 5.4 AgentDetail スキルビューの無効化

**場所**: `ui/src/pages/AgentDetail.tsx:882`

エージェント詳細ページの「skills」タブが無効化されている。Skills Manager は ROADMAP.md で ✅ 完了扱いだが、エージェント単位でのスキルを詳細表示する UI サーフェスが欠けている。

### 5.5 Company Import TUI のアダプター選択省略

**場所**: `cli/src/commands/client/company.ts:383`

会社インポート CLI の TUI が `claude_local` に一時フォールバックする。複数アダプターをサポートするエコシステムで、インポート時のアダプター選択がハードコードされている状態は、Codex・Gemini・OpenClaw ユーザーには機能制限となる。

### 5.6 `doc/SPEC.md` の複数箇所に `[DRAFT]` と `TBD` が残存

**場所**: `doc/SPEC.md` §1, §2, §9, §10

Board Governance の Approval Gates（「他のガバナンスゲートアクションは TBD」）、Budget Delegation（「TBD」）、Frontend Views（`[DRAFT]`）など、実装が進んだ後もドキュメントが更新されていない。新規参入者に誤った印象を与えるリスク。

### 5.7 `embedded-postgres` の beta バージョン使用

**場所**: `packages/db/` の依存関係 `embedded-postgres@18.1.0-beta.16`

ローカル開発向けとして合理的だが、beta リリースであることはプロダクション安定性のリスク。`doc/DEVELOPING.md` と `doc/DATABASE.md` に「本番は外部 Postgres を推奨」と明記されているが、デプロイガイドを読まないユーザーがデフォルトで beta embedded Postgres を本番利用するケースが考えられる。

---

## 6. さらに深く調査すべき区域

このトレースでカバーが不十分だった重要な領域:

| 区域 | 場所 | 理由 |
|------|------|------|
| **Evals フレームワーク** | `evals/` | `doc/plans/2026-03-13-agent-evals-framework.md` が存在。promptfoo ベースの LLM 評価基盤があるが内容未確認 |
| **Recovery システムの詳細** | `server/src/services/recovery/` | Watchdog・孤立ラン検出・自動 Recovery Issue 生成の具体的な閾値や実装が未確認 |
| **Memory Landscape** | `doc/memory-landscape.md` | メモリ/コンテキスト設計ドキュメントが存在するが内容未読。ROADMAP の「Memory/Knowledge」未実装項目と関連 |
| **Execution Semantics** | `doc/execution-semantics.md` | 実行セマンティクスの詳細仕様。ハートビート間の Issue 状態遷移の formal な定義が含まれる可能性 |
| **Smart Model Routing** | `doc/plans/2026-04-06-smart-model-routing.md` | モデル選択の自動化計画が 2026年4月に計画されたが、実装有無が不明 |
| **VS Code Task Interoperability** | `doc/plans/2026-04-12-vscode-task-interoperability-plan.md` | VS Code タスク連携計画が存在するが内容・進捗未確認 |
| **Agent OS Technical Report** | `doc/plans/2026-04-08-agent-os-technical-report.md` | エージェントOSとしての技術的検討の最新報告書。アーキテクチャの将来方向を理解する上で重要 |
| **MCP Server 詳細** | `packages/mcp-server/` | Paperclip データを MCP として公開する実装の詳細（公開するツール・リソース定義）が未確認 |
| **Skills ディレクトリ** | `skills/` | エージェントスキルの実際の markdown 定義群。Paperclip Skill（`skills/paperclip/SKILL.md`）の内容未確認 |
| **テスト・Eval カバレッジ** | `tests/`, `evals/` | E2E テスト（Playwright）と promptfoo evals の実際の網羅範囲が未確認 |
| **Untrusted PR Review** | `doc/UNTRUSTED-PR-REVIEW.md` | セキュリティ観点で重要な PR レビューポリシーが未読 |

---

## 7. 維護者・コミュニティへの確認事項

Discord `#dev` チャンネルまたは GitHub Issue で確認すべき点:

### 設計・ロードマップ

1. **ClipHub / Plugin Remote Registry の統合計画**: `doc/CLIPHUB.md` が設計されているが、`plugin-loader.ts` の `"registry"` type と連携するのか、独立したシステムか確認が必要。
2. **MAXIMIZER MODE の具体的な設計**: ロードマップに記載されているが、技術的な変更点（並列実行、承認ゲートの変更、ガバナンス境界）が不明。コア変更かプラグインとして実装するか。
3. **Budget Delegation（cascading）の実装優先度**: `doc/SPEC.md` §1 で「TBD」と記載。CEO から下位エージェントへの予算委任がどの Milestone で実装される予定か。

### 技術的な懸念

4. **`heartbeat.ts` のリファクタリング計画**: 9,000行超のファイルを分割する計画があるか。サービス分割の方針（例: SchedulerService, AdapterOrchestratorService 等）についての議論があれば参照先を教示してほしい。
5. **`acpx_local` / `pi_local` アダプターの正式名称と対象エージェント**: 外部ドキュメントに説明がなく、どのエージェントランタイムを対象としているか不明。
6. **Issue Worktree UI の再有効化時期**: `ui/src/adapters/runtime-json-fields.tsx` の `TODO(issue-worktree-support)` はいつ有効化される予定か。バックエンドは既に動作しているか。
7. **`embedded-postgres` beta からの移行計画**: `18.1.0-beta.16` の stable リリースを待っているのか、別の embedded DB（PGlite 等）への移行を検討しているか。

### ドキュメント整備

8. **`doc/SPEC.md` の `[DRAFT]` セクション更新**: §9, §10 の DRAFT マークはいつ取り除かれる予定か。現在の実装状態を反映したドキュメント更新の優先度について。
9. **`billing_code` / `request_depth` の UI サーフェス**: DB には存在するがダッシュボードで参照できない。将来の Cost Dashboard や Billing Ledger 機能（`doc/plans/2026-03-14-billing-ledger-and-reporting.md`）での利用を計画しているか。

---

*このドキュメントはトレースセッションの知識整理用であり、公式の技術ドキュメントではありません。⚠️ 未驗證 とマークした情報はコードやドキュメントで直接確認してください。*
