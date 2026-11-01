---
status: Draft
owner: {{担当者名}}
reviewers: [{{レビュアー}}]
last_updated: 2026-11-01
related: [./03_functional-requirements.md]
---

# 初期ルールカタログ（案）

> 移行元: `.kiro/specs/agent-guard/requirements.md`（Kiro spec）。内容は原文のまま章単位で分割している。
> Requirement 番号は全要件を通した通し番号で、他ファイルの要件も「Requirement N」で参照する（1〜19・24・25 は機能要件、20〜23・26 は非機能要件）。


組み込み Rule_Pack に含める初期ルールの一覧（要件レビュー用）。Rule ID の体系は `AG-<カテゴリ>-<連番>` とする。最終的な ID・パターンは設計フェーズで確定する。

| Rule ID | 概要 | 対象 | 重大度 | OWASP LLM 2025 | 対応要件 |
|---------|------|------|--------|----------------|----------|
| AG-PERM-001 | autoApprove にワイルドカード | Kiro mcp.json | High | LLM06 | 7.1 |
| AG-PERM-002 | autoApprove に書き込み系ツール | Kiro mcp.json | High | LLM06 | 7.2 |
| AG-PERM-003 | 無制限の Bash 許可 | Claude settings | Critical | LLM06 | 7.3 |
| AG-PERM-004 | 高リスクコマンドの接頭辞許可 | Claude settings | High | LLM06 | 7.4 |
| AG-PERM-005 | WebFetch / MCP サーバー全体の許可 | Claude settings | Medium | LLM06 / LLM01 | 7.5 |
| AG-PERM-006 | defaultMode: bypassPermissions | Claude settings | Critical | LLM06 | 7.6 |
| AG-PERM-007 | defaultMode: acceptEdits | Claude settings | Low | LLM06 | 7.7 |
| AG-PERM-008 | enableAllProjectMcpServers: true | Claude settings | High | LLM06 / LLM03 | 7.8 |
| AG-PERM-009 | deny ルールなし | Claude settings | Medium | LLM06 | 7.9 |
| AG-PERM-010 | 重要パスが deny で未保護 | Claude settings | Medium | LLM02 / LLM06 | 7.10 |
| AG-PERM-011 | additionalDirectories が広範 | Claude settings | High | LLM06 | 7.11 |
| AG-PERM-012 | hook による自動許可判定 | Kiro / Claude hooks | Medium | LLM06 | 7.12 |
| AG-PERM-013 | bypassPermissions 無効化の未設定 | Claude settings | Info | LLM06 | 7.13 |
| AG-MCP-001 | filesystem MCP の広範なルート | mcp.json / .mcp.json | High | LLM06 | 8.1 |
| AG-MCP-002 | Docker MCP の特権・ホストマウント | mcp.json / .mcp.json | Critical | LLM06 | 8.2 |
| AG-MCP-003 | シェル経由のインラインスクリプト起動 | mcp.json / .mcp.json | Medium | LLM06 / LLM03 | 8.3 |
| AG-SEC-001 | 設定値への Secret 直書き（パターン一致） | mcp.json / settings | Critical | LLM02 | 9.1 |
| AG-SEC-002 | 機微キーへの高エントロピーリテラル | mcp.json / settings | High | LLM02 | 9.2 |
| AG-SEC-003 | ドキュメント内の Secret | steering / CLAUDE.md | Critical | LLM02 | 9.4 |
| AG-SEC-004 | アカウント ID・リソース ID・内部エンドポイント | steering / CLAUDE.md | Medium | LLM02 | 9.5 |
| AG-SEC-005 | apiKeyHelper のリテラル Secret | Claude settings | Critical | LLM02 | 9.6 |
| AG-GIT-001 | .env が未除外 | .gitignore | High | LLM02 | 10.1 |
| AG-GIT-002 | settings.local.json が未除外 | .gitignore | Medium | LLM02 | 10.2 |
| AG-GIT-003 | .gitignore が存在しない | .gitignore | Low | LLM02 | 10.4 |
| AG-CMD-001 | リモートスクリプトのパイプ実行 | hooks / MCP / apiKeyHelper | Critical | LLM03 / LLM06 | 11.1 |
| AG-CMD-002 | 再帰的な強制削除 | hooks / MCP / apiKeyHelper | High | LLM06 | 11.2 |
| AG-CMD-003 | 外部送信 | hooks / MCP / apiKeyHelper | High | LLM02 | 11.3 |
| AG-CMD-004 | エンコード済みコマンド実行 | hooks / MCP / apiKeyHelper | Critical | LLM03 / LLM06 | 11.4 |
| AG-CMD-005 | 環境変数・認証情報ファイルの読み取り | hooks / MCP / apiKeyHelper | High | LLM02 | 11.5 |
| AG-CMD-006 | 広範トリガーでの外部コマンド実行 | Kiro / Claude hooks | Medium | LLM06 | 11.6 |
| AG-CMD-007 | 外部・不在スクリプトの参照 | Kiro / Claude hooks | Low | LLM03 | 11.7 |
| AG-SUP-001 | パッケージのバージョン未固定 | mcp.json / .mcp.json | Medium | LLM03 | 12.1 |
| AG-SUP-002 | Docker イメージのタグ未固定 | mcp.json / .mcp.json | Medium | LLM03 | 12.2 |
| AG-SUP-003 | 許可リスト外のパッケージ | mcp.json / .mcp.json | Low | LLM03 | 12.3 |
| AG-SUP-004 | タイポスクワッティングの疑い | mcp.json / .mcp.json | High | LLM03 | 12.4 |
| AG-SUP-005 | 非 TLS のリモート MCP | mcp.json / .mcp.json | High | LLM03 / LLM02 | 12.5 |
| AG-SUP-006 | 非公式レジストリの指定 | mcp.json / .mcp.json | Medium | LLM03 | 12.6 |
| AG-PI-001 | 外部コンテンツの無条件信頼 | steering / CLAUDE.md | High | LLM01 | 13.1 |
| AG-PI-002 | 確認省略・安全策無効化の指示 | steering / CLAUDE.md | Medium | LLM01 / LLM06 | 13.2 |
| AG-PI-003 | 不可視文字・双方向制御文字 | 全 Target_File | High | LLM01 | 13.3 |
| AG-PI-004 | HTML コメント内の命令・外部取り込み | steering / CLAUDE.md | Low | LLM01 | 13.4 |
| AG-META-001 | 未知の設定キー | 全 JSON Target_File | Info | - | 4.5 |
| AG-META-002 | 期限切れの抑制 | .agentguard.yml | Low | - | 19.4 |
| AG-META-003 | 未使用の抑制 | .agentguard.yml | Info | - | 19.5 |

## カバレッジレビュー所見

- 依頼書の初期ルールに対し、以下を追加提案した: filesystem MCP の広範なルート（AG-MCP-001）、Docker MCP の特権・ホストマウント（AG-MCP-002）、Claude Code の `additionalDirectories`（AG-PERM-011）・`apiKeyHelper`（AG-SEC-005、AG-CMD 系の対象化）・settings.json 内 hooks（AG-CMD 系の対象化）、Env_Reference とリテラルの区別（9.3）、不可視文字・双方向制御文字（AG-PI-003）、エンコード済みコマンド（AG-CMD-004）、認証情報ファイルの読み取り（AG-CMD-005）、Docker イメージのタグ未固定（AG-SUP-002）、非公式レジストリ（AG-SUP-006）、抑制の期限切れ・未使用（AG-META-002/003）
- OWASP Top 10 for LLM Applications 2025 のうち、静的設定解析で扱えるのは主に LLM01 / LLM02 / LLM03 / LLM06 である。LLM04（Data and Model Poisoning）・LLM05（Improper Output Handling）・LLM07（System Prompt Leakage）・LLM08（Vector and Embedding Weaknesses）・LLM09（Misinformation）・LLM10（Unbounded Consumption）は設定ファイルの静的解析では直接検出できないため、MVP では対象外とする。LLM07 については、steering / CLAUDE.md への Secret 記載検出（AG-SEC-003/004）が間接的に対応する
- ヒューリスティック系ルール（AG-SEC-002、AG-SUP-003/004、AG-PI-001/002）は誤検出が発生しやすいため、Severity 上書きと抑制機能で運用調整できることを前提とする
