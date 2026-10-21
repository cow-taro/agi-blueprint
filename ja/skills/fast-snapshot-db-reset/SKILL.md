---
name: fast-snapshot-db-reset
description: >-
  Percona XtraBackup の物理スナップショットによる高速 DB リセットの導入方法と利用法サンプル
  （ローカル・開発用 CI 向け）。毎回の論理ロードが遅くてスナップショットテストが重いとき、
  セットアップ・トラブルシュート・復旧手順を調べるときに使う。
disable-model-invocation: false
---

# XtraBackup による高速 DB リセット（導入・利用サンプル）

スナップショットテスト用の DB を、毎回の論理リセット（ダンプファイルの反映＋ seed 適用）ではなく、物理スナップから戻せるようにする**任意機能**。未設定なら従来どおりの論理ロードを使う。本番・検証のアプリ DB には使わない。

本サンプルの実装箇所（テストヘルパへ組み込む処理の仕様）:

- `prepare`: 論理リセット → `xtrabackup --backup` → `xtrabackup --prepare` → 完了マーカー書き込み
- `reset`: マーカーと checkpoints 確認 → DB 停止 → datadir クリア → copy-back → 権限修正 → DB 起動 → 接続待ち（seed 再適用なし。スナップに込み）

## 1. 目的

毎回の論理ロードが遅いので、ローカルおよび開発用 CI では物理スナップから戻せるようにする。

- 未設定／`sql`: 論理ロード（ダンプ＋seed）
- ローカル任意・開発用 CI: 物理スナップモード（環境変数 `SNAPSHOT_RESET_MODE=xtrabackup`）
- XtraBackup 非対応の開発機: 論理ロードで既存経路

MySQL Clone Plugin（mysqld 2 系統）は採用しない。

## 2. 前提・制約

- MySQL 8.0.x と、同メジャー向け XtraBackup（`percona-xtrabackup-80`）
- **sudo 前提**（mysqld の stop／start、datadir クリア、`chown`）
- **破壊的**: datadir を空にして copy-back する。**同一 mysqld 上の他スキーマもスナップ時点に戻る／消える**。同一インスタンス上に他アプリ DB を置いている場合は、有効化前にその DB をダンプしておく
- 失敗して MySQL が起動できないときは、起動できる状態にしてから論理リセットで復旧できる（「6-1. 失敗時の復旧（論理リセット）」）
- 物理スナップを git／S3 に載せない（スナップディレクトリは gitignore）
- テスト用の DB 接続設定が `.env` 等でテストと同一であること

## 3. セットアップ（開発機・一回）

### 3-1. Percona XtraBackup

Ubuntu 22.04 系の例:

```bash
cd /tmp
curl -fsSL -O https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo apt-get install -y ./percona-release_latest.generic_all.deb
sudo percona-release enable-only tools release
sudo apt-get update
sudo apt-get install -y percona-xtrabackup-80
xtrabackup --version
```

### 3-2. MySQL 権限

必要: `BACKUP_ADMIN` / `PROCESS` / `RELOAD`、および `SELECT ON performance_schema.*`（無いと backup が 1142 で失敗する）。

先に host を確認する。MySQL 8 では存在しない `user@host` への GRANT は ERROR 1410 になる。

```bash
sudo mysql -e "SELECT user, host FROM mysql.user WHERE user='<db_user>';"
```

`localhost` がある典型例:

```bash
sudo mysql -e "GRANT BACKUP_ADMIN, PROCESS, RELOAD ON *.* TO '<db_user>'@'localhost'; GRANT SELECT ON performance_schema.* TO '<db_user>'@'localhost'; FLUSH PRIVILEGES;"
```

`DB_HOST=127.0.0.1`（TCP）用に `@'%'` が無ければ、先に CREATE してから GRANT（パスワードは `.env` の実値。プレースホルダをそのまま書かない）:

```sql
CREATE USER IF NOT EXISTS '<db_user>'@'%' IDENTIFIED BY '<.env の DB パスワード>';
GRANT BACKUP_ADMIN, PROCESS, RELOAD ON *.* TO '<db_user>'@'%';
GRANT SELECT ON performance_schema.* TO '<db_user>'@'%';
FLUSH PRIVILEGES;
```

```bash
sudo mysql -e "SHOW GRANTS FOR '<db_user>'@'localhost';"
```

### 3-3. passwordless sudo（限定コマンド）

`<login_user>` を自分の Linux ログイン名に置き換える（`<login_user>` のまま残すと visudo が syntax error）。`SNAPSHOT_MYSQL_*_CMD` を変えた場合は、sudoers も**実際に実行するコマンド文字列**に合わせる。

```bash
sudo tee /etc/sudoers.d/snapshot-xtrabackup >/dev/null <<'EOF'
Cmnd_Alias SNAPSHOT_XBK = /usr/bin/systemctl stop mysql, /usr/bin/systemctl start mysql, /usr/bin/systemctl stop mysqld, /usr/bin/systemctl start mysqld, /usr/sbin/service mysql stop, /usr/sbin/service mysql start, /usr/bin/xtrabackup, /usr/bin/find, /usr/bin/chown, /bin/rm
<login_user> ALL=(root) NOPASSWD: SNAPSHOT_XBK
EOF
sudo chmod 440 /etc/sudoers.d/snapshot-xtrabackup
sudo visudo -cf /etc/sudoers.d/snapshot-xtrabackup
cat /etc/sudoers.d/snapshot-xtrabackup
```

一般ユーザーで確認（root シェルのままでは意味がない）:

```bash
sudo -n systemctl stop mysql
sudo -n systemctl start mysql
sudo -n xtrabackup --version
```

`NOPASSWD: ALL` は使わない。WSL で `systemctl` が使えないときは「3-4. 環境変数」の stop／start を `service mysql` にする。

### 3-4. 環境変数（`.env` またはシェル）

```bash
SNAPSHOT_RESET_MODE=xtrabackup
SNAPSHOT_XTRABACKUP_BIN=sudo -n xtrabackup
SNAPSHOT_MYSQL_STOP_CMD=sudo -n systemctl stop mysql
SNAPSHOT_MYSQL_START_CMD=sudo -n systemctl start mysql
SNAPSHOT_MYSQL_CHOWN_CMD=sudo -n chown -R mysql:mysql /var/lib/mysql
SNAPSHOT_MYSQL_CLEAR_CMD=sudo -n find /var/lib/mysql -mindepth 1 -maxdepth 1 -exec rm -rf {} +
SNAPSHOT_XTRABACKUP_RM_CMD=sudo -n rm -rf {snap}
```

任意: `SNAPSHOT_XTRABACKUP_DIR`（既定 `tests/db/xtrabackup-snap`）、`SNAPSHOT_MYSQL_DATADIR`（既定 `/var/lib/mysql`）。`CLEAR_CMD` で `{datadir}`、`RM_CMD` の `{snap}` はスナップディレクトリに置換。未設定時は素の `rm -rf`（CI 向け）。ローカルで `sudo xtrabackup` を使うとスナップ配下が mysql 所有になるため、再 prepare には `RM_CMD` が必要。sudoers の Cmnd_Alias に既にある `/bin/rm` で足りる（追加の NOPASSWD は不要）。

論理リセットに戻すとき: `SNAPSHOT_RESET_MODE=sql`（実行時指定が `.env` より優先）。

### 3-5. 黄金スナップ作成

```bash
<テストヘルパの prepare コマンド>
```

成功するとスナップディレクトリに `.snapshot_ready` と `data/xtrabackup_checkpoints` ができる。dump／seed 変更後は再実行。

### 3-6. セットアップ確認

```bash
SNAPSHOT_RESET_MODE=xtrabackup <テストヘルパの reset 呼び出し>
```

数秒〜数十秒で戻り、MySQL に再接続できること。

| 項目 | 完了 |
|---|---|
| `xtrabackup --version` が 8.0 系 | |
| `BACKUP_ADMIN` と `performance_schema` SELECT | |
| sudoers.d が `visudo -cf` OK | |
| `sudo -n` で stop/start/xtrabackup／clear | |
| `.env` の `SNAPSHOT_*` | |
| 他アプリ DB を同一インスタンスに置くなら事前ダンプ済み | |
| 黄金スナップ作成が成功 | |
| **直後にもう一度**スナップ作成が成功（旧スナップ削除＝`RM_CMD` の確認） | |
| reset（xtrabackup）成功 | |

`Permission denied` で旧スナップ削除に失敗するときは、素の `rm -rf` のままか `SNAPSHOT_XTRABACKUP_RM_CMD` 未設定を疑う。

## 4. テスト実行

**導入済みのときだけ**、全件は xtrabackup。dump／seed／スキーマ変更後は「3-5. 黄金スナップ作成」でスナップを取り直してから回す。スナップが古いことを理由に sql 全件へ逃げない（`skills/snapshot-test-debugging/SKILL.md` の「DB リセット方式の分岐（物理スナップあり／なし）」）。

```bash
<テストヘルパの prepare コマンド>
SNAPSHOT_RESET_MODE=xtrabackup <テスト実行コマンド>
```

**未導入の開発機**は従来どおり論理ロード（`SNAPSHOT_RESET_MODE` 未設定または `sql`）。prepare は不要。

```bash
SNAPSHOT_RESET_MODE=sql <テスト実行コマンド>
```

## 5. 処理の流れ

### prepare

1. 論理リセット（ダンプ＋ seed）
2. `xtrabackup --backup`
3. `xtrabackup --prepare`
4. `.snapshot_ready` を `data/` の外に書く

### reset（`SNAPSHOT_RESET_MODE=xtrabackup`）

1. マーカーと checkpoints 確認
2. stop → datadir クリア（`SNAPSHOT_MYSQL_CLEAR_CMD`）→ copy-back → chown → start → ready 待ち
3. seed 再適用なし（スナップに込み）

## 6. トラブルシュート

| 症状 | 確認 |
|---|---|
| `.snapshot_ready` が無い | 「3-5. 黄金スナップ作成」を先に実行 |
| sudo でパスワードを聞かれる | 「3-3. passwordless sudo」。一般ユーザーで `sudo -n` を確認。CLEAR／STOP 等が sudoers と一致しているか |
| `BACKUP_ADMIN` / LOCK 拒否 | 「3-2. MySQL 権限」 |
| `performance_schema` SELECT 1142 | 「3-2. MySQL 権限」の `GRANT SELECT ON performance_schema.*` |
| ERROR 1410 | 存在しない `user@host` へ GRANT。先に `mysql.user` を確認 |
| copy-back / 起動失敗 | MySQL を起動できる状態にしてから「6-1. 失敗時の復旧（論理リセット）」 |
| seed 変更が反映されない | 「3-5. 黄金スナップ作成」を再実行 |
| 他スキーマが消えた | 「2. 前提・制約」。事前ダンプから戻す |

### 6-1. 失敗時の復旧（論理リセット）

この機能はローカル任意の破壊的処理なので、失敗時の正攻法は**論理リセット経路に戻す**こと。

1. MySQL が起動できるか確認する（できない場合は OS／パッケージ側で datadir を直すか、mysqld を再導入する）
2. 論理リセット:

```bash
SNAPSHOT_RESET_MODE=sql <テストヘルパの reset 呼び出し>
```

3. 必要なら XtraBackup をやめて `.env` の `SNAPSHOT_RESET_MODE` を外すか `sql` にする

## 7. やらないこと

- 本番／検証アプリへの同設定、datadir の個人 chown、sudoers の無制限 NOPASSWD、物理スナップの git／S3 投入
- 開発用 CI の常時ゲート化（別承認）
- passwordless systemd や datadir ACL による sudo 削減は別タスク
