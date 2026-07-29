# AAP Notes

Personal reference material for [Ansible Automation Platform (AAP)](https://www.redhat.com/en/technologies/management/ansible). This repo was formerly **aap-rbac-notes**; RBAC content lives under `rbac/`.

## Contents

| Folder | Description |
|--------|-------------|
| [`rbac/`](rbac/) | RBAC concepts, role catalogs, hierarchy diagrams, organizational design, agent lookup |
| [`installer/`](installer/) | **RPM & containerized** — inventories, vars examples, install playbook task reference |
| [`openshift/`](openshift/) | **OpenShift operator** — CR and secret examples (not install playbooks) |
| [`ui/`](ui/) | Platform UI design tokens — colors, typography, dark mode, HTML report starter CSS |

Local Red Hat inputs (gitignored at repo root): [`.installer-dumps/`](.installer-dumps/) (installer tarballs), [`.crd-dumps/`](.crd-dumps/) (OpenShift operator CRDs).

## Quick links

- **RBAC overview** — [`rbac/README.md`](rbac/README.md)
- **Installer index** — [`installer/README.md`](installer/README.md)
- **OpenShift index** — [`openshift/README.md`](openshift/README.md)
- **OpenShift examples** — [2.4](openshift/AAP24/README.md) · [2.5](openshift/AAP25/README.md) · [2.6](openshift/AAP26/README.md) · [2.7](openshift/AAP27/README.md)
- **RPM inventories** — [2.4](installer/AAP24/rpm/README.md) · [2.5](installer/AAP25/rpm/README.md) · [2.6](installer/AAP26/rpm/README.md) (no RPM for 2.7)
- **Containerized inventories** — [2.5](installer/AAP25/containerized/README.md) · [2.6](installer/AAP26/containerized/README.md) · [2.7](installer/AAP27/containerized/README.md)
- **UI design / HTML reports** — [`ui/README.md`](ui/README.md)

## Using with Cursor / other agents

`@`-mention the folder that matches your question:

| Question | `@`-mention |
|----------|-------------|
| RBAC / roles | `rbac/AAP-RBAC-AGENT-CONTEXT.md` |
| UI colors / HTML report styling | `ui/AAP-UI-DESIGN-AGENT-CONTEXT.md` |
| Inventory or install playbook tasks | `installer/AAP25/containerized/` (match your version/method) |
| OpenShift CRs / secrets | `openshift/AAP27/` (match your version) |
| Refreshing inventories after a Red Hat update | `installer/maintainer/CONTAINERIZED_INVENTORY_CONTEXT.md` or `RPM_INVENTORY_CONTEXT.md` |

Most material under `rbac/` was derived from local forks of upstream AAP component repos; see [`rbac/README.md`](rbac/README.md).
