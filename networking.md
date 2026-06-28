# Networking

Canonical Nginx routing reference.

## Public routes (nginx :80)

| Route | Upstream | Internal port | Notes |
|-------|----------|---------------|-------|
| `/` | `odoo` | 8069 | Main Odoo HTTP |
| `/web/login` | `odoo` | 8069 | Login page |
| `/longpolling` | `odoo-chat` | 8072 | Bus notifications |
| `/websocket`, `/web/websocket`, `/im_livechat/websocket` | `odoo-chat` | 8072 | Live chat / bus |
| `/instploy/editor/` | `jupyterlab` | 8888 | JupyterLab + Odoo kernel |
| `/instploy/webshell/` | `jupyter-webshell` | 8889 | Browser terminal |
| `/instploy/manager/` | `manager-api` | 5001 | Platform-managed auth |
| `/hermes/` | `hermes-agent` | 8642 | Platform-managed auth |
| `/health` | nginx direct | — | Returns `200 healthy` |
| `/web/database/manager` | `odoo` | 8069 | Disabled in prod via `--no-database-list` |
| `/web/dataset/call_kw` | `odoo` | 8069 | JSON-RPC ORM (use with `session_id` cookie) |

## ORM over HTTP

Canonical: passwordless session + JSON-RPC `/web/dataset/call_kw`. See [odoo-runtime.md](odoo-runtime.md#programmatic--rpc-access-canonical).

Forbidden: XML-RPC (`/xmlrpc/2/*`), `/web/session/authenticate` with password, password-based `/jsonrpc`.

Config: `/etc/nginx/conf.d/default.conf`, `/etc/nginx/nginx.conf`

## Upstreams

| Upstream | Target |
|----------|--------|
| `odoo` | `127.0.0.1:8069` |
| `odoo-chat` | `127.0.0.1:8072` (fallback `8069`) |
| `jupyterlab` | `127.0.0.1:8888` |
| `jupyter-webshell` | `127.0.0.1:8889` |
| `manager-api` | `127.0.0.1:5001` |
| `hermes-agent` | `127.0.0.1:8642` |

## Blue/green upstream swap

`instploysh-restart http` rewrites `proxy_pass`/`server` lines in `/etc/nginx/conf.d/default.conf`:
- Temporarily routes 8069→9069, 8072→9072 (to odooB)
- Restarts odooA
- Routes back to 8069/8072
- Validates with `nginx -t` before reload; rolls back on failure
- Reload via `supervisorctl signal HUP nginx`

## Health checks

```bash
curl -s http://127.0.0.1/health                                    # expect: healthy
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8069/web/login  # odooA
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:9069/web/login  # odooB
```

## Out of scope

`/instploy/manager/` and `/hermes/` exist and are proxied. Authentication is platform-managed — do not document or bypass.
