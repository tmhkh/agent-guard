---
status: Draft
owner: {{担当者名}}
reviewers: [{{レビュアー}}]
last_updated: 2026-11-01
related: [../glossary.md, ./04_non-functional-req.md, ./07_initial-rule-catalog.md]
---

# 機能要件

> 移行元: `.kiro/specs/agent-guard/requirements.md`（Kiro spec）。内容は原文のまま章単位で分割している。
> Requirement 番号は全要件を通した通し番号で、他ファイルの要件も「Requirement N」で参照する（1〜19・24・25 は機能要件、20〜23・26 は非機能要件）。


## Requirement 1: ディレクトリの再帰スキャン

**User Story:** 開発者として、`agent-guard scan <path>` でリポジトリ全体をスキャンしたい。そうすることで、エージェント設定のリスクをコミット前に一括で確認できる。

### Acceptance Criteria

1. WHEN ユーザーが `agent-guard scan <path>` を実行したとき、THE Scanner SHALL `<path>` 配下を深さの制限なく再帰的に探索し、スキャンルート直下だけでなく任意の深さのサブディレクトリに存在するファイルのうち、相対パスの末尾が Glossary に定義した Target_File のパターン（`.kiro/settings/mcp.json`、`.kiro/hooks/` 直下の `*.json`、`.kiro/steering/` 直下の `*.md`、`.claude/settings.json`、`.claude/settings.local.json`、ファイル名が `.mcp.json`・`CLAUDE.md`・`.gitignore`・`.env`・`.env.` で始まるもの）に大文字小文字を区別して一致するファイルをすべて収集する
2. WHEN `<path>` が省略されたとき、THE Scanner SHALL コマンド実行時のカレントディレクトリをスキャンルートとして扱い、`<path>` にカレントディレクトリを指定した場合と同一の Target_File 集合を収集する
3. IF `<path>` が存在しない、読み取り権限がない、またはディレクトリではない（通常ファイル等である）とき、THEN THE CLI SHALL 原因（存在しない／権限がない／ディレクトリではない）を示す日本語のエラーメッセージを標準エラー出力に出力し、Scan_Result を出力せずに終了コード 2 で終了する
4. THE Scanner SHALL ディレクトリ名が `.git`、`node_modules`、`.venv` のいずれかに一致するディレクトリを、スキャンルートからの深さによらずデフォルトで除外し、その配下を探索しない。加えて、`--exclude` オプション（複数回指定可）および Suppression_File で指定された glob パターン（`*`・`**`・`?` を使用可）にスキャンルートからの相対パスが一致するファイル・ディレクトリを除外する
5. THE Scanner SHALL ファイルおよびディレクトリを指すシンボリックリンク（Windows のジャンクションを含む）をデフォルトで辿らず、辿らなかった各シンボリックリンクのスキャンルートからの相対パスを、Scan_Error ではなくスキップ情報として Scan_Result に記録する
6. THE Scanner SHALL 収集した Target_File を、パス区切り文字を `/` に正規化したスキャンルートからの相対パスの Unicode コードポイント順（大文字小文字を区別する昇順）で処理し、同一のディレクトリ内容に対しては OS によらず同一の処理順序とする
7. WHEN 除外適用後の Target_File が 1 件も見つからなかったとき、THE Reporter SHALL 解析対象ファイルが見つからなかったことを示す Info レベルの通知を Scan_Result に含める
8. WHEN 除外適用後の Target_File が 1 件も見つからず、Scan_Error も 0 件のとき、THE CLI SHALL 終了コード 0 で終了する
9. IF 探索中にスキャンルート配下のサブディレクトリの読み取り権限がないとき、THEN THE Scanner SHALL 当該ディレクトリのスキャンルートからの相対パスと日本語の原因説明を持つ Scan_Error を記録し、残りのディレクトリの探索を継続する
10. IF `--exclude` オプションに空文字列または glob として解釈できないパターンが指定されたとき、THEN THE CLI SHALL 不正なパターンを示す日本語のエラーメッセージを標準エラー出力に出力し、スキャンを開始せずに終了コード 2 で終了する

## Requirement 2: ユーザーレベル設定のスキャン（オプトイン）

**User Story:** 開発者として、必要なときだけホームディレクトリのエージェント設定もスキャンしたい。そうすることで、CI では誤検出を避けつつ、手元では個人設定のリスクも確認できる。

### Acceptance Criteria

1. WHILE `--include-user-config` オプションが指定されていない状態で、THE Scanner SHALL User_Config（ホームディレクトリ直下の `.kiro/settings/mcp.json` と `.claude/settings.json` の 2 ファイル）を Target_File として収集せず、スキャンルートがホームディレクトリまたはその祖先ディレクトリである場合も当該 2 ファイルを Finding・Scan_Error の対象に含めない
2. WHERE `--include-user-config` オプションが指定された場合、THE Scanner SHALL ホームディレクトリ配下の `~/.kiro/settings/mcp.json` と `~/.claude/settings.json` の 2 ファイルを、スキャンルート配下の Target_File に加えて Target_File として追加し、同一の実ファイルがスキャンルート配下からも収集された場合は 1 件の Target_File として 1 回だけ解析する
3. THE Scanner SHALL ホームディレクトリのパスを OS の環境変数（Windows では `USERPROFILE`、それ以外では `HOME`）から解決し、ソースコードにホームディレクトリの固定値を持たない
4. IF `--include-user-config` が指定され、ホームディレクトリ解決用の環境変数が未設定または空文字列である場合、THEN THE CLI SHALL ホームディレクトリを解決できない旨の日本語エラーメッセージを標準エラー出力に出力し、スキャンを実行せずに終了コード 2 で終了する
5. WHEN Reporter が User_Config に関する Finding・Scan_Error・スキップ情報を Text_Report / JSON_Report / SARIF_Report のいずれかに出力するとき、THE Reporter SHALL ファイルパスのホームディレクトリ部分を `~` に置換し、区切り文字を `/` に統一した表記（例: `~/.claude/settings.json`）で出力し、ホームディレクトリの絶対パス文字列をレポート中に 1 箇所も含めない
6. THE Rule_Engine SHALL User_Config の Finding の Fingerprint を、スキャンルートからの相対パスではなく `~` 置換後のパス表記を用いて算出し、同一内容の User_Config を異なるスキャンルート・異なるホームディレクトリの環境でスキャンしても同一の Fingerprint を生成する
7. IF `--include-user-config` が指定され、User_Config の 2 ファイルのいずれかが存在しない場合、THEN THE Scanner SHALL 存在しないファイルを Scan_Error ではなくスキップ情報として `~` 置換後のパスとともに Scan_Result に記録し、残りの Target_File のスキャンを継続し、この事象のみを理由に終了コードを変更しない
8. IF `--include-user-config` が指定され、存在する User_Config の読み取りに失敗した、または構文エラーがある場合、THEN THE Scanner SHALL 当該ファイルを他の Target_File と同じ扱いで Scan_Error として Scan_Result に記録し、残りの Target_File のスキャンを継続する

## Requirement 3: 設定ファイルの解析と堅牢性

**User Story:** 開発者として、壊れた設定ファイルや巨大なファイルが含まれていてもスキャンを最後まで完了させたい。そうすることで、1 ファイルの不具合で他の検出が失われない。

### Acceptance Criteria

1. WHEN Parser が JSON 形式の Target_File を読み取ったとき、THE Parser SHALL ファイル内容を Config_Model に変換し、各値の位置情報として行（1 始まり）、列（1 始まり、Unicode コードポイント単位）、JSON Pointer（ルートは空文字列）を保持する
2. THE Parser SHALL JSON 形式の Target_File に含まれる行コメント（`//`）、ブロックコメント（`/* */`）、オブジェクトと配列の末尾カンマを許容して解析し、コメントの有無で Config_Model の値と JSON Pointer を変えない
3. IF Target_File が構文として不正なとき、THEN THE Parser SHALL ファイルパス（スキャンルートからの相対パス）、最初に構文エラーを検出した行・列、日本語の原因説明を持つ Scan_Error を 1 ファイルにつき 1 件記録し、THE Scanner SHALL 残りの Target_File の解析を継続する
4. IF Target_File のサイズが最大ファイルサイズ（デフォルト 5 MiB = 5,242,880 バイト。`--max-file-size` で 1 バイト以上 104,857,600 バイト（100 MiB）以下の範囲で変更可能）を超えるとき、THEN THE Scanner SHALL 当該ファイルの内容を読み込まずに、ファイルパス・実サイズ・上限値を持つ Scan_Error を記録し、残りの解析を継続する。サイズが上限値ちょうどのファイルは解析対象とする
5. IF `--max-file-size` に整数でない値、または 1 バイト以上 104,857,600 バイト以下の範囲外の値が指定されたとき、THEN THE CLI SHALL スキャンを開始せずに、不正な値と許容範囲を示す日本語のエラーメッセージを標準エラー出力に出力し、終了コード 2 で終了する
6. IF Target_File が UTF-8 として復号できないとき、THEN THE Parser SHALL ファイルパスと最初に復号できなかったバイト位置（0 始まりのオフセット）を持つ Scan_Error を記録し、残りの解析を継続する。先頭に UTF-8 BOM を持つファイルは BOM を除いて復号可能として扱う
7. IF JSON のネスト深さ（ルートのオブジェクトまたは配列を深さ 1 とする）が最大深さ（デフォルト 64。Requirement 25 に従い変更可能）を超えるとき、THEN THE Parser SHALL 最大深さ + 1 に達した位置の行・列を持つ Scan_Error を記録し、残りの解析を継続する。深さが最大深さちょうどの JSON は解析対象とする
8. WHEN Parser が Markdown 形式の Target_File（steering、`CLAUDE.md`）を読み取ったとき、THE Parser SHALL 本文を 1 始まりの行番号付きテキストとして Config_Model に含め、ファイル 1 行目が `---` のみで始まり、その後に `---` のみの行で閉じられたブロックをフロントマター（YAML）として解析し Config_Model に含める
9. IF Markdown 形式の Target_File のフロントマターが YAML として不正なとき、または閉じの `---` 行が存在しないとき、THEN THE Parser SHALL フロントマターについての Scan_Error を記録し、フロントマター部分を含む全行を行番号付きテキストとして Config_Model に含めて解析を継続する
10. THE Parser SHALL `.gitignore` を Git の ignore パターン仕様に従って解釈し、少なくとも否定パターン `!`、先頭 `/` によるルート固定、末尾 `/` によるディレクトリ限定、`*`・`?`・`**`、`#` で始まるコメント行、空行、`\#`・`\!` のエスケープを Git と同じ一致結果で扱う
11. THE Config_Printer SHALL Config_Model を、Parser が構文エラーなく解析できる有効な JSON テキストとして UTF-8 で整形出力し、同一の Config_Model に対して常にバイト単位で同一のテキストを出力する
12. FOR ALL 有効な Config_Model について、Config_Printer で出力したテキストを Parser で解析した結果は、位置情報とコメントを除き、元の Config_Model と値・型・キー集合・配列要素の順序が等価である（ラウンドトリップ特性）
13. FOR ALL 0 バイト以上・最大ファイルサイズ以下のバイト列の入力について、THE Parser SHALL 呼び出し元に例外を送出せず、スキャン対象の内容を実行・評価せずに、Config_Model または Scan_Error のいずれか一方を返す
14. IF JSON 形式の Target_File が 0 バイト、または空白文字とコメントのみで構成されるとき、THEN THE Parser SHALL 内容が空であることを示す Scan_Error を記録し、残りの解析を継続する。Markdown 形式の Target_File が 0 バイトのときは、行を持たない Config_Model を返し Scan_Error を記録しない
15. WHEN Target_File について Scan_Error が記録されたとき（Requirement 3.9 のフロントマターエラーを除く）、THE Rule_Engine SHALL 当該ファイルの内容に対して Rule を適用せず、当該ファイルの内容に基づく Finding を生成しない
16. WHEN スキャンが完了し、Scan_Result に Scan_Error が 1 件以上含まれ、Fail_Threshold 以上の Finding が 0 件のとき、THE CLI SHALL 終了コード 0 で終了し（`--fail-on-scan-error` 指定時を除く）、THE Reporter SHALL Scan_Error の件数と各 Scan_Error のファイルパス・原因説明を Text_Report・JSON_Report・SARIF_Report のすべてに含める

## Requirement 4: スキーマ変更への追従

**User Story:** PM / リードとして、エージェントの設定スキーマが更新されてもツールが壊れず、追従漏れに気づきたい。そうすることで、ツールを継続的に信頼して運用できる。

### Acceptance Criteria

1. THE Parser SHALL エージェント種別（Kiro / Claude Code）とファイル種別の組ごとに、対応しているスキーマバージョンの一覧と、各スキーマバージョンで定義された既知キーの一覧を定義データとして保持する
2. IF Target_File に、Parser が適用したスキーマバージョンの既知キー一覧に含まれないキーが含まれるとき、THEN THE Parser SHALL 当該キーとその値を位置情報（行・列・JSON Pointer）付きで Config_Model に保持し、当該ファイルの解析を中断せずに継続し、未知キーの存在のみを理由に Scan_Error を記録しない
3. THE Parser SHALL Kiro hooks ファイルについて、`when` / `then` 形式と `version` / `hooks` 配列形式の 2 つの既知フォーマットを識別し、識別したフォーマットに対応する Config_Model に変換する
4. THE AgentGuard SHALL Rule_Pack をツール本体のバージョンと独立した `MAJOR.MINOR.PATCH` 形式のバージョンで管理し、WHEN ユーザーが `agent-guard rules list` を実行したとき、THE CLI SHALL 読み込まれているすべての Rule_Pack の名前・バージョン・スキーマバージョンと、各 Rule の ID・Severity・タイトル・有効 / 無効状態を Rule ID の辞書順で標準出力に出力し、終了コード 0 で終了する
5. WHEN Config_Model に未知キーが含まれるとき、THE Rule_Engine SHALL 未知キー 1 件につき 1 件の Severity Info の Finding を生成し、当該 Finding にキー名、JSON Pointer、Parser が適用したスキーマバージョンを含める
6. IF Target_File が宣言するスキーマバージョンが Parser の対応一覧に含まれない、またはスキーマバージョンを宣言していないとき、THEN THE Parser SHALL 当該ファイル種別の対応済み最新スキーマバージョンを適用して解析を継続し、THE Rule_Engine SHALL 宣言されたバージョン（未宣言の場合はその旨）と適用したバージョンを示す Severity Info の Finding を 1 ファイルにつき 1 件生成する
7. IF Kiro hooks ファイルが基準 3 の 2 つの既知フォーマットのいずれにも一致しないとき、THEN THE Parser SHALL ファイルパスと日本語の原因説明を持つ Scan_Error を記録し、THE Scanner SHALL 残りの Target_File の解析を継続する

## Requirement 5: データ駆動のルールパック

**User Story:** PM / リードとして、検出ルールをコード変更なしに追加・無効化したい。そうすることで、チームのポリシーや新しい脅威に素早く対応できる。

### Acceptance Criteria

1. THE Rule_Pack_Loader SHALL YAML 形式の Rule_Pack から、ID・Severity・タイトル・説明・修正例・参考情報（URL を含む）・OWASP カテゴリ・対象エージェント・対象ファイル種別・検出条件を持つ Rule を読み込む。このうち ID・Severity・タイトル・説明・対象エージェント・対象ファイル種別・検出条件を必須項目とし、Severity は Critical / High / Medium / Low / Info の 5 値のいずれかに限る
2. THE AgentGuard SHALL 組み込み Rule_Pack を同梱し、Requirement 7 から Requirement 13 に定義した検出をすべて Rule_Pack のデータとして提供する。組み込み Rule_Pack は、追加の Rule_Pack を指定しない状態でも読み込まれる
3. WHERE ユーザーが `--rule-pack <path>`（複数回指定可）または Suppression_File の設定で追加の Rule_Pack を指定したとき、THE Rule_Pack_Loader SHALL 組み込み Rule_Pack に加えて指定されたすべての Rule_Pack を読み込み、読み込んだ全 Rule_Pack のバージョンを Scan_Result のメタデータに記録する
4. WHERE ユーザーが Rule ID を `--disable-rule` オプション（複数回指定可）または Suppression_File の `disabled_rules` に指定したとき、THE Rule_Engine SHALL 当該 Rule を評価せず、当該 Rule ID の Finding を Scan_Result に 1 件も含めない
5. WHERE ユーザーが Rule の Severity の上書きを Suppression_File に指定したとき、THE Rule_Engine SHALL 上書き後の Severity で Finding を生成し、Fail_Threshold の判定にも上書き後の Severity を用いる
6. IF Rule_Pack ファイルが存在しない・読み取り権限がない・YAML として不正である・スキーマに違反する（必須項目の欠落、Severity が 5 値以外、未対応のスキーマバージョンを含む）、または組み込み Rule_Pack と追加 Rule_Pack を通じて Rule ID が重複するとき、THEN THE Rule_Pack_Loader SHALL 検出したすべての違反について Rule_Pack のファイルパス・行番号・該当 Rule ID（特定できる場合）・違反内容を日本語で標準エラー出力に出力し、THE CLI SHALL スキャンを実行せず、レポートを出力せずに終了コード 2 で終了する
7. THE Rule_Pack_Loader SHALL 検出条件として、JSON Pointer / パス式による値の参照、正規表現一致、値の列挙一致、存在・非存在判定、およびそれらの論理結合（all / any / not）を表現できるようにする
8. THE Rule_Engine SHALL 検出条件の正規表現を線形時間保証のある正規表現エンジンで評価し、補助的な安全策として 1 回の評価あたり 100 ミリ秒のタイムアウトを設ける。決定性（Requirement 22）は線形時間保証のある正規表現エンジンによって担保し、タイムアウトの発生有無に依存させない
9. FOR ALL 有効な Rule_Pack について、読み込んだ Rule 集合を YAML に整形出力し再度読み込んだ結果は元の Rule 集合と等価である（ラウンドトリップ特性）
10. IF 正規表現の評価が補助的な 100 ミリ秒のタイムアウトに達したとき、THEN THE Rule_Engine SHALL 当該 Target_File について Rule ID とファイルパスを含む Scan_Error を記録し、当該評価を不一致として扱い、残りの Rule と Target_File の評価を継続する
11. IF `--disable-rule`、Suppression_File の `disabled_rules`、または Severity の上書きに、読み込まれたどの Rule_Pack にも存在しない Rule ID が指定されたとき、THEN THE CLI SHALL 当該 Rule ID を示す日本語の警告を標準エラー出力に出力し、スキャンを継続する（終了コードは警告の有無によって変えない）
12. IF Suppression_File の Severity の上書きに Critical / High / Medium / Low / Info 以外の値が指定されたとき、THEN THE CLI SHALL 該当 Rule ID と指定値を日本語で標準エラー出力に出力し、スキャンを実行せず終了コード 2 で終了する

## Requirement 6: 検出結果（Finding）の構造

**User Story:** 開発者として、各検出で何が危ないか・どこにあるか・どう直すかを一目で把握したい。そうすることで、修正の優先順位をすぐに判断できる。

### Acceptance Criteria

1. THE Rule_Engine SHALL 各 Finding に、Rule ID、Severity（Critical / High / Medium / Low / Info のいずれか 1 つ）、空文字列でない日本語のタイトルと説明、スキャンルートからの相対ファイルパス、位置情報、修正例、参考情報、OWASP カテゴリ、Fingerprint をすべて含める
2. THE Rule_Engine SHALL Finding の相対ファイルパスを、OS に関係なく区切り文字を `/` に統一し、先頭に `./` や `/` を付けない表記で格納する（User_Config の場合は Requirement 2 の `~` 表記に従う）
3. THE Rule_Engine SHALL Finding の位置情報として、行と列を 1 始まりの番号（列は Unicode コードポイント単位）で格納し、JSON 形式の Target_File の場合は加えて検出対象の値を指す JSON Pointer を格納する
4. IF Finding が特定の行に対応しない（ファイルの存在・非存在、またはファイル内に記述が存在しないことを検出した）とき、THEN THE Rule_Engine SHALL 当該 Finding の行・列・JSON Pointer を「位置なし」として格納し、ファイル単位の Finding として出力する
5. THE Rule_Engine SHALL Fingerprint を Rule ID・相対ファイルパス・論理位置（JSON の場合は JSON Pointer、Markdown 等のテキストの場合は直近の見出しテキストと、前後の空白を除去し連続する空白を 1 文字に縮約した行内容、ファイル単位の Finding の場合は論理位置なし）のみから算出し、Secret の値・行番号・列番号・ツールのバージョン・実行日時を算出の入力に含めない
6. THE Rule_Engine SHALL Fingerprint が同一となる Finding を 1 件に集約し、集約後の Finding の位置情報として、集約対象のうち行・列が最も小さいものを保持する
7. THE Reporter SHALL Finding を Severity の降順、次に相対ファイルパスの辞書順（Unicode コードポイント順）、次に行の昇順、次に列の昇順、次に Rule ID の辞書順で並べ、同一ファイル内ではファイル単位の Finding を行・列を持つ Finding より前に並べる
8. FOR ALL 同一の Target_File 集合・Rule_Pack・Suppression_File による 2 回のスキャンについて、THE AgentGuard SHALL 実行 OS・実行日時・実行順に関係なく、同一の Finding 集合・同一の Fingerprint・同一の並び順を出力する（決定性特性）
9. FOR ALL JSON 形式の Target_File について、検出対象の値の JSON Pointer と値を変えずに、その前に空行・コメント・他のキーを追加して行番号を変えた場合でも、THE Rule_Engine SHALL 当該 Finding に同一の Fingerprint を付与する（位置ずれ耐性特性）
10. THE Rule_Engine SHALL Finding のタイトル・説明・修正例に含まれるスキャン対象由来の文字列を、Finding に格納する前に Masker により Secret を Masked_Token に置換する

## Requirement 7: 権限・自動承認の検出（LLM06 Excessive Agency）

**User Story:** PM / リードとして、エージェントに過剰な権限や確認なしの実行権限が与えられていないか確認したい。そうすることで、意図しない破壊的操作や情報持ち出しを防げる。

### Acceptance Criteria

1. WHEN Rule_Engine が Kiro の mcp.json の Config_Model を評価し、いずれかの MCP サーバー定義の `autoApprove` 配列に `*` のみからなる要素、または `*` を含む要素（例: `fs_*`）を検出したとき、THE Rule_Engine SHALL 該当する要素ごとに Severity High の Finding を生成する
2. WHEN Rule_Engine が Kiro の mcp.json の `autoApprove` 配列に、`*` を含まず、かつ Rule_Pack の書き込み系ツール名パターン（write / edit / delete / exec / run / shell / push / deploy 等）に一致する要素を検出したとき、THE Rule_Engine SHALL 該当する要素ごとに Severity High の Finding を生成する。一致判定は大文字小文字を区別せず、ツール名を区切り文字（`_`・`-`・`.`・`/`）で分割した語、または区切りのないツール名の部分文字列のいずれかがパターンに一致することとする
3. WHEN Rule_Engine が Claude Code の設定ファイル（`.claude/settings.json`・`.claude/settings.local.json`・指定時の User_Config）の `permissions.allow` 配列に、`Bash`、`Bash(*)`、`Bash(:*)`、`Bash()` のようにコマンドを限定しない Bash 許可の要素を検出したとき、THE Rule_Engine SHALL 該当する要素ごとに Severity Critical の Finding を生成する
4. WHEN Rule_Engine が Claude Code の `permissions.allow` 配列に、Rule_Pack の高リスクコマンドパターン（`curl`、`wget`、`git push`、`rm`、`ssh`、`scp`、`npx`、`docker` 等）を接頭辞ワイルドカード（`Bash(<コマンド>:*)`、`Bash(<コマンド> *)`、`Bash(<コマンド>*)` のいずれかの形式）で許可する要素を検出したとき、THE Rule_Engine SHALL 基準 3 に該当する要素を除き、該当する要素ごとに Severity High の Finding を生成する
5. WHEN Rule_Engine が Claude Code の `permissions.allow` 配列に、`WebFetch` または `WebFetch(*)` のように `domain:` 指定を持たない WebFetch 許可の要素、あるいは `mcp__<server>` または `mcp__<server>__*` のようにツール名を限定せずサーバー全体を許可する要素を検出したとき、THE Rule_Engine SHALL 該当する要素ごとに Severity Medium の Finding を生成する
6. WHEN Rule_Engine が Claude Code の設定ファイルのトップレベルの `defaultMode` または `permissions.defaultMode` の値が `bypassPermissions` であることを検出したとき、THE Rule_Engine SHALL 当該キーの位置に Severity Critical の Finding を生成する
7. WHEN Rule_Engine が Claude Code の設定ファイルのトップレベルの `defaultMode` または `permissions.defaultMode` の値が `acceptEdits` であることを検出したとき、THE Rule_Engine SHALL 当該キーの位置に Severity Low の Finding を生成する
8. WHEN Rule_Engine が Claude Code の設定ファイルの `enableAllProjectMcpServers` の値が真偽値 `true` であることを検出したとき、THE Rule_Engine SHALL 当該キーの位置に Severity High の Finding を生成する
9. IF スキャン対象に Claude Code の設定ファイルが 1 件以上存在し、そのいずれにも `permissions.deny` キーが存在しないとき、THEN THE Rule_Engine SHALL スキャンルートごとに 1 件の Severity Medium の Finding を、`.claude/settings.json` が存在する場合は当該ファイル、存在しない場合は `.claude/settings.local.json` のファイル先頭位置に生成する。Claude Code の設定ファイルが 1 件も存在しない場合、THE Rule_Engine SHALL 本 Finding を生成しない
10. IF スキャンルート内の全 Claude Code 設定ファイルの `permissions.deny` を合算した集合が、Rule_Pack の重要パス一覧（`.env`、`.env.*`、`~/.aws/**`、`~/.ssh/**`、秘密鍵ファイル等）のいずれかについて、当該パスを包含する `Read(<パスパターン>)` 要素またはパス指定のない `Read` 要素を含まないとき、THEN THE Rule_Engine SHALL 保護されていない重要パスごとに 1 件の Severity Medium の Finding を生成する。`permissions.deny` が全ファイルに存在しない場合は基準 9 のみを生成し、本基準の Finding は生成しない
11. WHEN Rule_Engine が Claude Code の `permissions.additionalDirectories` 配列に、ホームディレクトリ全体（`~`、`~/`、`$HOME`、`%USERPROFILE%`、または解決後のホームディレクトリと一致する絶対パス）、ドライブ / ファイルシステムのルート（`/`、`C:\`、`C:/` 等）、または Rule_Pack の重要パス一覧（`~/.ssh` 等）に一致・包含される要素を検出したとき、THE Rule_Engine SHALL 該当する要素ごとに Severity High の Finding を生成する。パス比較は末尾の区切り文字の有無と区切り文字の種類（`/` と `\`）を区別しない
12. WHEN Rule_Engine が Claude Code hooks の `PreToolUse` イベント、または Kiro hooks のツール実行前トリガーに登録された Shell コマンドの文字列に、Rule_Pack の承認返却パターン（`permissionDecision` と `allow` の組、`"decision"` と `approve` の組等）に一致する記述を検出したとき、THE Rule_Engine SHALL 該当するフック定義ごとに Severity Medium の Finding を生成する
13. IF `.claude/settings.json` が存在し、その `permissions.disableBypassPermissionsMode` の値が文字列 `disable` でない、またはキーが存在しないとき、THEN THE Rule_Engine SHALL 当該ファイルに 1 件の Severity Info の Finding を生成する
14. THE Rule_Engine SHALL 基準 1〜8・11・12 の Finding の位置として、検出対象となった配列要素またはキーの行・列・JSON Pointer を設定する
15. IF `autoApprove`、`permissions.allow`、`permissions.deny`、`permissions.additionalDirectories` の値が配列でない、または配列要素が文字列でないとき、THEN THE Rule_Engine SHALL 当該キーについて基準 1〜5・10・11 の評価を行わず、型が想定と異なることを示す Severity Info の Finding を当該キーの位置に生成し、他のキー・他の Target_File の評価を継続する

## Requirement 8: MCP サーバー構成の検出（LLM06 / LLM03）

**User Story:** 開発者として、MCP サーバーにファイルシステムやホスト環境への過剰なアクセスを与えていないか確認したい。そうすることで、MCP 経由の権限昇格や情報漏えいを防げる。

### Acceptance Criteria

1. WHEN Kiro の mcp.json（`.kiro/settings/mcp.json`、User_Config を含む）または Claude Code の `.mcp.json` の `mcpServers` 配下の MCP サーバー定義のうち、`command` または `args` に Rule_Pack の filesystem 系 MCP サーバーのパッケージ名一覧（例: `@modelcontextprotocol/server-filesystem`）のいずれかと一致する要素を含むサーバーの `args` 要素が、ホームディレクトリ全体（`~`、`~/`、`$HOME`、`${HOME}`、`%USERPROFILE%`、`${env:USERPROFILE}`、または解決後のホームディレクトリの絶対パスと末尾区切り文字を除いて一致するパス）、ドライブ / ファイルシステムのルート（`/`、`C:\`、`C:/` 等の任意ドライブ文字のルート）、または Requirement 7 で定義した Rule_Pack の重要パス一覧に一致するパスであるとき、THE Rule_Engine SHALL 一致した `args` 要素ごとに、当該要素の JSON Pointer を位置とする Severity High の Finding を 1 件生成する
2. WHEN `command` が `docker` または `podman`（パス付き表記および Windows の `.exe` 拡張子付き表記を含み、大文字小文字を区別しない）の MCP サーバーの `args` に、`--privileged`、`--network host` または `--network=host`、`--net host` または `--net=host`、`--pid host` または `--pid=host`、Docker ソケット（`/var/run/docker.sock`）をソースとするマウント、またはホームディレクトリ全体 / ドライブ・ファイルシステムのルートをソースとするボリュームマウント（`-v`、`--volume`、`--mount` のいずれの形式も対象）が含まれるとき、THE Rule_Engine SHALL 一致した指定ごとに、当該 `args` 要素の JSON Pointer を位置とする Severity Critical の Finding を 1 件生成する
3. WHEN MCP サーバーの `command` が `sh`・`bash`・`zsh`・`cmd`・`powershell`・`pwsh`（パス付き表記および `.exe` 拡張子付き表記を含み、大文字小文字を区別しない）で、`args` にインラインスクリプト指定（`-c`、`/c`、`/k`、`-Command`、`-EncodedCommand`、および PowerShell で大文字小文字を区別しないこれらの省略形）が含まれるとき、THE Rule_Engine SHALL 当該サーバー 1 件につき、インラインスクリプト指定の `args` 要素の JSON Pointer を位置とする Severity Medium の Finding を 1 件生成する
4. WHEN MCP サーバー定義の `disabled` が JSON の真偽値 `true` のとき、THE Rule_Engine SHALL 当該サーバーに対する本要件の Finding のうち、Suppression_File による Severity 上書き適用後の Severity が Medium 以上のものを 1 段階下げ（Critical→High、High→Medium、Medium→Low）、Low と Info は変更せず、Finding の説明に「無効化されたサーバー」であることを記載する
5. IF MCP サーバー定義の `disabled` が存在しない、`false`、または真偽値以外（文字列 `"true"` を含む）のとき、THEN THE Rule_Engine SHALL 当該サーバーを有効なサーバーとして扱い、Severity を下げない
6. IF filesystem 系 MCP サーバーの `args` 要素がホームディレクトリ配下のサブディレクトリ（例: `~/projects/app`）、またはドライブ / ファイルシステムのルート配下のサブディレクトリであり、かつ Rule_Pack の重要パス一覧に一致しないとき、THEN THE Rule_Engine SHALL 当該要素に対して Acceptance Criterion 1 の Finding を生成しない
7. IF MCP サーバー定義の `command` が存在しない、`args` が存在しない・配列でない、または `args` に文字列以外の要素が含まれるとき、THEN THE Rule_Engine SHALL 例外で異常終了せずに文字列として評価可能な要素のみを Acceptance Criteria 1〜3 の判定対象とし、残りの MCP サーバー定義と Target_File の評価を継続する

## Requirement 9: シークレット露出の検出（LLM02 Sensitive Information Disclosure）

**User Story:** 開発者として、設定ファイルやドキュメントに認証情報が直書きされていないか確認したい。そうすることで、Git へのシークレット流出を未然に防げる。

### Acceptance Criteria

1. WHEN mcp.json・`.mcp.json`・Claude Code settings の `env`・`args`・`headers`・`url` の値（配列要素およびネストしたオブジェクト内の文字列値を含む）の全体または一部が、Rule_Pack の Secret パターン（AWS アクセスキー、GitHub トークン、Slack トークン、OpenAI / Anthropic API キー、Google API キー、JWT、秘密鍵ブロック、URL 内の認証情報等）に一致するとき、THE Rule_Engine SHALL 一致した値 1 つにつき 1 件の Severity Critical の Finding を、当該値の行・列・JSON Pointer 付きで生成する
2. WHEN キー名が大文字・小文字を区別せずに Rule_Pack の機微キー名パターン（`*_TOKEN`、`*_KEY`、`*_SECRET`、`PASSWORD`、`Authorization` 等）に一致し、値が Env_Reference を除いたリテラル部分の長さ 8 文字以上で、かつそのリテラル部分のシャノンエントロピーが Rule_Pack のエントロピー閾値（デフォルト 3.5 bit/文字）以上のとき、THE Rule_Engine SHALL Severity High の Finding を生成する
3. WHEN 機微キー名の値が Env_Reference のみで構成される、または Env_Reference を除いたリテラル部分（`Bearer ` 等の固定接頭辞を含む）が基準 1 の Secret パターンに一致せず、基準 2 の長さ・エントロピー条件を満たさないとき、THE Rule_Engine SHALL Secret 直書きの Finding を生成しない
4. WHEN steering ファイルまたは `CLAUDE.md` のいずれかの行（コードブロック・フロントマター内を含む）に Rule_Pack の Secret パターンに一致する文字列が含まれるとき、THE Rule_Engine SHALL 一致箇所 1 つにつき 1 件の Severity Critical の Finding を、当該箇所の行・列付きで生成する
5. WHEN steering ファイルまたは `CLAUDE.md` に、前後を数字以外の文字または行頭・行末で区切られた 12 桁の数字列（AWS アカウント ID）、`<リージョン名>_<英数字 9 文字>` 形式の Cognito User Pool ID、`E` で始まる英大文字・数字 13〜14 文字の CloudFront Distribution ID、または Rule_Pack の内部エンドポイントパターン（プライベート IP アドレス、`*.internal`、`*.local`、`*.corp` 等）に一致する文字列が含まれるとき、THE Rule_Engine SHALL 一致箇所 1 つにつき 1 件の Severity Medium の Finding を生成する
6. WHEN Claude Code の `apiKeyHelper` がリテラルの Secret を出力するコマンド（`echo` 等の出力コマンドの引数に Rule_Pack の Secret パターンに一致する文字列が含まれる等）を含むとき、THE Rule_Engine SHALL Severity Critical の Finding を生成する
7. WHEN Claude Code の `apiKeyHelper` が設定されているとき、THE Rule_Engine SHALL 当該コマンドを Requirement 11 の危険コマンド判定の対象に含める
8. WHEN `.claude/settings.local.json` に基準 1 または基準 2 の Finding があり、かつ当該ファイルがスキャンルート配下のいずれの `.gitignore`（Requirement 3 の ignore パターン仕様で解釈）によっても除外されていない（`.gitignore` が存在しない場合を含む）とき、THE Rule_Engine SHALL 当該 Finding の Severity を元の Severity にかかわらず Critical とする
9. THE Reporter SHALL 本要件の Finding のメッセージ・位置情報・Fix_Suggestion・Text_Report・JSON_Report・SARIF_Report のいずれにも Secret の原文およびその部分文字列を含めず、Secret 該当部分を Masked_Token に置換して出力する
10. WHEN 同一の値が基準 1 と基準 2 の両方の条件を満たすとき、THE Rule_Engine SHALL 当該値について Severity Critical の Finding 1 件のみを生成し、Severity High の Finding を生成しない
11. WHEN 基準 1 または基準 2 の Finding を生成するとき、THE Rule_Engine SHALL 当該値を Env_Reference に置き換えた修正後設定例を Fix_Suggestion として Finding に含める

## Requirement 10: .gitignore による除外の検出（LLM02）

**User Story:** 開発者として、個人用設定や `.env` が Git にコミットされる状態になっていないか確認したい。そうすることで、ローカル専用の情報がリポジトリに流出しない。

### Acceptance Criteria

1. WHEN スキャンルート配下（Requirement 1 の除外ディレクトリを除く）に、ファイル名が `.env` と完全一致するファイル、または `.env.` で始まるファイル（ファイル名が `.env.example`・`.env.sample`・`.env.template` と完全一致するものを除く）が存在し、かつ当該ファイルが基準 5 の評価で除外されていないとき、THE Rule_Engine SHALL 当該ファイル 1 件につき 1 件の Severity High の Finding を、ファイルの内容を読み取らず、スキャンルートからの相対パスのみを位置情報として生成する
2. WHEN スキャンルート配下に `.claude/settings.local.json` が存在し、かつ当該ファイルが基準 5 の評価で除外されていないとき、THE Rule_Engine SHALL 当該ファイル 1 件につき 1 件の Severity Medium の Finding を生成する
3. WHEN スキャンルート配下の Target_File に Requirement 9 の Secret 直書きの Finding が 1 件以上あり、かつ当該 Target_File が基準 5 の評価で除外されていないとき、THE Rule_Engine SHALL 当該 Target_File の Secret 直書きの Finding すべての説明に「Git にコミットされる状態」であることを追記し、Severity は Requirement 9 基準 8 による変更以外では変更しない（User_Config の Finding には追記しない）
4. WHEN スキャンルート直下に `.gitignore` が存在しないとき、THE Rule_Engine SHALL `.gitignore` の不在を示す Severity Low の Finding を 1 件生成し、基準 1〜3 の判定は除外パターンが 0 件であるものとして継続する
5. THE Rule_Engine SHALL `.gitignore` による除外判定を、スキャンルートから対象ファイルの親ディレクトリまでの各階層にある `.gitignore` のパターンを Requirement 3 の `.gitignore` 解釈仕様（下位階層のパターンが上位階層より優先され、同一ファイル内では後に書かれたパターンが優先される）で評価するファイルシステム上の処理だけで行い、`git` コマンドを実行しない
6. THE Rule_Engine SHALL スキャンルートより外にある `.gitignore`、`.git/info/exclude`、Git のグローバル除外設定を除外判定に使わず、ファイルが Git で既に追跡されているかどうかも判定しない
7. WHEN 対象ファイルが基準 5 の評価で除外されているとき、THE Rule_Engine SHALL 当該ファイルについて基準 1・2 の Finding を生成せず、基準 3 の追記も行わない
8. IF 判定に使う `.gitignore` が Scan_Error（UTF-8 として復号できない、最大ファイルサイズ超過など）となったとき、THEN THE Rule_Engine SHALL 当該 `.gitignore` の除外パターンを 0 件として基準 1〜3 の判定を継続し、その判定で生成・追記した Finding の説明に「`.gitignore` を評価できなかった」ことを追記する

## Requirement 11: 危険なコマンドの検出（LLM06 / LLM03）

**User Story:** PM / リードとして、hooks や MCP 起動コマンドに危険な処理が含まれていないか確認したい。そうすることで、エージェントの自動実行を起点とした攻撃や事故を防げる。

### Acceptance Criteria

1. WHEN Config_Model 内の検査対象コマンド（Kiro hooks の実行コマンド、Claude Code hooks の `command`、Claude Code の `apiKeyHelper`、MCP サーバー定義の `command` と `args` の各要素を半角スペース 1 個で連結した文字列）に、リモートから取得した内容をシェル・インタプリタへ渡して実行する記述（`curl ... | sh`、`curl ... | bash`、`wget ... | sh`、`wget -O- ... | bash`、`iex (iwr ...)`、`Invoke-Expression` と `Invoke-WebRequest` / `Invoke-RestMethod` / `DownloadString` の組み合わせ、`irm ... | iex` 等）が含まれるとき、THE Rule_Engine SHALL Severity Critical の Finding を生成する
2. WHEN 検査対象コマンドに再帰的な強制削除（`rm -rf`・`rm -fr`・`rm -r -f` 等のオプション順序・分割の違いを含む、`Remove-Item -Recurse -Force`、`rd /s /q`、`rmdir /s /q`、`del /s /q`）が含まれるとき、THE Rule_Engine SHALL Severity High の Finding を生成する
3. WHEN 検査対象コマンドに外部送信（`curl` の `-d` / `--data*` / `-F` / `--form` / `-T` / `--upload-file` / `-X POST` / `-X PUT`、`wget` の `--post-data` / `--post-file`、`Invoke-WebRequest` / `Invoke-RestMethod` の `-Method Post` / `-Method Put` / `-Body` / `-InFile`、`nc`、`ncat`、`scp`）が含まれるとき、THE Rule_Engine SHALL Severity High の Finding を生成し、送信オプションを伴わない `curl` / `wget` / `Invoke-WebRequest` 単体の取得処理に対しては本項の Finding を生成しない
4. WHEN 検査対象コマンドにエンコードされたコマンドの実行（`powershell` / `pwsh` の `-EncodedCommand` およびその省略形 `-enc` / `-e`、`base64 -d | sh`、`base64 --decode | bash`、`[Convert]::FromBase64String` と `Invoke-Expression` / `iex` の組み合わせ等）が含まれるとき、THE Rule_Engine SHALL Severity Critical の Finding を生成する
5. WHEN 検査対象コマンドが環境変数一覧または認証情報ファイルを読み取るとき（後続コマンド引数を伴わない `env` / `printenv` / `set`、`Get-ChildItem env:`、`gci env:`、`dir env:`、および `~/.aws/credentials`・`$HOME/.aws/credentials`・`%USERPROFILE%\.aws\credentials`・`~/.ssh/` 配下・`$HOME/.ssh/` 配下・`%USERPROFILE%\.ssh\` 配下へのパス参照）、THE Rule_Engine SHALL Severity High の Finding を生成し、`env VAR=value <command>` のように環境変数を設定して別コマンドを起動する用法に対しては本項の Finding を生成しない
6. WHEN 外部コマンドを実行する hook のトリガーが広範なイベント（Kiro の `PreToolUse` / `PostToolUse` / `UserPromptSubmit` / `promptSubmit` / `SessionStart`、Claude Code の `PreToolUse` / `PostToolUse` / `UserPromptSubmit` / `SessionStart`）で、かつ matcher が未指定・空文字列・`*`・`.*` のいずれかであるとき、THE Rule_Engine SHALL 該当 hook 1 件につき 1 件の Severity Medium の Finding を生成する
7. WHEN hook の検査対象コマンドが拡張子 `.sh` / `.bash` / `.ps1` / `.py` / `.js` / `.bat` / `.cmd` のスクリプトファイルを参照し、その参照先がスキャンルート外（絶対パスでスキャンルート外を指す、または `..` によりスキャンルート外へ解決される）であるか、スキャンルート内に存在しないとき、THE Rule_Engine SHALL Severity Low の Finding を生成する。このとき相対パスはスキャンルートを基点に解決し、`$CLAUDE_PROJECT_DIR` はスキャンルートとして扱い、スキャンルート外のファイルの内容は読み取らない
8. THE Rule_Engine SHALL 検査対象コマンドをトークン化とパターン照合のみで解析し、コマンドの実行・シェル評価・変数展開・参照スクリプトの実行を一切行わない
9. THE Rule_Engine SHALL コマンド名・PowerShell コマンドレット名・エイリアス・オプション名を大文字小文字を区別せずに照合し、トークン間の空白の数・引用符（`'`、`"`）の有無の違いによって検出結果を変えない
10. WHEN 1 つの検査対象コマンドが第 1〜5 項の複数の検出条件に該当するとき、THE Rule_Engine SHALL 該当した検出条件ごとに 1 件ずつ Finding を生成し、同一の検出条件に同一コマンド内で複数回一致した場合は 1 件のみ生成し、各 Finding の位置には一致箇所を含む設定値の行・列・JSON Pointer を設定する
11. IF 検査対象コマンドのトークン化に失敗したとき（閉じられていない引用符等）、THEN THE Rule_Engine SHALL 未閉鎖の引用符以降を 1 つのトークンとして扱ってパターン照合を継続し、当該ファイルの他の Rule の評価を中断しない
12. WHEN 第 1〜7 項の Finding のメッセージに検査対象コマンドの抜粋を含めるとき、THE Rule_Engine SHALL 抜粋を Masker に通し、抜粋中の Secret（`Authorization` ヘッダ値、URL 中の認証情報、トークン等）を Masked_Token に置換した状態でのみ出力する

## Requirement 12: サプライチェーンリスクの検出（LLM03 Supply Chain）

**User Story:** 開発者として、MCP サーバーのパッケージが改ざん・なりすましのリスクを抱えていないか確認したい。そうすることで、悪意あるパッケージをエージェントに読み込ませずに済む。

### Acceptance Criteria

1. WHEN MCP サーバーの `command` が `npx`・`bunx`・`pnpm dlx`・`uvx`・`pipx run` のいずれか（判定はパスを除いたコマンド名で行い、大文字小文字および `.cmd`・`.exe` 拡張子の有無を区別しない）で、`args` 中のパッケージ指定が固定バージョンでないとき、THE Rule_Engine SHALL Severity Medium の Finding を生成する。固定バージョンとは、範囲演算子（`^`、`~`、`*`、`x`、`>`、`<`、`>=`、`<=`、`~=`、`!=`）を含まず、メジャー・マイナー・パッチの 3 要素すべてを数値で指定したバージョン（npm 形式 `name@1.2.3`・`@scope/name@1.2.3`、Python 形式 `name==1.2.3`、および `--from` 等で同等に指定したもの）を指し、バージョン省略・`latest` を含むタグ名指定・範囲指定はすべて固定バージョンでないものとして扱う
2. WHEN MCP サーバーの `command` が `docker`（判定方法は受け入れ基準 1 と同じ）で `args` に `run` サブコマンドを含み、イメージ指定にダイジェスト（`@sha256:`）がなく、かつタグがない、または `:latest` タグであるとき、THE Rule_Engine SHALL Severity Medium の Finding を生成する。このとき、レジストリホスト部のポート番号（例: `localhost:5000/image` の `:5000`）はタグとみなさない
3. WHEN 受け入れ基準 1 または 2 の対象となる MCP サーバーのパッケージ名またはイメージ名（バージョン・タグ・ダイジェストを除いた名前）が Package_Allowlist のいずれの名前とも完全一致せず、かつ受け入れ基準 4 の Finding が生成されないとき、THE Rule_Engine SHALL Severity Low の Finding を生成する
4. WHEN Typosquat_Detector が、Package_Allowlist のいずれの名前とも完全一致しないパッケージ名について、Package_Allowlist 内のいずれかの名前との編集距離が 1 以上かつ Rule_Pack の閾値以下であると判定したとき、または紛らわしい文字置換（`0`/`o`、`1`/`l`、`-`/`_`、スコープの欠落・差し替え）を正規化すると Package_Allowlist の名前に完全一致すると判定したとき、THE Rule_Engine SHALL 一致した Package_Allowlist の名前を含めた Severity High の Finding を生成する。閾値は 1 以上 5 以下の整数でデフォルトは 2 とし、範囲外の値は Requirement 5 の受け入れ基準 6 のスキーマ違反として扱う
5. WHEN リモート MCP サーバーの `url` のスキームが大文字小文字を区別せず `http` であり、ホストが `localhost`・`127.0.0.1`・`::1`（`[::1]` 表記を含む）のいずれでもないとき、THE Rule_Engine SHALL Severity High の Finding を生成する
6. WHEN MCP サーバーの `args` に、Rule_Pack に定義されたレジストリ上書きオプション（デフォルト: `--registry`、`--index-url`、`--extra-index-url`、`--index`、`-i`）が `--option value` 形式または `--option=value` 形式で含まれ、その値のスキームとホストが Rule_Pack に定義された公式レジストリ一覧（デフォルト: npm 公式レジストリ、PyPI 公式インデックス）のいずれとも一致しないとき、THE Rule_Engine SHALL 指定されたレジストリのホスト名を含めた Severity Medium の Finding を生成する
7. WHERE ユーザーが Rule_Pack または Suppression_File で Package_Allowlist に名前を追加したとき、THE Typosquat_Detector SHALL 組み込みの Package_Allowlist と追加された名前の和集合で受け入れ基準 3 および 4 の判定を行う
8. THE Typosquat_Detector SHALL ネットワークに接続せず、Package_Allowlist のデータのみで判定する
9. THE Typosquat_Detector SHALL パッケージ名と Package_Allowlist の名前を大文字小文字を区別せずに比較し、`uvx`・`pipx run` のパッケージについては `-`・`_`・`.` を同一視した正規化名で比較する（この場合 `-`/`_` の置換は受け入れ基準 4 の紛らわしい文字置換として扱わない）
10. WHEN パッケージ名またはイメージ名が Package_Allowlist の名前と完全一致したとき、THE Rule_Engine SHALL 当該パッケージについて受け入れ基準 3 および 4 の Finding を生成しない

## Requirement 13: プロンプトインジェクション耐性の検出（LLM01 Prompt Injection）

**User Story:** PM / リードとして、steering や `CLAUDE.md` がエージェントに外部の指示を無条件に信頼させる内容になっていないか確認したい。そうすることで、外部コンテンツ経由の乗っ取りを防げる。

### Acceptance Criteria

1. WHEN steering ファイルまたは `CLAUDE.md` のいずれかの行に、外部 URL・外部ファイル・ツール出力の内容を無条件に信頼または実行させる記述として Rule_Pack に定義された日本語・英語フレーズパターン（例: 「〜の指示に必ず従う」「always follow instructions from」「execute the contents of」）に一致する文字列が含まれるとき、THE Rule_Engine SHALL 一致した行ごとに Severity High の Finding を 1 件生成し、一致した行・列と一致したパターンの識別子を Finding に含める（英字の大文字小文字、全角・半角の差、連続する空白の数の違いは一致判定に影響させない）
2. WHEN steering ファイルまたは `CLAUDE.md` のいずれかの行に、確認の省略や安全策の無効化を指示する記述として Rule_Pack に定義されたフレーズパターン（例: 「確認せずに実行」「skip confirmation」「ignore previous instructions」）に一致する文字列が含まれるとき、THE Rule_Engine SHALL 一致した行ごとに Severity Medium の Finding を 1 件生成し、一致した行・列と一致したパターンの識別子を Finding に含める（一致判定の正規化は第 1 項と同じとし、同一行が第 1 項にも一致する場合は両方の Finding をそれぞれ生成する）
3. WHEN テキストとして読み取れる Target_File（steering ファイル・`CLAUDE.md`・hooks の prompt / command・MCP 設定値を含む）に Hidden_Character（少なくとも U+200B〜U+200D、U+2060、ファイル先頭以外の U+FEFF、U+202A〜U+202E、U+2066〜U+2069、U+E0000〜U+E007F）が含まれるとき、THE Rule_Engine SHALL Hidden_Character を含む行ごとに Severity High の Finding を 1 件生成し、当該行に含まれるすべての Hidden_Character のコードポイント（`U+XXXX` 形式）と行・列（1 始まり）を Finding に含める
4. WHEN steering ファイルまたは `CLAUDE.md` に、内容が第 1 項・第 2 項のフレーズパターンまたは Rule_Pack に定義された命令表現パターンに一致する HTML コメント（`<!--` から `-->` まで、複数行にまたがるものを含む）、または取り込み先が `http://`・`https://` の URL、絶対パス、もしくはスキャンルートの外へ解決される相対パスである取り込み指示（`#[[file:...]]` 等）が含まれるとき、THE Rule_Engine SHALL 該当する HTML コメントまたは取り込み指示ごとに Severity Low の Finding を 1 件生成し、取り込み先がスキャンルート内へ解決される取り込み指示に対しては本項の Finding を生成しない
5. WHEN steering ファイルのフロントマターの `inclusion` が `always` である、またはフロントマターもしくは `inclusion` キーが存在しない（常時読み込みとして扱う）とき、かつ同ファイルに本要件の第 1 項または第 2 項の Finding があるとき、THE Rule_Engine SHALL 当該 Finding の説明に「すべての対話に常時読み込まれる」ことを追記し、Severity は変更しない
6. THE Reporter SHALL 本要件の Finding のメッセージ・説明・抜粋において、Hidden_Character を原文の文字として出力せず `U+XXXX` 形式のエスケープ表記に置き換えて出力する
7. IF steering ファイルのフロントマターが YAML として解析できない場合、THEN THE Scanner SHALL 当該ファイルについて構文エラーを示す Scan_Error を記録し、THE Rule_Engine SHALL 本文に対する第 1 項〜第 4 項の判定を継続し、第 5 項の追記は行わない

## Requirement 14: 日本語テキストレポート

**User Story:** 開発者として、検出結果を日本語で読みたい。そうすることで、何が危ないかとどう直すかを素早く理解できる。

### Acceptance Criteria

1. WHEN `--format` が省略されたとき、または `--format text` が指定されたとき、かつ `--output` が指定されていないとき、THE Reporter SHALL Text_Report を UTF-8 で標準出力に出力する
2. THE Reporter SHALL Text_Report に Finding ごとに Severity・Rule ID・日本語のタイトル・スキャンルートからの相対ファイルパスと行番号・日本語の説明・Fix_Suggestion（unified diff 形式）・参考情報（URL を含む）・OWASP カテゴリを含め、Fix_Suggestion を持たない Finding には修正例がないことを示す日本語の表記を出力する
3. THE Reporter SHALL Text_Report 中の Finding を Requirement 6 の基準 7 と同じ順序（Severity 降順 → 相対ファイルパス辞書順 → 行昇順 → 列昇順 → Rule ID 辞書順、同一ファイル内ではファイル単位の Finding を先）で出力し、Scan_Error の一覧を相対ファイルパスの辞書順で出力する
4. THE Reporter SHALL Text_Report の末尾に、Severity 5 段階（Critical / High / Medium / Low / Info）それぞれの件数（0 件の段階も 0 と表示）、抑制済み件数、Scan_Error 件数、スキャンした Target_File 数を含むサマリーを出力し、Scan_Error が 1 件以上あるときは各 Scan_Error の相対ファイルパスと日本語の失敗理由を一覧で出力する
5. WHEN 抑制後の Finding が 0 件のとき、THE Reporter SHALL 検出がなかったことを示す日本語のメッセージとサマリー（全 Severity 0 件）を含む Text_Report を出力する
6. IF `--no-color` オプションが指定されている、標準出力が端末ではない、または `--output <file>` オプションが指定されているとき、THEN THE Reporter SHALL ANSI エスケープシーケンスを 1 つも含まない Text_Report を出力する
7. THE Reporter SHALL Text_Report に出力するスキャン対象由来の文字列（ファイルパス・設定値・抜粋等）に含まれる C0 / C1 制御文字（改行・タブを除く）、ANSI エスケープシーケンス、および Hidden_Character を `\xNN` または `\uNNNN` 形式の可視表現に置換して出力する
8. THE Reporter SHALL Text_Report に Secret の原文を含めず、Secret に該当する値を Masked_Token で置換して出力する
9. WHERE `--output <file>` オプションが指定されたとき、THE Reporter SHALL 指定された `--format` のレポートを BOM なし UTF-8 で指定ファイルに書き込み（既存ファイルは上書き）、レポート本文を標準出力に出力しない
10. IF `--output <file>` の出力先に書き込めない（親ディレクトリが存在しない、書き込み権限がない、ディレクトリを指している等）とき、THEN THE CLI SHALL 書き込み失敗とその出力先パスを示す日本語のエラーメッセージを標準エラー出力に出力し、終了コード 2 で終了する

## Requirement 15: JSON レポート

**User Story:** PM / リードとして、検出結果を機械可読な JSON で取得したい。そうすることで、ダッシュボードや他ツールと連携できる。

### Acceptance Criteria

1. WHEN `--format json` が指定されたとき、THE Reporter SHALL Scan_Result を 1 つの JSON ドキュメントからなる UTF-8 の JSON_Report として、`--output <file>` が指定されていない場合は標準出力に、指定されている場合は当該ファイルに出力し、JSON_Report 以外の文字列（進捗表示・警告等）を同じ出力先に含めない
2. THE Reporter SHALL JSON_Report に、`メジャー.マイナー` 形式の JSON_Report スキーマバージョン、ツールバージョン、Rule_Pack バージョン、スキャンルートの表示名（スキャンルートのディレクトリ名のみとし、絶対パス・ホームディレクトリのパスを含めない）、Finding 一覧、Scan_Error 一覧、抑制済み Finding の総件数と Rule ID 別件数、サマリー（Severity 別の Finding 件数、抑制済み件数、Scan_Error 件数、スキャンしたファイル数）を含める
3. THE Reporter SHALL JSON_Report の Finding 一覧を Requirement 6 の基準 7 と同じ順序（Severity 降順 → 相対パス（`/` 区切り）辞書順 → 行昇順 → 列昇順 → Rule ID 辞書順、同一ファイル内ではファイル単位の Finding を先）で並べ、Scan_Error 一覧を相対パスの辞書順で並べ、Finding と Scan_Error が 0 件の場合も各一覧を空配列として出力する
4. THE AgentGuard SHALL JSON_Report を読み込んで Scan_Result に復元するデシリアライザを提供する
5. FOR ALL Scan_Result について、JSON_Report に変換してから Scan_Result に復元した結果は、Finding・Scan_Error・抑制済み件数・メタデータのすべてのフィールド値と一覧の順序において元の Scan_Result と等価である（ラウンドトリップ特性）
6. THE Reporter SHALL JSON_Report の JSON Schema を AgentGuard の配布物に同梱し、FOR ALL Scan_Result について、出力される JSON_Report は同梱の JSON Schema の検証でエラーが 0 件である（スキーマ適合特性）
7. IF デシリアライザに渡された入力が JSON として構文不正であるとき、または同梱の JSON Schema に適合しないとき、THEN THE AgentGuard SHALL Scan_Result を部分的にも返さず、不適合の箇所（JSON の行番号または JSON Pointer）を示す日本語のエラーを返す
8. IF デシリアライザに渡された JSON_Report のスキーマバージョンのメジャー番号が AgentGuard の対応するメジャー番号と異なるとき、THEN THE AgentGuard SHALL 復元を行わず、検出したスキーマバージョンと対応するスキーマバージョンを示す日本語のエラーを返す

## Requirement 16: SARIF レポート

**User Story:** 開発者として、検出結果を GitHub Code Scanning に表示したい。そうすることで、PR のレビュー画面上でリスクに気づける。

### Acceptance Criteria

1. WHEN `--format sarif` が指定されたとき、THE Reporter SHALL SARIF 2.1.0 スキーマに適合し、run を 1 件だけ持つ SARIF_Report を UTF-8 で出力する（`--output <file>` 指定時は指定ファイル、未指定時は標準出力）。その際、抑制済みの Finding は result に含めない
2. THE Reporter SHALL SARIF_Report の `tool.driver` にツール名とツールバージョンを設定し、`tool.driver.rules` には当該スキャンで有効だった全 Rule（Finding が 0 件の Rule を含み、無効化された Rule を除く）について、ID・タイトル・説明・参考情報 URL をそれぞれ 1 エントリずつ含める
3. THE Reporter SHALL Severity を SARIF の `level` に対応づける（Critical / High → `error`、Medium → `warning`、Low / Info → `note`）とともに、`properties.security-severity` に数値（Critical 9.5、High 8.0、Medium 5.5、Low 3.0、Info 0.0）を設定する
4. THE Reporter SHALL SARIF_Report の各 result に、`ruleId` として Finding の Rule ID、`artifactLocation.uri` としてスキャンルートからの相対パス（区切り文字は `/`、絶対パスやホームディレクトリのパスを含まない）、`region` として 1 始まりの開始行・開始列、`partialFingerprints` として Finding の Fingerprint を設定し、Finding に行・列の位置情報がない場合（ファイル存在のみを検出対象とする Finding 等）は `region` を省略する
5. THE Reporter SHALL Fix_Suggestion を SARIF_Report の result の `fixes` ではなく `message` に Markdown として含め、`message.text` には Fix_Suggestion を除いた日本語の説明をプレーンテキストで、`message.markdown` には同じ説明に続けて unified diff 形式のコードブロックとした Fix_Suggestion を設定し、いずれの文字列でも Secret を Masked_Token に置換した状態で出力する
6. IF Scan_Result に Finding が 0 件のとき、THEN THE Reporter SHALL `results` を空配列とした、SARIF 2.1.0 スキーマに適合する SARIF_Report を出力する
7. IF Scan_Result に Scan_Error が 1 件以上含まれるとき、THEN THE Reporter SHALL 各 Scan_Error を result ではなく run の invocation の `toolExecutionNotifications` に、スキャンルートからの相対パスと日本語の原因説明を持つ通知として 1 件ずつ、相対パスの辞書順で記録し、Scan_Error を含まない Finding の result はすべて出力する
8. FOR ALL 同一の Target_File 群・Rule_Pack・Suppression_File・オプションによるスキャンについて、THE Reporter SHALL result を Requirement 6 の基準 7 と同じ順序（Severity 降順 → 相対パス辞書順 → 開始行昇順 → 開始列昇順 → Rule ID 辞書順、同一ファイル内ではファイル単位の Finding を先）で並べた、バイト単位で同一の SARIF_Report を出力する（決定性特性）

## Requirement 17: 終了コードと CI 連携

**User Story:** PM / リードとして、一定以上の重大度の検出があれば CI を失敗させたい。そうすることで、危険な設定が main にマージされるのを防げる。

### Acceptance Criteria

1. WHEN `agent-guard scan` が実行エラー（引数不正・Rule_Pack 不正・Suppression_File 不正・スキャンルート不在）なしに完了し、抑制適用後の Finding（期限切れ・未使用の Suppression_Entry を報告する Finding を含む）に Severity が Fail_Threshold 以上のものが 0 件であり、かつ基準 5 の終了コード 1 の条件にも該当しないとき、THE CLI SHALL 終了コード 0 で終了する
2. WHEN `agent-guard scan` が実行エラーなしに完了し、抑制適用後の Finding（期限切れ・未使用の Suppression_Entry を報告する Finding を含む）に Severity が Fail_Threshold 以上のものが 1 件以上存在するとき、THE CLI SHALL 終了コード 1 で終了する
3. THE CLI SHALL Fail_Threshold のデフォルトを High とし、`--fail-on` に小文字の `critical`・`high`・`medium`・`low`・`info`・`none` のいずれか 1 つを指定して変更できるようにする。「Fail_Threshold 以上」とは Severity の全順序 Critical > High > Medium > Low > Info で Fail_Threshold と同じかそれより上を指す（例: `--fail-on medium` のときは Critical・High・Medium が該当する）
4. WHERE `--fail-on none` が指定されたとき、THE CLI SHALL Finding の件数と Severity にかかわらず、実行エラーがなく、かつ基準 5 の終了コード 1 の条件にも該当しなければ終了コード 0 で終了する
5. WHERE `--fail-on-scan-error` オプションが指定されたとき、THE CLI SHALL Scan_Error が 1 件以上存在すれば、Finding の件数と `--fail-on` の値（`none` を含む）にかかわらず終了コード 1 で終了する。`--fail-on-scan-error` が指定されないときは、Scan_Error の件数が終了コードに影響してはならない。Requirement 1 の「解析対象ファイルが見つかりませんでした」という通知とシンボリックリンクのスキップ情報は Finding にも Scan_Error にも数えない
6. IF 引数が不正なとき（未定義のオプション、`--fail-on` に基準 3 の 6 値以外の値を指定した場合を含む）、または Rule_Pack・Suppression_File が不正なとき、THEN THE CLI SHALL エラー原因を示す日本語のエラーメッセージを標準エラー出力に出力し、終了コード 2 で終了する
7. IF 終了コード 2 の条件と、終了コード 1 または 0 の条件が同時に成り立つとき、THEN THE CLI SHALL 終了コード 2 で終了する（優先順位: 2 > 1 > 0）
8. WHEN THE CLI が終了コード 0 または 1 で終了するとき、THE Reporter SHALL 指定されたすべての形式（Text_Report / JSON_Report / SARIF_Report）のレポート出力を終了前に完了する。終了コードの値は出力形式の指定に左右されない
9. THE AgentGuard SHALL GitHub Actions から利用するためのワークフロー例をドキュメントとして提供する。この例には、`agent-guard scan` の実行、`--fail-on` の指定例、SARIF_Report の GitHub Code Scanning へのアップロードを含め、スキャンが終了コード 1 で終わったときもアップロードのステップが実行される構成にする

## Requirement 18: 修正提案（差分形式）

**User Story:** 開発者として、検出ごとに修正後の設定例を差分で見たい。そうすることで、調べ直さずにすぐ修正できる。

### Acceptance Criteria

1. WHEN Rule が構造的な修正（値の削除・置換・キーの追加）を定義している Finding を出力するとき、THE Reporter SHALL 当該 Finding 1 件分の修正のみを Config_Model に適用した結果を Config_Printer で整形し、元ファイルとの unified diff 形式の Fix_Suggestion を生成する。この Fix_Suggestion は、ヘッダーにスキャンルートからの相対パスを使い、各変更箇所の前後に最大 3 行のコンテキスト行を含める
2. WHEN Rule が構造的な修正を定義していない Finding を出力するとき、THE Reporter SHALL Rule_Pack の当該 Rule の修正例テキストを Fix_Suggestion として出力する
3. THE Reporter SHALL Fix_Suggestion 内のすべての Secret を Masked_Token に置換し、diff の追加行（`+` 行）で Secret の置換先として Env_Reference（例: `${GITHUB_TOKEN}`）を提示する。この Env_Reference の変数名には、Secret を保持していたキー名を英大文字・数字・アンダースコアに正規化した名前を使う
4. THE AgentGuard SHALL Fix_Suggestion を生成する際に Target_File の内容・サイズ・更新日時を変更せず、新しいファイルも作成しない
5. FOR ALL 構造的な修正を持つ Finding について、Fix_Suggestion 適用後の設定を Parser で解析した結果に対して同じ Rule を評価したとき、当該 Finding（同一 Fingerprint）は再検出されず、適用後の設定は Parser で Scan_Error なく解析できる（修正の有効性特性）
6. THE Reporter SHALL unified diff 形式の Fix_Suggestion で、修正対象の値・キーを含む行以外の行（キーの順序・インデント・改行コード・コメントを含む）を差分として出力しない
7. IF 構造的な修正を Config_Model に適用できないとき（修正対象の JSON Pointer が存在しない、修正後の Config_Model を Config_Printer で整形できない等）、THEN THE Reporter SHALL unified diff を出力せずに Rule_Pack の修正例テキストを Fix_Suggestion として出力し、当該 Finding とほかの Finding の出力を継続する
8. WHEN 1 つの Target_File に対して構造的な修正を持つ Finding が 2 件以上あるとき、THE Reporter SHALL Finding ごとに独立した Fix_Suggestion を生成し、ある Finding の Fix_Suggestion に別の Finding の修正内容を含めない

## Requirement 19: 抑制機能

**User Story:** PM / リードとして、許容済みのリスクを理由付きで抑制したい。そうすることで、既知の検出に埋もれずに新しいリスクに集中できる。

### Acceptance Criteria

1. WHEN `--config` を指定せずにスキャンを開始したとき、THE AgentGuard SHALL スキャンルート直下の `.agentguard.yml` を Suppression_File として読み込む。当該ファイルが存在しない場合はエラーを出力せず、Suppression_Entry を 0 件としてスキャンを継続する
2. WHEN Finding が Suppression_Entry の Rule ID と完全一致し、かつ Suppression_Entry に対象パスパターンが指定されている場合は Finding のスキャンルートからの相対パス（区切り文字 `/`）が当該パターンに一致し、Fingerprint が指定されている場合は Finding の Fingerprint と完全一致するとき、THE Rule_Engine SHALL 当該 Finding を出力対象の Finding から除外し、抑制済み件数に計上する。1 件の Finding が複数の Suppression_Entry に一致した場合も、抑制済み件数には 1 件として計上する
3. IF Suppression_Entry に理由（`reason`）が存在しない、または空文字列か空白文字のみのとき、THEN THE AgentGuard SHALL 当該 Suppression_Entry の Suppression_File 内のファイルパスと行番号を示す日本語のエラーを標準エラー出力に出力し、レポートを出力せず、THE CLI SHALL 終了コード 2 で終了する
4. IF Suppression_Entry の有効期限（`expires`）がスキャン実行時のローカル日付より前の日付のとき（`expires` と同日は有効とする）、THEN THE Rule_Engine SHALL 当該 Suppression_Entry を適用せずに一致する Finding を抑制せず出力し、さらに当該 Suppression_Entry の Suppression_File 内の行番号を位置として、期限切れであることを示す Severity Low の Finding を 1 件報告する
5. WHEN すべての Finding への抑制判定が完了した時点で、期限切れでない Suppression_Entry がいずれの Finding にも一致しなかったとき、THE Rule_Engine SHALL 当該 Suppression_Entry の Suppression_File 内の行番号を位置として、未使用の Suppression_Entry であることを示す Severity Info の Finding を 1 件報告する
6. FOR ALL Scan_Result と Suppression_File の組について、抑制適用後の Finding 集合は、抑制適用前の Finding 集合から期限切れでない Suppression_Entry のいずれかに一致する Finding を除いた集合と等しく、抑制済み件数は除かれた Finding の件数と等しい（抑制の正確性特性）
7. WHERE `--no-suppress` オプションが指定されたとき、THE Rule_Engine SHALL Suppression_Entry を 1 件も適用せずにすべての Finding を出力し、抑制済み件数を 0 とし、期限切れおよび未使用の Suppression_Entry に関する Finding を生成しない
8. FOR ALL 有効な Suppression_File について、読み込んだ内容を YAML に整形出力し再度読み込んだ結果は元の内容と等価である（ラウンドトリップ特性）
9. WHEN `--config <path>` が指定されたとき、THE AgentGuard SHALL `<path>` のファイルのみを Suppression_File として読み込み、スキャンルート直下の `.agentguard.yml` を読み込まない
10. IF `--config <path>` で指定したファイルが存在しない、または読み取り権限がないとき、THEN THE CLI SHALL 当該パスを示す日本語のエラーを標準エラー出力に出力し、レポートを出力せず、終了コード 2 で終了する
11. IF Suppression_File が YAML として構文エラーである、または Suppression_Entry に Rule ID が存在しない、または `expires` が `YYYY-MM-DD` 形式の実在する日付でないとき、THEN THE AgentGuard SHALL 違反箇所のファイルパスと行番号を示す日本語のエラーを標準エラー出力に出力し、レポートを出力せず、THE CLI SHALL 終了コード 2 で終了する

## Requirement 24: ローカル読み取り専用ビューア（Web UI）

**User Story:** PM / リードとして、スキャン結果をブラウザで絞り込みながら閲覧したい。そうすることで、レビュー時に重大度やファイル単位で状況を把握できる。

### Acceptance Criteria

1. WHEN ユーザーが `agent-guard serve --report <json_file>` を実行したとき、THE Viewer SHALL 指定された JSON_Report を読み込み、コマンド実行から 5 秒以内にローカルの HTTP サーバーで閲覧画面の提供を開始し、閲覧用 URL（`http://127.0.0.1:<port>/` 形式）を標準出力に出力する
2. WHILE Viewer が起動している間、THE Viewer SHALL 起動時に読み込んだ JSON_Report の内容のみを提供し、ユーザーが割り込み操作（Ctrl+C）を行ったとき、サーバーを停止して終了コード 0 で終了する
3. THE Viewer SHALL `127.0.0.1` のみにバインドし、ポート番号のデフォルトを 8765 とし、`--port` オプションまたは環境変数 `AGENTGUARD_VIEWER_PORT` で 1024〜65535 の範囲の値に変更できるようにし、両方が指定された場合は `--port` オプションの値を優先する
4. THE Viewer SHALL JSON_Report と画面表示用静的ファイルの読み取りのみを提供し、スキャンの実行・ファイルの書き込み・リクエストで指定された任意パスの読み取りを行う API を提供せず、Viewer の起動から終了までの間に `<json_file>` を含むいかなるファイルも作成・変更しない
5. THE Viewer SHALL Finding の一覧を Severity（複数選択）・Rule ID（複数選択）・ファイルパス（部分一致）・OWASP カテゴリ（複数選択）で絞り込めるようにし、異なる条件間は論理積、同一条件内の複数選択は論理和で評価し、絞り込み後の Finding 件数を一覧とともに表示する
6. WHEN ユーザーが一覧から Finding を選択したとき、THE Viewer SHALL 当該 Finding の Rule ID・Severity・ファイルパス・行番号・説明・Fix_Suggestion（unified diff のテキスト）・OWASP カテゴリを含む参考情報を表示する
7. THE Viewer SHALL React・TypeScript・MUI で実装し、画面幅 360px〜1920px の範囲で、ページ全体の横スクロールを発生させず、第 5 項の絞り込みと第 6 項の詳細表示のすべての操作を実行できるようにする
8. THE Viewer SHALL スキャン対象由来の文字列をすべてテキストとしてエスケープして表示し、HTML として解釈しない
9. THE Viewer SHALL すべての HTTP レスポンスに Content-Security-Policy ヘッダーを設定し、外部オリジンからのスクリプト・スタイル・フォントを読み込まない
10. IF 指定された `<json_file>` が存在しない、読み取り権限がない、JSON として解析できない、または JSON_Report のスキーマに適合しないとき、THEN THE Viewer SHALL HTTP サーバーを起動せずに原因を示す日本語のエラーメッセージを標準エラー出力に出力し、終了コード 2 で終了する
11. IF 指定されたポート番号が 1024〜65535 の範囲外である、数値でない、または既に他のプロセスに使用されているとき、THEN THE Viewer SHALL HTTP サーバーを起動せずに原因を示す日本語のエラーメッセージを標準エラー出力に出力し、終了コード 2 で終了する
12. WHEN JSON_Report の Finding が 0 件のとき、または絞り込み条件に一致する Finding が 0 件のとき、THE Viewer SHALL 該当する Finding がないことを示すメッセージを一覧領域に表示し、設定中の絞り込み条件を保持する

## Requirement 25: 設定値の外部化

**User Story:** 第三者の開発者として、自分の環境で AgentGuard をそのまま使いたい。そうすることで、ソースコードを書き換えずに導入できる。

### Acceptance Criteria

1. THE AgentGuard SHALL 環境依存の設定項目（Viewer のポート番号、最大ファイルサイズ、JSON の最大ネスト深さ）の値を、各項目について CLI オプション・環境変数・Suppression_File の 3 つの経路から取得する。環境変数名は `AGENTGUARD_` 接頭辞で統一する（`AGENTGUARD_VIEWER_PORT`、`AGENTGUARD_MAX_FILE_SIZE`、`AGENTGUARD_MAX_JSON_DEPTH`）。ソースコードには特定環境に紐づく値（絶対パス、ユーザー名、ホスト名、アカウント識別子）を記述せず、組み込みデフォルト値（最大ファイルサイズ 5 MiB、最大ネスト深さ 64、バインド先 `127.0.0.1`、デフォルトポート番号 8765）だけを持つ
2. THE AgentGuard SHALL ホームディレクトリを Requirement 2 第 3 項に従って OS の環境変数だけから解決し、ホームディレクトリの値をソースコード・Suppression_File の値として持たない
3. WHEN 同じ設定項目が複数の経路で指定されたとき、THE AgentGuard SHALL 項目ごとに優先順位「CLI オプション > 環境変数 > Suppression_File > 組み込みデフォルト」で最も上位の経路の値を採用し、下位の経路の値を無視する
4. IF 環境変数が定義されていても値が空文字列のとき、THEN THE AgentGuard SHALL 当該環境変数を未指定として扱い、次の優先順位の経路の値を採用する
5. IF 設定値が許容範囲外または不正な形式のとき（ポート番号が 1024〜65535 の整数でない、最大ファイルサイズが 1 バイト〜100 MiB の整数バイト値でない、最大ネスト深さが 1〜1024 の整数でない）、THEN THE CLI SHALL 不正な設定項目名と値の指定元（CLI オプション / 環境変数 / Suppression_File）を示す日本語のエラーメッセージを標準エラー出力に出力し、スキャンの開始と Viewer の起動を行わず、レポートファイルを書き込まずに終了コード 2 で終了する
6. THE AgentGuard SHALL 外部化したすべての設定項目について、CLI オプション名・環境変数名・Suppression_File のキー名・組み込みデフォルト値・許容範囲を一覧にしたドキュメントを提供する
