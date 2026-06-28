# PostgreSQL Runtime Contract

**Scope:** InstPloy Odoo Containers

This document defines the only supported way autonomous agents may access PostgreSQL inside an InstPloy Odoo container.

---

# Runtime Contract

Environment: **InstPloy Odoo Container**

Database Access: **Environment-managed**

Supported Interface:

```bash
psql
psql -c "<SQL>"
```

Unsupported Interfaces:

* Manual PostgreSQL connections
* Connection strings
* Manual credentials
* Manual host selection
* Manual port selection

This contract has **no exceptions** unless the user explicitly requests raw PostgreSQL connection commands.

---

# Golden Rule

Inside an InstPloy container, **`psql` is the PostgreSQL API.**

Agents must never construct PostgreSQL connections manually.

The runtime automatically resolves:

* database
* username
* password
* host
* port

from the container environment.

Agents must rely entirely on the runtime.

---

# AI Operational Rules

## ALWAYS

* Use `psql` for interactive access.
* Use `psql -c "<SQL>"` for one-shot queries.
* Use `psql -c "SELECT 1;"` to verify connectivity.
* Prefer `SELECT` statements for diagnostics.
* Look up actual user IDs before creating sessions.
* Verify database state before performing changes.
* Verify results after every write operation.

---

## NEVER

Never generate commands containing:

* `-h`
* `-p`
* `-U`
* `-d`
* `postgresql://`
* `PGPASSWORD=`
* `export PGPASSWORD`
* `psql -W`

Never:

* hardcode IP addresses
* hardcode `localhost`
* hardcode database names
* hardcode usernames
* hardcode passwords
* ask the user for PostgreSQL credentials
* manually assemble connection strings
* print `POSTGRES_PASSWORD`
* echo credentials to logs or chat

Even if environment variables exist, never generate commands such as:

```bash
psql -h "$PGHOST" \
     -p "$PGPORT" \
     -U "$POSTGRES_USER" \
     -d "$PGDATABASE"
```

The runtime already performs this binding.

---

## ONLY

Use

```bash
psql <database>
```

only when intentionally switching to another database on the **same PostgreSQL server**.

---

# Decision Rule

Need to execute SQL?

↓

Use

```bash
psql -c "<SQL>"
```

Need an interactive SQL shell?

↓

Use

```bash
psql
```

Need another database on the same PostgreSQL server?

↓

Use

```bash
psql <database>
```

Anything else?

↓

Do **not** construct a PostgreSQL connection.

---

# Preferred Patterns

| Intent            | Command               |
| ----------------- | --------------------- |
| Interactive shell | `psql`                |
| Execute SQL       | `psql -c "<SQL>"`     |
| Connectivity test | `psql -c "SELECT 1;"` |
| Switch database   | `psql <database>`     |

Examples:

```bash
psql

psql -c "SELECT 1;"

psql -c "SELECT id, login FROM res_users;"

psql another_database
```

---

# Anti-Patterns

The following commands violate the runtime contract.

## Manual host

```bash
psql -h db
psql -h localhost
psql -h 127.0.0.1
```

Reason:

Manual hosts bypass the runtime and may connect to the wrong database.

---

## Manual user

```bash
psql -U odoo
psql -U postgres
```

Reason:

The runtime already selects the correct user.

---

## Manual database

```bash
psql -d production
```

Reason:

The runtime already selects the active database.

---

## Manual password

```bash
PGPASSWORD=secret psql

export PGPASSWORD=...

psql -W
```

Reason:

Credentials are runtime-managed and must never be exposed.

---

## Connection strings

```bash
psql postgresql://user:pass@host/db
```

Reason:

Connection strings bypass the runtime contract.

---

# Investigation Priority

If PostgreSQL access fails:

1. Execute

```bash
psql -c "SELECT 1;"
```

2. Verify runtime environment.

3. Inspect:

```text
/home/odoo/logs/startup.log
```

4. Report infrastructure or environment problems.

Do **not** modify connection parameters during investigation.

---

# Failure Handling

If `psql` fails:

Do **NOT**

* add `-h`
* add `-U`
* add `-p`
* add `-d`
* ask for credentials
* construct a connection string

Instead verify:

```bash
echo "HOST=$PGHOST DB=$PGDATABASE USER=$POSTGRES_USER"

test -n "$POSTGRES_PASSWORD" \
&& echo "Password: [set]" \
|| echo "Password: [not set]"
```

Then inspect:

```bash
grep -iE "database|psycopg2|ERROR" \
/home/odoo/logs/startup.log | tail -20
```

If the environment is correct but `psql` still fails, treat the problem as an infrastructure or container configuration issue.

---

# Safety Classification

| Operation                     | Classification |
| ----------------------------- | -------------- |
| `psql -c "SELECT ..."`        | Safe           |
| `psql` (read-only inspection) | Safe           |
| INSERT / UPDATE / DELETE      | Caution        |
| DROP / TRUNCATE               | Dangerous      |
| Manual connection parameters  | Forbidden      |

---

# Verification

After any PostgreSQL operation:

```bash
psql -c "SELECT 1;"
```

Expected:

* Exit code: `0`
* One row returned
* No authentication errors

---

# Runtime Invariant

Autonomous agents must always assume:

> Inside an InstPloy container, PostgreSQL connectivity is owned by the runtime.

The correct solution is almost always:

```bash
psql
```

or

```bash
psql -c "<SQL>"
```

Never anything more.
