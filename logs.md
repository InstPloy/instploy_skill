# Logs

Canonical log path catalog. Symptom-to-log mapping: [troubleshooting.md](troubleshooting.md).

## Application logs

| Key | Path | Contents |
|-----|------|----------|
| `odoo_logs` | `/home/odoo/logs/odoo.log` | Main Odoo runtime |
| `startup` | `/home/odoo/logs/startup.log` | Startup, addons-path, DB connection |
| `install_log` | `/home/odoo/logs/install.log` | Module installation |
| `import_database` | `/home/odoo/logs/instploysh-import-database.log` | DB import |
| `restore` | `/home/odoo/logs/restore.log` | DB restore |
| `http_logs` | `/home/odoo/logs/access.log` | Nginx access |
| `http_error` | `/home/odoo/logs/error.log` | Nginx errors |
| `manager_instance` | `/home/odoo/logs/manager.log` | Session creation, manager Python ops |
| `backup` | `/home/odoo/logs/backup.log` | Scheduled backups |
| — | `/home/odoo/logs/odoo-bin-wrapper.log` | Wrapper args, env, addons check |
| — | `/home/odoo/logs/odoo-exec-error.log` | Odoo exec stderr (if enabled) |

## Supervisor logs

Base: `/home/odoo/logs/supervisor/`

| Service | stdout | stderr |
|---------|--------|--------|
| supervisord | `supervisord.log` | — |
| odooA / odooB | `odoo.log` | `odoo-error.log` |
| manager | `manager.log` | `manager-error.log` |
| nginx | `nginx.log` | `nginx-error.log` |
| jupyter_editor | `jupyter-editor.log` | `jupyter-editor-error.log` |
| jupyter_webshell | `jupyter-webshell.log` | `jupyter-webshell-error.log` |
| hermes_agent | `hermes-agent.log` | `hermes-agent-error.log` |

## Legacy paths

Manager UI may reference `/var/log/supervisor/` or `/var/log/supervisord.log`. Check both if file missing:

| Key | Path |
|-----|------|
| `supervisord_logs` | `/var/log/supervisord.log` |
| `manager_log` | `/var/log/supervisor/manager.out.log` |
| `manager_err` | `/var/log/supervisor/manager.err.log` |
| `supervisord_odoo_log` | `/var/log/supervisor/odoo.out.log` |
| `supervisord_odoo_err` | `/var/log/supervisor/odoo.err.log` |
| `supervisord_jupyterlab_log` | `/var/log/supervisor/jupyterlab.out.log` |
| `supervisord_jupyterlab_err` | `/var/log/supervisor/jupyterlab.err.log` |
| `supervisord_webshell_log` | `/var/log/supervisor/webshell.out.log` |
| `supervisord_webshell_err` | `/var/log/supervisor/webshell.err.log` |

## Log chain (read in order)

| Scenario | Order |
|----------|-------|
| Odoo won't start | `odoo-error.log` → `odoo-bin-wrapper.log` → `odoo.log` → `startup.log` |
| Install/upgrade failure | `odoo.log` (or `--logfile=` stdout) → `odoo-bin-wrapper.log` → `startup.log` |
| 502 HTTP | `odoo-error.log` → `nginx-error.log` |
| Session failure | `manager.log` |
| Addons missing | `startup.log` (grep `addons-path`) |

## Shell aliases

From `/etc/profile.d/instploy.sh`:

| Alias | Target |
|-------|--------|
| `tail-odoo` | `/home/odoo/logs/odoo.log` |
| `tail-manager` | `/home/odoo/logs/manager.log` |
| `tail-startup` | `/home/odoo/logs/startup.log` |
| `tail-backup` | `/home/odoo/logs/backup.log` |
| `tail-access` | `/home/odoo/logs/access.log` |

## Inspection patterns

```bash
tail -n 100 -f /home/odoo/logs/odoo.log
grep -iE "error|critical|traceback|failed" /home/odoo/logs/odoo.log | tail -40
grep "Final addons-path" /home/odoo/logs/startup.log
lnav /home/odoo/logs/odoo.log
ls -lhS /home/odoo/logs/
```

## Error signatures

| Pattern | Likely cause |
|---------|--------------|
| `CRITICAL ... Failed to load registry` | Broken module / migration |
| `psycopg2.*Error` | SQL/schema problem |
| `ParseError` / `XMLSyntaxError` | Bad view/data XML |
| `ImportError` / `ModuleNotFoundError` | Missing pip dep → `/opt/extra-packages` |
| `Module ... not found` | Wrong addons-path |
| `Traceback (most recent call last)` | Read lines below |

## Log env vars

| Variable | Effect |
|----------|--------|
| `LOG_LEVEL` | Odoo verbosity (`info` default) |
| `LOG_MANAGER` | Manager debug |
| `DEBUG_LOG` | Manager service debug |
