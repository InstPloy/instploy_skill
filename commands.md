# Commands

Canonical command registry. Define once; reference from playbooks and troubleshooting.

Safety: **R**=Read-only, **S**=Safe, **C**=Caution, **D**=Destructive, **I**=Irreversible

## instploysh commands

| Command | Safety | Purpose | When to use | When NOT to use | Expected output | Failure symptoms |
|---------|--------|---------|-------------|-----------------|-----------------|------------------|
| `instploysh-restart http` | S | Blue/green Odoo HTTP restart | After code deploy, config change | During active install/upgrade on odooB | `Blue/Green restart completed` | Timeout on endpoint poll |
| `instploysh-restart cron` | S | Restart cron worker | Cron/mail issues | HTTP-only problems | Completion message | supervisorctl error |
| `instploysh-import-database <file>` | D/I | Replace DB + filestore from backup | Explicit restore request | Routine deploys | `Finished.` exit 0 | `InvalidDumpError`, psql errors |
| `instploysh-storage` | R | Disk usage analyzer (ncdu) | Storage investigation | — | Interactive ncdu UI | ncdu not installed |
| `instploysh-sync-branch` | S | Sync web.base.url, neutralize banner | Manual post-restore sync | Normal ops (auto on startup) | DB params updated | API/DB connection warning |
| `instploysh-sql-access <cmd>` | C/D | External read-only PG access | Dedicated server BI setup | Development containers | Connection details | Platform callback error |
| `instploysh-dumpstacks` | S | Dump Odoo stack traces | Debug hung workers | — | Stack output | Platform callback error |

Paths: `/opt/instploy/instploysh/bin/instploysh-*`

## odoo commands

| Command | Safety | Purpose | When to use | When NOT to use | Expected output | Failure symptoms |
|---------|--------|---------|-------------|-----------------|-----------------|------------------|
| `odoo-bin` | C | Start Odoo (wrapper) | Never manually; supervisor uses it | One-off DB ops | Running server | See [logs.md](logs.md) chain |
| `odoo-bin shell` | C | Interactive ORM shell | Debug data, run ORM queries | Automated scripts | Python REPL with `env` | Registry/DB errors |
| `odoo-bin -i <mod> --stop-after-init --logfile=` | C | Install module(s) | Module state=uninstalled | Module already installed | Exit 0, state=installed | Registry/XML/traceback errors |
| `odoo-bin -u <mod> --stop-after-init --logfile=` | C | Upgrade module(s) | Code changed, state=installed | Fresh install (use `-i`) | Exit 0 | Same as install |
| `odoo-bin -u base --stop-after-init --logfile=` | S | Refresh module catalog | Module not in catalog | — | Module list updated | DB connection error |
| `odoo-update <mod>` | C | Upgrade wrapper | Preferred upgrade path | Install (use `-i`) | stdout log output | Same as `-u` |
| `odoo-update all` | C | Upgrade all modules | Explicit full upgrade request | Routine single-module deploy | Long-running output | Timeout, registry failure |
| `odoo-rpc` | C | Internal RPC server | Never manually | — | Running RPC | Port conflict |
| `odoo-kernel <conn>` | C | Jupyter Odoo kernel | Never manually | — | IPython kernel | Jupyter manages this |

Wrapper exec target: `/home/odoo/src/odoo/odoo-bin`
Wrapper log: `/home/odoo/logs/odoo-bin-wrapper.log`

## Python utilities

| Command | Safety | Purpose | When to use | When NOT to use |
|---------|--------|---------|-------------|-----------------|
| `python3 .../odoo_create_session.py <uid>` | C | Create web session JSON | HTTP API auth needed | When psql/shell suffices |
| `odoo_utils.py` | — | Library (not CLI) | — | Never run directly |

## supervisorctl

| Command | Safety | Purpose | When to use | When NOT to use |
|---------|--------|---------|-------------|-----------------|
| `supervisorctl status` | R | All program states | Every orient/verify step | — |
| `supervisorctl status odooA` | R | Single program state | Targeted check | — |
| `supervisorctl restart odooA` | C | Direct Odoo restart | User explicitly requests | Routine restart (use instploysh-restart) |
| `supervisorctl restart nginx` | C | Restart nginx | nginx config broken | Routine Odoo restart |
| `supervisorctl signal HUP nginx` | S | Reload nginx config | After upstream swap | — |
| `supervisorctl reread` | C | Reload supervisor config | After conf.d change | — |
| `supervisorctl update` | C | Apply config changes | After reread | — |

Socket: `unix:///var/run/supervisor/supervisor.sock`

## Shell utilities (profile)

| Alias | Maps to | Safety |
|-------|---------|--------|
| `psql` | env-aware psql function | R* |
| `tail-odoo` | follow odoo.log | R |
| `tail-manager` | follow manager.log | R |
| `tail-startup` | follow startup.log | R |
| `tail-backup` | follow backup.log | R |
| `tail-access` | follow access.log | R |
| `reload-profile` | source instploy.sh | S |
| `psaux` | ps auxwww | R |
| `http` | httpie monokai | R/C |

Source: `/etc/profile.d/instploy.sh`

## Backup

| Command | Safety | Purpose |
|---------|--------|---------|
| `/usr/local/bin/start_backup.py` | S | Scheduled backup execution |

Storage: `/home/odoo/backups/` | Log: `/home/odoo/logs/backup.log`

## pip

```bash
pip3 install --target=/opt/extra-packages <package>
```

Safety: Caution. Missing deps cause `ImportError` in module install/upgrade.

## Command paths

| Name | Path |
|------|------|
| odoo-bin | `/opt/instploy/instploysh/bin/odoo-bin` |
| odoo-update | `/opt/instploy/instploysh/bin/odoo-update` |
| odoo-rpc | `/opt/instploy/instploysh/bin/odoo-rpc` |
| odoo-kernel | `/opt/instploy/instploysh/bin/odoo-kernel` |
| instploysh-restart | `/opt/instploy/instploysh/bin/instploysh-restart` |
| instploysh-import-database | `/opt/instploy/instploysh/bin/instploysh-import-database` |
| instploysh-storage | `/opt/instploy/instploysh/bin/instploysh-storage` |
| instploysh-sync-branch | `/opt/instploy/instploysh/bin/instploysh-sync-branch` |
| instploysh-sql-access | `/opt/instploy/instploysh/bin/instploysh-sql-access` |
| instploysh-dumpstacks | `/opt/instploy/instploysh/bin/instploysh-dumpstacks` |
| odoo_create_session.py | `/opt/instploy/instploysh/lib/odoo_create_session.py` |
| Real Odoo binary | `/home/odoo/src/odoo/odoo-bin` |
| start_backup.py | `/usr/local/bin/start_backup.py` |
| startup.sh | `/usr/local/bin/startup.sh` |

Programs: [services.md](services.md)
