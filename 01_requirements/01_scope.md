---
status: Draft
owner: {{担当者名}}
reviewers: [{{レビュアー}}]
last_updated: 2026-11-01
related: [./00_project-overview.md, ./03_functional-requirements.md]
---

# スコープ定義

> 移行元: `.kiro/specs/agent-guard/requirements.md`（Kiro spec）。内容は原文のまま章単位で分割している。
> Requirement 番号は全要件を通した通し番号で、他ファイルの要件も「Requirement N」で参照する（1〜19・24・25 は機能要件、20〜23・26 は非機能要件）。

## スコープ外（MVP）

- 設定の自動修正（修正提案の提示のみ行う）
- Cursor / GitHub Copilot など Kiro / Claude Code 以外のエージェント対応（ルールパックとパーサの追加で拡張可能な構造にはする）
- AWS へのデプロイ、マルチユーザー、認証
- LLM による解説生成
- ネットワーク経由のパッケージ評判照会・脆弱性 DB 照会
