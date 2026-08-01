# AAP Migration Paths — Operator vs Installers

> What “migration” means in each install method. Names collide; behaviors do not.
>
> Sources: [`.operator-dumps/`](../.operator-dumps/), [`.installer-dumps/`](../.installer-dumps/). Companion: [AAP-BACKUP-RESTORE-OVERVIEW.md](AAP-BACKUP-RESTORE-OVERVIEW.md).

---

## Quick map (do not confuse these)

| Name in code | Where | What it actually does |
|--------------|-------|------------------------|
| **`migrate_data.yml`** | Operator (controller/hub/eda) | **Postgres cutover:** `pg_dump \| pg_restore` from old DB secret → managed PG pod |
| **`upgrade_postgres.yml`** | Operator | **Managed PG major upgrade** (e.g. 13 → 15) via dump\|restore between STS |
| **`migrate_schema.yml`** / `*-manage migrate` | Operator + installers | **Django schema** migrations |
| **`migrate_service_data`** / installer `migrate.yml` / `data_migration.yml` | RPM + containerized (+ Gateway) | **Org/team/user merge** from controller/hub/eda into Platform Gateway |
| **`post_migration_setup.yml`** | RPM 2.6 | **2.4→2.x auth import** to Gateway (`import_auth_config_to_gateway`) |
| Backup/restore `pg_dump` | All methods | DR tooling — not install-time DB cutover |

**Bottom line:** Only the **OpenShift operator** has an automated external/old → managed Postgres pipe. RPM and containerized installers do **not** — they use Django migrate + Gateway service-data merge, and expect you to point at the right DB (or use backup/restore / `aap_snapshot` for moves).

---

## 1. OpenShift operator — Postgres data migration

### 1.1 External / old DB → managed Postgres (`migrate_data.yml`)

Present on **Controller** (2.4–2.7), **Hub** (2.4–2.7), **EDA** (2.5–2.7). **Gateway has no `migrate_data.yml`.**

#### When it runs (controller)

Triggered from `database.yml` / `database_configuration.yml` when:

1. Secret `old_postgres_configuration_secret` (or default `{name}-old-postgres-configuration`) exists
2. Status field `migratedFromSecret` is **not** already set

Hub uses `postgres_migrant_configuration_secret` and status `migrantDatabaseConfigurationSecret`.  
EDA uses `old_postgres_configuration_secret` and then nulls that CR field after success (`cleanup_migration_references.yml`).

#### Ordered flow (controller — canonical)

1. Decode old secret → host / user / password / database / port  
2. Find managed Postgres pod (`app.kubernetes.io/instance=postgres-{{ supported_pg_version }}-{{ name }}`)  
3. Scale down controller task/web Deployments  
4. `k8s_exec` into the **managed** PG pod and run the pipe below  
5. Set `status.migratedFromSecret` (idempotency)

#### Commands (controller 2.6)

```text
# Built as facts, then executed inside the managed postgres pod:
pg_dump
  -h <old_host> -U <old_user> -d <old_db> -p <old_port>
  -F custom
  {{ pg_dump_suffix }}

pg_restore --clean --if-exists
  -U <database_username> -d <database_name>
  # no -h → restore targets localhost inside the managed pod
```

Pipe (exec’d on managed pod):

```bash
psql -c 'GRANT postgres TO <awx_postgres_user>'
PGPASSWORD="$PGPASSWORD_OLD" pg_dump … | PGPASSWORD="$POSTGRES_PASSWORD" pg_restore …
psql -c 'REVOKE postgres FROM <awx_postgres_user>'
```

- `PGPASSWORD_OLD` is injected on the managed STS from the **old** secret when present  
- Keepalive loop prints `Migrating data from old database...` every 60s  
- Success when stdout contains `Successful`

#### Hub / EDA differences

| | Hub | EDA (2.5+) |
|---|-----|------------|
| Trigger secret | `postgres_migrant_configuration_secret` | `old_postgres_configuration_secret` |
| GRANT/REVOKE dance | No | Yes (like controller) |
| Post-success | Status `migrantDatabaseConfigurationSecret` | Nulls CR `spec.old_postgres_configuration_secret` |
| Dump format | `-F custom` | `-F custom` (+ `pg_dump_suffix`) |

#### Important limitations

- Restore side is **hard-wired to the managed pod** (tools + `POSTGRES_PASSWORD` live there). This is **old/external → managed**, not a general external→external pipe.
- Does **not** scrub Instance UUIDs after restore (see overview notes).
- Not the same as Backup/Restore CRs (those write/read a PVC layout).

### 1.2 Managed Postgres major upgrade (`upgrade_postgres.yml`)

When the CR uses **managed** DB and an older managed pod still exists with `PG_VERSION < supported_pg_version` (15 across these dumps):

1. Recreate postgres configuration secret with new host `{name}-postgres-15`  
2. Create new STS; wait ready  
3. Dump from old in-cluster SVC → restore into new pod (same dump\|restore pattern; often **no** `--clean` on restore)  
4. Delete old STS/SVC (`{name}-postgres-13`, etc.)  
5. Set `status.upgradedPostgresVersion`

### 1.3 Django schema migrate (operator)

| Version | How |
|---------|-----|
| Controller 2.4 | `awx-manage migrate --noinput` via `k8s_exec` on task pod |
| Controller 2.5+ | Job from `migrate_schema.yml` / `jobs/migration.yaml.j2` if `showmigrations` shows pending |

Hub/EDA/Gateway also run their manage `migrate` during normal reconcile/install — schema only.

---

## 2. RPM installer — what “migrate” means

**No `old_postgres` / no `migrate_data.yml` Postgres cutover** in the automation_platform_installer dumps (2.4–2.6).

### 2.1 Django schema (install)

| Component | Command | Typical task |
|-----------|---------|--------------|
| Controller | `awx-manage migrate --noinput` | controller tasks |
| Gateway (2.5+) | `aap-gateway-manage migrate` | gateway `main.yml` |
| EDA (2.5+) | `aap-eda-manage migrate` | provision_api |
| Hub | `pulpcore-manager migrate --no-input` | pulp_database_config |

### 2.2 Gateway service-data merge (`migrate_service_data`)

Brings organizations/teams/users (and related service records) from controller / hub / eda into the Platform Gateway DB.

**RPM 2.5** — per-component file `roles/automationgateway/tasks/data_migration.yml`:

```text
aap-gateway-manage migrate_service_data \
  --username <gateway_admin> \
  --merge-organizations true \
  --api-slug <controller|galaxy|eda>
```

Called from each component’s `post_install_setup` (skipped on restore when `_migrate_data: false`).

**RPM 2.6** — single call in `roles/automationgateway/tasks/post_install_setup.yml`:

```text
aap-gateway-manage migrate_service_data --username <gateway_admin>
```

Retries until `rc == 0`; `changed_when` if stdout matches `Items remaining: [^0]$`.

### 2.3 2.4 → 2.x auth import (RPM 2.6 only)

`roles/automationcontroller/tasks/post_migration_setup.yml` — only if marker `/var/lib/awx/is_aap_24` exists:

```text
awx-manage import_auth_config_to_gateway
# then remove marker
```

Marker is created during package install when upgrading from controller 2.4. Related flags: `_aap24_upgrade` in preflight/config for SSO/resource-management settings during upgrade. Full SSO/Keycloak **role** exists only in **RPM 2.4** dumps.

### 2.4 Postgres moves on RPM

Point inventory at the desired (external or installer-managed) DB, or use **backup/restore** playbooks. There is no operator-style automated dump\|restore cutover during `setup.sh` install.

---

## 3. Containerized installer — what “migrate” means

**No `old_postgres` / no operator-like DB cutover** (2.5–2.7 dumps).

### 3.1 Django schema

Each component `tasks/init.yml` runs manage `migrate --noinput` inside an init container (controller, gateway, eda, hub; lightspeed 2.6+).

### 3.2 Gateway service-data merge

**Containerized 2.5** — `roles/automationgateway/tasks/migrate.yml` waits for gateway proxy HTTP 200, then `data_migration.yml` per component via podman:

```text
aap-gateway-manage migrate_service_data \
  --username=<gateway_admin> \
  --merge-organizations=true \
  --api-slug=<controller|eda|galaxy>
```

**Containerized 2.6 / 2.7** — `migrate.yml` only (no separate `data_migration.yml`):

```text
# podman run … automation-gateway-init
aap-gateway-manage migrate_service_data --username=<gateway_admin>
```

Wired from `playbooks/install.yml` play “Migrate component resources” after components are up.

### 3.3 Postgres moves on containerized

Logical dumps + filesystem PG volume appear in **backup/restore** playbooks only. Install expects inventory DB settings; no `pg_dump|pg_restore` cutover role during install. PG container recreate on image change is not a major-version `pg_upgrade` path.

---

## 4. Side-by-side comparison

| Capability | Operator | RPM | Containerized |
|------------|----------|-----|---------------|
| Old/external DB → managed PG pipe | **Yes** (`migrate_data.yml`) | No | No |
| Managed PG major version upgrade | **Yes** (`upgrade_postgres.yml`) | No automated equivalent | No automated equivalent |
| Django schema migrate | Yes | Yes | Yes |
| Gateway `migrate_service_data` | Via gateway reconcile / separate from `migrate_data` | Yes (2.5+) | Yes (2.5+) |
| 2.4 auth → Gateway import | N/A (platform CR era) | Yes (2.6 marker) | No |
| Cross-platform (RPM→OCP) | N/A | Use **`aap_snapshot`**, not these tasks | Same |

---

## 5. Practical guidance

1. **Moving Postgres into operator-managed DB on OCP**  
   Use `old_postgres_configuration_secret` / hub migrant secret and let reconcile run `migrate_data.yml`. Scale-down and dump\|restore happen for you.

2. **Moving Postgres on RPM/containerized**  
   There is no install-time pipe. Options: repoint to external DB with a fresh/restored dump, use installer **backup/restore**, or (cross-platform) **`ansible.aap_snapshot`**.

3. **Seeing `migrate.yml` / `data_migration` in installers**  
   That is **Gateway service sync**, not a database host migration. Safe to re-run conceptually (retries until “Items remaining: 0”), but it is not `pg_dump`.

4. **Seeing `migrate_data` in operator logs**  
   That *is* the Postgres cutover. Check CR status `migratedFromSecret` / hub migrant status before assuming it will run again.

---

## 6. Source pointers

```text
# Operator (example 2.6)
.operator-dumps/2.6/controller/roles/installer/tasks/migrate_data.yml
.operator-dumps/2.6/controller/roles/installer/tasks/upgrade_postgres.yml
.operator-dumps/2.6/controller/roles/installer/tasks/migrate_schema.yml
.operator-dumps/2.6/hub/roles/postgres/tasks/migrate_data.yml
.operator-dumps/2.6/eda/roles/eda/tasks/migrate_data.yml
.operator-dumps/2.6/eda/roles/eda/tasks/cleanup_migration_references.yml

# RPM 2.6
.installer-dumps/2.6/.../automationgateway/tasks/post_install_setup.yml   # migrate_service_data
.installer-dumps/2.6/.../automationcontroller/tasks/post_migration_setup.yml

# Containerized 2.6
.installer-dumps/2.6/.../automationgateway/tasks/migrate.yml
```
