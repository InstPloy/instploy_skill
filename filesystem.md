# Filesystem

Canonical path reference. Do not duplicate elsewhere.

## Directory tree

```
/home/odoo/
├── src/
│   ├── odoo/                 # Core; real binary: src/odoo/odoo-bin
│   │   ├── addons/
│   │   └── odoo/addons/
│   ├── enterprise/           # If ENTERPRISE=true
│   ├── themes/               # If ENTERPRISE=true
│   ├── user/                 # Custom addons (mounted)
│   └── task/                 # Scheduled tasks (mounted)
├── data/
│   ├── filestore/<db>/       # Attachments
│   └── sessions/             # HTTP sessions
├── logs/                     # Application logs
│   └── supervisor/           # Per-service supervisor logs
├── backups/                  # Backup storage (mounted)
├── .config/odoo/odoo.conf    # Odoo config (mounted)
├── .ssh/                     # Platform-managed
└── .bash_history             # Mounted

/opt/
├── instploy/
│   ├── instploysh/bin/       # CLI wrappers
│   ├── instploysh/lib/       # Python helpers
│   ├── jupyterlab/           # JupyterLab
│   └── image_manager         # Manager binary (platform-managed)
├── extra-packages/           # User pip installs
└── jupyter/packages/

/etc/
├── nginx/conf.d/default.conf
├── supervisor/conf.d/
└── profile.d/instploy.sh
```

## Volume mounts (typical)

| Host | Container |
|------|-----------|
| `./addons` | `/home/odoo/src/user` |
| `./data_dir` | `/home/odoo/data` |
| `./logs` | `/home/odoo/logs` |
| `./etc/odoo.conf` | `/home/odoo/.config/odoo/odoo.conf` |
| `./backups` | `/home/odoo/backups` |
| `./task` | `/home/odoo/src/task` |
| SSH keys | `/home/odoo/.ssh` |

## Key paths

| Path | Purpose |
|------|---------|
| `/opt/instploy/instploysh/bin/odoo-bin` | Odoo wrapper (use this) |
| `/home/odoo/src/odoo/odoo-bin` | Real Odoo binary (exec target) |
| `/opt/instploy/instploysh/lib/odoo_create_session.py` | Session creation |
| `/home/odoo/data/filestore/$PGDATABASE` | Active filestore |
| `/home/odoo/.config/odoo/odoo.conf` | Odoo config |
| `/etc/supervisor/conf.d/*.conf` | Supervisor programs |
| `/usr/local/bin/start_backup.py` | Scheduled backup |
| `/usr/local/bin/startup.sh` | Container startup |

## Data directory

| Path | Purpose |
|------|---------|
| `/home/odoo/data` | Odoo `--data-dir` |
| `/home/odoo/data/filestore/<PGDATABASE>` | Attachments |
| `/home/odoo/data/sessions/` | Web session files |

After DB import, filestore lands at `/home/odoo/data/filestore/<db_name>`.

## Addons-path resolution

`odoo-bin` wrapper calls `odoo_utils.get_odoo_args()`:

1. Parse `ADDONS_PATH` env (comma-separated)
2. Always include: `/home/odoo/src/odoo/addons`, `/home/odoo/src/odoo/odoo/addons`
3. Include path only if it contains valid addon (`__manifest__.py` in subdirectory)
4. Recursively discover nested parents under `/home/odoo/src/user`
5. If `ENTERPRISE=true`: add `/home/odoo/src/enterprise`, `/home/odoo/src/themes` when valid
6. Fallback: core paths only

Verify resolved path:

```bash
grep "Final addons-path" /home/odoo/logs/startup.log
```

## Inspection commands

```bash
find /home/odoo/src/user -name __manifest__.py | head -20
du -sh /home/odoo/data/filestore/$PGDATABASE
ls /home/odoo/data/sessions/ 2>/dev/null | wc -l
ls -la /home/odoo/src/odoo/odoo-bin
ss -tlnp | grep -E '8069|9069|8072|9072'
```

## pip installs

```bash
pip3 install --target=/opt/extra-packages <package>
```

`PIP_TARGET` and `PYTHONPATH` set in `/etc/profile.d/instploy.sh`.
