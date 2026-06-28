---
name: instploy-odoo-container
description: >-
  Operate inside instploy Odoo Docker containers: AI behavioral rules, decision
  trees, execution playbooks, filesystem, logs, PostgreSQL, Odoo runtime, Supervisor,
  Nginx, and instploy wrappers. Use when debugging Odoo in instploy containers,
  installing/upgrading modules, reading logs, obtaining credentials/sessions, or
  restarting services.
---

# instploy Odoo Container

Operational knowledge base for autonomous agents inside instploy Odoo Docker containers.

## Purpose

Enable agents to orient, plan, execute, verify, and recover container operations with minimal reasoning cost.

## Scope

| In scope | Out of scope |
|----------|--------------|
| Filesystem, env vars, logs, ports, routes | Manager API auth internals |
| instploy wrappers (`odoo-bin`, `odoo-update`, `instploysh-*`) | Hermes auth internals |
| PostgreSQL via env credentials | Platform secret keys / branch tokens |
| Odoo shell, sessions, module install/upgrade | Entrypoint obfuscation |
| Supervisor, Nginx, blue/green restart | SSH key management |
| Symptom-based troubleshooting | Reverse-engineering platform callbacks |

## Quick Start

Run before any operation:

```bash
echo "DB=$PGDATABASE STAGE=$ODOO_STAGE VERSION=$ODOO_VERSION APP=$app"
echo "HOST=$PGHOST PORT=${PGPORT:-${PGDBPORT:-5432}} USER=$POSTGRES_USER"
supervisorctl status
curl -s http://127.0.0.1/health
psql -c "SELECT 1 AS ok;"
```

Container identity: user `odoo`, home `/home/odoo`, process manager `supervisord`, HTTP proxy `nginx:80`, tooling `/opt/instploy/instploysh/bin/`.

## AI Operational Rules

Highest priority. Apply before every action.

### Always

- Read [environment.md](environment.md) env vars; never hardcode credentials.
- Prefer instploy wrappers over raw binaries (`odoo-bin` wrapper, not `/home/odoo/src/odoo/odoo-bin` directly).
- Prefer `instploysh-restart http` over `supervisorctl restart odooA` unless user explicitly requests direct restart.
- Verify after every mutating operation (see [Verification](#verification)).
- Use `curl -s http://127.0.0.1/health` after any HTTP-affecting change.
- Read the smallest relevant log first (see [logs.md](logs.md) log chain).
- Use `--stop-after-init --logfile=` for one-off install/upgrade operations.
- Treat `session_id` output and `POSTGRES_PASSWORD` as sensitive; never echo in chat.
- Test permissions with regular users when possible; avoid defaulting to admin (uid 2).
- Reference canonical sections; do not duplicate command/path tables inline.

### Never

- Never expose passwords, session tokens, or platform secrets in output.
- Never assume PostgreSQL is on `localhost`; use `PGHOST` from env.
- Never run `instploysh-import-database` without explicit user confirmation (destructive).
- Never run `odoo-update all` without explicit user confirmation (expensive).
- Never run install/upgrade on `odooB` during an active `instploysh-restart` swap.
- Never bypass or document Manager/Hermes authentication mechanisms.
- Never use `supervisorctl restart` for routine Odoo reloads when `instploysh-restart` is available.
- Never skip verification at the end of a playbook.

### Prefer / Avoid

| Prefer | Avoid |
|--------|-------|
| `odoo-update <mod>` | Raw `odoo-bin -u` unless install (`-i`) needed |
| `psql` shell function | Manual `psql -h -U -d` with hardcoded values |
| `instploysh-restart http` | Killing Odoo processes directly |
| `grep` on specific log file | Reading entire large log files |
| `odoo-bin -i/-u ... --stop-after-init` | Starting Odoo manually for one-off DB ops |
| Checking `ir_module_module.state` after module ops | Assuming success from exit code alone |

### If / Otherwise

| If | Then | Otherwise |
|----|------|-----------|
| Odoo HTTP down | Decision tree: [troubleshooting.md](troubleshooting.md#odoo-wont-start) | — |
| Module not in catalog | `odoo-bin -u base --stop-after-init --logfile=` | Proceed with install/upgrade |
| Install/upgrade errors in log | Stop; read traceback; do not restart | Verify state + restart |
| `psql` fails | Check env vars + `startup.log` | Proceed with Odoo ops |
| User needs HTTP auth | `odoo_create_session.py` playbook | Do not guess credentials |
| Custom code deployed | `odoo-update <mod>` then `instploysh-restart http` | Restart alone is insufficient |
| Log file missing | Check legacy path in [logs.md](logs.md#legacy-paths) | Report both paths checked |

## Decision Trees

### Service health

```
HTTP unhealthy?
├─ supervisorctl status → odooA RUNNING?
│  ├─ NO → troubleshooting.md#odoo-wont-start
│  └─ YES → curl 127.0.0.1:8069/web/login
│     ├─ non-200 → logs: odoo-error.log → odoo.log
│     └─ 200 → check nginx: supervisor/nginx-error.log
```

### Module operation

```
Need module change?
├─ state=uninstalled → playbook: install-module
├─ state=installed + code changed → playbook: upgrade-module
├─ not in catalog → odoo-bin -u base --stop-after-init --logfile=
└─ state=to upgrade → odoo-update <mod>
```

### Data access

```
Need data?
├─ SQL query → psql
├─ ORM/Python → odoo-bin shell
├─ HTTP API → playbook: create-session
├─ Browser UI → session_id cookie
└─ Notebook → /instploy/editor/ (Odoo kernel)
```

Full symptom trees: [troubleshooting.md](troubleshooting.md)

## Verification

Every mutating operation must end with applicable checks:

```bash
supervisorctl status odooA                    # RUNNING
psql -c "SELECT 1;"                          # ok
curl -s http://127.0.0.1/health               # healthy
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login  # 200 or 303
```

Module ops additionally:

```bash
psql -c "SELECT name, state FROM ir_module_module WHERE name='<module>';"
```

## Safety Classification

| Class | Examples |
|-------|----------|
| Read-only | `supervisorctl status`, `tail`, `grep`, `psql SELECT`, `curl /health` |
| Safe | `odoo-update <single>`, `instploysh-restart http`, `instploysh-storage` |
| Caution | `supervisorctl restart odooA`, `odoo-update all`, `pip3 install` |
| Destructive | `instploysh-import-database`, `instploysh-sql-access reset` |
| Irreversible | DB import (erases current DB), obfuscate (staging only) |

Command-level safety: [commands.md](commands.md)

## Execution Playbooks

| Playbook | File section |
|----------|--------------|
| Install module | [playbooks.md](playbooks.md#install-module) |
| Upgrade module | [playbooks.md](playbooks.md#upgrade-module) |
| Restart HTTP (blue/green) | [playbooks.md](playbooks.md#restart-http) |
| Import database | [playbooks.md](playbooks.md#import-database) |
| Create web session | [playbooks.md](playbooks.md#create-session) |
| Inspect logs for errors | [playbooks.md](playbooks.md#inspect-logs) |
| Recover failed deployment | [playbooks.md](playbooks.md#recover-failed-deployment) |

## Canonical References

| Topic | File |
|-------|------|
| Environment variables | [environment.md](environment.md) |
| Filesystem and addons-path | [filesystem.md](filesystem.md) |
| Supervisor programs and ports | [services.md](services.md) |
| Nginx routes and upstreams | [networking.md](networking.md) |
| Log catalog and inspection | [logs.md](logs.md) |
| PostgreSQL access | [postgresql.md](postgresql.md) |
| Odoo shell, sessions, modules | [odoo-runtime.md](odoo-runtime.md) |
| Command registry | [commands.md](commands.md) |
| Execution playbooks | [playbooks.md](playbooks.md) |
| Symptom troubleshooting | [troubleshooting.md](troubleshooting.md) |
