# Command Registry

Canonical APIs only. Violations: [SKILL.md](SKILL.md#runtime-contract-violations).

Safety: **R** Read-only · **S** Safe · **C** Caution · **D** Destructive · **I** Irreversible

| Task | Command | Safety | When NOT |
|------|---------|--------|----------|
| DB query | `psql` / `psql -c` | R* | Never `-h/-p/-U/-d` |
| Session obtain | Read `/home/odoo/.cache/instploy/session.json` → validate | R | Never skip cache |
| Session create | `odoo_create_session.py <uid>` (cache miss/invalid only) | C | Never per-request |
| JSON-RPC ORM | `curl .../web/dataset/call_kw -b session_id=` | C | Never XML-RPC |
| Upgrade module | `odoo-update <mod>` | C | Never `odoo-bin -u` |
| Install module | `odoo-bin -i <mod> --stop-after-init --logfile=` | C | — |
| Catalog refresh | `odoo-bin -u base --stop-after-init --logfile=` | S | — |
| Restart HTTP | `instploysh-restart http` | S | Never `kill -9` |
| Restart cron | `instploysh-restart cron` | S | — |
| Service status | `supervisorctl status` | R | Never `systemctl` |
| Reload nginx | `supervisorctl signal HUP nginx` | S | — |
| Direct odoo restart | `supervisorctl restart odooA` | C | Only if user explicit |
| Odoo shell | `odoo-bin shell` | C | — |
| DB import | `instploysh-import-database <file>` | D/I | Without user confirm |
| Storage | `instploysh-storage` | R | — |
| Logs follow | `tail-odoo` etc. | R | Never `cat` huge logs |
| Pip dep | `pip3 install --target=/opt/extra-packages` | C | — |
| Health | `curl -s http://127.0.0.1/health` | R | — |

\*Writes if SQL mutates.

## Paths

| Name | Path |
|------|------|
| odoo-bin | `/opt/instploy/instploysh/bin/odoo-bin` |
| odoo-update | `/opt/instploy/instploysh/bin/odoo-update` |
| instploysh-restart | `/opt/instploy/instploysh/bin/instploysh-restart` |
| odoo_create_session.py | `/opt/instploy/instploysh/lib/odoo_create_session.py` |
| Real binary (do not call) | `/home/odoo/src/odoo/odoo-bin` |

Subsystems: [postgresql.md](postgresql.md) · [jsonrpc.md](jsonrpc.md) · [odoo-runtime.md](odoo-runtime.md) · [services.md](services.md)
