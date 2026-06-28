# Odoo Runtime Contract

**Scope:** InstPloy Odoo Container — shell, modules, import.

---

## Runtime Contract

| Field | Value |
|-------|-------|
| Supported entry | `odoo-bin` wrapper at `/opt/instploy/instploysh/bin/odoo-bin` |
| Unsupported | Direct `/home/odoo/src/odoo/odoo-bin`, manual `--addons-path`, manual DB flags |
| Golden rule | Wrapper owns addons-path, DB creds, ports |
| Session/RPC | See [jsonrpc.md](jsonrpc.md) — not covered here |
| SQL | See [postgresql.md](postgresql.md) — `psql` only |

---

## Decision Tree

```
Need Odoo operation?
├─ SQL → psql ([postgresql.md](postgresql.md))
├─ ORM in-process → odoo-bin shell
├─ ORM over HTTP → [jsonrpc.md](jsonrpc.md)
├─ Install module → odoo-bin -i <mod> --stop-after-init --logfile=
├─ Upgrade module → odoo-update <mod>
├─ Restart after deploy → instploysh-restart http
└─ Notebook → /instploy/editor/ (kernel managed)
```

---

## Odoo Shell

| Field | Value |
|-------|-------|
| Canonical | `odoo-bin shell` |
| Safety | Caution (ORM writes) |
| Commit | `env.cr.commit()` required |

```python
env['res.users'].search_read([('active', '=', True)], ['login', 'name'])
env.cr.commit()
```

Uncommitted shell changes can block the database.

---

## Modules Runtime Contract

| Operation | Canonical | Forbidden |
|-----------|-----------|-----------|
| Upgrade | `odoo-update <mod>` | `odoo-bin -u <mod>` |
| Install | `odoo-bin -i <mod> --stop-after-init --logfile=` | Manual SQL on `ir_module_module` |
| Catalog refresh | `odoo-bin -u base --stop-after-init --logfile=` | — |
| Upgrade all | `odoo-update all` (user explicit only) | — |
| Post-deploy | `odoo-update <mod>` then `instploysh-restart http` | Restart without upgrade |

Required flags for one-off ops: `--stop-after-init --logfile=`

### Verify module state

```bash
psql -c "SELECT name, state, latest_version FROM ir_module_module WHERE name='<module>';"
```

| State | Action |
|-------|--------|
| `uninstalled` | `odoo-bin -i` |
| `installed` + code change | `odoo-update` |
| not in catalog | `odoo-bin -u base --stop-after-init --logfile=` |

Playbooks: [playbooks.md#install-module](playbooks.md#install-module) · [upgrade-module](playbooks.md#upgrade-module)

---

## Database Import

| Field | Value |
|-------|-------|
| Canonical | `instploysh-import-database <backup.zip>` |
| Safety | Destructive / Irreversible |
| Filestore | `/home/odoo/data/filestore/<PGDATABASE>` |

Requires explicit user confirmation before execution.

---

## Jupyter Kernel

| Field | Value |
|-------|-------|
| Route | `/instploy/editor/` |
| Script | `/opt/instploy/instploysh/bin/odoo-kernel` |
| Managed by | Supervisor `jupyter_editor` |
| Agent action | Do not run manually |

---

## Investigation Priority

| Scenario | Order |
|----------|-------|
| Module install/upgrade fail | operation log → `odoo.log` → `odoo-bin-wrapper.log` → `startup.log` |
| Module not found | `startup.log` (addons-path) → `filesystem.md` user addons |
| Shell/registry error | `odoo.log` traceback |

---

## Anti-Patterns

| Violation | Regenerate as |
|-----------|---------------|
| `odoo-bin -u sale` (upgrade) | `odoo-update sale` |
| `/home/odoo/src/odoo/odoo-bin` direct | `odoo-bin` wrapper |
| `--addons-path=...` manual | Check `grep addons-path startup.log` |
| `--db_host` / `--db_user` manual | Use wrapper (reads env) |
| Skip `--stop-after-init` on install | Add `--stop-after-init --logfile=` |

---

## Verification

After module ops:

```bash
psql -c "SELECT name, state FROM ir_module_module WHERE name='<module>';"
instploysh-restart http
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
```

Sessions/JSON-RPC: [jsonrpc.md](jsonrpc.md)
