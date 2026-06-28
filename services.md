# Services

Canonical Supervisor and internal port reference.

## Supervisor programs

| Program | Command | Autostart | User | Logs |
|---------|---------|-----------|------|------|
| `nginx` | `/usr/sbin/nginx -g "daemon off;"` | yes | root | [logs.md](logs.md#supervisor-logs) |
| `odooA` | `/opt/instploy/instploysh/bin/odoo-bin` | yes | odoo | supervisor/odoo.log |
| `odooB` | `/opt/instploy/instploysh/bin/odoo-bin` | no | odoo | supervisor/odoo.log |
| `jupyter_editor` | jupyter-lab | yes | odoo | supervisor/jupyter-editor.log |
| `jupyter_webshell` | sh-webshell | yes | odoo | supervisor/jupyter-webshell.log |
| `manager` | `/opt/instploy/image_manager` | yes | root | supervisor/manager.log |
| `hermes_agent` | `/usr/local/bin/hermes-gateway` | yes | odoo | supervisor/hermes-agent.log |

Config: `/etc/supervisor/conf.d/*.conf`
Socket: `unix:///var/run/supervisor/supervisor.sock`
Main config: `/etc/supervisor/supervisord.conf`

## Internal ports

| Service | Program | HTTP | Gevent | User |
|---------|---------|------|--------|------|
| Nginx | `nginx` | 80 | — | root |
| odooA | `odooA` | 8069 | 8072 | odoo |
| odooB | `odooB` | 9069 | 9072 | odoo |
| JupyterLab | `jupyter_editor` | 8888 | — | odoo |
| Web shell | `jupyter_webshell` | 8889 | — | odoo |
| Manager | `manager` | 5001 | — | root |
| Hermes | `hermes_agent` | 8642 | — | odoo |

Port defaults and overrides: [environment.md](environment.md#port-selection-logic)

## Blue/green (odooA / odooB)

- `odooA`: primary, autostart, ports 8069/8072
- `odooB`: warm-up slot, autostart=false, ports 9069/9072
- `instploysh-restart http` orchestrates swap via nginx upstream rewrite
- Never run install/upgrade on `odooB` during active swap

## supervisorctl

| Action | Command | Safety |
|--------|---------|--------|
| Status | `supervisorctl status` | Read-only |
| Status one | `supervisorctl status odooA` | Read-only |
| Restart Odoo | `instploysh-restart http` | Safe (preferred) |
| Direct restart | `supervisorctl restart odooA` | Caution (brief downtime) |
| Reload nginx | `supervisorctl signal HUP nginx` | Safe |
| After config change | `supervisorctl reread && supervisorctl update` | Caution |

Nginx routes: [networking.md](networking.md)

## odoo-bin wrapper responsibilities

- Build `--addons-path` from env
- Inject DB credentials
- Set ports from `odoo_port`/`chat_port`
- Run `instploysh-sync-branch` on startup
- Log to `/home/odoo/logs/odoo-bin-wrapper.log`
- Exec real binary: `/home/odoo/src/odoo/odoo-bin`

Special wrapper-only subcommand: `obfuscate` (staging, Odoo 16+).

## Backup service

| Item | Path |
|------|------|
| Script | `/usr/local/bin/start_backup.py` |
| Storage | `/home/odoo/backups/` |
| Log | `/home/odoo/logs/backup.log` |

Configured via task volume JSON schedule.
