# AAP Backup / Restore — Agent Context

> **Purpose:** Fast facts for agents answering backup, restore, UUID/secret, or cross-platform migration questions.
>
> **Companion:** [AAP-BACKUP-RESTORE-OVERVIEW.md](AAP-BACKUP-RESTORE-OVERVIEW.md) (full detail + source paths)
>
> **Sources:** `.installer-dumps/` (RPM + containerized), `.crd-dumps/` (CRDs), `.operator-dumps/` (live operator `/opt/ansible` for 2.4–2.7), Red Hat AAP migration docs (`ansible.aap_snapshot`).

---

## Decision tree

1. **Same install method, disaster recovery?** → Use that method’s backup/restore (RPM `setup.sh -b/-r`, containerized playbooks, or OCP Backup/Restore CRs).
2. **Changing install method (RPM ↔ containerized ↔ OCP)?** → **Not** installer backup into OCP Restore CR (or reverse). Use **`ansible.aap_snapshot`** (same AAP version) or RH-documented manual import.
3. **User asks about “migrate” / `migrate_data`?** → Naming trap — see [AAP-MIGRATION-PATHS.md](AAP-MIGRATION-PATHS.md). Operator `migrate_data.yml` = Postgres dump\|restore cutover. Installer `migrate.yml` / `data_migration` = Gateway `migrate_service_data`. Installers have **no** operator-style old→managed PG pipe.
4. **User asks “does it restore every table?”** → **Yes today** — full `pg_dump --clean --create`. Older dumps that omitted `main_instance*` fail an assert; modern backups include those tables.

---

## Full DB dump (not selective tables)

```text
postgresql_db: target_opts: '--clean --create'   # all component DBs
```

RPM restore still greps the controller dump for `main_instance`, `main_instancegroup`, `main_instancegroup_instances` as a **compat check**. Comment in code: backups **used to** exclude those tables and re-inject on restore; **now** the full DB is dumped and nothing is injected.

---

## What must NOT be overridden on restore

| Keep from **backup** | Keep on **target** |
|----------------------|--------------------|
| Controller / EDA / Gateway `SECRET_KEY` | `CLUSTER_HOST_ID` / `cluster_host_id.py` |
| Hub `database_fields.symmetric.key` | DB connection / Redis / channels conf (RPM excludes these from conf restore) |
| Other crypto: resource_server secrets, channels secret, Lightspeed OAuth (containerized) | Destination TLS, receptor.conf, redis cluster ACL/nodes files |
| Full DB contents | Operator CR hostnames/routes (OCP) |

**Rule:** Restored DB + new `SECRET_KEY` = unreadable encrypted credentials.

Gateway `gateway.py` resource-server key (RPM): **not** restored from conf; rewritten from restored gateway DB in post_db_setup.

---

## Instance / UUID behavior

| Platform | Behavior |
|----------|----------|
| **RPM** | `ha.py` `SYSTEM_UUID` travels with host conf. After restore: deprovision orphans → `provision_instance` (control/hybrid use `system_uuid`; execution uses worker UUID; hop gets new UUID). |
| **Containerized** | Destination `_system_uuid` (hardware/machine_id). After DB restore, `provision_instance` with dest UUID; orphans deprovisioned. `receptor.conf` stays dest. |
| **OCP** | CRDs have no installer UUID remapping story; restore reconstitutes operator-managed deploy from Backup PVC/CR. |

`INSTALL_UUID` / `TOWER_UUID`: not specially handled in installer roles; if present, they live in the DB dump.

---

## Platform cheat sheet

### RPM

- Entry: `setup.sh -b` / `-r`
- Artifact: `automation-platform-backup-<ts>.tar.gz` (+ `…-latest` symlink)
- Conf excludes: `SECRET_KEY` (copied to `common/`), `postgres.py`, `channels.py`, `caching.py`, `cluster_host_id.py`, `gateway.py`
- Projects: backs up `/var/lib/awx/projects` minus SCM dirs `.*_<digits>__.*`
- Not in backup: Redis data, receptor work dirs
- Version lock: backup `VERSION` must match installer and installed platform

### Containerized

- Entry: `ansible.containerized_installer.backup` / `.restore`
- Artifact: `./backups/<component>_<inventory_hostname>.tar.gz`
- Secrets: extract selected names from Podman secrets JSON via `restore_secrets.yml` — do not blind-overwrite dest secrets dir
- Managed PG: filesystem volume archive **plus** logical dumps
- Inventory hostnames must match backup ↔ restore
- PCP tasks exist but are **not** in backup/restore playbooks

### OpenShift

- Platform Backup/Restore Ansible lives in the **gateway** operator (2.5+); components have their own backup/restore roles
- PVC layout: `*-openshift-backup-<ts>/{aap|tower}.db` (`pg_dump -F custom`), `*_object` CR YAML, `secrets.yml`
- Restore: Secrets from backup → scale down → `pg_restore` → redeploy CRs (platform also spawns child Restore CRs)
- `SYSTEM_UUID` = pod UID; no Instance UUID scrub after restore
- No path for RPM/containerized tarballs

---

## Cross-platform migration (RPM → OCP etc.)

```bash
# Export (example: RPM source)
ansible-playbook -i <inventory> ansible.aap_snapshot.artifact_export -e aap_platform=rpm
```

- Same AAP **version** only (e.g. 2.6 → 2.6).
- Artifact: `manifest.yml`, `secrets.yml`, per-component `.pgc` dumps, optional hub content tar — **not** the same as installer or operator backups.
- After import: re-register execution nodes, fix IdP redirects if hostnames changed, expect custom `conf.d` **not** applied on OCP.
- Docs: [Migration overview (2.6)](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html-single/ansible_automation_platform_migration/index)

**Wrong:** Restore CR pointing at `automation-platform-backup-*.tar.gz` or containerized `./backups/*`.

**Right for same-platform DR:** matching backup/restore tool.

**Right for cross-platform:** `aap_snapshot` (or RH manual procedure).

---

## How agents should answer

1. Identify **source** and **target** install methods.
2. If they differ → migration/`aap_snapshot`, explain why Restore CRs reject installer backups.
3. If same → summarize that method’s flow; stress full-DB dumps + SECRET_KEY preservation + topology kept on target.
4. Cite overview doc sections or dump paths when the user wants depth.
5. Prefer `@backup-restore/` over inventing steps from memory after installer async bumps — re-read dumps if versions moved.
