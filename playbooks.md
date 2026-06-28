# Execution Playbooks

Deterministic sequences. Every playbook: **Commands → Verification → Expected Result → Failure Recovery → Rollback**.

Violations: [SKILL.md](SKILL.md#runtime-contract-violations). Canonical APIs: [SKILL.md](SKILL.md#canonical-apis).

---

## Orient

**Commands:**

```bash
echo "DB=$PGDATABASE STAGE=$ODOO_STAGE VERSION=$ODOO_VERSION APP=$app"
supervisorctl status
curl -s http://127.0.0.1/health
psql -c "SELECT 1;"
```

**Verification:** odooA RUNNING, health=`healthy`, psql exit 0.

**Expected:** Container ready for operations.

**Failure recovery:** [troubleshooting.md](troubleshooting.md)

---

## Install Module

**Goal:** Install `<module>`.

**Prerequisites:** Orient. Module in `/home/odoo/src/user` with `__manifest__.py`.

**Commands:**

```bash
psql -c "SELECT name, state FROM ir_module_module WHERE name='<module>';"
# if not in catalog:
odoo-bin -u base --stop-after-init --logfile=
odoo-bin -i <module> --stop-after-init --logfile=/home/odoo/logs/install.log
grep -iE "error|critical|traceback" /home/odoo/logs/install.log | tail -20 || true
```

**Verification:**

```bash
psql -c "SELECT name, state FROM ir_module_module WHERE name='<module>';"  # installed
instploysh-restart http
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
curl -s http://127.0.0.1/health
```

**Expected:** state=`installed`, HTTP 200/303, no critical in log.

**Failure recovery:** Read traceback → fix code/XML/deps → re-run. [troubleshooting.md#module-installupgrade-fails](troubleshooting.md)

**Rollback:** Uninstall via `odoo-bin shell` or DB restore.

---

## Upgrade Module

**Goal:** Upgrade `<module>` after code deploy.

**Commands:**

```bash
psql -c "SELECT name, state FROM ir_module_module WHERE name='<module>';"
odoo-update <module>
grep -iE "error|critical|traceback" /home/odoo/logs/odoo.log | tail -20 || true
instploysh-restart http
```

**Verification:** Same as install.

**Expected:** `latest_version` updated, HTTP healthy.

**Failure recovery:** STOP on traceback; do not restart until fixed.

**Rollback:** Revert code + `odoo-update` or DB restore.

---

## Restart HTTP

**Goal:** Zero-downtime Odoo HTTP restart.

**Commands:**

```bash
instploysh-restart http
```

**Verification:**

```bash
supervisorctl status odooA
supervisorctl status odooB    # STOPPED after completion
curl -s http://127.0.0.1/health
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
psql -c "SELECT 1;"
```

**Expected:** `Blue/Green restart completed`, HTTP 200/303.

**Failure recovery:** [troubleshooting.md#odoo-wont-start](troubleshooting.md). If nginx swap failed, config auto-rolled back.

**Rollback:** `instploysh-restart http` retry after fixing root cause.

---

## Create Session

**Goal:** Resolve reusable session (cache-first) + JSON-RPC ORM access.

**Rules:** [session-lifecycle.md](session-lifecycle.md) · [jsonrpc.md](jsonrpc.md). Never password auth. Never XML-RPC. Never create session without cache check.

**User:** default uid 2 (admin). No PostgreSQL lookup unless a specific non-admin user is required.

**Commands:**

```bash
CACHE=/home/odoo/.cache/instploy/session.json

# 1. Read + validate cache
if [ -f "$CACHE" ]; then
  SID=$(python3 -c "import json; print(json.load(open('$CACHE'))['session_id'])")
  CODE=$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/web/dataset/call_kw \
    -H "Content-Type: application/json" -b "session_id=$SID" \
    -d '{"jsonrpc":"2.0","method":"call","params":{"model":"res.users","method":"search_count","args":[[]],"kwargs":{}}}')
  [ "$CODE" != "200" ] && rm -f "$CACHE"
fi

# 2. Create only if cache missing/invalid (default uid 2; atomic create + persist)
if [ ! -f "$CACHE" ]; then
  mkdir -p /home/odoo/.cache/instploy
  python3 /opt/instploy/instploysh/lib/odoo_create_session.py 2 2>&1 | grep -o '{.*}' \
    | python3 -c "import sys,json,datetime,os; d=json.load(sys.stdin); assert 'session_id' in d, d.get('error'); now=datetime.datetime.utcnow().isoformat()+'Z'; d['created_at']=d['validated_at']=now; tmp='$CACHE.tmp'; json.dump(d,open(tmp,'w')); os.replace(tmp,'$CACHE')"
  test -s "$CACHE" && echo "cache OK" || echo "CACHE MISSING — contract violation"
  SID=$(python3 -c "import json; print(json.load(open('$CACHE'))['session_id'])")
fi

# 3. ORM call
curl -s http://127.0.0.1/web/dataset/call_kw \
  -H "Content-Type: application/json" -b "session_id=$SID" \
  -d '{"jsonrpc":"2.0","method":"call","params":{"model":"res.users","method":"search_count","args":[[]],"kwargs":{}}}'
```

**Verification:** Cache exists (`test -s`) or reused; validation HTTP 200; `search_count` returns integer.

**Expected:** Normal path: no `odoo_create_session.py` call. Exceptional path: one create, then reuse.

**Failure recovery:** Delete cache → create once → validate. Second failure → [session-lifecycle.md#failure-recovery](session-lifecycle.md). Never loop.

**Rollback:** `rm -f /home/odoo/.cache/instploy/session.json`

---

## Import Database

**Goal:** Restore from backup. **Requires explicit user confirmation.**

**Commands:**

```bash
instploysh-import-database --logfile=/home/odoo/logs/import.log /path/to/backup.zip
instploysh-restart http
```

**Verification:**

```bash
psql -c "SELECT latest_version FROM ir_module_module WHERE name='base';"
curl -s http://127.0.0.1/health
```

**Expected:** `Finished.` exit 0.

**Failure recovery:** Check import log. Invalid format → correct backup type.

**Rollback:** Another backup restore only.

---

## Inspect Logs

**Goal:** Find root cause of failure.

**Commands:** Use investigation priority from [logs.md](logs.md#investigation-priority).

```bash
grep -iE "error|critical|traceback" <relevant-log> | tail -40
```

**Verification:** Actionable error identified.

**Expected:** Specific module/file/line in traceback.

**Failure recovery:** Apply subsystem playbook after fix.

---

## Recover Failed Deployment

**Decision:**

```
Install/upgrade failed?
├─ YES → read traceback → fix → re-run odoo-update / odoo-bin -i
└─ Restart failed?
         ├─ supervisorctl status
         ├─ logs per [logs.md](logs.md)
         └─ instploysh-restart http after fix
```

**Verification:** Full orient checklist passes.

**Rollback:** DB restore if data corrupted.
