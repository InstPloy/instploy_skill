# InstPloy Odoo Container — Runtime Specification

Production-grade runtime specification for autonomous agents (Claude Code, Cursor, Codex, Gemini, Hermes, MCP).

**Invariant:** Canonical InstPloy wrappers only. Generic implementations are runtime contract violations.

## Install

```bash
cp -r instploy_skill ~/.cursor/skills/instploy-odoo-container
# or
npx skills add https://github.com/InstPloy/instploy_skill --skill instploy-odoo-container
```

## Architecture

| File | Role |
|------|------|
| `SKILL.md` | Entry: activation, philosophy, canonical APIs, violations, trees |
| `postgresql.md` | PostgreSQL runtime contract |
| `jsonrpc.md` | JSON-RPC ORM contract |
| `session-lifecycle.md` | Session cache, reuse, validation |
| `odoo-runtime.md` | Odoo shell, modules, import |
| `services.md` | Supervisor runtime contract |
| `networking.md` | Nginx runtime contract |
| `filesystem.md` | Paths, addons runtime contract |
| `logs.md` | Logs runtime contract |
| `environment.md` | Env boundaries (read-only reference) |
| `commands.md` | Command registry |
| `playbooks.md` | Execution playbooks with verification |
| `troubleshooting.md` | Symptom decision trees |

## Activation Triggers

Odoo · InstPloy · PostgreSQL · Supervisor · JSON-RPC · session_id · odoo_create_session · addons · module · restart · nginx · database · Docker container

Overrides generic Docker, PostgreSQL, Linux, Python, and Odoo knowledge.

## Author

instploy — [https://instploy.com](https://instploy.com)
