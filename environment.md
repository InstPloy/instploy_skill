# Environment

Canonical environment variable reference. All credentials and connection targets come from env — never hardcode.

## Identity

| Variable | Purpose | Default |
|----------|---------|---------|
| `PGDATABASE` | Active Odoo database name | `master` / `odoo` |
| `ODOO_VERSION` | Version string (e.g. `18.0`) | — |
| `ODOO_STAGE` | `development`, `staging`, `production` | — |
| `app` | Active Odoo slot: `odooA` or `odooB` | `odooA` |

## PostgreSQL

| Variable | Purpose | Default |
|----------|---------|---------|
| `PGHOST` | Database host | `db` (often external IP, not localhost) |
| `PGPORT` / `PGDBPORT` | Database port | `5432` |
| `POSTGRES_USER` | DB username | `odoo` |
| `POSTGRES_PASSWORD` | DB password | — |
| `PGUSER` | Alias for `POSTGRES_USER` | — |
| `PGPASSWORD` | Alias; set from `POSTGRES_PASSWORD` by profile | — |

Safe read (never print password):

```bash
echo "DB=$PGDATABASE HOST=$PGHOST PORT=${PGPORT:-${PGDBPORT:-5432}} USER=$POSTGRES_USER"
test -n "$POSTGRES_PASSWORD" && echo "Password: [set]" || echo "Password: [not set]"
```

## Odoo runtime

| Variable | Purpose | Default |
|----------|---------|---------|
| `ADDONS_PATH` | Comma-separated addons roots (resolved at startup) | — |
| `WORKERS` | Worker count; `0` = single process | `0` |
| `ENTERPRISE` | Include enterprise/themes paths | `true` |
| `odoo_port` | HTTP port for current slot | 8069 (odooA) / 9069 (odooB) |
| `chat_port` | Gevent/websocket port | 8072 (odooA) / 9072 (odooB) |
| `LOG_LEVEL` | Odoo log verbosity | `info` |
| `ODOO_KERNEL` | `1` = notebook mode (no HTTP, workers=0) | `0` |
| `ODOO_STAGE` | Controls `--without-demo` and neutralize banner | — |
| `DB_MAXCONN` | Max DB connections | `16` |
| `LIMIT_MEMORY_HARD` | Memory hard limit (bytes) | `2684354560` |
| `LIMIT_MEMORY_SOFT` | Memory soft limit (bytes) | `2147483648` |
| `MAX_CRON_THREADS` | Cron thread count | `2` |
| `SMTP_HOST` / `SMTP_PORT` | SMTP relay | `127.0.0.1` / `25` |

## Logging / debug

| Variable | Purpose |
|----------|---------|
| `LOG_MANAGER` | Manager debug logging |
| `DEBUG_LOG` | Manager service debug mode |

## PYTHONPATH

Set automatically by wrappers:

```
/opt/extra-packages:/opt/instploy:/home/odoo/src/odoo
```

`/opt/extra-packages` overrides core when same module name exists.

## Port selection logic

- `app=odooA` → HTTP 8069, gevent 8072
- `app=odooB` → HTTP 9069, gevent 9072
- Override: `odoo_port`, `chat_port`
- Odoo 18+: `--http-port`; Odoo 17-: `--xmlrpc-port`
- `--gevent-port` only when `WORKERS > 0`

## Odoo version flags

| Version | HTTP flag | Notes |
|---------|-----------|-------|
| 17 and below | `--xmlrpc-port` | Standard workers + gevent |
| 18+ | `--http-port` | Registry API changes in kernel |
| 19+ | `--http-port` | `db_name` is a list in config |

Check: `echo $ODOO_VERSION`

## Shell profile

`/etc/profile.d/instploy.sh` exports:
- `PGPASSWORD` from `POSTGRES_PASSWORD`
- `PIP_TARGET=/opt/extra-packages`
- `PYTHONPATH` with extra-packages
- Log tail aliases (`tail-odoo`, etc.)
- `psql` shell function (env-aware)

Reload: `reload-profile` or `source /etc/profile.d/instploy.sh`
