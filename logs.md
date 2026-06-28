# Logs Runtime Contract

**Scope:** InstPloy Odoo Container — log inspection.

---

## Runtime Contract

| Field | Value |
|-------|-------|
| Supported | `tail-odoo`, `tail-startup`, `tail-manager`, `grep` on known paths, `lnav` |
| Unsupported | `cat` on large logs, `tail -f` on unknown paths, reading entire log files |
| Golden rule | Read smallest relevant log first; use known paths only |
| Decision rule | Match symptom → log chain below |
| Failure rule | If path missing, check legacy paths; never invent log locations |

---

## Canonical Log APIs

| Intent | Canonical | Forbidden |
|--------|-----------|-----------|
| Follow Odoo log | `tail-odoo` | `tail -f /var/log/...` (unknown) |
| Follow startup | `tail-startup` | — |
| Follow manager | `tail-manager` | — |
| Search errors | `grep -iE "error\|critical\|traceback" <known-path>` | `cat odoo.log` |
| Addons-path check | `grep "Final addons-path" /home/odoo/logs/startup.log` | — |

Aliases from `/etc/profile.d/instploy.sh`: `tail-odoo`, `tail-manager`, `tail-startup`, `tail-backup`, `tail-access`

---

## Log Catalog

### Application (`/home/odoo/logs/`)

| Key | Path |
|-----|------|
| `odoo_logs` | `odoo.log` |
| `startup` | `startup.log` |
| `install_log` | `install.log` |
| `import_database` | `instploysh-import-database.log` |
| `http_logs` | `access.log` |
| `http_error` | `error.log` |
| `manager_instance` | `manager.log` |
| `backup` | `backup.log` |
| wrapper | `odoo-bin-wrapper.log` |

### Supervisor (`/home/odoo/logs/supervisor/`)

| Service | stdout | stderr |
|---------|--------|--------|
| odooA/B | `odoo.log` | `odoo-error.log` |
| nginx | `nginx.log` | `nginx-error.log` |
| manager | `manager.log` | `manager-error.log` |
| jupyter | `jupyter-editor.log` | `jupyter-editor-error.log` |
| hermes | `hermes-agent.log` | `hermes-agent-error.log` |

### Legacy (if primary missing)

`/var/log/supervisor/*.out.log`, `/var/log/supervisor/*.err.log`, `/var/log/supervisord.log`

---

## Investigation Priority

| Scenario | Order |
|----------|-------|
| Odoo won't start | `odoo-error.log` → `odoo-bin-wrapper.log` → `odoo.log` → `startup.log` |
| Install/upgrade fail | operation output → `odoo.log` → `odoo-bin-wrapper.log` → `startup.log` |
| HTTP 502 | `odoo-error.log` → `nginx-error.log` |
| Session fail | `manager.log` |
| Addons missing | `startup.log` |
| Restart issues | `startup.log` → supervisor stderr → `odoo.log` → wrapper → infrastructure |

---

## Error Signatures

| Pattern | Cause |
|---------|-------|
| `Failed to load registry` | Broken module / migration |
| `psycopg2.*Error` | DB/schema |
| `ParseError` / `XMLSyntaxError` | Bad XML |
| `ImportError` | Missing pip dep → `/opt/extra-packages` |
| `Module ... not found` | Addons-path |

---

## Anti-Patterns

| Violation | Regenerate as |
|-----------|---------------|
| `cat /home/odoo/logs/odoo.log` | `grep ... \| tail -40` or `tail-odoo` |
| `journalctl` | Known path + `grep` |
| `docker logs` (inside container) | Supervisor/app logs above |
| Invented log path | Use catalog table |

---

## Verification

After log investigation, confirm fix with operation-specific verification from [SKILL.md](SKILL.md#verification-mandatory-last-step).

Symptom trees: [troubleshooting.md](troubleshooting.md)
