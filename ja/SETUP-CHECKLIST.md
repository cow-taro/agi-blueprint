# SETUP-CHECKLIST（初期設定の棚卸し）

このテンプレートをプロジェクトへ採用するときに、入力・調整すべき項目の正本一覧。

- AI に `skills/setup-wizard/SKILL.md` を読み込ませると、この順に質問・サジェストして埋めさせられる。
- 既存プロジェクトへの適用では、setup-wizard がソースコードとドキュメントを解析し、`templates/adoption-analysis-template.md` に構造化して、A 章の値をサジェストする。
- 記入済みの判定: 各項目の「書き込み先」を開き、プレースホルダが実値に置き換わっていれば済み。
- 未入力の A 章項目が残ったまま運用を開始してはならない。

---

## A. 絶対に最初に入力する項目（必須）

### A1. 最上位目的（羅針盤）

- 書き込み先: `rules/00-governance-core.md` の「0. 最上位目的（羅針盤）」
- 形式: 1〜5 行。このプロジェクトが存在する目的を、実名（製品名・対象業務）で書く
- 記入例: 「複数の外部システムと連携する業務システムを、必要な機能とともに安定提供すること」
- すべての判断はここに立ち返る。あらゆる実装・調査・修正の方向は「この目的を満たすために何が必要か」から決める

### A2. 採用言語

- 選択: 日本語（この `ja/` ツリー）／英語（`../en/` ツリー）
- 採用した方だけをプロジェクトへ入れる。両方入れると参照が二重化し、取り違えが起きる

### A3. ドキュメント正本ディレクトリ

- 書き込み先: `rules/01-workflow-gates.md` の「2. doc-first」
- 形式: 仕様・手順・確認事項の md を置くディレクトリの実パス
- 記入例: `docs/forAI/` と `docs/`

### A4. 作業計画 md・テスター用 md の置き場所

- 書き込み先: `rules/01-workflow-gates.md` の「2. doc-first」と `skills/backend-test-first/SKILL.md` の「0. 作業計画」
- 記入例: `docs/plans/`

### A5. テスト基盤の実名

- 書き込み先: `rules/03-test-before-implementation.md`、`skills/backend-test-first/SKILL.md`、`skills/snapshot-test-debugging/SKILL.md`
- 形式: テストフレームワーク名・実行コマンド・スイートの分け方（マーカー等）・golden（現挙動を固定する期待値ファイル）の概念の有無
- 自動テストを導入しない場合は C2 に従って該当 skills を削除する
- 書き込み先（追加）: `templates/test-runbook-template.md` を複製した**テスト実行手順書**へ、そのまま実行して成功する確定コマンドを記入する。手順書は生きた正本として、環境変更・コマンド変更・失敗のたびに追記編集する

### A6. 保護ブランチ対応表

- 書き込み先: `skills/git-repo-operations/SKILL.md` の「0. ブランチ対応表（サンプルを実名へ置き換える）」
- 形式: 本番用・検証用・feature 起点・残置ブランチの実名
- 記入例: 本番 `main`、検証と feature 起点 `staging`、残置 `master`

### A7. プロジェクト固有前提の初期入力

- 書き込み先: `templates/project-premises-template.md` を複製した前提ファイル
- 形式: 環境ごとの差異・外部依存の停止スケジュール・実行環境（サンドボックス・権限）の癖など、繰り返し踏みそうな罠

---

## B. 案件に合わせてカスタマイズ・選択する項目（推奨）

### B1. モード名の読み替え

- 書き込み先: `rules/01-workflow-gates.md` の「5. モード選択とリスク」
- 利用するエージェントツールのモード体系（Ask / Plan / Debug / Agent 等の実名）に表を読み替える

### B2. テスター用成果物の形式

- 書き込み先: `skills/backend-test-first/SKILL.md` の「9. テスター用 md（終点）」
- 選択肢: md ＋ スプレッドシート用 tsv（既定）／ md のみ／テスター運用なし（この選択は推奨しない。人間の検収ゲートが消える）

### B3. golden（現挙動固定）運用の採用

- 書き込み先: `skills/backend-test-first/SKILL.md` の「固定手順（この順以外で実装禁止）」
- golden の概念がないテスト基盤の場合は、該当ステップをその基盤に読み替える

### B4. git 戦略

- 書き込み先: `skills/git-repo-operations/SKILL.md` の「3. ブランチ作成ルール」
- 選択肢: 検証ブランチを feature 起点にする（既定）／ trunk based（本番ブランチから直接 feature を切る）等

### B5. commit / push の運用

- 書き込み先: `rules/01-workflow-gates.md` の「3. 編集前承認」と `skills/git-repo-operations/SKILL.md` の「4. push・commit・マージのルール」
- 既定は「AI は実行せず常に人間手動」。条件付き許可にする場合は、許可条件を明文化する

### B6. 要確認事項・問題の起票先

- 書き込み先: `skills/backend-test-first/SKILL.md` の「バグ挙動の扱い」
- 仕様確認が必要な発見・既知の問題を起票するリスト（ファイル or issue 管理）の実名を書く

---

## C. 必要なければ削除してよい項目

削除するときは、残るファイルから削除対象への参照箇所もあわせて外す（`skills/setup-wizard/SKILL.md` が削除提案を出す。削除は必ず人間の承認を経る）。

### C1. DB を使わないプロジェクト

- 削除候補: `skills/query-tuning/SKILL.md`、`skills/mysql-performance-diagnosis/SKILL.md`、`skills/fast-snapshot-db-reset/SKILL.md`

### C2. 自動テスト基盤を持たない・導入しないプロジェクト

- 削除候補: `skills/backend-test-first/SKILL.md`、`skills/snapshot-test-debugging/SKILL.md`、`templates/desired-red-gate-checklist.md`
- この場合も `rules/03-test-before-implementation.md` の要約は、代替の証拠手順（手動確認・実ログ提示）に読み替えて残す

### C3. git を AI に触らせない運用

- 削除候補: `skills/git-repo-operations/SKILL.md`

### C4. 物理スナップを別方式にする場合

- 書き換え候補: `skills/fast-snapshot-db-reset/SKILL.md` の XtraBackup 固有の節（セットアップ・トラブルシュート）を、採用する別方式（論理ダンプの差分適用等）へ置き換える
