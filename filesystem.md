# Filesystem Runtime Contract

**Scope:** InstPloy Odoo Container — paths and addons.

---

## Runtime Contract

| Field | Value |
|-------|-------|
| Supported | Read known paths; deploy to `/home/odoo/src/user`; verify via `startup.log` |
| Unsupported | Manual `--addons-path`, rebuilding `odoo.conf`, relocating data dir |
| Golden rule | Runtime resolves addons-path; agent reads, does not reconstruct |
| Decision rule | Custom module → `/home/odoo/src/user`; verify `__manifest__.py` exists |
| Failure rule | Check `grep addons-path startup.log`; never hand-build addons-path |

---

## Key Paths

| Path | Purpose |
|------|---------|
| `/home/odoo/src/user` | Custom addons (deploy here) |
| `/home/odoo/src/odoo` | Odoo core |
| `/home/odoo/data/filestore/$PGDATABASE` | Attachments |
| `/home/odoo/data/sessions/` | HTTP sessions |
| `/home/odoo/.config/odoo/odoo.conf` | Odoo config (runtime-managed) |
| `/opt/instploy/instploysh/bin/` | Canonical CLI wrappers |
| `/opt/extra-packages` | Pip installs |

Full tree: see directory layout in prior reference — do not duplicate mount tables elsewhere.

---

## Addons-Path (runtime-owned)

Resolved by `odoo-bin` wrapper via `odoo_utils.get_odoo_args()`:

1. `ADDONS_PATH` env + core paths
2. Valid paths only (`__manifest__.py` present)
3. Nested discovery under `/home/odoo/src/user`
4. Enterprise/themes if `ENTERPRISE=true`

Verify (do not construct):

```bash
grep "Final addons-path" /home/odoo/logs/startup.log
find /home/odoo/src/user -name __manifest__.py | head -20
```

---

## Investigation Priority

1. `__manifest__.py` exists under `/home/odoo/src/user`
2. `grep addons-path startup.log`
3. `ENTERPRISE` env if enterprise module expected
4. `odoo-bin-wrapper.log`

---

## Anti-Patterns

| Violation | Regenerate as |
|-----------|---------------|
| `--addons-path=/custom/...` | Deploy to `/home/odoo/src/user`; restart |
| Edit `odoo.conf` addons manually | Let wrapper resolve; check startup.log |
| Pip to system site-packages | `pip3 install --target=/opt/extra-packages` |

---

## Verification

```bash
grep "Final addons-path" /home/odoo/logs/startup.log
find /home/odoo/src/user -name __manifest__.py -path "*<module>*"
```

Volume mounts: typical `./addons` → `/home/odoo/src/user` (platform-managed).
