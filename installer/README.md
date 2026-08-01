# AAP installer reference

Annotated **RPM** and **containerized** installer inventories, optional `vars-example.yml` files, and generated **install playbook task** references. These are learning/reference configs — not production templates.

OpenShift operator examples live in the sibling [`openshift/`](../openshift/) folder (operator CRs, not install playbooks).

Backup/restore behavior (what is dumped, secrets/UUIDs, cross-platform limits): [`backup-restore/`](../backup-restore/).

Local Red Hat installer tarballs (gitignored): [`.installer-dumps/`](../.installer-dumps/) at repo root.

## Layout

Each version and method is one folder with everything co-located:

```
installer/
├── maintainer/          # how to refresh inventories after upstream updates
├── scripts/             # generators and audit
└── AAP{XY}/
    ├── rpm/             # 2.4–2.6 only (no RPM for 2.7)
    └── containerized/   # 2.5–2.7
        ├── inventory-example
        ├── inventory-growth-example   # containerized only
        ├── vars-example.yml
        ├── install-playbook-tasks.md  # generated from install.yml
        └── README.md
```

## Version index

| AAP | RPM | Containerized | OpenShift |
|-----|-----|---------------|-----------|
| 2.4 | [AAP24/rpm](AAP24/rpm/README.md) | — | [openshift/AAP24](../openshift/AAP24/README.md) |
| 2.5 | [AAP25/rpm](AAP25/rpm/README.md) | [AAP25/containerized](AAP25/containerized/README.md) | [openshift/AAP25](../openshift/AAP25/README.md) |
| 2.6 | [AAP26/rpm](AAP26/rpm/README.md) | [AAP26/containerized](AAP26/containerized/README.md) | [openshift/AAP26](../openshift/AAP26/README.md) |
| 2.7 | — | [AAP27/containerized](AAP27/containerized/README.md) | [openshift/AAP27](../openshift/AAP27/README.md) |

## Scripts

Run from repo root or `installer/`:

| Script | Purpose |
|--------|---------|
| [`scripts/build_inventory.py`](scripts/build_inventory.py) | Regenerate containerized inventories (`--version 2.5\|2.6\|2.7\|all`) |
| [`scripts/build_rpm_inventory.py`](scripts/build_rpm_inventory.py) | Regenerate RPM inventories (`--version 2.4\|2.5\|2.6\|all`) |
| [`scripts/audit_inventories.py`](scripts/audit_inventories.py) | Check enabled-line match and catalog conventions |
| [`scripts/extract_install_playbook_tasks.py`](scripts/extract_install_playbook_tasks.py) | Regenerate `install-playbook-tasks.md` (`--version X.Y --method rpm\|containerized`) |

Requires Python 3 and PyYAML. Install playbook extraction reads from `.installer-dumps/{version}/` at repo root.

## Maintainer docs

- [Containerized inventory maintenance](maintainer/CONTAINERIZED_INVENTORY_CONTEXT.md)
- [RPM inventory maintenance](maintainer/RPM_INVENTORY_CONTEXT.md)

## Using with Cursor / other agents

`@`-mention the folder for your version and method — inventories and install playbook tasks are in the same directory:

| Question | `@`-mention |
|----------|-------------|
| Inventory variables / host groups | `installer/AAP26/containerized/` |
| Install playbook task order / roles | same folder — `install-playbook-tasks.md` |
| Refreshing after a Red Hat update | `installer/maintainer/CONTAINERIZED_INVENTORY_CONTEXT.md` or `RPM_INVENTORY_CONTEXT.md` |

Do **not** run Format Document on generated `install-playbook-tasks.md` files (table formatters inflate column width).
