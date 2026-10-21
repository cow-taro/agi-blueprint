---
name: mysql-performance-diagnosis
description: >-
  MySQL のスロークエリ・重い API の原因を証拠ベースで調査するための汎用診断スキル。
  EXPLAIN / EXPLAIN ANALYZE の読み方、performance_schema での犯人特定、インデックス設計の
  原則、クエリ書き換えパターンをまとめる。「なぜ遅いか分からない」「どのインデックスが
  必要か」「EXPLAIN の見方」を調べるときに使う。このテンプレートの書き換え方針は query-tuning
  スキルを併用する。
disable-model-invocation: false
---

# MySQL パフォーマンス診断（EXPLAIN・performance_schema）

出典: 公開スキル [jsamuelsen11/claude-config `performance-tuning`](https://github.com/jsamuelsen11/claude-config/tree/main/plugins/ccfg-mysql/skills/performance-tuning) から、汎用プロジェクトで役立つ部分だけを抽出・日本語化したもの。

書き換え方針（巨大 JOIN の分割・ビューのネスト回避）は `skills/query-tuning/SKILL.md` を、テスト実行・性能実測の手順は `skills/snapshot-test-debugging/SKILL.md` を併用する。

## ⚠️ 最重要の安全注意: サーバー設定変更は「提案はする・実行は人がやる」

本スキルの元ネタには `innodb_buffer_pool_size` / `innodb_flush_log_at_trx_commit` / `innodb_io_capacity` / `SET GLOBAL ...` などの**MySQL サーバー設定（`my.cnf`・グローバル変数）の変更**に関する記述が多く含まれる。サーバー設定の調整が最適なチューニングである場合は珍しくない。**その場合はためらわずに、証拠（EXPLAIN・performance_schema の実測）とともに設定変更を提案する。** 設定変更が最適解なのに、それを避けてクエリを無理に複雑化させる（コードを汚す）のは本末転倒。

ただし、**本番・検証環境のサーバー設定変更を「実行」するのは必ず人（DBA/管理者）**。

- AI は**提案までを行う**: 「どの設定が原因の可能性が高いか」「推奨値と根拠」「変更のトレードオフ」「ロールバック手順」を証拠つきで示す。
- **実行（`SET GLOBAL`・パラメータグループ変更・`my.cnf` 編集・再起動）は人が GUI / 手作業で行う。** 本番・検証環境に対する CLI 操作を AI が実行しない。
- ローカル開発 DB に対する**読み取り系の診断クエリ（EXPLAIN・SHOW・performance_schema の SELECT）は AI が自分で実行してよい**。ローカルの設定変更検証が必要な場合も、影響と手順を提示してから行う。

**優先順位**: プロジェクト固有のスキル・ルール（`skills/query-tuning/SKILL.md` 等）が最優先。本スキルの汎用ノウハウはそれらと矛盾しない範囲で使い、矛盾する場合はプロジェクト側を優先する。

## 1. まず測る（推測で修正しない）

- 「たぶん JOIN が遅い」「たぶんインデックス不足」と**推測で修正に入らない**。必ず EXPLAIN と実測（`skills/snapshot-test-debugging/SKILL.md` の性能検証手順）で犯人を特定してから直す。
- 変更は 1 つずつ。1 つ直したら EXPLAIN / 実測で効果を確認し、次に進む。

## 2. EXPLAIN の読み方

まず `EXPLAIN FORMAT=TREE`（実行フローの把握）、必要なら `EXPLAIN ANALYZE`（実行して実測時間つき・MySQL 8.0.18+）を使う。`FORMAT=JSON` はコスト詳細を見たいとき。

### アクセスタイプ `type`（良い → 悪い）

- `const` / `eq_ref`: PK・ユニークインデックスでの単一行引き（最良）
- `ref`: 非ユニークインデックスでの複数行引き（良い）
- `range`: インデックスの範囲スキャン（`BETWEEN` / `<` / `>` / `IN`）
- `index`: インデックスのフルスキャン（`ALL` よりはマシ）
- `ALL`: **フルテーブルスキャン（最悪）**。大きいテーブルで出たら要対応

### `Extra` の警告サイン

- `Using filesort`: インデックスで並び順を賄えず、別途ソートしている。→ WHERE 列 + ORDER BY 列の複合インデックスで解消できることが多い。
- `Using temporary`: 一時テーブルを作っている（GROUP BY / DISTINCT 等）。→ GROUP BY / DISTINCT 対象列のインデックスで解消できることが多い。
- `Using index`: **カバリングインデックス**でテーブル本体を読まずに済んでいる（良い状態）。

### rows / filtered

- `rows`（調査対象の推定行数）が返却行数より**桁違いに多い**なら、絞り込みが効いていない（インデックス不足の典型）。`EXPLAIN ANALYZE` の `actual rows` と `loops` で実際の再実行回数も見る。
- `EXPLAIN FORMAT=TREE` で `Materialize` / `Union all materialize` が出ていたら、ビューやサブクエリが全件展開されている疑い（`skills/query-tuning/SKILL.md` のビュー全 materialize 問題を参照）。

## 3. performance_schema で犯人を特定する（ローカル調査用）

`performance_schema` は稼働中に自動収集される統計。ローカル DB では有効（`performance_schema=ON`）。以下は**読み取りのみの診断クエリ**なので AI が実行してよい。

```sql
-- 総実行時間の重いクエリ TOP10
SELECT DIGEST_TEXT AS q, COUNT_STAR AS cnt,
       ROUND(SUM_TIMER_WAIT/1e12,3) AS total_sec,
       ROUND(AVG_TIMER_WAIT/1e12,3) AS avg_sec,
       SUM_ROWS_EXAMINED AS examined, SUM_ROWS_SENT AS sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;

-- examined/sent 比が悪い（＝インデックス不足の疑い）クエリ TOP10
SELECT DIGEST_TEXT AS q, COUNT_STAR AS cnt,
       SUM_ROWS_EXAMINED AS examined, SUM_ROWS_SENT AS sent,
       ROUND(SUM_ROWS_EXAMINED/NULLIF(SUM_ROWS_SENT,0),1) AS ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_SENT > 0
ORDER BY SUM_ROWS_EXAMINED/NULLIF(SUM_ROWS_SENT,0) DESC LIMIT 10;

-- フルテーブルスキャンしているステートメント
SELECT * FROM sys.statements_with_full_table_scans
ORDER BY no_index_used_count DESC LIMIT 10;

-- 使われていないインデックス（書き込みコストの無駄・削除候補）
SELECT * FROM sys.schema_unused_indexes
WHERE object_schema NOT IN ('mysql','performance_schema','sys');

-- 冗長なインデックス
SELECT * FROM sys.schema_redundant_indexes
WHERE table_schema NOT IN ('mysql','performance_schema','sys');
```

> 注意: `performance_schema` の統計は「起動以降の累積」。直前の作業で計測したいときは、対象クエリを実行する前後で digest を見るか、テストで対象 API だけを叩いて比較する（`skills/snapshot-test-debugging/SKILL.md` の「2. 実行前に必ずやること」）。

## 4. インデックス設計の原則

- **複合インデックスは leftmost prefix**: `INDEX(a,b,c)` は `WHERE a=..`、`WHERE a=.. AND b=..` に効くが、`WHERE b=..` 単独には効かない。先頭列から順に使われる。
- **範囲条件は以降の列を止める**: `WHERE a=.. AND b>.. AND c=..` だと `a,b` までしかインデックスを活かせず、`c` はフィルタ評価になる。等値条件を前、範囲条件を後ろに置く。
- **カバリングインデックス**: WHERE/JOIN 列 → ORDER BY 列 → SELECT 列 の順で複合 index に含めると、テーブル本体を読まずに済む（`Using index`）。頻繁・性能クリティカルなクエリにだけ使う。
- **暗黙の型変換に注意**: `WHERE phone = 5551234567`（`phone` が VARCHAR）は全行を数値変換して比較しフルスキャンになる。必ず型を合わせる（`WHERE phone = '5551234567'`）。
- **インデックス列に関数を使わない**: `WHERE YEAR(created_at)=2025` はインデックスを使えない。範囲条件 `created_at >= '2025-01-01' AND created_at < '2026-01-01'` に書き換える。
- **過剰インデックス禁止**: インデックスは読みを速くするが書き込みを遅くする。実際のクエリパターンに基づいて必要なものだけ作る。EXPLAIN で使われることを確認する。

欠損インデックスを追加するときは**直接 `ALTER TABLE` を実行しない**。必ずマイグレーションファイルを作り、その経由で追加する（`skills/query-tuning/SKILL.md` の Step 0 と同旨）。

## 5. クエリ書き換えパターン

- **相関サブクエリ → JOIN / EXISTS**: `WHERE id IN (SELECT ...)` は外側の行ごとに再評価されがち。`JOIN` か `EXISTS` に書き換える（`skills/query-tuning/SKILL.md` の「IN（相関サブクエリ）を JOIN（導出テーブル）に書き換える」も参照）。
- **SELECT 句のスカラサブクエリ → LEFT JOIN + 集約**: 行ごとに副問い合わせが走るのを避ける。
- **OFFSET ページング → キーセット（カーソル）ページング**: `LIMIT 20 OFFSET 100000` は先頭を捨てるため深いページほど遅い。`WHERE (created_at,id) < (?,?) ORDER BY ... LIMIT 20` にする。
- **異なる列への OR → UNION [ALL]**: 複数列にまたがる OR は単一インデックスで捌けない。各条件を単一テーブル・単一条件の SELECT に分けて UNION する（重複除去に注意）。認可条件（tenant / dept / self）の再構築でも有効。`skills/query-tuning/SKILL.md` 参照。
- **バルク操作後は `ANALYZE TABLE`**: 大量 INSERT/UPDATE/DELETE 後は統計が古くなり最適化が狂う。`ANALYZE TABLE <table>` で統計を更新する（これはデータではなく統計更新なので比較的安全だが、本番での実行は人の判断で行う）。

## 6. このスキルの守備範囲と、他スキルとの分担

- **本スキル**: EXPLAIN の読み方・performance_schema での犯人特定・インデックス設計原則・一般的なクエリ書き換え（汎用的な MySQL 調査ノウハウ）。
- **`skills/query-tuning/SKILL.md`**: プロジェクトの事情に寄せた書き換え方針（UNION ALL ビューの全 materialize、ビューのネスト、型サフィックス付き共有 ID、認可条件の UNION 分割、巨大 JOIN の分割方針）。
- **`skills/snapshot-test-debugging/SKILL.md`**: テスト実行・残留プロセス確認・DB リセット要否・性能実測・失敗解析の手順。

調査は「本スキルで EXPLAIN / performance_schema から犯人を特定」→「query-tuning で方針に沿って書き換え」→「snapshot-test-debugging で回帰と実測を確認」の順で連携させる。
