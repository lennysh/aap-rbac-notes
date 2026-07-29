# AAP UI Design Notes

Reference material for mimicking the Ansible Automation Platform (Platform UI) look and feel in standalone HTML reports, emails, or internal tools.

Derived from the `ansible-ui-fork` monorepo (PatternFly 6.3.1).

## Files

| File | Audience | Purpose |
|------|----------|---------|
| [AAP-UI-DESIGN-AGENT-CONTEXT.md](AAP-UI-DESIGN-AGENT-CONTEXT.md) | **AI agents** | Colors, typography, dark mode, UI region mapping, source file pointers |
| [report-starter.css](report-starter.css) | **Humans & agents** | Copy-paste CSS starter kit for HTML reports |

## Using with Cursor / other agents

1. `@`-mention `ui/AAP-UI-DESIGN-AGENT-CONTEXT.md` when building HTML that should match the Platform UI.
2. Link or embed `report-starter.css` in generated HTML, or load PatternFly 6.3.1 CSS from unpkg (see agent context doc).

## Source repo

- `ansible-ui-fork` — Platform UI monorepo (`/framework`, `/platform`, PatternFly imports in `platform/main/PlatformMain.tsx`)

Re-derive when PatternFly version bumps or `framework/PageFramework.css` overrides change.
