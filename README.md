# AAP RBAC Notes

Reference material for Ansible Automation Platform role-based access control (RBAC), derived from local forks of the upstream repos.

## Files

| File | Audience | Purpose |
|------|----------|---------|
| [AAP-RBAC-GUIDE.md](./AAP-RBAC-GUIDE.md) | **Humans** | Concepts, mermaid diagrams, walkthroughs, common mistakes |
| [AAP-RBAC-ROLE-HIERARCHY.md](./AAP-RBAC-ROLE-HIERARCHY.md) | **Humans** | Full role tree (text + mermaid) from superuser down to object roles |
| [AAP-RBAC-AGENT-CONTEXT.md](./AAP-RBAC-AGENT-CONTEXT.md) | **AI agents** | Exhaustive lookup tables, API reference, goal → role mapping |

## Using with Cursor / other agents

1. Add this folder to your workspace, or `@`-mention `AAP-RBAC-AGENT-CONTEXT.md` in chat.
2. Ask goal-oriented questions, e.g. *"What role does user Jane need to add people to the Ops team?"*
3. The agent should answer with: role name, assignment scope (object), required permissions, and what **not** to assign.

## Source repos (local forks)

- `django-ansible-base-fork` — authoritative RBAC library
- `awx-fork` — Controller managed roles
- `eda-server-fork` — EDA org and resource roles
- `galaxy_ng-fork` — Hub / galaxy roles
- `ansible-ui-fork` — UI labels and Gateway fixtures (display only)

Re-derive this document when managed roles change upstream.
