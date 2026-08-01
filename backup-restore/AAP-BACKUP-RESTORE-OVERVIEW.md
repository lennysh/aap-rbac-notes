# AAP Backup & Restore — Overview

> Derived from local installer dumps (RPM 2.4–2.6, containerized 2.5–2.7) and OpenShift operator CRD dumps (2.4–2.7), plus Red Hat migration docs. Re-check after async installer/operator updates.

---

## 1. Three incompatible backup ecosystems

| Method | How you invoke it | Artifact shape | Restored by |
|--------|-------------------|----------------|-------------|
| **RPM installer** | `setup.sh -b` / `-r` | One `automation-platform-backup-<ts>.tar.gz` | Same RPM installer restore playbook |
| **Containerized installer** | `ansible.containerized_installer.backup` / `.restore` | Per-host tarballs under `./backups/` | Same containerized restore playbook |
| **OpenShift operator** | `AnsibleAutomationPlatformBackup` / `Restore` CRs (2.5+), or per-component Backup/Restore CRs | PVC + directory recorded on Backup CR status | Matching operator Restore CR |

These are **same-platform disaster-recovery tools**, not migration tools. Pointing an OCP Restore CR at an RPM/containerized tarball (or vice versa) is outside what the APIs/playbooks accept.

**Cross-platform migration** (e.g. RPM → OpenShift) uses a separate path: the `ansible.aap_snapshot` collection. See [§7 Cross-platform migration](#7-cross-platform-migration).

---

## 2. Databases: full dump, not selective tables

### Current behavior (2.4+ RPM already; containerized same)

Every component database dump uses:

```text
pg_dump / postgresql_db  target_opts: '--clean --create'
```

There is **no** `--exclude-table` / include-table list in current backup roles. That means:

- Controller (`awx` / automationcontroller DB)
- Hub (`pulp`)
- EDA (`eda`)
- Gateway (`gateway`)
- Lightspeed / Metrics (containerized 2.6+)

…are dumped **entirely**, including instance/mesh-related tables.

### Historical note (why “instance tables” still come up)

RPM restore still asserts the controller dump contains:

- `main_instance`
- `main_instancegroup`
- `main_instancegroup_instances`

Comment in RPM `roles/restore/tasks/postgres.yml`:

> The backup role **used to exclude** tables in the pg_dump command, and those tables were replaced on the restore side. **Now we backup the full database**, and don't inject the tables during restore.

If an old backup is missing those `CREATE TABLE` lines, restore fails with a compatibility error. That is a version/format guard — not evidence that modern restores still skip those tables.

### After controller DB restore (RPM)

Running jobs are force-canceled:

```text
UnifiedJob.objects.filter(status='running').update(status='canceled', ...)
```

### Dual path for managed Postgres (containerized)

When a `database` inventory group exists:

1. Restore the **filesystem** Postgres data directory (`postgresql_<host>.tar.gz`)
2. Each component still applies its **logical** dump into `template1` (`--clean --create`)

External Postgres: only the logical dumps matter.

---

## 3. What is backed up vs not

### RPM installer (2.6)

**Included**

| Content | Notes |
|---------|--------|
| Full DB dumps | Controller, Hub, EDA, Gateway |
| Controller `/etc/tower/` (most of it) | Per controller host |
| `/etc/tower/SECRET_KEY` | Once, under `common/` |
| Manual project trees under `/var/lib/awx/projects/` | SCM sync dirs matching `.*_<digits>__.*` **excluded** |
| Hub `/var/lib/pulp/` | If `automationhub_backup_collections` |
| Hub `database_fields.symmetric.key`, GPG signing keys | |
| EDA + Gateway `SECRET_KEY` files | |
| Internal platform CA | If present |
| Platform `VERSION` / `upgrade_from` | Version lock on restore |

**Explicitly excluded from controller conf rsync**

```text
SECRET_KEY                    # copied separately to common/
conf.d/postgres.py            # keep target DB endpoints
conf.d/channels.py            # keep target Redis/channels
conf.d/caching.py
conf.d/cluster_host_id.py     # keep target CLUSTER_HOST_ID
conf.d/gateway.py             # resource-server key rewritten post-DB
```

**Not backed up**

- Redis data (stop/start + TLS re-sign on restore)
- Receptor work directories (mesh re-registered after restore)
- Live job output beyond what is already in Postgres

**Final archive**

```text
{setup_dir}/automation-platform-backup-{timestamp}.tar.gz
→ symlink automation-platform-backup-latest.tar.gz
```

Inner pieces: per-host controller archives, `common.tar.gz`, `postgres.tar.gz`, `ca.tar.gz`, `automationhub.tar.gz`, `automationedacontroller.tar.gz`, `automationgateway.tar.gz`, `VERSION`.

### Containerized installer (2.6)

**Included (per component, hostname-keyed tarball)**

| Archive | Typical contents |
|---------|------------------|
| `controller_<host>.tar.gz` | Podman secrets store, `awx.db`, projects, controller etc/nginx, TLS (archived but not applied), metrics data (2.6+) |
| `hub_<host>.tar.gz` | Secrets, `pulp` dump, hub tree + content volume |
| `eda_<host>.tar.gz` | Secrets, `eda` dump, eda tree (+ event-persistence DB in 2.7) |
| `gateway_<host>.tar.gz` | Secrets, `gateway` dump, gateway + gatewayproxy |
| `lightspeed_*` / `automationmetrics_*` / `mcp_*` | 2.6+ |
| `receptor_<host>.tar.gz` | Receptor data + TLS |
| `redis_<host>.tar.gz` | Redis data + TLS |
| `postgresql_<host>.tar.gz` | Managed PG data dir (if `database` group) |

**Universally kept on destination (excluded from unarchive)**

- Destination TLS / CA
- Blind restore of Podman secrets directory (selected secrets are re-applied by name instead)
- `cluster_host_id.py`, `settings.py`, `receptor.conf`, `redis_nodes.conf`, `redis-users.acl`, gateway `envoy.yaml`, etc.

**PCP** has backup/restore task files but is **not** wired into the backup/restore playbooks.

### OpenShift operator (from operator `/opt/ansible` dumps)

Verified from live manager pods under [`.operator-dumps/`](../.operator-dumps/) (gitignored).

**Where the Ansible lives**

| Kind | Operator container (2.5+) |
|------|---------------------------|
| `AnsibleAutomationPlatformBackup` / `Restore` | **Gateway** operator (`manager`) — roles `ansibleautomationplatformbackup` / `…restore` |
| `AutomationControllerBackup` / `Restore` | Controller operator |
| Hub / EDA / Metrics Backup/Restore | Matching component operators |

Platform backup (gateway) is an **orchestrator**: dumps gateway DB + secrets + `aap_object`, then creates child Backup CRs for controller/hub/eda/(metrics/mcp) that share the same PVC layout.

**PVC artifact layout (controller example; platform similar)**

```text
/backups/tower-openshift-backup-<timestamp>/   # controller
  tower.db          # pg_dump --clean --create -F custom  (full DB)
  awx_object        # AutomationController CR spec + secret name refs
  secrets.yml       # K8s Secret YAML (secretKeySecret, admin, postgres, receptor, TLS, …)

/backups/aap-openshift-backup-<timestamp>/     # platform (gateway)
  aap.db
  aap_object
  secrets.yml
  (+ child component backup dirs created by child Backup CRs)
```

**Restore order (controller)** — secrets → deploy CR → `pg_restore` (scale task/web to 0 first).  
**Restore order (platform)** — secrets → managed postgres if needed → restore gateway DB → recreate AAP CR / child Restore CRs → verify.

**UUIDs on OCP:** settings templates use `SYSTEM_UUID = MY_POD_UID` and `CLUSTER_HOST_ID = socket.gethostname()` (pod-local). Restore does **not** scrub Instance UUIDs from the DB; stale instances are an expected post-restore cleanup concern (`awx-manage deprovision_instance`), not an operator step.

Optional `pg_dump_suffix` can exclude table *data* (e.g. job events) — default is still a full dump. Restore locates artifacts only via Backup CR status (`backupClaim` / `backupDirectory`) or the same PVC layout — **not** installer tarballs.

---

## 4. Secrets, UUIDs, and what must NOT be overridden

Mental model:

> **Crypto identity comes from the backup. Topology identity stays on the target.**

### Must restore from backup (do not regenerate)

| Artifact | Why |
|----------|-----|
| Controller `SECRET_KEY` | Fernet/Django encryption of credentials & secrets in DB |
| Hub `database_fields.symmetric.key` | Pulp encrypted fields |
| EDA / Gateway `SECRET_KEY` | Same class of problem for those DBs |
| Containerized: `*_resource_server`, `controller_channels`, Lightspeed OAuth client secrets, etc. | Continuity with restored DB / gateway resource server |
| DB contents themselves | Full dumps include settings, users, orgs, instance rows, etc. |

If you restore the DB but leave a **new** install’s `SECRET_KEY`, encrypted credential blobs become unreadable.

### Must keep target / regenerate on target

| Artifact | Why |
|----------|-----|
| `CLUSTER_HOST_ID` (`cluster_host_id.py`) | Bound to **this** host’s identity; excluded from backup/restore of conf |
| `postgres.py` / Redis channels / caching conf (RPM) | Point at **this** install’s DB/Redis |
| TLS certs / CA trust on destination | Target install owns TLS |
| Receptor config / Redis cluster membership files (containerized) | Destination mesh topology |
| Operator-managed CR hostnames, routes, ingress (OCP) | Cluster-local |

### Instance / SYSTEM_UUID behavior

**RPM**

- `SYSTEM_UUID` lives in `/etc/tower/conf.d/ha.py` and **is** included in the host conf archive (not in the exclude list).
- After DB restore, restore playbook: receptor preflight → **deprovision** DB instances not in inventory → **re-`provision_instance`** for inventory nodes.
- UUID sources on re-register:
  - control/hybrid → `system_uuid` from `ha.py`
  - execution → ansible-runner worker UUID
  - hop → new `uuidgen`

**Containerized**

- `_system_uuid` fact comes from **destination** hardware (`ansible_product_uuid` / `machine_id`).
- After DB restore, `init.yml` runs `awx-manage provision_instance --hostname=… --uuid={{ _system_uuid }}`, then deprovisions orphans.
- Source instance rows that do not match inventory are cleaned; receptor identity stays destination via excluded `receptor.conf`.

**`INSTALL_UUID` / `TOWER_UUID`**

Not specially handled in installer backup/restore roles. If they exist as settings, they ride along in the **full DB dump**. No installer task regenerates them during restore.

### Gateway resource-server key (RPM)

`conf.d/gateway.py` is **not** restored from the conf archive. After DB restore, gateway resource keys are read from the restored gateway DB and rewritten into components (`replace_resource_key` / post_db_setup). Do not invent a new resource-server secret by hand after restore.

---

## 5. Restore flows (ordered)

### RPM (`playbooks/restore.yml` + `roles/restore`)

1. Preflight (gateway admin password, EDA/Redis checks, CA)
2. Grant temporary `CREATEDB` where needed; stop controller / hub / EDA / gateway / Redis
3. Upload & extract tarball; **assert backup VERSION matches installer + installed platform VERSION**
4. Restore projects, conf/secrets/CA, hub pulp content
5. Postgres restore (full dumps into `template1`)
6. `post_db_setup` — EDA initial data, gateway resource keys into components, hub credential refresh
7. Start controller → cleanup orphan instances → register instances/peers
8. Start Redis / hub / EDA / gateway; deregister/register gateway service nodes

### Containerized (`playbooks/restore.yml`)

1. Preflight; expect archives named for **destination** `inventory_hostname`
2. Restore managed PG volume (if any) → Redis → Receptor
3. Per component: unarchive (with excludes) → reapply selected Podman secrets from backup → recreate containers → logical DB restore → init
4. Inventory hostnames must match between backup and restore

### OpenShift

1. Deploy / ensure target Deployment / AAP CR(s) exist (`deployment_name`)
2. Create Restore CR referencing Backup CR (or PVC + dir from that backup’s status)
3. Operator: recreate Secrets from `secrets.yml` → scale down → `pg_restore` custom-format dump → redeploy CR(s)
4. Platform Restore (2.5+) also drives child component Restore CRs (controller/hub/eda/…)
5. Optional `force_drop_db`, `spec_overrides`, `cluster_name` for cross-cluster domain

---

## 6. Version deltas worth knowing

| Area | Notes |
|------|--------|
| RPM 2.4 | Still backs up SSO/Keycloak; no Gateway |
| RPM 2.5+ | Gateway DB + SECRET_KEY; SSO backup removed; DB file often `automationcontroller.db` (restore still accepts legacy `tower.db`) |
| Containerized 2.5 → 2.6 | Adds Lightspeed, MCP, Metrics; controller metrics data in archive |
| Containerized 2.7 | EDA event-persistence DB; MCP no longer restores Podman secrets from backup |
| OCP 2.4 | Component Backup/Restore only |
| OCP 2.5+ | Platform `AnsibleAutomationPlatformBackup` / `Restore` |
| OCP 2.6+ | MetricsService Backup/Restore; `use_db_compression` defaults true |

Same-version restore is assumed. RPM explicitly version-locks the backup `VERSION` file.

---

## 7. Install-time “migration” (not backup/restore)

Operator and installers both use the word *migrate* for different jobs. Full detail: [AAP-MIGRATION-PATHS.md](AAP-MIGRATION-PATHS.md).

| | Operator | RPM / containerized |
|---|----------|---------------------|
| Old/external DB → managed PG | **`migrate_data.yml`** (`pg_dump \| pg_restore` into managed pod) | **None** |
| Managed PG major upgrade | **`upgrade_postgres.yml`** | None automated |
| Django schema | `migrate_schema` / manage `migrate` | manage `migrate` during install |
| Gateway org/team sync | Separate from `migrate_data` | **`aap-gateway-manage migrate_service_data`** (`migrate.yml` / `data_migration.yml`) |

---

## 8. Cross-platform migration

### Why installer backup ≠ OCP restore

| Concern | Reality |
|---------|---------|
| Artifact format | RPM single tarball / containerized hostname tarballs ≠ operator PVC directory + Backup CR status |
| Secrets packaging | Files under `/etc/...` or Podman secrets JSON ≠ operator Secret objects named in Hub Backup status |
| Config model | File-based `conf.d` / Podman mounts ≠ operator CR specs |
| API surface | Restore CR only accepts `backup_source: CR \| PVC` tied to operator backups |
| Node identity | Receptor/host UUID reconciliation assumes installer inventory, not K8s pods |

So: **you cannot feed an RPM or containerized backup into an `AnsibleAutomationPlatformRestore` (or component Restore) CR** and expect it to work. That matches observed behavior.

### Official path: `ansible.aap_snapshot`

Red Hat documents migration between installation types at the **same AAP version** (e.g. RPM 2.6 → Operator 2.6, not 2.4 → 2.6).

For **RPM → OpenShift**, use the snapshot collection rather than `setup.sh -b` + Restore CR:

```bash
ansible-playbook -i <rpm-inventory> ansible.aap_snapshot.artifact_export -e aap_platform=rpm
# then import into a prepared OCP AAP deployment (see RH migration docs)
```

`aap_platform` accepts `rpm` | `containerized` | `operator`.

**Snapshot artifact (high level)**

| Piece | Role |
|-------|------|
| `manifest.yml` | Schema, AAP version, topology, checksums |
| `secrets.yml` | SECRET_KEYs, DB creds, encryption keys (`0600`) |
| `controller/controller.pgc` (etc.) | PostgreSQL **custom-format** dumps per component |
| `hub/hub_content.tar` | Pulp content (optional via `export_hub_content`) |

**Not automatically handled / needs reconcile**

- Execution node re-registration (DB rows may migrate; nodes without heartbeat get deprovisioned — re-register)
- Custom TLS
- Custom `/etc/tower/conf.d/` files (exported for RPM but **not** applied on OCP — operator manages config)
- System settings outside the DBs; host metrics/facts

Auth settings stored **in** the component DBs (LDAP/SAML/etc.) migrate with the dump; IdP redirect URIs may still need updates if hostnames change.

Docs:

- [AAP 2.6 Migration overview](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html-single/ansible_automation_platform_migration/index)
- [Export RPM via aap_snapshot](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/migrate-export_an_rpm_based_deployment_using_the_aap_snapshot_collection)

### Same-platform moves that *do* use backup/restore

| Scenario | Tool |
|----------|------|
| RPM → new RPM hosts (e.g. RHEL 8 → 9 mirror) | Installer backup/restore (docs call this out for 2.4-era full mirror restores; consulting often recommended) |
| Containerized → containerized (same inventory names) | Containerized backup/restore |
| OCP → OCP (same or new Deployment CR name) | Operator Backup → Restore CRs |
| RPM/containerized ↔ OCP | **`aap_snapshot`** (or documented manual DB+secret import), **not** Restore CR on installer tarball |

---

## 9. Key source pointers (local dumps)

### RPM 2.6

```text
.installer-dumps/2.6/ansible-automation-platform-setup-2.6-7/.../automation_platform_installer/
  playbooks/backup.yml
  playbooks/restore.yml
  roles/backup/tasks/{main,postgres,conf,projects,automationhub,download}.yml
  roles/restore/tasks/{main,postgres,conf,post_db_setup}.yml
  roles/receptor/tasks/{cleanup_restored_instances,register_instances}.yml
  roles/misc/templates/ha.py
```

### Containerized 2.6

```text
.installer-dumps/2.6/ansible-automation-platform-containerized-setup-2.6-10/.../containerized_installer/
  playbooks/backup.yml
  playbooks/restore.yml
  roles/*/tasks/backup.yml
  roles/*/tasks/restore.yml
  roles/common/tasks/restore_secrets.yml
  roles/postgresql/tasks/{backup,restore}.yml
```

### OpenShift CRDs

```text
.crd-dumps/{2.5,2.6,2.7}/aap.ansible.com_ansibleautomationplatform{backups,restores}.yaml
.crd-dumps/{2.4–2.7}/automationcontroller.ansible.com_automationcontroller{backups,restores}.yaml
…
```

### OpenShift operator Ansible (live `/opt/ansible` exports)

```text
.operator-dumps/{2.4–2.7}/{gateway,controller,hub,eda,metrics}/
  watches.yaml
  roles/backup|restore|…          # component operators
  roles/ansibleautomationplatformbackup|restore   # gateway / platform (2.5+)
  artifacts/full-export-info.txt  # pod, CSV, image digest
```

---

## 10. Practical checklist

**Same-platform DR**

1. Back up with the tool that matches how AAP was installed.
2. Restore onto a compatible install of the **same AAP version**.
3. Preserve backup `SECRET_KEY`s / encryption keys; do not “rotate” them as part of restore.
4. Expect instance/mesh cleanup and re-registration.
5. Do not expect Redis/receptor runtime state to come back bit-for-bit.

**Cross-platform (especially RPM → OCP)**

1. Do **not** use OCP Restore CRs against installer backups.
2. Use `ansible.aap_snapshot` at matching version (or follow RH manual import for other paths).
3. Stand up a fresh operator deploy first; import snapshot; replace/align secrets; reconcile nodes and gateway services.
4. Plan for hostname/IdP redirect and execution-node re-registration work.
