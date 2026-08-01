# AAP on OpenShift — operator examples

Example Kubernetes manifests for deploying Ansible Automation Platform with the **Red Hat AAP Operator** on OpenShift. These use Custom Resources and secrets — not the RPM or containerized install playbooks.

RPM and containerized inventory references live in [`installer/`](../installer/).

Backup/Restore CRs, what they consume, and why installer backups cannot feed them: [`backup-restore/`](../backup-restore/).

Local operator CRD dumps (gitignored): [`.crd-dumps/`](../.crd-dumps/) at repo root.

## Version index

| AAP | Model | README |
|-----|-------|--------|
| 2.4 | Separate CRs per component (`AutomationController`, `AutomationHub`, `EDA`) | [AAP24](AAP24/README.md) |
| 2.5+ | Platform-first — one `AnsibleAutomationPlatform` CR | [AAP25](AAP25/README.md) · [AAP26](AAP26/README.md) · [AAP27](AAP27/README.md) |

## Regenerating CR examples

Use [generate-aap-cr-examples](https://github.com/lennysh/openshift-playground/tree/main/generate-aap-cr-examples) from openshift-playground. Point `--output-dir` at the matching folder here (for example `/path/to/aap-notes/openshift/AAP27`) and `--crd-dir` at `.crd-dumps/{version}/`.

## Using with Cursor / other agents

`@`-mention `openshift/AAP27/` (or your version) for CR specs, secrets layout, and deployment order.
