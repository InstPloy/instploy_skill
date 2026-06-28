# Nginx Runtime Contract

**Scope:** InstPloy Odoo Container — HTTP routing.

---

## Runtime Contract

| Field | Value |
|-------|-------|
| Supported | Health check `curl /health`; internal checks on `127.0.0.1` |
| Unsupported | Manual upstream edits, `nginx -s` as non-root workaround |
| Golden rule | Upstream swaps only via `instploysh-restart http` |
| Decision rule | HTTP issue → check odoo first, then nginx logs |
| Failure rule | Never edit nginx config manually unless explicit user request |

Config: `/etc/nginx/conf.d/default.conf`

---

## Routes

| Route | Upstream | Port |
|-------|----------|------|
| `/` | odoo | 8069 |
| `/web/dataset/call_kw` | odoo | 8069 (JSON-RPC) |
| `/longpolling`, `/websocket` | odoo-chat | 8072 |
| `/instploy/editor/` | jupyterlab | 8888 |
| `/instploy/webshell/` | jupyter-webshell | 8889 |
| `/instploy/manager/` | manager-api | 5001 (platform auth) |
| `/hermes/` | hermes-agent | 8642 (platform auth) |
| `/health` | direct | 200 `healthy` |

JSON-RPC: [jsonrpc.md](jsonrpc.md). Forbidden on ORM: XML-RPC, `/web/session/authenticate`.

---

## Canonical Health Checks

```bash
curl -s http://127.0.0.1/health
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:9069/web/login
```

Inside container: use `127.0.0.1`, not external `web.base.url`.

---

## Investigation Priority

1. `curl 127.0.0.1:8069/web/login`
2. `/home/odoo/logs/supervisor/odoo-error.log`
3. `/home/odoo/logs/supervisor/nginx-error.log`
4. `supervisorctl status nginx`
5. Infrastructure / SSL (external layer)

---

## Anti-Patterns

| Violation | Regenerate as |
|-----------|---------------|
| Manual nginx upstream edit for restart | `instploysh-restart http` |
| External URL for in-container JSON-RPC | `http://127.0.0.1/web/dataset/call_kw` |
| Bypass nginx to hit odoo on public domain from inside | `127.0.0.1:8069` |

---

## Verification

```bash
curl -s http://127.0.0.1/health
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login
```

Supervisor reload: `supervisorctl signal HUP nginx` ([services.md](services.md))
