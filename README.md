# instploy Odoo Container Skill

Production-grade operational knowledge base for AI agents inside instploy Odoo Docker containers.

## Install

### Cursor

```bash
cp -r instploy_skill ~/.cursor/skills/instploy-odoo-container
```

### skills.sh

```bash
npx skills add <owner>/instploy_skill --skill instploy-odoo-container
```

## Usage

```
@instploy-odoo-container install module sale and verify
```

## Architecture

| File | Role |
|------|------|
| `SKILL.md` | Entry: AI rules, decision trees, verification, index |
| `environment.md` | Canonical env vars |
| `filesystem.md` | Canonical paths, addons-path resolution |
| `services.md` | Supervisor programs, ports |
| `networking.md` | Nginx routes, upstreams |
| `logs.md` | Canonical log catalog |
| `postgresql.md` | DB access |
| `odoo-runtime.md` | Shell, sessions, module ops |
| `commands.md` | Command registry with safety classes |
| `playbooks.md` | Deterministic execution sequences |
| `troubleshooting.md` | Symptom-based decision trees |

Design: define once in canonical files; reference elsewhere. No duplication.

## Triggers

- Operating inside instploy Odoo containers
- Installing/upgrading Odoo modules
- Reading logs, obtaining credentials/sessions
- Restarting Odoo (blue/green)
- Debugging startup, DB, module, or HTTP issues

## Author

Rightechs Solutions — [https://www.rightechs.net/](https://www.rightechs.net/)
