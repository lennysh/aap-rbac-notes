# AAP Backup, Restore & Migration Notes

How Ansible Automation Platform backs up and restores data across **RPM**, **containerized**, and **OpenShift operator** install methods — and why those tools are **not** interchangeable for cross-platform moves.

## Files

| File | Audience | Purpose |
|------|----------|---------|
| [AAP-BACKUP-RESTORE-OVERVIEW.md](AAP-BACKUP-RESTORE-OVERVIEW.md) | Humans | Full notes: what is dumped, what is excluded, UUID/secret rules, platform comparison |
| [AAP-MIGRATION-PATHS.md](AAP-MIGRATION-PATHS.md) | Humans | Operator `migrate_data` vs installer Gateway/Django “migrate” — commands and triggers |
| [AAP-BACKUP-RESTORE-AGENT-CONTEXT.md](AAP-BACKUP-RESTORE-AGENT-CONTEXT.md) | **AI agents** | Compact lookup for backup/restore and migration naming traps |

## Quick answers

| Question | Short answer |
|----------|--------------|
| Does restore restore every table? | **Yes (today)** — full `pg_dump --clean --create` per component DB. Older backups that excluded `main_instance*` tables are rejected. |
| Can I restore an RPM/containerized backup with an OCP Restore CR? | **No.** Operator Restore CRs only consume operator Backup CR / PVC artifacts. |
| How do I migrate RPM → OpenShift? | Use **`ansible.aap_snapshot`** (same AAP version), not installer `setup.sh -b/-r` or OCP Backup/Restore CRs. |
| Do installers have operator `migrate_data` (old PG → managed)? | **No.** Installer “migrate” = Gateway `migrate_service_data` or Django schema. See [AAP-MIGRATION-PATHS.md](AAP-MIGRATION-PATHS.md). |
| What must not be overridden on restore? | Component **`SECRET_KEY`s**, hub **`database_fields.symmetric.key`**, and other crypto keys that encrypt DB fields. Keep target host topology (`cluster_host_id`, DB/Redis endpoints, TLS). |

## Sources (local, gitignored)

Derived primarily from:

- [`.installer-dumps/`](../.installer-dumps/) — RPM + containerized installer collections
- [`.crd-dumps/`](../.crd-dumps/) — OpenShift operator Backup/Restore CRDs
- [`.operator-dumps/`](../.operator-dumps/) — live `/opt/ansible` from operator manager pods (2.4–2.7)
- Red Hat docs: [AAP 2.6 Migration](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.6/html-single/ansible_automation_platform_migration/index)

`aap_snapshot` is documented for AAP **2.6** (gateway-era / 2.5+ architecture); not a supported 2.4 path.

## Related folders

- [`installer/`](../installer/) — inventories and install playbook task refs
- [`openshift/`](../openshift/) — operator CR/secret examples

## Using with Cursor / other agents

`@`-mention `backup-restore/AAP-BACKUP-RESTORE-AGENT-CONTEXT.md` for backup/restore or migration questions.
