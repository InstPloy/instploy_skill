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

## CRITICAL: Persist On First Creation

The most common failure: creating a session but never writing it to the cache, so every run recreates "for the first time".

Rule: **A session that is created MUST be written to the cache in the same step.** Creating a session without immediately persisting it to `/home/odoo/.cache/instploy/session.json` is a runtime contract violation.

- Never print the session JSON to stdout and stop.
- Never pipe creation through `grep` for display only.
- Creation and cache write are a single atomic operation (see [Create Session](#create-session-exceptional-path--expensive)).
- After creating, the agent MUST confirm the cache file exists before continuing.

If the cache write is skipped, the contract is violated even if the ORM call succeeds.

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

Only when cache missing, stale, or validation failed. Create and persist in ONE atomic step:

```bash
mkdir -p /home/odoo/.cache/instploy
python3 /opt/instploy/instploysh/lib/odoo_create_session.py <user_id> 2>&1 | grep -o '{.*}' \
  | python3 -c "import sys,json,datetime,os; \
d=json.load(sys.stdin); \
assert 'session_id' in d, d.get('error','no session'); \
now=datetime.datetime.utcnow().isoformat()+'Z'; \
d['created_at']=d['validated_at']=now; \
p='/home/odoo/.cache/instploy/session.json'; tmp=p+'.tmp'; \
json.dump(d, open(tmp,'w')); os.replace(tmp,p); \
print('cache written:', p)"
```

Then confirm the cache exists before any ORM call:

```bash
test -s /home/odoo/.cache/instploy/session.json && echo "cache OK" || echo "CACHE MISSING — contract violation"
```

The pipeline writes atomically (`tmp` then `os.replace`) so a partial file is never left. Do not run the bare `odoo_create_session.py` without this persistence pipeline.

ORM calls after session resolved (read sid from cache): [jsonrpc.md](jsonrpc.md).

---

## AI Rules

### Always

- Read cache before `odoo_create_session.py`.
- On creation, write the cache in the SAME step (atomic create + persist).
- Confirm `/home/odoo/.cache/instploy/session.json` exists after creation.
- Reuse valid cached sessions.
- Validate before reuse.
- Update `validated_at` after successful validation.
- Use cached `session_id` for JSON-RPC `call_kw`.

### Never

- Create a session without immediately writing it to the cache (top violation).
- Print session JSON to stdout as the final step instead of persisting it.
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
| Create session, never write cache (recreates every run) | Atomic create + persist; confirm file exists |
| `... odoo_create_session.py <uid> 2>&1 \| grep -o '{.*}'` as final step | Pipe into the persistence command in [Create Session](#create-session-exceptional-path--expensive) |
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

- `test -s /home/odoo/.cache/instploy/session.json` succeeds (file written, non-empty)
- Cache contains all schema fields including `created_at` / `validated_at`
- Validation succeeds after create
- JSON-RPC ORM call succeeds
- Next operation reuses the cache (does NOT recreate)

---

## Goal

| Path | Frequency |
|------|-----------|
| Read cache → validate → reuse | **Normal** (every operation) |
| Generate new session | **Exceptional** (missing or invalid cache only) |

Session creation is a relatively expensive runtime operation. Minimize invocations of `odoo_create_session.py`.
