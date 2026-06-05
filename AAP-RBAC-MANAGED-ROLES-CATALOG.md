# AAP Managed Roles Catalog

Complete reference of **built-in (managed) role definitions** and the permissions each grants, derived from local forks of `django-ansible-base`, `awx`, `eda-server`, `galaxy_ng`, and Gateway fixtures.

**Related:** [AAP-RBAC-AGENT-CONTEXT.md](./AAP-RBAC-AGENT-CONTEXT.md) · [AAP-RBAC-GUIDE.md](./AAP-RBAC-GUIDE.md) · [AAP-RBAC-ROLE-HIERARCHY.md](./AAP-RBAC-ROLE-HIERARCHY.md)

---

## How to use this catalog

| Column / field | Meaning |
|----------------|---------|
| **Scope** | What object the role is assigned *on* (`system`, `organization`, `team`, or a resource type) |
| **Permissions** | API slugs (`service.action_model`) unless noted as Django codenames for Hub legacy mapping |
| **Managed** | All roles here are `managed: true` — do not edit in production |

**Permission slug format:** `{service}.{codename}` — e.g. `awx.execute_jobtemplate`, `shared.member_team`, `eda.view_activation`.

### Verify on your deployment

Managed roles are synced at migrate time. Exact permission lists grow as services register models:

```http
GET /api/gateway/v1/role_definitions/?managed=true
GET /api/gateway/v1/role_definitions/{id}/
GET /api/controller/v2/role_definitions/?managed=true
GET /api/eda/v1/role_definitions/?managed=true
```

---

## Not a role definition: Superuser

| | |
|---|---|
| **Scope** | System |
| **Permissions** | **All** — RBAC checks bypassed |
| **Notes** | Django `is_superuser`; not a `RoleDefinition` record |

---

## Gateway / shared roles (`shared.*`)

Source: `ansible-ui/.../roleDefinitions.json`, `django-ansible-base/ansible_base/rbac/managed.py`

### Platform Auditor

| | |
|---|---|
| **Scope** | System (`content_type: null`) |
| **Description** | Has view permissions to all objects |

**Permissions (fixture minimum):**

- `shared.view_organization`
- `shared.view_team`

**Plus (dynamic at migrate):** every registered `view_*` permission across all services (Controller, EDA, Hub, …). AWX migration adds all permissions where `codename.startswith('view')`.

---

### Organization Admin

| | |
|---|---|
| **Scope** | Organization |
| **Description** | Has all permissions to a single organization and all objects inside of it |

**Permissions (Gateway `shared.*` — fixed list):**

- `shared.add_team`
- `shared.change_organization`
- `shared.change_team`
- `shared.delete_organization`
- `shared.delete_team`
- `shared.member_organization`
- `shared.member_team`
- `shared.view_organization`
- `shared.view_team`

**Plus (dynamic):** all permissions for every model registered under that organization in each connected service (see Controller and EDA sections — same role name merges in unified Platform).

---

### Organization Member

| | |
|---|---|
| **Scope** | Organization |
| **Description** | Has member permission to a single organization |

**Permissions:**

- `shared.member_organization`
- `shared.view_organization`

> **No operational privileges alone** — membership marker only. See [AAP-RBAC-GUIDE.md](./AAP-RBAC-GUIDE.md).

---

### Team Admin

| | |
|---|---|
| **Scope** | Team |
| **Description** | Can manage a single team and inherits all role assignments to the team |

**Permissions:**

- `shared.change_team`
- `shared.delete_team`
- `shared.member_team`
- `shared.view_team`

---

### Team Member

| | |
|---|---|
| **Scope** | Team |
| **Description** | Inherits all role assignments to a single team |

**Permissions:**

- `shared.member_team`
- `shared.view_team`

---

## Controller / AWX managed roles (`awx.*`)

Source: `awx/main/migrations/_dab_rbac.py` → `setup_managed_role_definitions`  
Registered models: `Project`, `Team`, `WorkflowJobTemplate`, `JobTemplate`, `Inventory`, `Organization`, `Credential`, `NotificationTemplate`, `ExecutionEnvironment`, `InstanceGroup`

**Naming patterns:**

| Pattern | Scope | Permissions rule |
|---------|-------|------------------|
| `{Model} Admin` | One resource | All permissions on that model **except** `add_{model}` |
| `{Model} {Action}` | One resource | `{action}_{model}` + `view_{model}` |
| `Organization {Model} Admin` | Organization | All permissions on that model + `add_{model}` + `awx.view_organization` |
| `Organization Admin` | Organization | Union of all org-child model permissions (excludes `InstanceGroup`) |
| `Organization Audit` | Organization | All `view_*` on org resources + `awx.audit_organization` |
| `Organization Execute` | Organization | Subset below |
| `Organization Approval` | Organization | Subset below |

**Legacy name:** `System Auditor` → migrated to **Platform Auditor** (see above).

---

### Organization-level Controller bundles

#### Organization Admin (Controller portion)

| | |
|---|---|
| **Scope** | Organization |
| **Permissions** | **All** `awx.*` permissions on org-child models (dynamic union of registered models, except `InstanceGroup`) |

Includes at minimum permissions from: Project, JobTemplate, WorkflowJobTemplate, Inventory, Credential, NotificationTemplate, ExecutionEnvironment, Organization, Team.

#### Organization Audit

| | |
|---|---|
| **Scope** | Organization |

**Permissions:**

- `awx.audit_organization`
- All `awx.view_*` permissions on org-child resources (dynamic)

#### Organization Execute

| | |
|---|---|
| **Scope** | Organization |

**Permissions:**

- `awx.view_organization`
- `awx.view_jobtemplate`
- `awx.execute_jobtemplate`
- `awx.view_workflowjobtemplate`
- `awx.execute_workflowjobtemplate`

#### Organization Approval

| | |
|---|---|
| **Scope** | Organization |

**Permissions:**

- `awx.view_organization`
- `awx.view_workflowjobtemplate`
- `awx.approve_workflowjobtemplate`

---

### Project roles

| Role | Scope | Permissions |
|------|-------|-------------|
| **Project Admin** | Project | `awx.change_project`, `awx.delete_project`, `awx.view_project`, `awx.update_project`, `awx.use_project` |
| **Project Update** | Project | `awx.view_project`, `awx.update_project` |
| **Project Use** | Project | `awx.view_project`, `awx.use_project` |
| **Organization Project Admin** | Organization | All Project permissions including `awx.add_project`, plus `awx.view_organization` |

---

### Job Template roles

| Role | Scope | Permissions |
|------|-------|-------------|
| **Job Template Admin** | Job template | `awx.change_jobtemplate`, `awx.delete_jobtemplate`, `awx.view_jobtemplate`, `awx.execute_jobtemplate` |
| **Job Template Execute** | Job template | `awx.view_jobtemplate`, `awx.execute_jobtemplate` |
| **Organization Job Template Admin** | Organization | All job template permissions including `awx.add_jobtemplate`, plus `awx.view_organization` |

---

### Workflow Job Template roles

| Role | Scope | Permissions |
|------|-------|-------------|
| **Workflow Job Template Admin** | Workflow job template | `awx.change_workflowjobtemplate`, `awx.delete_workflowjobtemplate`, `awx.view_workflowjobtemplate`, `awx.execute_workflowjobtemplate`, `awx.approve_workflowjobtemplate` |
| **Workflow Job Template Execute** | Workflow job template | `awx.view_workflowjobtemplate`, `awx.execute_workflowjobtemplate` |
| **Workflow Job Template Approve** | Workflow job template | `awx.view_workflowjobtemplate`, `awx.approve_workflowjobtemplate` |
| **Organization Workflow Job Template Admin** | Organization | All workflow job template permissions including `awx.add_workflowjobtemplate`, plus `awx.view_organization` |

---

### Inventory roles

| Role | Scope | Permissions |
|------|-------|-------------|
| **Inventory Admin** | Inventory | `awx.change_inventory`, `awx.delete_inventory`, `awx.view_inventory`, `awx.use_inventory`, `awx.adhoc_inventory`, `awx.update_inventory` |
| **Inventory Use** | Inventory | `awx.view_inventory`, `awx.use_inventory` |
| **Inventory Update** | Inventory | `awx.view_inventory`, `awx.update_inventory` |
| **Inventory Adhoc** | Inventory | `awx.view_inventory`, `awx.adhoc_inventory`, `awx.use_inventory` |
| **Organization Inventory Admin** | Organization | All inventory permissions including `awx.add_inventory`, plus `awx.view_organization` |

---

### Credential roles

| Role | Scope | Permissions |
|------|-------|-------------|
| **Credential Admin** | Credential | `awx.change_credential`, `awx.delete_credential`, `awx.view_credential`, `awx.use_credential` |
| **Credential Use** | Credential | `awx.view_credential`, `awx.use_credential` |
| **Organization Credential Admin** | Organization | All credential permissions including `awx.add_credential`, plus `awx.view_organization` |

---

### Notification Template roles

| Role | Scope | Permissions |
|------|-------|-------------|
| **Notification Template Admin** | Notification template | `awx.change_notificationtemplate`, `awx.delete_notificationtemplate`, `awx.view_notificationtemplate` |
| **Organization Notification Template Admin** | Organization | All notification template permissions including `awx.add_notificationtemplate`, plus `awx.view_organization` |

---

### Execution Environment roles

| Role | Scope | Permissions |
|------|-------|-------------|
| **Execution Environment Admin** | Execution environment | `awx.change_executionenvironment`, `awx.delete_executionenvironment`, `awx.view_executionenvironment` |
| **Organization Execution Environment Admin** | Organization | All execution environment permissions including `awx.add_executionenvironment`, plus `awx.view_organization` |

---

### Instance Group roles

| Role | Scope | Permissions |
|------|-------|-------------|
| **Instance Group Admin** | Instance group | `awx.change_instancegroup`, `awx.delete_instancegroup`, `awx.view_instancegroup`, `awx.use_instancegroup` |

> **Note:** `InstanceGroup` is **not** org-scoped — no `Organization Instance Group Admin` role.

---

## EDA managed roles (`eda.*`)

Source: `eda-server/.../create_initial_data.py` (`ORG_ROLES`, `_create_obj_roles`)

Permission prefix: `eda.` (e.g. `eda.view_activation`).

---

### EDA organization roles

All scoped to **organization** unless noted.

#### Organization Admin (EDA)

**Permissions by resource:**

| Resource | Actions granted |
|----------|-----------------|
| activation | add, view, change, delete, enable, disable, restart |
| rulebook_process | view |
| audit_rule | view |
| organization | view, change, delete |
| team | add, view, change, delete, member |
| project | add, view, change, delete, sync |
| rulebook | view |
| decision_environment | add, view, change, delete |
| eda_credential | add, view, change, delete |
| credential_input_source | add, view, change, delete |
| event_stream | add, view, change, delete |

**As slugs (examples):** `eda.add_activation`, `eda.enable_activation`, `eda.member_team`, `eda.sync_project`, …

#### Organization Member (EDA)

| Resource | Actions |
|----------|---------|
| organization | view, member |

> Membership only — no operational EDA access alone.

#### Organization Editor

| Resource | Actions |
|----------|---------|
| activation | add, view, change |
| rulebook_process, audit_rule, rulebook | view |
| organization, team | view |
| project, decision_environment, eda_credential, credential_input_source, event_stream | add, view, change |

#### Organization Contributor

Same as Editor, plus activation: **enable, disable, restart**.

#### Organization Operator

| Resource | Actions |
|----------|---------|
| activation | view, enable, disable, restart |
| All other listed EDA types | view only |

#### Organization Auditor / Organization Viewer

| Resource | Actions |
|----------|---------|
| All listed EDA types | view only |

---

### EDA per-resource managed roles (auto-generated)

For each org-child model registered in EDA (except `organization`), migrate creates:

| Role pattern | Scope | Permissions |
|--------------|-------|-------------|
| **`{Model} Admin`** | One object | All model permissions **except** `add_*` (includes child model perms where applicable) |
| **`{Model} Use`** | One object | All `view_*` on that model (and children) |
| **`Organization {Model} Admin`** | Organization | All model permissions **including** `add_*`, plus `eda.view_organization` |

**Naming exceptions:**

- Project object/org roles prefixed with **`EDA`** (e.g. **EDA Project Admin**, **Organization EDA Project Admin**)
- `EdaCredential` → **EDA Credential Admin**, etc.

**Generated for these resource types:**

| Resource | Object Admin | Object Use | Org Admin |
|----------|--------------|------------|-----------|
| Activation | Activation Admin | Activation Use | Organization Activation Admin |
| EDA Project | EDA Project Admin | EDA Project Use | Organization EDA Project Admin |
| EDA Credential | EDA Credential Admin | EDA Credential Use | Organization EDA Credential Admin |
| Credential Input Source | Credential Input Source Admin | Credential Input Source Use | Organization Credential Input Source Admin |
| Decision Environment | Decision Environment Admin | Decision Environment Use | Organization Decision Environment Admin |
| Event Stream | Event Stream Admin | Event Stream Use | Organization Event Stream Admin |
| Team | Team Admin | *(no Use role)* | Organization Team Admin |

**Child models** (permissions bundled into parent Admin roles, not separate top-level roles):

- **Rulebook** → included in Project Admin
- **Rulebook Process**, **Audit Rule** → included in Activation Admin

**Activation Admin** includes: `add`, `view`, `change`, `delete`, `enable`, `disable`, `restart` on activation + related child view permissions.

**EDA Project Admin** includes: CRUD + `sync_project` on project + rulebook view permissions.

---

## Automation Hub managed roles (`galaxy.*`)

Source: `galaxy_ng/app/migrations/0029_move_perms_to_roles.py`  
Hub syncs DAB `RoleDefinition` with legacy Pulp role names. **`galaxy.auditor`** ↔ **Platform Auditor** (system-wide view).

Permissions below use Django `app_label.codename` as stored in migration; DAB slugs use `galaxy.*` role names.

---

### System / global Hub roles

#### galaxy.auditor (Platform Auditor on Hub)

| | |
|---|---|
| **Scope** | System |
| **Permissions** | View-only across registered Hub models (synced with Platform Auditor) |

---

#### galaxy.content_admin

**Permissions:**

- `galaxy.add_namespace`, `galaxy.change_namespace`, `galaxy.delete_namespace`, `galaxy.upload_to_namespace`
- `ansible.delete_collection`, `ansible.change_collectionremote`, `ansible.view_collectionremote`, `ansible.modify_ansible_repo_content`
- `container.delete_containerrepository`, `container.namespace_change_containerdistribution`, `container.namespace_modify_content_containerpushrepository`, `container.namespace_push_containerdistribution`, `container.add_containernamespace`, `container.change_containernamespace`, `container.manage_roles_containernamespace`
- `galaxy.add_containerregistryremote`, `galaxy.change_containerregistryremote`, `galaxy.delete_containerregistryremote`

---

#### galaxy.collection_admin

**Permissions:**

- `galaxy.add_namespace`, `galaxy.change_namespace`, `galaxy.delete_namespace`, `galaxy.upload_to_namespace`
- `ansible.delete_collection`, `ansible.change_collectionremote`, `ansible.view_collectionremote`, `ansible.modify_ansible_repo_content`

---

#### galaxy.collection_publisher

**Permissions:**

- `galaxy.add_namespace`, `galaxy.change_namespace`, `galaxy.upload_to_namespace`

---

#### galaxy.collection_curator

**Permissions:**

- `ansible.change_collectionremote`, `ansible.view_collectionremote`, `ansible.modify_ansible_repo_content`

---

#### galaxy.execution_environment_admin

**Permissions:**

- `container.delete_containerrepository`, `container.namespace_change_containerdistribution`, `container.namespace_modify_content_containerpushrepository`, `container.namespace_push_containerdistribution`, `container.add_containernamespace`, `container.change_containernamespace`, `container.manage_roles_containernamespace`
- `galaxy.add_containerregistryremote`, `galaxy.change_containerregistryremote`, `galaxy.delete_containerregistryremote`

---

#### galaxy.execution_environment_publisher

**Permissions:**

- `container.namespace_change_containerdistribution`, `container.namespace_modify_content_containerpushrepository`, `container.namespace_push_containerdistribution`, `container.add_containernamespace`, `container.change_containernamespace`

---

#### galaxy.group_admin

**Permissions:**

- `galaxy.view_group`, `galaxy.add_group`, `galaxy.change_group`, `galaxy.delete_group`

---

#### galaxy.user_admin

**Permissions:**

- `galaxy.view_user`, `galaxy.add_user`, `galaxy.change_user`, `galaxy.delete_user`

---

#### galaxy.task_admin

**Permissions:**

- `core.view_task`, `core.change_task`, `core.delete_task`

---

#### galaxy.synclist_owner (global definition)

**Permissions:**

- `galaxy.add_synclist`, `galaxy.change_synclist`, `galaxy.delete_synclist`, `galaxy.view_synclist`

---

### Hub object-scoped roles

Assign on a **specific object** (namespace, synclist, etc.).

#### galaxy.collection_namespace_owner

**Permissions:**

- `galaxy.change_namespace`, `galaxy.upload_to_namespace`

*(Global variant also exists with `galaxy.add_namespace`, `galaxy.delete_namespace` — same role name, different assignment scope.)*

---

#### galaxy.execution_environment_namespace_owner

**Permissions:**

- `container.change_containernamespace`, `container.namespace_push_containerdistribution`, `container.namespace_change_containerdistribution`, `container.namespace_modify_content_containerpushrepository`, `container.manage_roles_containernamespace`

---

#### galaxy.execution_environment_collaborator

**Permissions:**

- `container.namespace_push_containerdistribution`, `container.namespace_change_containerdistribution`, `container.namespace_modify_content_containerpushrepository`

---

#### galaxy.synclist_owner (object)

**Permissions:**

- `galaxy.add_synclist`, `galaxy.change_synclist`, `galaxy.delete_synclist`, `galaxy.view_synclist`

---

#### galaxy.ansible_repository_owner

**Permissions:** *(object-scoped; see Hub access policies — repository management)*

> UI descriptions for all `galaxy.*` roles: `ansible-ui/frontend/hub/access/roles/hooks/useManagedRolesWithDescription.tsx`

---

## Quick index — all managed role names

### Gateway / shared (5)

`Platform Auditor` · `Organization Admin` · `Organization Member` · `Team Admin` · `Team Member`

### Controller (47+ unique names from registered models)

**Org bundles:** Organization Admin · Organization Audit · Organization Execute · Organization Approval

**Per resource type** (×7 org-child types): `{Type} Admin` · `{Type} {Action}` · `Organization {Type} Admin`

**Types:** Project · Job Template · Workflow Job Template · Inventory · Credential · Notification Template · Execution Environment

**Special actions:** Update · Use · Execute · Adhoc · Approve (where applicable)

**Standalone:** Instance Group Admin

### EDA (7 org roles + ~3 roles × ~7 resource types)

**Org:** Organization Admin · Organization Member · Organization Editor · Organization Contributor · Organization Operator · Organization Auditor · Organization Viewer

**Object + org pairs:** Activation · EDA Project · EDA Credential · Credential Input Source · Decision Environment · Event Stream · Team — each with Admin / Use / Organization … Admin variants

### Hub (14+ galaxy.* roles)

`galaxy.auditor` · `galaxy.content_admin` · `galaxy.collection_admin` · `galaxy.collection_publisher` · `galaxy.collection_curator` · `galaxy.collection_namespace_owner` · `galaxy.execution_environment_admin` · `galaxy.execution_environment_publisher` · `galaxy.execution_environment_namespace_owner` · `galaxy.execution_environment_collaborator` · `galaxy.group_admin` · `galaxy.user_admin` · `galaxy.task_admin` · `galaxy.synclist_owner` · `galaxy.ansible_repository_owner`

---

## Source files (update this catalog when upstream changes)

| Service | File |
|---------|------|
| DAB shared templates | `django-ansible-base-fork/ansible_base/rbac/managed.py` |
| Gateway fixture | `ansible-ui-fork/platform/access/authenticators/components/mocks/roleDefinitions.json` |
| Controller | `awx-fork/awx/main/migrations/_dab_rbac.py` |
| Controller model perms | `awx-fork/awx/main/models/{projects,jobs,inventory,credential,workflow,organization,ha}.py` |
| EDA org roles | `eda-server-fork/src/aap_eda/core/management/commands/create_initial_data.py` → `ORG_ROLES`, `_create_obj_roles` |
| EDA models | `eda-server-fork/src/aap_eda/core/models/__init__.py` (registry) |
| Hub | `galaxy_ng-fork/galaxy_ng/app/migrations/0029_move_perms_to_roles.py` |
