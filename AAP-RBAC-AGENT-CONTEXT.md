# AAP RBAC Agent Context

> **Purpose:** Single source of truth for AI agents answering Ansible Automation Platform (AAP) RBAC questions — which roles to assign, at what scope, and why.
>
> **Audience:** Agents helping admins configure access in the unified Platform UI (Gateway) and component services (Controller/AWX, EDA, Hub).
>
> **Last derived from:** Local forks of `django-ansible-base`, `awx`, `eda-server`, `galaxy_ng`, and `ansible-ui`.
>
> **Companion docs:** [AAP-RBAC-GUIDE.md](./AAP-RBAC-GUIDE.md) · [AAP-RBAC-ROLE-HIERARCHY.md](./AAP-RBAC-ROLE-HIERARCHY.md) · [AAP-RBAC-MANAGED-ROLES-CATALOG.md](./AAP-RBAC-MANAGED-ROLES-CATALOG.md)

---

## How agents should use this document

1. **Identify the goal** (what the user must be able to do in the UI or API).
2. **Find the goal** in [Goal → role lookup](#goal--role-lookup) or derive the required **permission codename(s)**.
3. **Pick the narrowest role** that satisfies the goal (prefer object-scoped over org-scoped over system-scoped).
4. **State assignment clearly:** role name + content type + target object + assignee (user or team).
5. **Call out exceptions:** external auth (`MANAGE_ORGANIZATION_AUTH`), superuser, or service-specific gaps (Hub groups vs Platform teams).

When unsure, recommend verifying on a live system:

```http
GET /api/gateway/v1/role_definitions/
GET /api/gateway/v1/role_metadata/
GET /api/gateway/v1/role_user_assignments/?object_id=<id>&content_type__model=<model>
```

---

## Architecture (three layers)

```text
┌─────────────────────────────────────────────────────────────┐
│  Platform UI (ansible-ui) + Gateway API (/api/gateway/v1/)  │
│  Shared access: organizations, teams, users, role defs      │
└───────────────────────────┬─────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
   Controller            EDA Server         Automation Hub
   (awx / awx.*)         (eda / eda.*)      (galaxy / galaxy.*)
   /api/controller/v2/   /api/eda/v1/       /api/galaxy/
```

| Layer | Owns | RBAC library | Typical API prefix |
|-------|------|--------------|-------------------|
| **Gateway (shared)** | Organizations, teams, users, platform roles, auth | django-ansible-base (DAB) | `/api/gateway/v1/` |
| **Controller** | Job templates, inventories, projects, credentials, jobs, … | DAB + AWX legacy role bridge | `/api/controller/v2/` |
| **EDA** | Activations, rulebooks, projects, decision environments, … | DAB | `/api/eda/v1/` |
| **Hub** | Collections, namespaces, execution environments, remotes, … | DAB + Pulp roles (synced) | `/api/galaxy/` |

**Authoritative RBAC implementation:** django-ansible-base (`ansible_base.rbac`). Each service registers its models and creates managed role definitions at migrate time.

---

## Core concepts (DAB RBAC)

### Permissions

A permission = **action** + **content type**, expressed as an API slug:

```text
{service}.{action}_{model}
```

Examples: `shared.member_team`, `shared.view_organization`, `awx.change_jobtemplate`, `eda.view_activation`, `galaxy.upload_to_namespace`.

Standard CRUD actions: `add`, `view`, `change`, `delete`. Special actions include `member_team`, `member_organization`, `execute`, `use`, `approve`, `audit`, etc.

### Role definitions ("roles")

A role definition lists permissions and optionally restricts them to a **content type**:

| `content_type` | Scope | Effect |
|----------------|-------|--------|
| `null` | **System-wide** | Permissions apply globally to all objects of those types |
| `shared.organization` | **Organization** | Permissions apply to all objects inside that organization |
| `shared.team` | **Team** | Permissions apply in context of that team |
| `awx.inventory`, `eda.activation`, … | **Object** | Permissions apply to one specific resource instance |

Managed roles (`managed: true`) are created by the platform and should not be edited.

### Assignments

| Assignment type | API | Meaning |
|-----------------|-----|---------|
| **User + role + object** | `role_user_assignments` | User gets role permissions for that object |
| **Team + role + object** | `role_team_assignments` | All team members inherit those permissions |

**Team membership is special:**

- The `member_team` permission marks team membership and causes inheritance of all roles granted **to** that team.
- To **make someone a member** of a team, assign them the **Team Member** role on that team.
- To **manage who is on a team** (add/remove members), assign **Team Admin** (or Organization Admin), not Team Member.

**Hard rules:**

- **Organization Member alone confers no operational privileges** — it only marks org membership for inheritance; assign additional roles (team, resource, or org-level) for any real access.
- Teams cannot be assigned Organization Admin/Member or Team Admin/Member roles.
- `member_team` cannot appear in system-wide (`content_type: null`) custom roles.
- Organization-level roles that include `member_team` apply across all teams in that org.

---

## Goal → role lookup

Use the **minimum** role listed first. Assignments are **user → role → object** unless noted.

### Platform / Gateway — organizations, teams, users

| Goal | Required permission(s) | Assign this role | Scope (object) | Notes |
|------|------------------------|------------------|----------------|-------|
| **Add/remove users on a team** | `shared.member_team` (+ UI typically needs `shared.change_team`) | **Team Admin** | Target **team** | Do **not** give Team Member to the manager — that role is for people who should *be on* the team. |
| Add/remove users on **any** team in an org | `shared.member_team`, `shared.add_team`, `shared.change_team`, … | **Organization Admin** | Parent **organization** | Broader; full org admin across services. |
| View team metadata only | `shared.view_team` | Custom role or **Platform Auditor** | Team or system | |
| Create/delete teams in an org | `shared.add_team`, `shared.delete_team`, … | **Organization Admin** | Organization | |
| Manage org settings | `shared.change_organization`, … | **Organization Admin** | Organization | |
| Be part of an org (prerequisite for member-targeted roles) | `shared.member_organization` | **Organization Member** | Organization | **Alone: no operational privileges.** Only marks membership; user still needs Team Member, resource roles, or other assignments. |
| View all orgs/teams (read-only, global) | `shared.view_*` | **Platform Auditor** | System (`content_type: null`) | |
| Full system administration | All permissions | Django **superuser** / Gateway admin | System | Bypasses RBAC. |

#### Worked example: "User X can add and remove users from team Ops"

```text
Assign:  User X  →  Team Admin  →  team "Ops" (object_id = Ops team pk)

Permissions included:
  shared.change_team
  shared.delete_team
  shared.member_team
  shared.view_team

UI: Access Management → Teams → Ops → Access → Add user → Role: Team Admin

Do NOT assign: Team Member (makes X a member, not a team manager)
```

**Broader alternative:** Organization Admin on the org containing Ops — only if they need org-wide admin too.

**Setting caveat:** If `MANAGE_ORGANIZATION_AUTH` is false with external SSO, Organization Admin may be blocked from team membership changes on Controller-backed teams. Team Admin on the specific team is the reliable narrow fix.

### Controller (Automation Execution)

| Goal | Assign this role | Scope |
|------|------------------|-------|
| Full control of one job template | **Job Template Admin** | That job template |
| Run a job template | **Job Template Execute** | That job template |
| View job template settings | **Job Template Use** or read roles | That job template |
| Manage all job templates in org | **Organization Job Template Admin** | Organization |
| Manage all inventories in org | **Organization Inventory Admin** | Organization |
| Manage one inventory | **Inventory Admin** | That inventory |
| Run ad hoc commands on inventory | **Inventory Ad Hoc** | That inventory |
| Use inventory in a job template | **Inventory Use** | That inventory |
| Manage all projects in org | **Organization Project Admin** | Organization |
| Sync/update one project | **Project Update** or **Project Admin** | That project |
| Use project in job template | **Project Use** | That project |
| Manage credentials in org | **Organization Credential Admin** | Organization |
| Use credential in job template | **Credential Use** | That credential |
| Run anything runnable in org | **Organization Execute** | Organization |
| Approve workflow nodes in org | **Organization Approval** | Organization |
| Read everything in org | **Organization Audit** | Organization |
| Full org control (all controller resources) | **Organization Admin** | Organization |

**Auto-generated managed role naming** (AWX `setup_managed_role_definitions`):

| Pattern | Content type | Meaning |
|---------|--------------|---------|
| `{Resource} Admin` | Resource model | All permissions on one instance (except `add_` on that type) |
| `Organization {Resource} Admin` | `shared.organization` | All permissions for that resource type inside org |
| `{Resource} {Action}` | Resource model | Special action + view (Execute, Use, Update, Ad Hoc, Approve, …) |
| `Organization Admin` | Organization | All registered model permissions in org |
| `Organization Audit` / `Execute` / `Approval` | Organization | Bundled org-level action sets |

Legacy implicit role names (`admin_role`, `read_role`, `use_role`, …) still appear in Controller UI during migration. Map via `awx/main/constants.py` → `role_name_to_perm_mapping`.

### EDA (Automation Decisions)

| Goal | Assign this role | Scope |
|------|------------------|-------|
| Full EDA admin within org | **Organization Admin** | Organization |
| View-only in org | **Organization Auditor** or **Organization Viewer** | Organization |
| Create/edit EDA resources (no delete) | **Organization Editor** | Organization |
| Enable/disable/restart activations | **Organization Contributor** or **Organization Operator** | Organization |
| Manage one activation | **Activation Admin** | That activation |
| View one activation | **Activation Use** | That activation |
| Manage all activations in org | **Organization Activation Admin** | Organization |

EDA org roles: `eda-server` → `create_initial_data.py` → `ORG_ROLES`. Per-resource `{Model} Admin` and `{Model} Use` roles are auto-created for each registered model (except `team`).

### Hub (Automation Content)

| Goal | Role(s) | Scope |
|------|---------|-------|
| Upload/publish collections | **galaxy.collection_publisher** or **galaxy.collection_namespace_owner** | Global or namespace object |
| Curate/sync collections from remotes | **galaxy.collection_curator** | Global |
| Full collection admin | **galaxy.collection_admin** | Global |
| Manage execution environments | **galaxy.execution_environment_admin** | Global |
| Manage Hub users | **galaxy.user_admin** | Global |
| Manage Hub groups | **galaxy.group_admin** | Global |
| View-only across Hub | **Platform Auditor** / **galaxy.auditor** | System |

**Important:** Hub content roles are often **global** or **namespace object-level**, not org-level. Organization/team use `shared.*` like Gateway. See `galaxy_ng/docs/dev/developer_guide/rbac.md`.

---

## Gateway managed roles (complete reference)

Sources: `ansible-ui/platform/access/authenticators/components/mocks/roleDefinitions.json`, `django-ansible-base/ansible_base/rbac/managed.py`.

> **Full permission lists for every built-in role:** [AAP-RBAC-MANAGED-ROLES-CATALOG.md](./AAP-RBAC-MANAGED-ROLES-CATALOG.md)

### Platform Auditor (system-wide)

| Field | Value |
|-------|-------|
| **content_type** | `null` |
| **Permissions** | `shared.view_organization`, `shared.view_team` (+ service `view_*` per registration) |
| **Use for** | Read-only access across the platform |

### Organization Admin

| Field | Value |
|-------|-------|
| **content_type** | `shared.organization` |
| **Permissions (shared)** | `shared.change_organization`, `shared.delete_organization`, `shared.member_organization`, `shared.view_organization`, `shared.add_team`, `shared.change_team`, `shared.delete_team`, `shared.member_team`, `shared.view_team` |
| **Plus** | All service permissions for models registered under that org |
| **Use for** | Full administrative control of one organization |

### Organization Member

| Field | Value |
|-------|-------|
| **content_type** | `shared.organization` |
| **Permissions** | `shared.member_organization`, `shared.view_organization` |
| **Use for** | Marks that a user belongs to the organization (membership hook for inheritance) |

> **Critical:** Organization Member **alone does not confer any operational privileges.** It does not let a user run jobs, manage teams, edit inventories, or access org resources. The listed permissions are membership markers — `member_organization` enables inheritance when **other** roles are assigned (to the user, to a team they join, or to org members as a group). Always assign additional roles for any capability beyond “is in this org.”

### Team Admin

| Field | Value |
|-------|-------|
| **content_type** | `shared.team` |
| **Permissions** | `shared.change_team`, `shared.delete_team`, `shared.member_team`, `shared.view_team` |
| **Use for** | Manage team settings and membership; inherits roles assigned to the team |

### Team Member

| Field | Value |
|-------|-------|
| **content_type** | `shared.team` |
| **Permissions** | `shared.member_team`, `shared.view_team` |
| **Use for** | Makes user a **member** of the team. **Not** for users who manage membership of others. |

---

## Hub managed roles (galaxy.*)

Common roles and intent (from `galaxy_ng/app/migrations/0029_move_perms_to_roles.py`):

| Role | Typical use |
|------|-------------|
| `galaxy.content_admin` | Broad content management (namespaces, collections, EEs, remotes) |
| `galaxy.collection_admin` | Full collection lifecycle |
| `galaxy.collection_publisher` | Create/upload collections |
| `galaxy.collection_curator` | Sync/approve collections from remotes |
| `galaxy.collection_namespace_owner` | Manage one namespace |
| `galaxy.execution_environment_admin` | Full EE and registry management |
| `galaxy.execution_environment_publisher` | Push/manage EEs |
| `galaxy.execution_environment_namespace_owner` | EE namespace ownership |
| `galaxy.execution_environment_collaborator` | Modify existing EEs |
| `galaxy.user_admin` | Hub user CRUD |
| `galaxy.group_admin` | Hub group CRUD |
| `galaxy.task_admin` | View/cancel tasks |
| `galaxy.synclist_owner` | Manage synclists |
| `galaxy.ansible_repository_owner` | Manage ansible repositories |

UI descriptions: `ansible-ui/frontend/hub/access/roles/hooks/useManagedRolesWithDescription.tsx`.

---

## Custom roles

Create when managed roles are too broad.

```http
POST /api/gateway/v1/role_definitions/
{
  "name": "Team membership manager",
  "description": "Can add/remove team members only",
  "content_type": "shared.team",
  "permissions": ["shared.member_team", "shared.view_team"]
}
```

Assign via `POST /api/gateway/v1/role_user_assignments/`. Valid permissions: `GET /api/gateway/v1/role_metadata/`.

**Agent rules:**

- Prefer managed roles unless least-privilege custom permissions are required.
- Never put `shared.member_team` in a system-wide role — validation rejects it.
- Compare raw permission slugs, not translated UI labels.

---

## UI navigation (Platform)

| Task | Path |
|------|------|
| View/create role definitions | **Access Management → Roles** |
| Assign org roles | **Access Management → Organizations → {org} → Access** |
| Assign team roles | **Access Management → Teams → {team} → Access** |
| Assign resource roles | Resource detail → **Access** tab |

---

## API quick reference

| Resource | Endpoint |
|----------|----------|
| List role definitions | `GET /api/gateway/v1/role_definitions/` |
| Role metadata | `GET /api/gateway/v1/role_metadata/` |
| Assign user | `POST /api/gateway/v1/role_user_assignments/` |
| Assign team | `POST /api/gateway/v1/role_team_assignments/` |
| Controller roles | `GET /api/controller/v2/role_definitions/` |
| EDA roles | `GET /api/eda/v1/role_definitions/` |

Assignment body:

```json
{
  "role_definition": 3,
  "object_id": 42,
  "user": 7
}
```

Use `"team": <team_id>` instead of `"user"` for team-to-object assignments.

---

## Settings that change effective access

| Setting | Service | Effect |
|---------|---------|--------|
| `MANAGE_ORGANIZATION_AUTH` | Controller | When false with external auth, org admins may be blocked from team membership changes |
| `ORG_ADMINS_CAN_SEE_ALL_USERS` | Controller | Whether org admins see all users or only org-associated users |
| Superuser | All | Bypasses RBAC |

---

## Source files (re-derive when updating)

| Topic | Path in local fork |
|-------|---------------------|
| DAB RBAC concepts | `django-ansible-base-fork/docs/apps/rbac/` |
| Shared managed role templates | `django-ansible-base-fork/ansible_base/rbac/managed.py` |
| Controller managed roles | `awx-fork/awx/main/migrations/_dab_rbac.py` |
| Controller legacy role names | `awx-fork/awx/main/models/rbac.py`, `awx/main/constants.py` |
| Gateway role fixtures | `ansible-ui-fork/platform/access/authenticators/components/mocks/roleDefinitions.json` |
| EDA org roles | `eda-server-fork/src/aap_eda/core/management/commands/create_initial_data.py` |
| Hub role maps | `galaxy_ng-fork/galaxy_ng/app/migrations/0029_move_perms_to_roles.py` |
| Hub RBAC overview | `galaxy_ng-fork/docs/dev/developer_guide/rbac.md` |

---

## Agent response template

```markdown
**Goal:** {restated goal}

**Minimum role:** {Role Name}
**Assign to:** User {name}
**On object:** {Organization | Team | Resource} "{name}" (content_type: {slug}, id: {pk})

**Why:** Includes `{permission}` which grants {capability}.

**Alternative (broader):** {optional}

**Not sufficient:** {wrong roles and why — e.g. Organization Member alone for any operational goal}

**Verify:** Access Management → {path} or GET {endpoint}
```

---

## Known limitations

- Legacy Controller role APIs coexist with `role_definitions` during DAB migration.
- Hub org-scoped content RBAC is incomplete vs Gateway org model.
- EDA `feature/rbac` branch may differ from `main`.
- UI strings in `ansible-ui` hooks are display-only — not authoritative for logic.
- Does not cover Lightspeed, Analytics, or licensing roles.
