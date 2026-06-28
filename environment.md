# Environment Runtime Contract

**Scope:** InstPloy Odoo Container — runtime-provided configuration.

---

## Runtime Contract

| Field | Value |
|-------|-------|
| Purpose | Reference only — agents READ env, never CONSTRUCT connections from it |
| Golden rule | Env vars are consumed by wrappers (`psql`, `odoo-bin`), not by agent-assembled CLI flags |
| Forbidden | `echo $POSTGRES_PASSWORD`, building `psql -h $PGHOST ...`, exporting creds |

---

## Pre-Configured Assumptions

Inside container, runtime has already set:

| Variable | Role |
|----------|------|
| `PGDATABASE` | Active Odoo database |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` | DB credentials (password via `PGPASSWORD`) |
| `PGHOST` | DB host (often external, not localhost) |
| `PGPORT` / `PGDBPORT` | DB port (default 5432) |
| `ODOO_VERSION` | e.g. `18.0` |
| `ODOO_STAGE` | development / staging / production |
| `app` | odooA or odooB |
| `odoo_port` / `chat_port` | HTTP / gevent ports |
| `ADDONS_PATH` | Addons roots (wrapper resolves) |
| `WORKERS` | Worker count |
| `ENTERPRISE` | Enterprise/themes paths |
| `LOG_LEVEL` | Odoo verbosity |

Safe diagnostic (no secrets):

```bash
echo "DB=$PGDATABASE STAGE=$ODOO_STAGE VERSION=$ODOO_VERSION APP=$app"
echo "HOST=$PGHOST PORT=${PGPORT:-${PGDBPORT:-5432}} USER=$POSTGRES_USER"
test -n "$POSTGRES_PASSWORD" && echo "Password: [set]" || echo "Password: [not set]"
```

---

## Port Logic

| app | HTTP | Gevent |
|-----|------|--------|
| odooA | 8069 | 8072 |
| odooB | 9069 | 9072 |

Odoo 18+: `--http-port`. Odoo 17-: `--xmlrpc-port`. Gevent when `WORKERS > 0`.

---

## PYTHONPATH

```
/opt/extra-packages:/opt/instploy:/home/odoo/src/odoo
```

Set by wrappers and `/etc/profile.d/instploy.sh`.

---

## Anti-Patterns

| Violation | Why |
|-----------|-----|
| Reconstruct connection from env vars in CLI | Wrappers already bind env |
| `export POSTGRES_PASSWORD=...` | Runtime owns creds |
| Change `PGHOST` to `localhost` | Wrong host; breaks connection |
| Manual `ADDONS_PATH` export to fix modules | Deploy code; check startup.log |

---

## Investigation Priority

1. Orient echo commands above
2. `psql -c "SELECT 1;"`
3. `/home/odoo/logs/startup.log`
4. Platform/infrastructure

PostgreSQL usage: [postgresql.md](postgresql.md)
