# Execution Playbooks

Deterministic sequences. Every playbook ends with verification. Commands defined in [commands.md](commands.md).

## Orient

**Goal:** Establish container state before any operation.

**Prerequisites:** Shell access as `odoo`.

```bash
echo "DB=$PGDATABASE STAGE=$ODOO_STAGE VERSION=$ODOO_VERSION APP=$app"
echo "HOST=$PGHOST PORT=${PGPORT:-${PGDBPORT:-5432}} USER=$POSTGRES_USER"
supervisorctl status
curl -s http://127.0.0.1/health
psql -c "SELECT 1 AS ok;"
```

**Verification:** All services RUNNING (or odooB STOPPED is normal), health=`healthy`, psql returns `ok`.

---

## Install module

**Goal:** Install Odoo module `<module>` into current database.

**Prerequisites:** Orient complete. Module exists in addons-path. State=`uninstalled` or absent from catalog.

**Commands:**

```bash
# If not in catalog:
odoo-bin -u base --stop-after-init --logfile=

# Install:
odoo-bin -i <module> --stop-after-init --logfile=/home/odoo/logs/install.log
```

**Monitor errors:**

```bash
grep -iE "error|critical|traceback|failed" /home/odoo/logs/install.log | tail -20
```

**Verification:**

```bash
psql -c "SELECT name, state FROM ir_module_module WHERE name='<module>';"   # state=installed
instploysh-restart http
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
curl -s http://127.0.0.1/health
```

**Rollback:** Uninstall via `odoo-bin shell`: `env['ir.module.module'].search([('name','=','<module>')]).button_immediate_uninstall()` then `env.cr.commit()`. Or restore from backup.

**Common failures:**

| Failure | Fix |
|---------|-----|
| Module not found | Check addons-path in `startup.log`; verify `__manifest__.py` in `/home/odoo/src/user` |
| ParseError | Fix XML in module; re-run install |
| ImportError | `pip3 install --target=/opt/extra-packages <dep>` |
| Registry failed | Read full traceback; fix migration/SQL |

**Expected output:** Exit 0. No critical/traceback in log. State=`installed`.

---

## Upgrade module

**Goal:** Upgrade module `<module>` after code change.

**Prerequisites:** Orient complete. Code deployed to `/home/odoo/src/user`. State=`installed`.

**Commands:**

```bash
odoo-update <module>
# or with log capture:
odoo-bin -u <module> --stop-after-init --logfile=/home/odoo/logs/update.log
```

**Monitor errors:**

```bash
grep -iE "error|critical|traceback|failed" /home/odoo/logs/update.log | tail -20
# live alternative:
tail -n 0 -f /home/odoo/logs/odoo.log | grep -iE "error|critical|traceback"
```

**Verification:**

```bash
psql -c "SELECT name, state, latest_version FROM ir_module_module WHERE name='<module>';"
instploysh-restart http
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
curl -s http://127.0.0.1/health
```

**Rollback:** Revert code in `/home/odoo/src/user`; `odoo-update <module>` with previous version. Or DB restore.

**Common failures:** Same as install-module. Additionally: migration script errors in upgrade hooks.

**Expected output:** Exit 0. `latest_version` updated. HTTP 200/303.

---

## Restart HTTP

**Goal:** Zero-downtime Odoo HTTP restart (blue/green).

**Prerequisites:** Orient complete. No active install/upgrade on odooB.

**Commands:**

```bash
instploysh-restart http
```

**Verification:**

```bash
supervisorctl status odooA    # RUNNING
supervisorctl status odooB    # STOPPED (normal after completion)
curl -s http://127.0.0.1/health
curl -s -o /dev/null -w "odooA: %{http_code}\n" http://127.0.0.1:8069/web/login
psql -c "SELECT 1;"
```

**Rollback:** If restart fails mid-swap, check nginx upstream in `/etc/nginx/conf.d/default.conf`. `supervisorctl signal HUP nginx` after manual fix.

**Common failures:**

| Failure | Fix |
|---------|-----|
| odooB timeout | Check `odoo-error.log`; fix startup issue; `supervisorctl stop odooB` |
| nginx switch failed | Config rolled back automatically; check `nginx-error.log` |
| odooA won't start | [troubleshooting.md](troubleshooting.md#odoo-wont-start) |

**Expected output:** `Blue/Green restart completed successfully in Xs`

---

## Import database

**Goal:** Replace current database and filestore from backup.

**Prerequisites:** Orient complete. **Explicit user confirmation required.**

**Commands:**

```bash
instploysh-import-database --logfile=/home/odoo/logs/import.log /path/to/backup.zip
instploysh-restart http
```

**Verification:**

```bash
psql -c "SELECT latest_version FROM ir_module_module WHERE name='base';"
ls /home/odoo/data/filestore/$PGDATABASE | head -5
curl -s http://127.0.0.1/health
```

**Rollback:** None in-place. Requires another backup restore.

**Common failures:**

| Failure | Fix |
|---------|-----|
| InvalidDumpError | Use Odoo manager `.zip` or instploy plain format |
| Custom format dump | Convert to plain SQL dump |
| psql restore errors | Check filtered warnings in import log |

**Expected output:** `Finished.` exit 0.

---

## Create session

**Goal:** Obtain authenticated Odoo HTTP session for user `<uid>`.

**Prerequisites:** Orient complete. User exists and is active.

**Commands:**

```bash
psql -c "SELECT id, login FROM res_users WHERE active ORDER BY id LIMIT 10;"
python3 /opt/instploy/instploysh/lib/odoo_create_session.py <uid> 2>&1 | grep -o '{.*}'
```

**Verification:**

```bash
SESSION=$(python3 /opt/instploy/instploysh/lib/odoo_create_session.py <uid> 2>&1 | grep -o '{.*}')
echo "$SESSION" | python3 -c "import sys,json; d=json.load(sys.stdin); assert 'session_id' in d"
SID=$(echo "$SESSION" | python3 -c "import sys,json; print(json.load(sys.stdin)['session_id'])")
curl -s -b "session_id=$SID" http://127.0.0.1/web/session/get_session_info | python3 -m json.tool
```

**Rollback:** N/A (stateless session creation).

**Common failures:**

| Failure | Fix |
|---------|-----|
| `User with ID N not found` | Verify uid in `res_users` |
| DB connection error | Check env vars + `psql -c "SELECT 1;"` |
| Empty JSON | Check `manager.log` for exception |

**Expected output:** `{"session_id":"...","login":"...","user_id":N,"db":"..."}`

**Security:** Never log `session_id` in chat output.

---

## Inspect logs

**Goal:** Find errors related to a failed operation.

**Prerequisites:** Know symptom or recent operation type.

**Commands:**

```bash
# By scenario — see logs.md chain
tail -n 50 /home/odoo/logs/supervisor/odoo-error.log
tail -n 50 /home/odoo/logs/odoo-bin-wrapper.log
grep -iE "error|critical|traceback" /home/odoo/logs/odoo.log | tail -40
grep "addons-path\|Database" /home/odoo/logs/startup.log | tail -20
```

**Verification:** Root cause identified in traceback or error line.

**Expected output:** Actionable error message with file/module reference.

---

## Recover failed deployment

**Goal:** Restore service after failed module deploy or restart.

**Prerequisites:** Orient complete. Failure already observed.

**Decision:**

```
Install/upgrade failed?
├─ YES → Read traceback in odoo.log or install.log
│        ├─ Fix code/XML/deps
│        ├─ Re-run odoo-update <module>
│        └─ If unrecoverable: instploysh-import-database (with user approval)
└─ Restart failed?
         ├─ supervisorctl status
         ├─ Read odoo-error.log → odoo-bin-wrapper.log
         ├─ Fix root cause
         └─ instploysh-restart http
```

**Commands:**

```bash
supervisorctl status
tail -n 50 /home/odoo/logs/supervisor/odoo-error.log
grep -iE "error|critical|traceback" /home/odoo/logs/odoo.log | tail -30
# after fix:
instploysh-restart http
```

**Verification:** Full orient verification passes.

**Rollback:** DB restore if module migration corrupted data.

---

## Full deploy loop

Checklist for code deploy → upgrade → restart:

```
[ ] Orient
[ ] Code in /home/odoo/src/user with __manifest__.py
[ ] psql: module state=installed
[ ] odoo-update <module>
[ ] grep errors in log
[ ] instploysh-restart http
[ ] curl /health + /web/login
```
