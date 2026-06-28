# Troubleshooting

Symptom-based decision trees. Logs: [logs.md](logs.md). Commands: [commands.md](commands.md).

## Odoo won't start

**Symptoms:** `odooA` FATAL/EXITED in supervisor; HTTP non-200 on :8069.

**Decision tree:**

1. `supervisorctl status odooA`
2. `tail -n 50 /home/odoo/logs/supervisor/odoo-error.log`
3. `tail -n 50 /home/odoo/logs/odoo-bin-wrapper.log`
4. `tail -n 50 /home/odoo/logs/odoo.log`
5. `grep -E "addons-path|Database|ERROR" /home/odoo/logs/startup.log`

**Likely causes:**

| Cause | Indicator | Fix |
|-------|-----------|-----|
| DB unreachable | `psycopg2.OperationalError` in startup.log | Verify `PGHOST`, credentials; `psql -c "SELECT 1;"` |
| Bad addons-path | `Module not found` | Check `startup.log` Final addons-path; verify `__manifest__.py` |
| Port conflict | `Address already in use` | `ss -tlnp \| grep 8069`; stop conflicting process |
| Registry failure | `Failed to load registry` | Broken module; uninstall or fix code |
| Missing binary | `Odoo binary not found` | Verify `/home/odoo/src/odoo/odoo-bin` exists |

**Verify fix:** `supervisorctl restart odooA` or `instploysh-restart http` → HTTP 200 on `/web/login`.

---

## HTTP 502

**Symptoms:** Nginx returns 502; `/health` may still work.

**Decision tree:**

1. `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login`
2. If non-200 → [Odoo won't start](#odoo-wont-start)
3. If 200 → `tail -n 50 /home/odoo/logs/supervisor/nginx-error.log`
4. Check upstream in `/etc/nginx/conf.d/default.conf`

**Likely causes:**

| Cause | Fix |
|-------|-----|
| odooA down | Restart Odoo |
| Wrong upstream after failed swap | Verify ports 8069/8072 in nginx config; `supervisorctl signal HUP nginx` |
| odooB left running incorrectly | `supervisorctl stop odooB`; fix nginx upstream |

---

## Workers crashed

**Symptoms:** Intermittent 500s, worker timeout, memory errors.

**Decision tree:**

1. `grep -i "Worker\|Memory\|Killed" /home/odoo/logs/odoo.log | tail -30`
2. Check `WORKERS`, `LIMIT_MEMORY_*` in [environment.md](environment.md)
3. `psql -c "SELECT count(*) FROM pg_stat_activity WHERE datname='$PGDATABASE';"`

**Likely causes:**

| Cause | Fix |
|-------|-----|
| Memory limit hit | Reduce workers or increase limits (platform config) |
| Long-running request | Identify module; optimize code |
| DB connection exhaustion | Check `DB_MAXCONN`; kill idle connections |

---

## Module missing

**Symptoms:** `Module <name> not found` during install/upgrade.

**Decision tree:**

1. `find /home/odoo/src/user -name __manifest__.py | grep <module>`
2. `grep "Final addons-path" /home/odoo/logs/startup.log`
3. `psql -c "SELECT name, state FROM ir_module_module WHERE name='<module>';"`

**Likely causes:**

| Cause | Fix |
|-------|-----|
| Code not deployed | Place module in `/home/odoo/src/user` |
| Path not in addons-path | Fix directory structure; restart Odoo |
| Not in catalog | `odoo-bin -u base --stop-after-init --logfile=` |

---

## Registry failed

**Symptoms:** `CRITICAL ... Failed to load registry` in odoo.log.

**Decision tree:**

1. Read full traceback in `/home/odoo/logs/odoo.log`
2. Identify failing module from traceback
3. `psql -c "SELECT name, state FROM ir_module_module WHERE state IN ('to upgrade','to install');"`

**Likely causes:**

| Cause | Fix |
|-------|-----|
| Bad migration | Fix migration script; re-upgrade |
| Corrupt view XML | Fix XML; upgrade module |
| Python import error | Install dep to `/opt/extra-packages` |

**Recovery:** Fix code → `odoo-update <module>`. If corrupted: DB restore.

---

## Database unavailable

**Symptoms:** `psql` fails; Odoo cannot connect.

**Decision tree:**

1. `echo "HOST=$PGHOST DB=$PGDATABASE USER=$POSTGRES_USER"`
2. `psql -c "SELECT 1;"`
3. `grep -i "database\|psycopg2" /home/odoo/logs/startup.log | tail -20`

**Likely causes:**

| Cause | Fix |
|-------|-----|
| Wrong PGHOST | Use env value (often external IP, not localhost) |
| Credentials expired/changed | Platform must update env |
| DB server down | Platform/infrastructure issue |
| Network partition | Check connectivity to PGHOST:PGPORT |

---

## Sessions fail

**Symptoms:** `odoo_create_session.py` returns error JSON; HTTP 401 with session cookie.

**Decision tree:**

1. `psql -c "SELECT id, login, active FROM res_users WHERE id=<uid>;"`
2. `tail -n 30 /home/odoo/logs/manager.log`
3. `psql -c "SELECT 1;"`

**Likely causes:**

| Cause | Fix |
|-------|-----|
| Invalid user id | Use active user from `res_users` |
| DB unreachable | Fix DB connection |
| Odoo registry not loaded | Ensure Odoo running |

---

## Module install/upgrade fails

**Symptoms:** Non-zero exit; errors in install/update log.

**Decision tree:**

1. `grep -iE "error|critical|traceback" /home/odoo/logs/odoo.log | tail -40`
2. Match pattern to [logs.md#error-signatures](logs.md#error-signatures)
3. Fix root cause
4. Re-run operation

Playbook: [playbooks.md](playbooks.md#install-module) or [upgrade-module](playbooks.md#upgrade-module)

---

## Performance issue

**Symptoms:** Slow UI, timeouts, high CPU.

**Decision tree:**

1. `grep -i "slow\|timeout\|WARNING" /home/odoo/logs/odoo.log | tail -30`
2. `psql -c "SELECT pid, state, query_start, left(query,80) FROM pg_stat_activity WHERE datname='$PGDATABASE' AND state='active';"`
3. `instploysh-storage` if disk suspected

**Likely causes:**

| Cause | Fix |
|-------|-----|
| Missing DB index | Add index via module |
| Long cron job | Check cron logs in odoo.log |
| Full disk | Clean backups/logs; expand storage |

---

## Disk full

**Symptoms:** Write failures, backup errors, ncdu shows high usage.

**Decision tree:**

1. `instploysh-storage`
2. `du -sh /home/odoo/data/filestore/$PGDATABASE`
3. `du -sh /home/odoo/logs/ /home/odoo/backups/`
4. `ls -lhS /home/odoo/logs/ | head -10`

**Likely causes:**

| Cause | Fix |
|-------|-----|
| Large filestore | Platform storage management |
| Log rotation needed | Truncate old logs (caution) |
| Too many backups | Adjust `max_backup` in task config |

---

## Backup issues

**Symptoms:** Errors in `/home/odoo/logs/backup.log`.

**Decision tree:**

1. `tail -n 50 /home/odoo/logs/backup.log`
2. `ls -la /home/odoo/backups/`
3. Check task volume schedule config

---

## Addons not loading

**Symptoms:** Custom module invisible; wrong addons-path.

**Decision tree:**

1. `grep "Final addons-path" /home/odoo/logs/startup.log`
2. `find /home/odoo/src/user -name __manifest__.py`
3. Verify `ENTERPRISE` env if enterprise module expected

See [filesystem.md](filesystem.md#addons-path-resolution).

---

## Quick symptom index

| Symptom | Section |
|---------|---------|
| Odoo won't start | [above](#odoo-wont-start) |
| HTTP 502 | [above](#http-502) |
| Workers crashed | [above](#workers-crashed) |
| Module missing | [above](#module-missing) |
| Registry failed | [above](#registry-failed) |
| Database unavailable | [above](#database-unavailable) |
| Sessions fail | [above](#sessions-fail) |
| Install/upgrade fails | [above](#module-installupgrade-fails) |
| Performance issue | [above](#performance-issue) |
| Disk full | [above](#disk-full) |
| Backup issues | [above](#backup-issues) |
| Addons not loading | [above](#addons-not-loading) |
