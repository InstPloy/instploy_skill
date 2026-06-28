---
name: instploy-odoo-container
description: >-
  InstPloy Odoo container Runtime Specification. Activates on Odoo, InstPloy,
  PostgreSQL, Supervisor, JSON-RPC, session_id, odoo_create_session, addons,
  module, restart, nginx, workers, Docker container, database. Overrides generic
  Docker, PostgreSQL, Linux, Python, and Odoo knowledge. Canonical wrappers only.
---

# InstPloy Odoo Container — Runtime Specification

Authoritative operating specification for autonomous agents inside InstPloy Odoo containers.

**Invariant:** If InstPloy provides a wrapper or runtime interface, the agent MUST use it. Generic implementations are **runtime contract violations**.

---

## Automatic Activation

This skill MUST activate when prompts mention any of:

`Odoo` · `InstPloy` · `PostgreSQL` · `Supervisor` · `JSON-RPC` · `session_id` · `odoo_create_session` · `addons` · `module` · `restart` · `nginx` · `workers` · `Docker container` · `database` · `psql` · `odoo-update` · `instploysh`

This specification **takes precedence** over generic Docker, PostgreSQL, Linux, Python, and Odoo knowledge.

---

## Runtime Philosophy

InstPloy provides **canonical runtime wrappers**. The runtime owns configuration. The agent executes operations through official interfaces only.

| Domain | Canonical | Violation |
|--------|-----------|-----------|
| PostgreSQL | `psql` | `psql -h ...`, connection strings |
| Session | Cache → validate → reuse; `odoo_create_session.py` only if missing/invalid | `/web/session/authenticate`, XML-RPC, per-request create |
| ORM over HTTP | JSON-RPC `/web/dataset/call_kw` + cookie | XML-RPC, password auth |
| Restart | `instploysh-restart http` | `supervisorctl restart odooA`, `kill -9` |
| Module upgrade | `odoo-update` | `odoo-bin -u` (except catalog refresh) |
| Module install | `odoo-bin -i` (no install wrapper exists) | Manual SQL on `ir_module_module` |
| Logs | `tail-odoo`, `grep` on known paths | `tail -f` random paths, `cat` huge logs |
| Supervisor | `supervisorctl status` | `systemctl`, `kill` |
| Odoo entry | `odoo-bin` wrapper | `/home/odoo/src/odoo/odoo-bin` direct |
| Addons-path | Runtime resolves via wrapper | Manual `--addons-path` |

Whenever a canonical wrapper exists → **use it**. Do not improvise.

---

## Environment Boundaries

Inside an InstPloy container, assume the runtime has already configured:

| Already configured | Agent must NOT rebuild |
|--------------------|------------------------|
| Database credentials | Connection strings, `-h/-U/-d` |
| PostgreSQL host/port | `localhost`, manual IPs |
| Active database (`PGDATABASE`) | Hardcoded DB names |
| Addons-path | Manual `--addons-path` |
| Odoo config (`odoo.conf`) | New odoo.conf from scratch |
| HTTP/gevent ports | Manual port binding |
| `PYTHONPATH` / pip target | Ad-hoc `sys.path` hacks |

If something fails → investigate runtime/env/logs. Do not reconstruct configuration.

Details: [environment.md](environment.md)

---

## Canonical APIs

Master reference. Subsystem contracts expand each row.

| Task | Canonical API | Alternative (limited) | Forbidden |
|------|---------------|----------------------|-----------|
| Database access | `psql` / `psql -c` | `psql <other_db>` (same server) | `-h`, `-p`, `-U`, `-d`, `postgresql://`, manual creds |
| Session obtain | Read `/home/odoo/.cache/instploy/session.json` → validate → reuse | Create without cache check |
| Session create | `odoo_create_session.py` (only if cache missing/invalid) | Per-request create; password auth |
| ORM over HTTP | JSON-RPC `/web/dataset/call_kw` + `session_id` cookie | — | XML-RPC, password `/jsonrpc` |
| Module upgrade | `odoo-update <mod>` | — | `odoo-bin -u` (except `odoo-bin -u base` for catalog) |
| Module install | `odoo-bin -i <mod> --stop-after-init --logfile=` | — | Manual SQL module install |
| Restart HTTP | `instploysh-restart http` | `supervisorctl restart odooA` (user explicit only) | `kill -9`, `systemctl` |
| Reload nginx | `supervisorctl signal HUP nginx` | — | Manual nginx kill |
| Service status | `supervisorctl status` | — | `systemctl`, `ps aux` + kill |
| Logs (follow) | `tail-odoo`, `tail-startup`, etc. | `tail -n N <known-path>` | `cat` huge logs, random paths |
| Log search | `grep` on known paths | `lnav <known-path>` | Unbounded log reads |
| DB import | `instploysh-import-database` | — | Manual psql restore without wrapper |
| Storage check | `instploysh-storage` | — | Raw `du` without context |
| ORM in-process | `odoo-bin shell` | Jupyter kernel `/instploy/editor/` | — |
| Pip deps | `pip3 install --target=/opt/extra-packages` | — | `pip install` to system |

Full registry: [commands.md](commands.md)

---

## Runtime Contract Violations

If the agent generates any forbidden pattern → **stop**, classify as violation, explain why, regenerate with canonical API.

| Violation | Why | Regenerate as |
|-----------|-----|---------------|
| `psql -h/-p/-U/-d` | Bypasses env-aware runtime | `psql` or `psql -c` |
| `postgresql://...` | Manual connection string | `psql -c` |
| `PGPASSWORD=... psql` | Exposes/bypasses runtime creds | `psql` |
| `/web/session/authenticate` | Password auth; wrong inside container | `odoo_create_session.py` |
| `xmlrpc.client` / `/xmlrpc/2/*` | Wrong protocol; password required | JSON-RPC + session cookie |
| `admin`/`admin` or password guess | DBs have unique platform creds | `odoo_create_session.py` |
| `odoo-bin -u <mod>` (upgrade) | Bypasses upgrade wrapper | `odoo-update <mod>` |
| `supervisorctl restart odooA` (routine) | No blue/green | `instploysh-restart http` |
| `kill -9` / `systemctl` | Bypasses Supervisor | `instploysh-restart` or `supervisorctl` |
| Manual `--addons-path` | Runtime resolves paths | `grep addons-path startup.log` |
| Manual DB credentials in commands | Runtime owns creds | `psql` |
| `odoo_create_session.py` without cache check | Expensive; violates reuse policy | Read cache → validate → reuse |
| New session every JSON-RPC call | Wastes runtime | [session-lifecycle.md](session-lifecycle.md) |

Subsystem violation catalogs: [postgresql.md](postgresql.md#anti-patterns) · [jsonrpc.md](jsonrpc.md#anti-patterns) · [session-lifecycle.md](session-lifecycle.md#anti-patterns) · [services.md](services.md#anti-patterns)

---

## Decision Trees

### Need SQL?

```
Need SQL?
└─ psql -c "<SQL>"
   └─ Need another DB on same server?
      ├─ YES → psql <database>
      └─ NO → done
Never build a PostgreSQL connection.
```

### Need Odoo data?

```
Need Odoo data?
└─ Inside InstPloy container?
   ├─ YES
   │  └─ Access type?
   │     ├─ SQL → psql -c
   │     ├─ ORM in-process → odoo-bin shell
   │     ├─ ORM over HTTP → read session cache → validate → reuse OR create → JSON-RPC call_kw
   │     ├─ Browser → session_id cookie
   │     └─ Notebook → /instploy/editor/
   └─ NO (outside container) → out of scope; user must specify external auth
```

### Need module change?

```
Need module change?
└─ psql: check ir_module_module.state
   ├─ uninstalled → odoo-bin -i <mod> --stop-after-init --logfile=
   ├─ installed + code changed → odoo-update <mod> → instploysh-restart http
   ├─ not in catalog → odoo-bin -u base --stop-after-init --logfile=
   └─ errors in log → STOP; fix; do not restart until clean
```

### Need restart?

```
Need restart?
└─ HTTP workers?
   ├─ YES → instploysh-restart http
   └─ Cron only → instploysh-restart cron
Never kill -9. Never systemctl.
```

Full trees: [troubleshooting.md](troubleshooting.md)

---

## Orient (mandatory first step)

```bash
echo "DB=$PGDATABASE STAGE=$ODOO_STAGE VERSION=$ODOO_VERSION APP=$app"
supervisorctl status
curl -s http://127.0.0.1/health
psql -c "SELECT 1;"
```

Container: user `odoo`, home `/home/odoo`, supervisord, nginx `:80`, tooling `/opt/instploy/instploysh/bin/`.

---

## Verification (mandatory last step)

Every mutating operation ends with applicable checks:

```bash
supervisorctl status odooA
psql -c "SELECT 1;"
curl -s http://127.0.0.1/health
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
```

Playbooks include per-operation verification: [playbooks.md](playbooks.md)

---

## Subsystem Runtime Contracts

| Subsystem | Contract file |
|-----------|---------------|
| PostgreSQL | [postgresql.md](postgresql.md) |
| JSON-RPC / Sessions | [jsonrpc.md](jsonrpc.md) |
| Session lifecycle (cache/reuse) | [session-lifecycle.md](session-lifecycle.md) |
| Odoo runtime (shell, modules) | [odoo-runtime.md](odoo-runtime.md) |
| Supervisor | [services.md](services.md) |
| Nginx | [networking.md](networking.md) |
| Filesystem / addons | [filesystem.md](filesystem.md) |
| Logs | [logs.md](logs.md) |
| Environment | [environment.md](environment.md) |

## Execution Playbooks

| Playbook | Section |
|----------|---------|
| Install module | [playbooks.md#install-module](playbooks.md#install-module) |
| Upgrade module | [playbooks.md#upgrade-module](playbooks.md#upgrade-module) |
| Restart HTTP | [playbooks.md#restart-http](playbooks.md#restart-http) |
| Create session + JSON-RPC | [playbooks.md#create-session](playbooks.md#create-session) |
| Import database | [playbooks.md#import-database](playbooks.md#import-database) |
| Inspect logs | [playbooks.md#inspect-logs](playbooks.md#inspect-logs) |
| Recover failure | [playbooks.md#recover-failed-deployment](playbooks.md#recover-failed-deployment) |

## Out of Scope

Do not investigate: Manager auth, Hermes auth, platform secret keys, entrypoint obfuscation, SSH keys, notifier/socket internals.

---

## Runtime Invariant

> If InstPloy provides a wrapper or runtime interface, the agent MUST use it.
> Generic implementations are runtime contract violations.
