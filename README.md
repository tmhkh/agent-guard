# AgentGuard 仕様駆動開発ドキュメント

AgentGuard（`agent-guard`）は、AI コーディングエージェント（Kiro / Claude Code）の設定ファイルを静的解析して、次のような記述を検出するツールです。検出したものについて「何が危ないか」と「どう直すか」を日本語で示します。

- 過剰な権限・自動承認
- シークレットの露出
- サプライチェーンリスク
- 危険なコマンド
- プロンプトインジェクションを誘発する記述

- 中核: Python 3.12 製 CLI（日本語テキスト / JSON / SARIF レポート）
- 検出ルール: YAML のルールパック（データ駆動）
- 付属: CLI が出力した JSON を閲覧するローカル・読み取り専用ビューア（FastAPI + React + MUI）
- AWS デプロイ・認証・LLM 解説生成は MVP のスコープ外

ドキュメント体系は `agentic-web` と同じ構成です。番号は開発工程の順番を表します。

> 要件定義は Kiro spec（`.kiro/specs/agent-guard/requirements.md`）から移行し、章単位で分割しています。
> Requirement 番号は全ファイルを通した通し番号です。

---

## 1. ドキュメント体系

```
agent-guard/
├─ README.md                      ← 本ファイル（入口・読む順番）
├─ rules.md                       ← チーム共有ルール（人間 & AI 共通）
├─ glossary.md                    ← 用語集
│
├─ 01_requirements/               ← ① 要件定義（WHAT / WHY）
│   ├─ 00_project-overview.md         プロジェクト概要
│   ├─ 01_scope.md                    スコープ外（MVP）
│   ├─ 03_functional-requirements.md  機能要件（Requirement 1〜19・24・25）
│   ├─ 04_non-functional-req.md       非機能要件（Requirement 20〜23・26）
│   ├─ 06_constraints-assumptions.md  決定事項と前提（D-1〜D-12）
│   └─ 07_initial-rule-catalog.md     初期ルールカタログ（案）
│
├─ 02_architecture/               ← ② 概要設計（未作成）
├─ 03_detailed-design/            ← ③ 詳細設計（未作成）
├─ 04_quality/                    ← ④ 品質担保（未作成）
├─ 05_implementation/             ← 実装ガイド（予約カテゴリ）
├─ 06_infrastructure-deployment/  ← CI / 配布（予約カテゴリ）
├─ 07_operations/                 ← 運用・保守（予約カテゴリ）
│
└─ 08_process/                    ← ⑤ 進め方
    ├─ adr/                           意思決定記録
    └─ templates/                     ADR・変更要求・議事録テンプレート
```

`02`〜`07` は工程カテゴリの定位置として確保しています。中身は各フォルダの `README.md` を参照してください。

## 2. 読む順番

| 役割 | 最初に読むべきドキュメント |
|------|---------------------------|
| 全員 | `rules.md` → `glossary.md` → `01_requirements/00_project-overview.md` |
| PM / PdM | `01_requirements/01_scope.md`、`01_requirements/06_constraints-assumptions.md` |
| 開発者・AIエージェント | `rules.md` → `01_requirements/03_functional-requirements.md` → `04_non-functional-req.md` |

## 3. ステータス定義

| ステータス | 意味 | 下流作業の可否 |
|-----------|------|---------------|
| `Draft` | 作成中 | 実装不可 |
| `Reviewing` | レビュー中 | 実装不可 |
| `Agreed` | 合意済み | 実装可 |
| `Frozen` | 変更凍結 | 実装可・変更は変更管理経由 |

各ドキュメント冒頭のメタ情報ブロックに記載します。

```md
---
status: Draft            # Draft | Reviewing | Agreed | Frozen
owner: {{担当者名}}
reviewers: [{{レビュアー}}]
last_updated: YYYY-MM-DD
related: [関連ドキュメントのパス]
---
```
