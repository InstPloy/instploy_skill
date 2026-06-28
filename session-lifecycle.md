# Session Lifecycle

**Scope:** InstPloy Odoo Container — canonical session management.

Parent: [jsonrpc.md](jsonrpc.md). Creation interface: `odoo_create_session.py` (internals out of scope).

---

## Runtime Contract

| Field | Value |
|-------|-------|
| Environment | InstPloy Odoo Container |
| Supported | Read cache → validate → reuse; create only when missing/invalid |
| Unsupported | New session per request; skip validation; ignore cache |
| Golden rule | Sessions are reusable runtime resources |
| Decision rule | Need auth → read cache first; create only if absent or invalid |
| Failure rule | Retry session creation exactly once; then report runtime/infrastructure issue |

Agents MUST treat Odoo sessions as **reusable runtime resources**.

Agents MUST NOT create a new session for every request.

Agents MUST first attempt to reuse an existing session.

---

## Session Cache

Deterministic cache location:

```
/home/odoo/.cache/instploy/session.json
```

Agents MUST read this file before calling `odoo_create_session.py`.

### Cache schema

| Field | Type | Purpose |
|-------|------|---------|
| `db` | string | Database name (`PGDATABASE`) |
| `user_id` | integer | Odoo user id |
| `login` | string | User login |
| `session_id` | string | Cookie value |
| `created_at` | string (ISO 8601 UTC) | Creation timestamp |
| `validated_at` | string (ISO 8601 UTC) | Last successful validation |

Example (never log `session_id` in chat):

```json
{
  "db": "db_example",
  "user_id": 2,
  "login": "admin",
  "session_id": "<redacted>",
  "created_at": "2026-06-28T12:00:00Z",
  "validated_at": "2026-06-28T12:05:00Z"
}
```

Ensure cache directory exists before write:

```bash
mkdir -p /home/odoo/.cache/instploy
```

---

## Session Reuse Policy

Before creating a session:

1. Read cached session from `/home/odoo/.cache/instploy/session.json`.
2. If no cached session exists → create via `odoo_create_session.py` → write cache.
3. If cached session exists → validate it.
4. If validation succeeds → reuse; update `validated_at`.
5. If validation fails → delete cache → create new session → replace cache.

Match cache `db` to current `PGDATABASE` and `user_id` to intended user. If mismatch → discard cache.

---

## Session Validation

Never assume a cached session is valid. Always validate before use.

Validation MUST be lightweight: minimal authenticated JSON-RPC `search_count`.

Preferred models (pick one):

| Model | Method | Args |
|-------|--------|------|
| `res.users` | `search_count` | `[[]]` |
| `ir.module.module` | `search_count` | `[[]]` |
| `res.company` | `search_count` | `[[]]` |

Requirements: requires authentication · returns quickly · minimal DB cost.

```bash
SID=$(python3 -c "import json; print(json.load(open('/home/odoo/.cache/instploy/session.json'))['session_id'])")
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/web/dataset/call_kw \
  -H "Content-Type: application/json" \
  -b "session_id=$SID" \
  -d '{"jsonrpc":"2.0","method":"call","params":{"model":"res.users","method":"search_count","args":[[]],"kwargs":{}}}'
```

| HTTP | Result |
|------|--------|
| 200 + valid JSON-RPC result | Reuse session; update `validated_at` |
| 401 / error / invalid JSON | Discard cache; create new session |

---

## Canonical Workflow

```
Need authenticated access?
        ↓
Read /home/odoo/.cache/instploy/session.json
        ↓
Session exists AND db/user_id match?
        ↓
   NO ──────────────────────────────┐
        ↓                          │
Run odoo_create_session.py <uid>   │
        ↓                          │
Write session.json                 │
        ↓                          │
Continue ◄─────────────────────────┘
        ↑
   YES ─┘
        ↓
Validate (search_count via call_kw)
        ↓
Valid?
   YES → Reuse → update validated_at → Continue
   NO  → Delete session.json
              ↓
         Run odoo_create_session.py
              ↓
         Write session.json
              ↓
         Continue
```

### Create session (exceptional path — expensive)

Only when cache missing, stale, or validation failed:

```bash
python3 /opt/instploy/instploysh/lib/odoo_create_session.py <user_id> 2>&1 | grep -o '{.*}'
```

Write result to `/home/odoo/.cache/instploy/session.json` with `created_at` and `validated_at`.

ORM calls after session resolved: [jsonrpc.md](jsonrpc.md).

---

## AI Rules

### Always

- Read cache before `odoo_create_session.py`.
- Reuse valid cached sessions.
- Validate before reuse.
- Update `validated_at` after successful validation.
- Use cached `session_id` for JSON-RPC `call_kw`.

### Never

- Generate a session per request.
- Regenerate without validating existing cache first.
- Ignore the session cache.
- Use generic Odoo login endpoints inside InstPloy (`/web/session/authenticate`, XML-RPC auth, password guess).
- Explain or depend on `odoo_create_session.py` internals.
- Repeatedly regenerate sessions in a loop.

---

## Failure Recovery

| Step | Action |
|------|--------|
| 1 | Validation fails → delete `/home/odoo/.cache/instploy/session.json` |
| 2 | Run `odoo_create_session.py` once → write cache |
| 3 | Validate new session |
| 4 | Second failure → stop; treat as Odoo runtime or infrastructure issue |

Do NOT retry session creation more than once per operation cycle.

Investigation: [jsonrpc.md#investigation-priority](jsonrpc.md#investigation-priority) · [troubleshooting.md#sessions-fail](troubleshooting.md#sessions-fail)

---

## Anti-Patterns

| Violation | Regenerate as |
|-----------|---------------|
| `odoo_create_session.py` on every ORM call | Read cache → validate → reuse |
| Skip validation of cached session | Validate with `search_count` first |
| New session without deleting invalid cache | Delete cache, then create once |
| Session loop (3+ creates) | Stop; investigate runtime |
| Cache in `/tmp` or random path | `/home/odoo/.cache/instploy/session.json` only |

---

## Verification

Normal path (reuse):

- Cache read succeeds
- Validation HTTP 200
- `search_count` returns integer
- No `odoo_create_session.py` invoked

Exceptional path (create):

- Cache written with all schema fields
- Validation succeeds after create
- JSON-RPC ORM call succeeds

---

## Goal

| Path | Frequency |
|------|-----------|
| Read cache → validate → reuse | **Normal** (every operation) |
| Generate new session | **Exceptional** (missing or invalid cache only) |

Session creation is a relatively expensive runtime operation. Minimize invocations of `odoo_create_session.py`.
