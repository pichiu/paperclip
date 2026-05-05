# Web Search 発見まとめ

## 検索結果サマリー

### 1. プロジェクト公式情報

| リソース | URL | 内容 |
|---------|-----|------|
| 公式サイト | https://paperclip.ing/ | ランディングページ |
| 公式ドキュメント | https://docs.paperclip.ing/ | Mintlify ベースの公開ドキュメント |
| GitHub | https://github.com/paperclipai/paperclip | ソースコード |
| アーキテクチャドキュメント | https://docs.paperclip.ing/start/architecture | 公式アーキテクチャ説明 |
| Discord | https://discord.gg/m4HZY7xNG3 | コミュニティ |
| Twitter/X | https://x.com/papercliping | 公式アカウント |

### 2. 記事・解説

| 記事 | URL | キー takeaway |
|-----|-----|--------------|
| Towards AI 解説 | https://pub.towardsai.net/paperclip-the-open-source-operating-system-for-zero-human-companies-2c16f3f22182 | "Open-Source Operating System for Multi-Agent Companies" として位置づけ |
| MindStudio ハートビート解説 | https://www.mindstudio.ai/blog/what-is-heartbeat-pattern-paperclip-ai-agents-24-7 | ハートビートパターンの詳細な説明 |
| MindStudio AI チーム構築ガイド | https://www.mindstudio.ai/blog/how-to-build-multi-agent-company-paperclip-claude-code | Claude Code + Paperclip の実践ガイド |
| DEV Community Deep Dive | https://dev.to/truongpx396/paperclip-deep-dive-a-build-guide-for-an-ai-company-control-plane-dda | コードレベルの深掘り解説 |
| Zeabur デプロイガイド | https://zeabur.com/blogs/deploy-paperclip-ai-agent-orchestration | Docker デプロイ手順 |
| Flowtivity 解説 | https://flowtivity.ai/blog/zero-human-company-paperclip-ai-agent-orchestration/ | ビジネス向け解説 |
| Medium 解説 | https://medium.com/@creativeaininja/paperclip-the-open-source-platform-turning-ai-agents-into-an-actual-company-7348015c5bf7 | 2026年3月の解説記事 |
| jimmysong.io | https://jimmysong.io/ai/paperclip/ | 日本語圏向け解説（実際は中国語） |

### 3. キー発見事項

#### 発足と成長
- 開発者: 匿名（@dotta）
- 公開: 2026年3月4日
- 3週間で GitHub Stars 30,000 超（2026年Q1 最速の OSS AI リポジトリの一つ）

#### ハートビートパターンについての説明（MindStudio）
> "The heartbeat pattern is a scheduling and context-management mechanism that wakes an AI agent at regular intervals, injects fresh context into its working memory, and allows it to operate without any human trigger."

これは Paperclip の核心的な設計パターン。エージェントを「スケジュール駆動」で起動し、セッション状態を維持することで、24/7 自律運用を実現する。

#### アーキテクチャドキュメント（公式）
https://docs.paperclip.ing/start/architecture

Paperclip の公式アーキテクチャドキュメントが存在し、主要コンポーネントの関係を説明している:
- Control Plane（Paperclip Server）
- Agent Adapters（Claude Code, Codex, OpenClaw, etc.）
- Board Dashboard（React UI）
- Storage（Embedded Postgres or external）

#### Awesome Paperclip（コミュニティプラグイン集）
- https://github.com/gsxdsm/awesome-paperclip

コミュニティが作成したプラグインやツールのキュレーションリスト。プラグインエコシステムの存在を示す。

#### GitHub Releases
- https://github.com/paperclipai/paperclip/releases/tag/v2026.318.0
- リリースはバージョン形式 `v{year}.{dayOfYear}.{patch}` を使用

### 4. 関連・競合プロジェクト

| プロジェクト | 関係 |
|------------|------|
| OpenClaw | Paperclip が使用するエージェントの一つ（OpenClaw ≒ 従業員、Paperclip ≒ 会社） |
| Claude Code (Anthropic) | Paperclip が使用する主要なエージェントの一つ |
| OpenAI Codex | サポートされるエージェント |
| Cursor | サポートされるエージェント（cursor_local adapter） |
| OpenCode | サポートされるエージェント |

### 5. コミュニティとサポート

- Discord の #dev チャンネルが主要な開発者議論の場
- Contribution は PR テンプレート + Greptile スコア 5/5 が必須
- プラグインシステムを通じた拡張が推奨（コアへの直接 PR は慎重に）

### 6. 注意事項・未解決の疑問

- `paperclipai.net` という別ドメインが存在するが公式との関係は ⚠️ 未確認
- `agencyenterprise/paperclip-ai` というフォークが存在する
- ClipMart（ワンクリックで会社テンプレートを配布するマーケットプレイス）は COMING SOON 状態
