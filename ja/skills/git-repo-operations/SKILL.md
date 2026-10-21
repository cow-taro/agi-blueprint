---
name: git-repo-operations
description: >-
  git 操作（ブランチ作成・push・PR・保護ブランチ判定）の安全ルール。git checkout /
  branch / push / merge / PR を実行する前、またはどのブランチが保護対象か・どこに push して
  よいか迷ったときに使う。SETUP-CHECKLIST A6 で実名へ置き換えたブランチ対応表を正とする。
disable-model-invocation: false
---

# git リポジトリ操作の安全ルール

## 0. ブランチ対応表（サンプルを実名へ置き換える）

下の表はサンプルである。**SETUP-CHECKLIST A6（保護ブランチ対応表）で、このプロジェクトの実名へ置き換えたものを正とする。** どれが実名か未確定のまま運用しない。

| ブランチ（サンプル） | 用途 |
|---|---|
| `main` | **本番**。デプロイに直結 |
| `develop` | **検証。日常の feature 起点** |

**同じ取り違えを繰り返さない:**
- md やスキルに古いブランチ名があっても、本スキルの対応表（実名版）を正とする。判断に迷ったら CI ワークフロー定義（`.github/workflows/*.yml` の `on.push.branches`）を確認する。

## 1. リポジトリ・ワークスペース構成の基本

- git 操作は必ず対象リポジトリのディレクトリに `cd` してから実行する（ワークスペースルートで `git init` / `commit` / `push` しない）。ルートに `.git/` があれば誤作成なので、ユーザー確認後に削除を提案する。
- 設定ファイル・秘匿情報・共有ドキュメント等、**どのリポジトリにも属さないもの**を commit 提案しない。対象リポジトリの `.gitignore` を確認する。
- ドキュメントだけを管理するリポジトリがある場合、その運用（feature ブランチを切るか・常に main か）を SETUP-CHECKLIST A6 の対応表に明記する。

## 2. 保護ブランチ（直接 push 絶対禁止）

- 対応表の本番・検証系ブランチ（サンプルでは `main` / `staging` / `develop` / `test`）への直接 push は、理由の如何を問わず絶対禁止。force push は言うまでもなく通常 push も禁止。
- 残置ブランチへも直接 push しない（存在するだけで現行ではない）。
- マージは **GitHub PR 経由のみ**。エージェントが保護ブランチに直接マージ・push しない。
- どのブランチがデプロイに直結するかは、CI ワークフロー定義の実体で確認する（対応表に反映しておく）。

## 3. ブランチ作成ルール

- **新規ブランチを切る前に、必ずベースブランチをリモートの最新に pull してから分岐する。** ローカルの古いベースから分岐すると、後で最新との差分・コンフリクトの原因になる。
- 日常のベースは対応表の「feature 起点」ブランチ（サンプルでは `staging`）。本番ブランチ・残置ブランチから切らない（B4 で trunk based を採用した場合はその限りではない）。

```bash
git checkout staging
git pull origin staging
git checkout -b feature/xxx
```

- ❌ 禁止: `git checkout -b feature/xxx origin/保護ブランチ`（upstream が保護ブランチに設定され、誤 push の原因になる）。必ずトラッキングなしで作成する。
- pull 前に `git fetch` でリモートの状態を確認してよいが、**分岐前の pull 自体を省略しない**。

## 4. push・commit・マージのルール

- **push は新規ブランチのみ**: `git push -u origin feature/xxx` で新しいリモートブランチに push する。
- **`git commit` / `git push` / `git push -u origin` の直前で必ず止まり、コマンドをユーザーに提示して実行を求める。** 「進めて」「実装して」等の依頼語は commit/push の許可にならない。feature ブランチ・新規ブランチも例外なし（SETUP-CHECKLIST B5 で条件付き許可にしていない限り）。
- 秘匿情報（`.env` の実値・認証情報・個人情報の生データ）は commit しない。commit 前に `git status` で対象を確認し、混入していないか目視する。
- 破壊的操作（`git reset --hard`・force push・history rewrite 等）はユーザーの明示指示がある場合のみ。事前に対象と影響を編集計画で提示してから実行する。

## 5. 作業終了後の状態確認

ブランチをまたぐ操作（マージ済み報告を受けた後、cherry-pick 後など）の後は、思い込みで進めず実際の状態を確認してから次の作業に着手する。

```bash
git status --short --branch
git log --oneline -3
```

## 6. 保護ブランチ間の移植（cherry-pick）の手順

検証ブランチにマージ済みの修正を、本番ブランチへ移植するときの手順（サンプル: `staging` → `main`）。

日常フロー: feature → PR → 検証ブランチ → 本番反映時は本番ブランチ向けブランチを作り同一修正を cherry-pick 等 → PR → 本番ブランチ → 自動デプロイ。

### 6-1. 前提・禁止事項

- 本番と検証は日常的に履歴が離れる。**ブランチ全体の `git diff` で移植対象を決めない。** 移植するコミット（または内容）だけを特定する。
- 残置ブランチへ移植しない・そこから取り込まない。
- 判断前に本スキルの「0. ブランチ対応表」と、デプロイ環境の正本 md（あれば）を先に読む（doc-first）。

### 6-2. 移植対象コミットの特定（`git cherry` を鵜呑みにしない）

`git cherry` は、squash / rebase 由来で内容は既に入っているのに別 SHA として「未適用」と表示されることがある。必ず内容ベースで確認する。

```bash
git cherry -v main staging
git grep -n "<その変更の特徴的な文字列>" main
git ls-tree -r --name-only main
```

- 機能変更が別 SHA で既に本番ブランチに入っている場合は**二重適用しない**（対象から除外）。
- 実際に新規のコミットだけを、適用する起点コミット以降に絞り込む。

### 6-3. ブランチ作成と cherry-pick

```bash
git checkout main
git pull origin main
git checkout -b feature/apply-xxx-to-main
git cherry-pick -x <c1> <c2> ... <cN>
```

- 衝突が出たら**自動解決せず**、対象ファイルを報告してユーザー判断を仰ぐ。
- 中断状態は `.git/CHERRY_PICK_HEAD` の有無で判定できる。

### 6-4. 健全性検証（push 前に必須）

- コンフリクトマーカーの残留確認（`<<<<<<< ` 等の検索）。
- 重要ファイル（テスト基盤・マイグレーション・設定）の存在確認。
- **マーカー付きテスト等での緑確認**（`skills/snapshot-test-debugging/SKILL.md` の残留プロセス確認を忘れない）。

### 6-5. push / PR はユーザー手動

- push・PR 作成の直前で必ず止まり、コマンドを提示する（「4. push・commit・マージのルール」）。
- **PR の base は本番ブランチ（デプロイに直結する）**である点を明示して注意喚起する。
- マイグレーションが含まれる場合、各環境への適用状況（冪等か・適用済みか）も併記する。
