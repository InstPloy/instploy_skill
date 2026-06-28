# Supervisor Runtime Contract

**Scope:** InstPloy Odoo Container — process management.

---

## Runtime Contract

| Field | Value |
|-------|-------|
| Supported | `supervisorctl` (status, signal HUP), `instploysh-restart` |
| Unsupported | `systemctl`, `kill -9`, `killall`, manual process management |
| Golden rule | Restart Odoo HTTP via `instploysh-restart http` |
| Decision rule | Need restart → instploysh-restart; need status → supervisorctl status |
| Failure rule | Never kill processes; investigate logs then use canonical restart |

---

## Programs

| Program | Port(s) | Autostart | User |
|---------|---------|-----------|------|
| `nginx` | 80 | yes | root |
| `odooA` | 8069 / 8072 | yes | odoo |
| `odooB` | 9069 / 9072 | no | odoo |
| `jupyter_editor` | 8888 | yes | odoo |
| `jupyter_webshell` | 8889 | yes | odoo |
| `manager` | 5001 | yes | root |
| `hermes_agent` | 8642 | yes | odoo |

Config: `/etc/supervisor/conf.d/*.conf`
Socket: `unix:///var/run/supervisor/supervisor.sock`

Blue/green: `instploysh-restart http` swaps odooA/odooB via nginx. Never install/upgrade on odooB during swap.

---

## Canonical Operations

| Task | Canonical | Forbidden |
|------|-----------|-----------|
| Check status | `supervisorctl status` | `ps aux` + kill |
| Restart HTTP | `instploysh-restart http` | `supervisorctl restart odooA` (routine) |
| Restart cron | `instploysh-restart cron` | — |
| Reload nginx | `supervisorctl signal HUP nginx` | `kill` nginx master |
| After conf change | `supervisorctl reread && supervisorctl update` | Edit without reread |

Direct `supervisorctl restart odooA` only when user **explicitly** requests it.

---

## Investigation Priority

1. `supervisorctl status`
2. `/home/odoo/logs/supervisor/odoo-error.log`
3. `/home/odoo/logs/startup.log`
4. `/home/odoo/logs/odoo.log`
5. `/home/odoo/logs/odoo-bin-wrapper.log`
6. Infrastructure / platform

---

## Anti-Patterns

| Violation | Why | Regenerate as |
|-----------|-----|---------------|
| `kill -9 <odoo-pid>` | Bypasses blue/green | `instploysh-restart http` |
| `systemctl restart odoo` | No systemctl in container | `instploysh-restart http` |
| `supervisorctl restart odooA` (routine) | No zero-downtime | `instploysh-restart http` |
| `pkill -f odoo` | Destructive, no rollback | `instploysh-restart http` |

---

## Verification

```bash
supervisorctl status odooA    # RUNNING
curl -s http://127.0.0.1/health
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
```

Nginx routes: [networking.md](networking.md)
