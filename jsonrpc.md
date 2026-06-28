# JSON-RPC Runtime Contract

**Scope:** InstPloy Odoo Container — programmatic ORM access over HTTP.

---

## Runtime Contract

| Field | Value |
|-------|-------|
| Environment | InstPloy Odoo Container |
| Supported interface | `odoo_create_session.py` → `session_id` cookie → `/web/dataset/call_kw` |
| Unsupported | `/web/session/authenticate`, XML-RPC, password `/jsonrpc`, login pages |
| Golden rule | Sessions are passwordless; reuse cached session when valid |
| Decision rule | [session-lifecycle.md](session-lifecycle.md) first; then JSON-RPC call_kw |
| Failure rule | Never fall back to password auth; never create session without cache check |

---

## Inside vs Outside InstPloy

| Context | Session method | ORM method |
|---------|----------------|------------|
| **Inside container** | `odoo_create_session.py` (mandatory) | JSON-RPC `/web/dataset/call_kw` + cookie |
| **Outside container** | Out of scope — user must specify | Out of scope |

Inside InstPloy: agents MUST NOT use `/web/session/authenticate`, login pages, XML-RPC `authenticate()`, or password guessing.

---

## Session Resolution (mandatory first step)

Before any JSON-RPC call, follow [session-lifecycle.md](session-lifecycle.md):

1. Read `/home/odoo/.cache/instploy/session.json`
2. Validate cached `session_id` (or create if missing/invalid)
3. Reuse validated session for all `call_kw` requests in the same operation cycle

Do NOT call `odoo_create_session.py` until cache read and validation complete.

---

## Canonical Workflow (after session resolved)

```
psql -c "SELECT id, login FROM res_users WHERE active LIMIT 10;"
        ↓
[session-lifecycle.md] read cache → validate → reuse OR create once
        ↓
Cookie: session_id=<sid>
        ↓
POST /web/dataset/call_kw (JSON-RPC body)
```

### Step 1 — Resolve user id

```bash
psql -c "SELECT id, login FROM res_users WHERE active AND share=false ORDER BY id LIMIT 10;"
```

Never assume uid 2. Never guess `admin`/`admin`.

Never assume uid 2. Never guess `admin`/`admin`.

### Step 2 — Resolve session_id

See [session-lifecycle.md](session-lifecycle.md). Use cached `session_id` when valid.

Create only when cache missing or validation failed:

```bash
python3 /opt/instploy/instploysh/lib/odoo_create_session.py <uid> 2>&1 | grep -o '{.*}'
```

Script path: `/opt/instploy/instploysh/lib/odoo_create_session.py`

| Output field | Use |
|--------------|-----|
| `session_id` | HTTP cookie value; persist to cache |
| `login` | Reference only |
| `user_id` | Reference only |
| `db` | Reference only |

Log: `/home/odoo/logs/manager.log` (sid masked). Never print `session_id` in chat.

### Step 3 — Extract sid (if not from cache)

```bash
SID=$(python3 -c "import json; print(json.load(open('/home/odoo/.cache/instploy/session.json'))['session_id'])")
```

Or after create, write cache then read `session_id` from cache.

### Step 4 — JSON-RPC ORM call

Endpoint: `http://127.0.0.1/web/dataset/call_kw` (via nginx: port 80)

```bash
curl -s http://127.0.0.1/web/dataset/call_kw \
  -H "Content-Type: application/json" \
  -b "session_id=$SID" \
  -d '{"jsonrpc":"2.0","method":"call","params":{
        "model":"res.users","method":"search_read",
        "args":[[],["login","name"]],"kwargs":{"limit":5}}}' | python3 -m json.tool
```

Template:

```json
{"jsonrpc":"2.0","method":"call","params":{
  "model":"<model>","method":"<method>",
  "args":[<positional>],"kwargs":{<keyword>}}}
```

### Step 5 — Verify

```bash
curl -s -b "session_id=$SID" http://127.0.0.1/web/session/get_session_info | python3 -m json.tool
```

Expected: JSON with uid, username, db. HTTP 200.

---

## Investigation Priority

1. Read session cache `/home/odoo/.cache/instploy/session.json`
2. Validate cached session (search_count)
3. `psql -c` user exists in `res_users`
4. `odoo_create_session.py` only if cache invalid (once)
5. `manager.log` if session script fails
6. JSON-RPC response / HTTP status on `call_kw`
7. `odoo-error.log` / `odoo.log` if Odoo unreachable
8. Infrastructure (nginx, supervisor) if HTTP down

---

## Anti-Patterns

| Violation | Why | Regenerate as |
|-----------|-----|---------------|
| `POST /web/session/authenticate` + password | Password auth forbidden inside container | `odoo_create_session.py` |
| `xmlrpc.client.ServerProxy(...authenticate...)` | XML-RPC + password forbidden | JSON-RPC + cookie |
| `login='admin', password='admin'` | Credentials are unique per DB; defaults don't exist | `odoo_create_session.py` |
| JSON-RPC without cookie | Unauthenticated call_kw fails | Add `-b "session_id=$SID"` |
| External HTTPS URL from `web.base.url` for in-container calls | Use `127.0.0.1` inside container | `http://127.0.0.1/web/dataset/call_kw` |
| MCP postgres for ORM when session+JSON-RPC suffices | Wrong layer for Odoo ORM | JSON-RPC call_kw |

---

## Failure Handling

| Failure | Action |
|---------|--------|
| `{"error":"User with ID N not found"}` | Query `res_users`; pick valid uid |
| Empty JSON from session script | Read `manager.log`; check `psql -c "SELECT 1;"` |
| 401 on call_kw | Validate cache; recreate session once per [session-lifecycle.md](session-lifecycle.md) |
| Odoo unreachable | [troubleshooting.md](troubleshooting.md#odoo-wont-start) |

Do NOT fall back to password authentication.

---

## Verification

```bash
curl -s -b "session_id=$SID" http://127.0.0.1/web/session/get_session_info | python3 -m json.tool
curl -s http://127.0.0.1/web/dataset/call_kw \
  -H "Content-Type: application/json" -b "session_id=$SID" \
  -d '{"jsonrpc":"2.0","method":"call","params":{"model":"res.users","method":"search_count","args":[[]],"kwargs":{}}}'
```

Expected: session info populated; search_count returns integer.

Playbook: [playbooks.md#create-session](playbooks.md#create-session) · Lifecycle: [session-lifecycle.md](session-lifecycle.md)
