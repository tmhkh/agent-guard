---
status: Draft
owner: {{担当者名}}
reviewers: [{{レビュアー}}]
last_updated: 2026-11-01
related: [./README.md]
---

# 用語集（Glossary）

> 移行元: `.kiro/specs/agent-guard/requirements.md`（Kiro spec）。内容は原文のまま章単位で分割している。
> Requirement 番号は全要件を通した通し番号で、他ファイルの要件も「Requirement N」で参照する（1〜19・24・25 は機能要件、20〜23・26 は非機能要件）。


- **AgentGuard**: 本仕様で定義するツール全体（CLI、解析エンジン、レポート出力、Viewer を含む）
- **CLI**: `agent-guard` コマンド。`scan` / `serve` 等のサブコマンドを提供するコマンドラインインターフェース
- **Scanner**: 指定パスを再帰探索して Target_File を収集し、Parser と Rule_Engine に渡すコンポーネント
- **Target_File**: 解析対象となる設定ファイル。MVP では Kiro の `.kiro/settings/mcp.json`・`.kiro/hooks/*.json`・`.kiro/steering/*.md`、Claude Code の `.claude/settings.json`・`.claude/settings.local.json`・`.mcp.json`・`CLAUDE.md`、共通の `.gitignore`、および存在確認対象としての `.env` 系ファイル
- **User_Config**: ユーザーレベルの設定ファイル（`~/.kiro/settings/mcp.json`、`~/.claude/settings.json` 等）。ホームディレクトリは環境変数から解決する
- **Parser**: Target_File を読み取り、エージェント種別・ファイル種別ごとの Config_Model に変換するコンポーネント
- **Config_Model**: Parser が生成する、設定内容の型付き内部表現。各値はファイル内の位置情報（行・列・JSON Pointer）を保持する
- **Config_Printer**: Config_Model を JSON 等の設定ファイル形式のテキストに整形出力するコンポーネント（修正提案の生成に使用）
- **Rule**: 1 つの検出条件。ID・重大度・タイトル・説明・修正例・参考情報・OWASP カテゴリ・対象エージェント・対象ファイル種別・検出条件を持つ
- **Rule_Pack**: 複数の Rule を YAML で定義したデータファイル。バージョンとスキーマバージョンを持つ
- **Rule_Pack_Loader**: Rule_Pack を読み込み、スキーマ検証して Rule の集合に変換するコンポーネント
- **Rule_Engine**: Config_Model に Rule を適用して Finding を生成するコンポーネント
- **Finding**: 1 件の検出結果。Rule ID・重大度・ファイルパス・位置・メッセージ・修正提案・Fingerprint を持つ
- **Severity**: 重大度。Critical > High > Medium > Low > Info の全順序を持つ
- **Fingerprint**: Finding を一意に識別する値。Rule ID、スキャンルートからの相対パス、JSON Pointer 等の論理位置から算出し、シークレット値を入力に含めない
- **Scan_Error**: Target_File の読み取り失敗・構文エラー・サイズ超過など、解析を完了できなかった事象の記録
- **Scan_Result**: 1 回のスキャンで得られた Finding、Scan_Error、抑制済み件数、メタデータ（ツールバージョン、Rule_Pack バージョン）の集合
- **Masker**: 文字列中のシークレットを検出し、Masked_Token に置換するコンポーネント
- **Secret**: API キー、アクセストークン、パスワード、秘密鍵、接続文字列中の認証情報など、漏えいにより損害が生じうる値
- **Masked_Token**: Secret を置換する固定表現（例: `<REDACTED:aws_access_key:len=20>`）。Secret の原文の部分文字列を含まない
- **Env_Reference**: `${VAR}`・`${env:VAR}`・`$VAR` のように環境変数を参照する記述。Secret の直書きとは区別する
- **Reporter**: Scan_Result を Text_Report / JSON_Report / SARIF_Report に変換するコンポーネント
- **Text_Report**: 日本語の人間向けテキストレポート
- **JSON_Report**: 機械可読な JSON 形式のレポート。バージョン付きスキーマを持つ
- **SARIF_Report**: SARIF 2.1.0 形式のレポート（GitHub Code Scanning 連携用）
- **Fix_Suggestion**: Finding ごとの修正後設定例。unified diff 形式で表現する
- **Suppression_File**: 特定の Finding を理由付きで抑制する設定ファイル（デフォルト `.agentguard.yml`）
- **Suppression_Entry**: Suppression_File 内の 1 件の抑制定義。Rule ID、対象パスパターン（任意）、Fingerprint（任意）、理由（必須）、有効期限（任意）を持つ
- **Fail_Threshold**: 終了コードを非 0 にする最低 Severity
- **Package_Allowlist**: 公式・著名と見なす MCP サーバーパッケージ名・イメージ名の一覧。Rule_Pack に同梱し、ユーザーが拡張できる
- **Typosquat_Detector**: パッケージ名と Package_Allowlist の名前の編集距離および紛らわしい文字置換から、タイポスクワッティングの疑いを判定するコンポーネント
- **Hidden_Character**: ゼロ幅文字、双方向制御文字（Bidi override）、Unicode タグ文字など、表示上見えない、または表示順を偽装できる文字
- **Viewer**: JSON_Report を読み込み、ブラウザで閲覧するためのローカル起動・読み取り専用 Web UI（FastAPI + React + MUI）
