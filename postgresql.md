# PostgreSQL

Canonical database access reference.

## Credentials

Source: environment variables only. See [environment.md](environment.md#postgresql).

| Variable | Role |
|----------|------|
| `PGDATABASE` | Database name |
| `POSTGRES_USER` | Username |
| `POSTGRES_PASSWORD` | Password (sensitive) |
| `PGHOST` | Host (often external, not localhost) |
| `PGPORT` / `PGDBPORT` | Port |

Profile sets `PGPASSWORD` from `POSTGRES_PASSWORD` automatically.

## Connection

| Command | Safety | Purpose |
|---------|--------|---------|
| `psql` | Read-only* | Interactive shell (env-aware function) |
| `psql -c "SELECT 1;"` | Read-only | Connectivity test |
| `psql -c "SELECT ..."` | Read-only* | One-shot query |

*Writes possible if SQL contains INSERT/UPDATE/DELETE.

### Connectivity test

```bash
psql -c "SELECT 1 AS ok;"
```

Failure: check `PGHOST`, `PGDATABASE`, `POSTGRES_USER` + [logs.md](logs.md) `startup.log`.

## Common queries

```bash
# Users
psql -c "SELECT id, login, name, active FROM res_users WHERE active ORDER BY id;"

# Module state
psql -c "SELECT name, state, latest_version FROM ir_module_module WHERE name='<module>';"

# Installed modules
psql -c "SELECT name, state, latest_version FROM ir_module_module WHERE state='installed' ORDER BY name LIMIT 20;"

# Base version
psql -c "SELECT latest_version FROM ir_module_module WHERE name='base';"

# Web base URL
psql -c "SELECT value FROM ir_config_parameter WHERE key='web.base.url';"

# Neutralized flag (staging)
psql -c "SELECT value FROM ir_config_parameter WHERE key='database.is_neutralized';"
```

## Module states

| State | Meaning |
|-------|---------|
| `uninstalled` | Not installed |
| `installed` | Active |
| `to install` | Pending install |
| `to upgrade` | Pending upgrade |

## User lookup

```bash
psql -c "SELECT id, login FROM res_users WHERE id=2;"                          # admin
psql -c "SELECT id, login, active FROM res_users WHERE share=false ORDER BY id;" # internal
psql -c "SELECT id, login FROM res_users WHERE share=true AND active LIMIT 10;"  # portal
```

## External SQL access

```bash
instploysh-sql-access status|init|reset|disable|kill
```

Dedicated servers only. Read-only remote access for BI tools. Platform callback — out of scope for internals.

## Security

- Never echo `POSTGRES_PASSWORD` in chat or logs
- Prefer `SELECT` for diagnostics
- Test permissions with regular users, not admin (uid 2)
