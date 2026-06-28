# Odoo Runtime

Canonical Odoo shell, session, RPC, and module operation reference.

## Odoo shell

```bash
odoo-bin shell
```

| Attribute | Value |
|-----------|-------|
| Safety | Caution (ORM writes possible) |
| Access | Superuser by default |
| Commit | `env.cr.commit()` required for persistence |

```python
env['res.users'].search_read([('active', '=', True)], ['login', 'name'])
env['res.partner'].search_read([], ['name', 'email'], limit=5)
env.cr.commit()
```

Warning: uncommitted changes in open shell can block the database.

## Programmatic / RPC access (CANONICAL)

Rule: obtain access **passwordless** via `odoo_create_session.py`. The script reads the DB registry and computes the session token directly — no login password required.

Forbidden:
- Never guess passwords (`admin`/`admin`). Each database has unique platform-generated credentials; defaults do not exist.
- Never use XML-RPC. Use JSON-RPC web endpoints with the session cookie.
- Never call `/web/session/authenticate`, `/xmlrpc/2/common`, or `/jsonrpc` with a password for ORM access.

### Step 1 — Create session (passwordless)

Script: `/opt/instploy/instploysh/lib/odoo_create_session.py`

```bash
python3 /opt/instploy/instploysh/lib/odoo_create_session.py <user_id> 2>&1 | grep -o '{.*}'
```

| Attribute | Value |
|-----------|-------|
| Safety | Caution (creates real auth session) |
| Auth | None required (DB registry + token compute) |
| Output | JSON: `session_id`, `login`, `user_id`, `db` |
| Log | `/home/odoo/logs/manager.log` (sid masked) |
| Cookie | Use `session_id` value as HTTP cookie |
| Scope | Run inside container; sid valid for that Odoo instance |

Success: `{"session_id":"<sid>","login":"admin","user_id":2,"db":"<db>"}`
Error: `{"error":"User with ID 99 not found"}`

Find a valid user id first: see [postgresql.md](postgresql.md#user-lookup). Do not assume uid 2 exists or is desired.

### Step 2 — Extract sid

```bash
SID=$(python3 /opt/instploy/instploysh/lib/odoo_create_session.py <uid> 2>&1 | grep -o '{.*}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['session_id'])")
```

### Step 3 — JSON-RPC ORM call (session cookie, no password)

Use the web JSON endpoint `/web/dataset/call_kw` with the `session_id` cookie:

```bash
curl -s http://127.0.0.1/web/dataset/call_kw \
  -H "Content-Type: application/json" \
  -b "session_id=$SID" \
  -d '{"jsonrpc":"2.0","method":"call","params":{
        "model":"res.users","method":"search_read",
        "args":[[],["login","name"]],"kwargs":{"limit":5}}}' | python3 -m json.tool
```

Generic ORM template:

```json
{"jsonrpc":"2.0","method":"call","params":{
  "model":"<model>","method":"<method>",
  "args":[<positional>],"kwargs":{<keyword>}}}
```

### Verify session

```bash
curl -s -b "session_id=$SID" http://127.0.0.1/web/session/get_session_info | python3 -m json.tool
```

### Internal RPC server

`/opt/instploy/instploysh/bin/odoo-rpc` — supervisor-managed, dispatches as superuser. Do not run manually unless debugging.

Playbook: [playbooks.md](playbooks.md#create-session)

## Module install

No `odoo-install` wrapper. Use `odoo-bin -i`:

```bash
odoo-bin -i <module> --stop-after-init --logfile=
odoo-bin -i sale,account,stock --stop-after-init --logfile=
```

| Flag | Purpose |
|------|---------|
| `--stop-after-init` | Exit after operation (required) |
| `--logfile=` | Output to stdout for live monitoring |

Refresh module catalog if module missing:

```bash
odoo-bin -u base --stop-after-init --logfile=
```

Playbook: [playbooks.md](playbooks.md#install-module)

## Module upgrade

```bash
odoo-update <module>
odoo-update sale,account
odoo-update all    # caution: expensive
```

Equivalent: `odoo-bin -u <module> --stop-after-init --logfile=`

After custom code deploy: `odoo-update <mod>` then `instploysh-restart http`.

Playbook: [playbooks.md](playbooks.md#upgrade-module)

## Jupyter Odoo kernel

| Item | Value |
|------|------|
| Script | `/opt/instploy/instploysh/bin/odoo-kernel` |
| Env | `ODOO_KERNEL=1` |
| Namespace | `env`, `odoo`, `self` (admin) |
| Route | `/instploy/editor/` |
| Managed by | JupyterLab supervisor program |

## Database import

```bash
instploysh-import-database <backup.zip>
instploysh-import-database --logfile=/home/odoo/logs/import.log <backup.zip>
```

| Attribute | Value |
|-----------|-------|
| Safety | Destructive (erases current DB) |
| Formats | Odoo `.zip` (dump.sql + filestore) or instploy plain (.json + .sql.gz) |
| Filestore target | `/home/odoo/data/filestore/<PGDATABASE>` |

Playbook: [playbooks.md](playbooks.md#import-database)

## odoo_utils.py (library)

Not run directly. Used by wrappers:
- `get_odoo_args()` — build CLI args from env
- `get_obfuscate_args()` — obfuscate args
- `database_exists()` — test DB connectivity

## Access decision

```
SQL only          → psql ([postgresql.md](postgresql.md))
ORM (in-process)  → odoo-bin shell
ORM (RPC)         → odoo_create_session.py + JSON-RPC /web/dataset/call_kw
Browser UI        → session_id cookie
Notebook          → /instploy/editor/
```

Never: XML-RPC, or any password-based authentication.
