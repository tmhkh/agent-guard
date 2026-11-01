---
status: Draft
owner: {{担当者名}}
reviewers: [{{レビュアー}}]
last_updated: 2026-11-01
related: [../glossary.md, ./03_functional-requirements.md]
---

# 非機能要件

> 移行元: `.kiro/specs/agent-guard/requirements.md`（Kiro spec）。内容は原文のまま章単位で分割している。
> Requirement 番号は全要件を通した通し番号で、他ファイルの要件も「Requirement N」で参照する（1〜19・24・25 は機能要件、20〜23・26 は非機能要件）。

## Requirement 20: シークレットのマスキング

**User Story:** 開発者として、レポートやログ経由でシークレットが二次流出しないことを保証したい。そうすることで、CI ログや SARIF を安心して共有できる。

### Acceptance Criteria

1. THE Masker SHALL Text_Report・JSON_Report・SARIF_Report・Fix_Suggestion・Scan_Error のメッセージ・ログ・標準出力・標準エラー出力・Viewer の表示に出力されるすべてのスキャン対象由来の文字列（Finding の説明・抜粋・Fix_Suggestion の差分行と前後の文脈行を含む）について、出力前に Rule_Pack の Secret パターンに一致する部分を、Secret の長さ（1 文字以上）にかかわらず Masked_Token に置換する
2. THE Masker SHALL Masked_Token を、Rule_Pack で定義された Secret の種別識別子（機微キー名のみで検出した場合は汎用の種別識別子）と Secret の原文の文字数（Unicode コードポイント数）の 2 項目のみを含む固定表現とし、Secret の原文の部分文字列・ハッシュ値・先頭や末尾の文字を含めない
3. FOR ALL Secret を含む入力について、AgentGuard のすべての出力（Text_Report・JSON_Report・SARIF_Report・Fix_Suggestion・Scan_Error のメッセージ・ログ・標準出力・標準エラー出力・Viewer の表示）に、長さ 4 文字以上の Secret の原文の部分文字列が含まれない（非露出特性）
4. WHEN 機微キー名の値として Finding になったリテラル値を出力するとき、THE Masker SHALL Rule_Pack の Secret パターンに一致しない場合でも当該値を Masked_Token に置換し、同一スキャン内の他の出力箇所（他の Finding・Fix_Suggestion・Scan_Error のメッセージ）に同じ値が現れた場合もそのすべての出現箇所を Masked_Token に置換する
5. WHEN AgentGuard の処理中に例外が発生したとき、THE AgentGuard SHALL スタックトレース・例外メッセージ・デバッグログにスキャン対象のファイル内容由来の値を含めず、スキャンルートからの相対ファイルパスと行・列のみを出力する
6. FOR ALL 文字列について、Masker を 2 回適用した結果は 1 回適用した結果と等しく、Masked_Token 自体が Secret として再検出されない（冪等性）
7. WHEN 1 つの文字列に複数の Secret が含まれる、または複数の Secret パターンの一致範囲が重なるとき、THE Masker SHALL すべての一致範囲の和集合を置換対象とし、一致範囲外の文字列を変更せずに保持する
8. IF Masker が文字列の処理中にエラーとなったとき、THEN THE AgentGuard SHALL 当該文字列全体を汎用の種別識別子を持つ Masked_Token に置換して出力し、マスキング処理が失敗したことを示す Scan_Error を記録して残りの処理を継続する

## Requirement 21: スキャン対象の非実行・非変更

**User Story:** 開発者として、ツールがスキャン対象を勝手に実行したり書き換えたりしないことを保証したい。そうすることで、悪意ある設定ファイルを含むリポジトリも安全にスキャンできる。

### Acceptance Criteria

1. THE AgentGuard SHALL Target_File および User_Config に記述されたコマンド・スクリプト・URL・MCP サーバー定義を、子プロセスの起動、シェルによる評価、コードとしての評価、ネットワーク接続、DNS 名前解決のいずれの方法でも実行・評価・起動・アクセスしない
2. THE Scanner SHALL Target_File および User_Config を読み取り専用モードで開き、当該ファイルに対して書き込み・切り詰め・削除・名前変更・権限変更・更新日時の変更を行わない
3. WHEN `agent-guard scan` が終了したとき（終了コード 0・1・2 のいずれの場合も）、THE AgentGuard SHALL スキャンルート配下（`--include-user-config` 指定時は User_Config を含む）の全ファイルのバイト列・更新日時、およびファイル・ディレクトリの一覧を、スキャン開始前と同一に保つ（`--output` で明示指定された出力ファイル 1 件のみを例外とし、一時ファイル・キャッシュファイルもスキャンルート配下に作成しない）
4. IF `--output` の出力先を絶対パスに正規化した結果（シンボリックリンク解決後、Windows では大文字小文字を区別せずに比較）が、当該スキャンで収集される Target_File または User_Config のいずれかのパスと一致する場合、THEN THE CLI SHALL スキャンとファイル書き込みを行わずに、衝突したパスを示すエラーメッセージを標準エラー出力に出力し、終了コード 2 で終了し、衝突したファイルの内容を変更しない
5. THE Parser および Rule_Pack_Loader SHALL YAML（Target_File、Rule_Pack、Suppression_File を含む）の解析に、YAML タグによる任意オブジェクト生成およびコード実行を行わない安全なローダーを使用する
6. IF Rule_Pack または Suppression_File に、任意オブジェクト生成を要求する YAML タグ（言語固有オブジェクトのタグ等）が含まれる場合、THEN THE CLI SHALL 当該タグを解釈せず、オブジェクト生成とコード実行を行わずに、該当ファイルパスと行番号を示すエラーメッセージを標準エラー出力に出力し、終了コード 2 で終了する
7. IF Target_File に任意オブジェクト生成を要求する YAML タグが含まれる場合、THEN THE Scanner SHALL 当該タグを解釈せずに、該当ファイルの Scan_Error を Scan_Result に記録し、残りの Target_File のスキャンを継続する

## Requirement 22: オフライン動作と決定性

**User Story:** PM / リードとして、スキャン内容が外部に送信されず、同じ入力から常に同じ結果が得られることを保証したい。そうすることで、機密リポジトリでも利用でき、CI の結果を再現できる。

### Acceptance Criteria

1. WHILE `scan` サブコマンドを実行している間（開始から終了コードを返すまで。`--include-user-config` 指定時を含む）、THE AgentGuard SHALL ネットワーク通信（DNS 名前解決、ループバック以外のホストへの接続・送信を含む）を 1 回も行わない
2. THE AgentGuard SHALL すべてのサブコマンドで、テレメトリ・利用統計・クラッシュレポートを収集・送信せず、これらを有効化するオプションや設定項目も持たない
3. FOR ALL 同一のスキャン対象（Target_File の内容とスキャンルートからの相対パス構成）・Rule_Pack・Suppression_File・コマンドラインオプションの組で、各 Suppression_Entry の有効期限判定結果が同一である場合について、AgentGuard は実行回数、OS（Windows / macOS / Linux）、カレントディレクトリ、スキャンルートの絶対パスの違いにかかわらず、実行日時メタデータを除いてバイト単位で同一の JSON_Report・SARIF_Report・Text_Report と同一の終了コードを出力する。レポートの文字コードは UTF-8（BOM なし）、改行コードは LF とする（決定性特性）
4. WHEN `--no-timestamp` オプションが指定されたとき、THE Reporter SHALL JSON_Report・SARIF_Report・Text_Report のいずれにも実行日時メタデータを出力しない。実行日時メタデータ以外の内容は、同じ入力でオプションを指定しない場合の出力と同一にする
5. THE Reporter SHALL レポートに出力するすべてのファイルパスを、スキャンルートからの相対パス（先頭の `./` なし、区切り文字は OS にかかわらず `/`）で表記し、スキャンルートやホームディレクトリの絶対パスを含めない。User_Config のパスは `~/` から始まる表記とする
6. THE Reporter SHALL すべてのレポート形式で、Finding を Requirement 6 の基準 7 と同じ順序（Severity 降順 → スキャンルートからの相対パス辞書順 → 行番号昇順 → 列番号昇順 → Rule ID 辞書順、同一ファイル内ではファイル単位の Finding を先）で並べ、Scan_Error をスキャンルートからの相対パスの辞書順で並べる
7. WHEN ネットワークインターフェースがすべて無効な環境で `scan` サブコマンドを実行したとき、THE AgentGuard SHALL ネットワーク接続のある環境で同じ入力に対して実行した場合と同一の Scan_Result と終了コードを出力し、ネットワーク不通を原因とする Scan_Error・警告を出力しない
8. WHERE `--no-timestamp` オプションが指定されていない場合、THE Reporter SHALL スキャン開始日時を UTC の ISO 8601 形式（秒単位）の実行日時メタデータとして、JSON_Report・SARIF_Report・Text_Report のそれぞれに 1 か所だけ出力する

## Requirement 23: 性能

**User Story:** 開発者として、pre-commit や CI で待たされずにスキャンを終えたい。そうすることで、日常の開発フローに組み込める。

### Acceptance Criteria

1. WHEN ユーザーが、Target_File が 200 件以下・合計サイズ 10 MiB 以下・Target_File 以外のファイルとディレクトリが合計 10,000 件以下のディレクトリを、同梱の Rule_Pack とデフォルト設定で `agent-guard scan` によりスキャンしたとき、THE AgentGuard SHALL GitHub Actions の標準 Linux ランナー上で、プロセス起動から終了までの経過時間 10 秒以内にスキャンを終え、Text_Report・JSON_Report・SARIF_Report のうち指定された 1 形式のレポート出力を完了する
2. WHILE スキャンを実行している間、THE AgentGuard SHALL プロセスのピーク常駐メモリ（RSS）を「最大ファイルサイズ設定の 10 倍 + 200 MiB」以下に保つ。この上限は、スキャン対象の Target_File の合計サイズが最大ファイルサイズ設定の 10 倍を超える場合にも適用する
3. WHEN ユーザーが、Target_File が 20 件以下・合計サイズ 1 MiB 以下のディレクトリを、同梱の Rule_Pack とデフォルト設定で `agent-guard scan` によりスキャンしたとき、THE AgentGuard SHALL GitHub Actions の標準 Linux ランナー上で、プロセス起動から終了までの経過時間 2 秒以内にスキャンとレポート出力を完了する
4. WHEN スキャンで生成された Finding が 1,000 件以下のとき、THE AgentGuard SHALL 受け入れ基準 1 と 3 の時間上限を、Finding の件数にかかわらず満たす
5. IF 受け入れ基準 1 の条件下でスキャン対象に Scan_Error となる Target_File（構文エラー・サイズ超過・UTF-8 復号失敗）が含まれるとき、THEN THE AgentGuard SHALL 受け入れ基準 1 と同じ 10 秒以内の時間上限でスキャンとレポート出力を完了する

## Requirement 26: 品質保証と自己スキャン

**User Story:** PM / リードとして、ツール自体の品質と安全性が継続的に検証されていることを確認したい。そうすることで、セキュリティツールとして信頼して導入できる。

### Acceptance Criteria

1. THE AgentGuard SHALL Python 3.12 で動作し、全モジュールの全関数・メソッドの引数と戻り値に型ヒントを付与し、型チェッカー（mypy または pyright）のエラー件数 0 件、および Ruff の lint エラー件数 0 件・format 差分 0 ファイルの状態を維持する
2. THE AgentGuard SHALL Parser・Rule_Engine・Rule_Pack_Loader・Masker・Reporter・抑制処理の各コンポーネントについて、pytest で計測したラインカバレッジをコンポーネントごとに 80% 以上に維持する
3. THE AgentGuard SHALL 非露出特性・非変更特性・解析の堅牢性・抑制の正確性・ラウンドトリップ特性・決定性特性の 6 特性それぞれについて、Hypothesis によるプロパティベーステストを 1 件以上持ち、各プロパティを 1 回の実行につき 100 例以上の生成入力で検証する
4. WHEN main ブランチへの push が行われたとき、または main ブランチ宛ての pull request が作成もしくは更新されたとき、THE AgentGuard の CI（GitHub Actions `ci.yml`）SHALL Ruff の lint / format チェック、型チェック、pytest（カバレッジ計測とプロパティベーステストを含む）、フロントエンドのビルドをすべて実行する
5. WHEN CI が実行されたとき、THE AgentGuard の CI SHALL AgentGuard 自身でリポジトリ全体を Fail_Threshold High（`--fail-on high`）かつ `--fail-on-scan-error` を指定してスキャンし、Suppression_File で抑制されていない Severity Critical または High の Finding が 1 件以上ある場合にジョブを失敗させる。ただし、Rule の検出用テストフィクスチャは理由付きの Suppression_Entry によってのみ抑制し、それ以外の Finding は抑制しない
6. THE AgentGuard SHALL 組み込み Rule_Pack の各 Rule について、当該 Rule ID の Finding が 1 件以上生成される入力例と、当該 Rule ID の Finding が 0 件となる入力例を各 1 件以上テストフィクスチャとして持ち、両方の期待結果を pytest で検証する
7. IF CI の lint・format チェック・型チェック・テスト・フロントエンドのビルドのいずれかがエラーを報告したとき、またはいずれかのコンポーネントのラインカバレッジが 80% 未満のとき、THEN THE AgentGuard の CI SHALL ジョブを失敗として報告し、失敗したステップ名を CI ログに出力する
8. IF CI の自己スキャン（`--fail-on-scan-error` を指定して実行）で Scan_Error が 1 件以上記録されたとき、THEN THE AgentGuard の CI SHALL Finding の有無にかかわらず、`--fail-on-scan-error` による終了コード 1 を受けてジョブを失敗として報告し、Scan_Error の対象ファイルの相対パスを Secret の値を含めずに CI ログに出力する
9. IF 組み込み Rule_Pack に、検出される入力例または検出されない入力例のテストフィクスチャを持たない Rule が 1 件以上存在するとき、THEN THE AgentGuard のテストスイート SHALL 失敗し、該当する Rule ID を出力する
