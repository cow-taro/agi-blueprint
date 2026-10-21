---
name: fast-snapshot-db-reset
description: >-
  Introduction and usage sample of a fast DB reset via Percona XtraBackup physical
  snapshots (for local machines and dev CI). Use when the logical load makes
  snapshot tests heavy every run, or when looking up setup, troubleshooting, and
  recovery steps.
disable-model-invocation: false
---

# Fast DB reset via XtraBackup (introduction and usage sample)

An **optional feature** that restores the snapshot-test DB from a physical snapshot instead of a logical reset (dump load + seed application) every run. When unset, the conventional logical load is used. Never used for production/verification application DBs.

Implementation points to embed into your test helpers (processing spec):

- `prepare`: logical reset → `xtrabackup --backup` → `xtrabackup --prepare` → write completion marker
- `reset`: check marker and checkpoints → stop DB → clear datadir → copy-back → fix ownership → start DB → wait for connections (seeds are NOT re-applied; included in the snapshot)

## 1. Purpose

The logical load is slow every run, so local machines and dev CI can restore from a physical snapshot.

- Unset / `sql`: logical load (dump + seeds)
- Local optional / dev CI: physical snapshot mode (env var `SNAPSHOT_RESET_MODE=xtrabackup`)
- Machines without XtraBackup support: conventional logical load

The MySQL Clone Plugin (a second mysqld) is not adopted.

## 2. Prerequisites and constraints

- MySQL 8.0.x and the matching XtraBackup major (`percona-xtrabackup-80`)
- **sudo required** (mysqld stop/start, datadir clearing, `chown`)
- **Destructive**: the datadir is emptied and copy-back runs. **Other schemas on the same mysqld also revert to the snapshot point / disappear**. If you keep non-project DBs on the same instance, dump them before enabling
- If MySQL fails to start, recover by the logical reset after making it startable (see "6-1. Recovery on failure (logical reset)")
- Never put physical snapshots in git/S3 (gitignore the snapshot directory)
- The test DB connection settings must match the test's (usually from `.env`)

## 3. Setup (once per dev machine)

### 3-1. Percona XtraBackup

Ubuntu 22.04 example:

```bash
cd /tmp
curl -fsSL -O https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo apt-get install -y ./percona-release_latest.generic_all.deb
sudo percona-release enable-only tools release
sudo apt-get update
sudo apt-get install -y percona-xtrabackup-80
xtrabackup --version
```

### 3-2. MySQL privileges

Required: `BACKUP_ADMIN` / `PROCESS` / `RELOAD`, plus `SELECT ON performance_schema.*` (without it backup fails with 1142).

Check the host first. On MySQL 8, granting to a non-existent `user@host` fails with ERROR 1410.

```bash
sudo mysql -e "SELECT user, host FROM mysql.user WHERE user='<db_user>';"
```

Typical case with `localhost`:

```bash
sudo mysql -e "GRANT BACKUP_ADMIN, PROCESS, RELOAD ON *.* TO '<db_user>'@'localhost'; GRANT SELECT ON performance_schema.* TO '<db_user>'@'localhost'; FLUSH PRIVILEGES;"
```

If `@'%'` is missing for `DB_HOST=127.0.0.1` (TCP), CREATE first, then GRANT (the password is the real value from `.env`; never write placeholders):

```sql
CREATE USER IF NOT EXISTS '<db_user>'@'%' IDENTIFIED BY '<DB password from .env>';
GRANT BACKUP_ADMIN, PROCESS, RELOAD ON *.* TO '<db_user>'@'%';
GRANT SELECT ON performance_schema.* TO '<db_user>'@'%';
FLUSH PRIVILEGES;
```

```bash
sudo mysql -e "SHOW GRANTS FOR '<db_user>'@'localhost';"
```

### 3-3. Passwordless sudo (limited commands)

Replace `<login_user>` with your Linux login name (leaving `<login_user>` as is causes a visudo syntax error). If you change any `SNAPSHOT_MYSQL_*_CMD`, align the sudoers with **the actual command strings to be executed**.

```bash
sudo tee /etc/sudoers.d/snapshot-xtrabackup >/dev/null <<'EOF'
Cmnd_Alias SNAPSHOT_XBK = /usr/bin/systemctl stop mysql, /usr/bin/systemctl start mysql, /usr/bin/systemctl stop mysqld, /usr/bin/systemctl start mysqld, /usr/sbin/service mysql stop, /usr/sbin/service mysql start, /usr/bin/xtrabackup, /usr/bin/find, /usr/bin/chown, /bin/rm
<login_user> ALL=(root) NOPASSWD: SNAPSHOT_XBK
EOF
sudo chmod 440 /etc/sudoers.d/snapshot-xtrabackup
sudo visudo -cf /etc/sudoers.d/snapshot-xtrabackup
cat /etc/sudoers.d/snapshot-xtrabackup
```

Verify as the regular user (verifying inside a root shell is meaningless):

```bash
sudo -n systemctl stop mysql
sudo -n systemctl start mysql
sudo -n xtrabackup --version
```

Do not use `NOPASSWD: ALL`. If `systemctl` is unavailable on WSL, switch the stop/start in "3-4. Environment variables" to `service mysql`.

### 3-4. Environment variables (`.env` or shell)

```bash
SNAPSHOT_RESET_MODE=xtrabackup
SNAPSHOT_XTRABACKUP_BIN=sudo -n xtrabackup
SNAPSHOT_MYSQL_STOP_CMD=sudo -n systemctl stop mysql
SNAPSHOT_MYSQL_START_CMD=sudo -n systemctl start mysql
SNAPSHOT_MYSQL_CHOWN_CMD=sudo -n chown -R mysql:mysql /var/lib/mysql
SNAPSHOT_MYSQL_CLEAR_CMD=sudo -n find /var/lib/mysql -mindepth 1 -maxdepth 1 -exec rm -rf {} +
SNAPSHOT_XTRABACKUP_RM_CMD=sudo -n rm -rf {snap}
```

Optional: `SNAPSHOT_XTRABACKUP_DIR` (default `tests/db/xtrabackup-snap`), `SNAPSHOT_MYSQL_DATADIR` (default `/var/lib/mysql`). `{datadir}` works in `CLEAR_CMD`; `{snap}` in `RM_CMD` is replaced with the snapshot directory. When unset, plain `rm -rf` is used (CI-oriented). With local `sudo xtrabackup`, the snapshot files become mysql-owned, so re-prepare requires `RM_CMD`. The `/bin/rm` already in the sudoers Cmnd_Alias suffices (no additional NOPASSWD needed).

To return to the logical reset: `SNAPSHOT_RESET_MODE=sql` (a run-time value overrides `.env`).

### 3-5. Creating the golden snapshot

```bash
<test helper prepare command>
```

On success, `.snapshot_ready` and `data/xtrabackup_checkpoints` appear in the snapshot directory. Re-run after dump/seed changes.

### 3-6. Setup verification

```bash
SNAPSHOT_RESET_MODE=xtrabackup <test helper reset invocation>
```

It should return in seconds to tens of seconds, and MySQL should accept connections again.

| Item | Done |
|---|---|
| `xtrabackup --version` is 8.0-series | |
| `BACKUP_ADMIN` and `performance_schema` SELECT | |
| sudoers.d passes `visudo -cf` | |
| `sudo -n` works for stop/start/xtrabackup/clear | |
| `SNAPSHOT_*` in `.env` | |
| Pre-dump done if other DBs share the instance | |
| Golden snapshot creation succeeded | |
| **Immediately re-run** snapshot creation succeeds (verifies old-snapshot deletion = `RM_CMD`) | |
| reset (xtrabackup) succeeded | |

If old-snapshot deletion fails with `Permission denied`, suspect plain `rm -rf` or a missing `SNAPSHOT_XTRABACKUP_RM_CMD`.

## 4. Running tests

**Only when introduced**, full-suite runs use xtrabackup. After dump/seed/schema changes, retake the snapshot per "3-5. Creating the golden snapshot" before running. Never flee to the sql full-suite "because the snapshot is old" (see `skills/snapshot-test-debugging/SKILL.md`, the "Branching the DB reset mode (physical snapshot available / not)" section).

```bash
<test helper prepare command>
SNAPSHOT_RESET_MODE=xtrabackup <test run command>
```

**Machines without it** run the conventional logical load (`SNAPSHOT_RESET_MODE` unset or `sql`). No prepare needed.

```bash
SNAPSHOT_RESET_MODE=sql <test run command>
```

## 5. Processing flow

### prepare

1. Logical reset (dump + seeds)
2. `xtrabackup --backup`
3. `xtrabackup --prepare`
4. Write `.snapshot_ready` outside `data/`

### reset (`SNAPSHOT_RESET_MODE=xtrabackup`)

1. Check marker and checkpoints
2. stop → clear datadir (`SNAPSHOT_MYSQL_CLEAR_CMD`) → copy-back → chown → start → wait for ready
3. No seed re-application (included in the snapshot)

## 6. Troubleshooting

| Symptom | Check |
|---|---|
| `.snapshot_ready` missing | Run "3-5. Creating the golden snapshot" first |
| sudo asks for a password | "3-3. Passwordless sudo". Verify `sudo -n` as the regular user; CLEAR/STOP commands match sudoers |
| `BACKUP_ADMIN` / LOCK denial | "3-2. MySQL privileges" |
| `performance_schema` SELECT 1142 | `GRANT SELECT ON performance_schema.*` in "3-2. MySQL privileges" |
| ERROR 1410 | GRANT to a non-existent `user@host`. Check `mysql.user` first |
| copy-back / startup failure | Make MySQL startable, then "6-1. Recovery on failure (logical reset)" |
| Seed changes not reflected | Re-run "3-5. Creating the golden snapshot" |
| Other schemas disappeared | "2. Prerequisites and constraints". Restore from the pre-dump |

### 6-1. Recovery on failure (logical reset)

This feature is an optional destructive local operation, so the correct recovery path is **returning to the logical reset route**.

1. Confirm MySQL can start (if not, fix the datadir at the OS/package level or reinstall mysqld)
2. Logical reset:

```bash
SNAPSHOT_RESET_MODE=sql <test helper reset invocation>
```

3. If necessary, drop XtraBackup and remove `SNAPSHOT_RESET_MODE` from `.env` or set it to `sql`

## 7. Out of scope / future

- Applying the same settings to production/verification apps, personal chown of the datadir, unrestricted NOPASSWD in sudoers, physical snapshots in git/S3
- Making the dev-CI physical snapshot a permanent gate (separate approval)
- Reducing sudo via passwordless systemd or datadir ACLs is a separate task
