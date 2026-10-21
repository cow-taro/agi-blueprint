# 日本語版ガイド — agi-blueprint/ja

汎用 AI 統治フレームワークの日本語版。`../en/` に同一内容の英語版がある。**1 プロジェクトへ入れるのはどちらか一方。**

## 構成と読み込まれ方

| 種別 | 場所 | 読み込まれ方 | 収めるもの |
|---|---|---|---|
| rules | `rules/` | 常時（毎ターン適用） | 毎回適用される高頻度規約のみ |
| skills | `skills/` | 随時（該当作業のとき） | 稀にしか適用されない手順・ノウハウ |
| templates | `templates/` | 随時（複製して使う） | 記入式ひな形 |

ツリー直下の付録 `SAFE-AUTOMATION.md`（安全な AI 自動化の設計指針）は、エージェントが常時ロードする対象ではなく、採用時に人間が読んで範囲を決める参照文書である。

## ファイル一覧と役割

### rules（常時ロード）

| ファイル | 役割 |
|---|---|
| `rules/00-governance-core.md` | 最上位規約の正本。羅針盤（最上位目的）の記述枠・一意参照規約・生コード垂れ流し禁止・文書トポロジー固定・繰り返す失敗のエスカレーション |
| `rules/01-workflow-gates.md` | 計画と実行の分離・証拠ベース計画・編集前承認・doc-first・検収後の md 反映・モード選択・コストと自律度 |
| `rules/02-response-self-check.md` | レスポンス出力前の自己チェック（フィードバック処理）。ルール違反を検出したら修正してから出力する |
| `rules/03-test-before-implementation.md` | 実装前のテスト先行手順の要約（詳細は skills へ委譲） |

### skills（随時ロード）

| ファイル | 役割 |
|---|---|
| `skills/setup-wizard/SKILL.md` | 対話式初期設定ウィザード。`SETUP-CHECKLIST.md` の項目を質問して各ファイルへ書き込む |
| `skills/ask-when-unclear/SKILL.md` | 指示が曖昧・矛盾・不完全なときに推測で進めず聞き返す |
| `skills/edit-plan-approval/SKILL.md` | 編集計画 → 人間の承認 → 実行 の手順の正本 |
| `skills/git-repo-operations/SKILL.md` | git 操作の安全ルール（保護ブランチ・ブランチ作成・commit/push 前停止・cherry-pick 移植） |
| `skills/backend-test-first/SKILL.md` | 既存挙動を変える開発の一気通貫手順（実装前ゲート → desired red → 実装 → 緑 → テスター用 md） |
| `skills/snapshot-test-debugging/SKILL.md` | テストの正しい実行方法と想定外失敗の解析手順（出力握りつぶし禁止・証拠ファースト） |
| `skills/query-tuning/SKILL.md` | MySQL の重いクエリを高速化する方針（巨大 JOIN の分割・ビュー全 materialize 回避・認可境界の検証） |
| `skills/mysql-performance-diagnosis/SKILL.md` | EXPLAIN / performance_schema での診断・インデックス設計原則（サーバー設定変更は提案のみ・実行は人間） |
| `skills/fast-snapshot-db-reset/SKILL.md` | XtraBackup 物理スナップによる高速 DB リセットの導入方法・利用法サンプル |

### templates（複製して使う）

| ファイル | 役割 |
|---|---|
| `templates/desired-red-gate-checklist.md` | 実装前・失敗予測ゲートの自律セルフチェック用ひな形 |
| `templates/work-log-skeleton.md` | 人間が手書きせず AI が随時整合同期し続ける作業ログの骨子 |
| `templates/project-premises-template.md` | プロジェクト固有前提の追記先（複製して運用） |
| `templates/adoption-analysis-template.md` | 既存プロジェクト適用時の解析結果（構造化 md）のひな形。setup-wizard の解析モードで使用 |
| `templates/test-runbook-template.md` | 環境固有の確定コマンドと厳密な実行手順（残留プロセス停止・コンフリクト回避）を、生きた正本として追記編集し続ける手順書 |

### 付録（参照文書・複製して使うものではない）

| ファイル | 役割 |
|---|---|
| `SAFE-AUTOMATION.md` | AI を安全に自動化させるための設計指針。絶対禁止作業・特権作業の 2 経路・踏み台の常時停止・爆発半径の限定・量子化警告・採用チェックリスト |

## 導入手順

1. `ja/` ツリーの中身をプロジェクトへ配置する。Roo / Claude 系のエージェントツールなら `rules/` を rules ディレクトリへ、`skills/` を skills ディレクトリへ置く。規約機構がないツールなら `rules/` の内容をシステムプロンプトの**先頭**に固定する（コンテキストが長くなっても先頭のルールは参照されるため、ルール忘れを防げる）。
2. `SETUP-CHECKLIST.md` を開き、A 章必須項目を埋める。AI に `skills/setup-wizard/SKILL.md` を読み込ませると、質問に答えるだけで初期設定が完了する。
3. B 章で案件に合わせて選択・調整し、C 章に該当するスキル・節を削除する。
4. `templates/project-premises-template.md` を複製して前提ファイルとし、運用開始する。

## 運用中の拡張（繰り返す失敗の排除）

- 同じ失敗を 3 度したら: 個別の環境事情なら前提ファイルへ追記、汎用手順として再利用できるなら新規スキル化する（`rules/00-governance-core.md` の「6. 繰り返す失敗のエスカレーション」）。
- 前提ファイルへの追記は、ユーザーに提案してから行う。個別事案・一時的な判断はスキル化しない。
- 場当たり的に新しい番号付きルールファイルを作らない。
