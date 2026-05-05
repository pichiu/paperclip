# Trace Metadata

## 分支資訊
- **Base Branch**: claude/generate-codebase-docs-JPMK3
- **Trace Branch**: claude/generate-codebase-docs-JPMK3

## 最後 Trace 資訊
- **Base Commit Hash**: d6bee62f02de058a4f12ac9a10dedf8c58c4b1d5
- **日期**: 2026-05-05
- **Trace 類型**: full
- **涵蓋範圍**: 全部（server / ui / cli / packages / adapters / plugins）

## 文件清單

| 文件 | 對應 Base Commit | 最後更新日期 |
|------|-----------------|-------------|
| INDEX.md | d6bee62 | 2026-05-05 |
| ARCHITECTURE.md | d6bee62 | 2026-05-05 |
| DATA_MODEL.md | d6bee62 | 2026-05-05 |
| API_SURFACE.md | d6bee62 | 2026-05-05 |
| DEV_GUIDE.md | d6bee62 | 2026-05-05 |
| CODEBASE_MAP.md | d6bee62 | 2026-05-05 |
| DISCOVERY_LOG.md | d6bee62 | 2026-05-05 |

## 變更歷程

| 日期 | 類型 | Base Commit 範圍 | 更新的文件 | 摘要 |
|------|------|-----------------|-----------|------|
| 2026-05-05 | full | initial..d6bee62 | 全部 | 初次 trace |

## Trace 範囲メモ

プロジェクト規模: 2,007 ファイル（TypeScript 1,149 + TSX 304 + その他）

完全 trace 対象:
- `server/` — Express API サーバー（ルート・サービス・アダプター・middleware）
- `ui/` — React ダッシュボード
- `cli/` — CLI ツール
- `packages/db/` — DB スキーマ・マイグレーション
- `packages/shared/` — 共有型・定数
- `packages/adapter-utils/` — アダプターユーティリティ
- `packages/adapters/` — 全エージェントアダプター
- `packages/plugins/sdk/` — プラグイン SDK

部分 trace（ファイルが多く概要のみ）:
- `docs/` — Mintlify 公開ドキュメント（内容は README/既存ドキュメントと重複部分大）
- `evals/` — LLM 評価スクリプト（promptfoo）
- `tests/e2e/` — E2E テスト実装詳細
- `scripts/` — ビルド・リリーススクリプト詳細
