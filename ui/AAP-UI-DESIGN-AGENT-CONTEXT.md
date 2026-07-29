# AAP UI Design Agent Context

> **Purpose:** Single source of truth for AI agents building HTML reports, emails, or tools that should visually match the Ansible Automation Platform (Platform UI).
>
> **Audience:** Agents working outside `ansible-ui-fork` who need colors, typography, layout cues, and starter CSS.
>
> **Last derived from:** `ansible-ui-fork` monorepo, PatternFly **6.3.1** (`@patternfly/patternfly@6.3.1`).
>
> **Companion files:** [report-starter.css](report-starter.css) · [README.md](README.md)

---

## How agents should use this document

1. **Prefer PatternFly semantic tokens** (`--pf-t--global--*`) over hardcoded hex when PF CSS is loaded.
2. **Primary UI accent is blue (`#0066cc`)**, not Ansible red — use red only for logo/mark or danger status.
3. **Enable dark mode** by adding `class="pf-v6-theme-dark"` to `<html>`.
4. **Load PF CSS** for best fidelity, or use hex fallbacks from the cheat sheets below.
5. **Copy layout patterns** from masthead → page header → content/table (see [UI region mapping](#ui-region-mapping)).

Minimal HTML head:

```html
<link rel="stylesheet" href="https://unpkg.com/@patternfly/patternfly@6.3.1/patternfly-addons.css">
<link rel="stylesheet" href="https://unpkg.com/@patternfly/patternfly@6.3.1/patternfly-base.css">
<link rel="stylesheet" href="report-starter.css">
```

---

## Design system foundation

| Item | Value |
|------|-------|
| UI framework | [PatternFly 6](https://www.patternfly.org/) |
| PF version in ansible-ui | **6.3.1** |
| Custom SCSS theme | **None** — PF tokens + a few CSS overrides |
| Semantic token prefix | `--pf-t--global--*` |
| Component token prefix | `--pf-v6-c-*` |
| PF CSS entry (Platform) | `platform/main/PlatformMain.tsx` imports `patternfly-addons.css`, `patternfly-base.css`, `patternfly-charts.css` |

The app does **not** define a custom color palette. It inherits PatternFly 6 semantic tokens and applies a handful of overrides in `framework/PageFramework.css`.

---

## Repo-specific overrides (important)

**File:** `ansible-ui-fork/framework/PageFramework.css`

```css
#app {
  --pf-t--global--border--color--default: light-dark(#ccc, #505050);
  --pf-t--chart--global--success--color--100: var(--pf-t--chart--color--green--200);
}

table tbody tr td {
  border-bottom: 1px solid var(--pf-t--global--border--color--default);
}
```

**Pre-load splash:** `platform/main/Platform.tsx` sets `document.body.style.backgroundColor = '#222'` before React mounts.

**Semantic color aliases in app code:** `framework/components/pfcolors.tsx`

| Export | CSS variable |
|--------|--------------|
| `pfSuccess` | `--pf-t--global--color--status--success--default` |
| `pfDanger` | `--pf-t--global--color--status--danger--default` |
| `pfWarning` | `--pf-t--global--color--status--warning--default` |
| `pfInfo` | `--pf-t--global--color--status--info--default` |
| `pfDisabled` | `--pf-t--global--text--color--disabled` |
| `pfLink` | `--pf-t--global--text--color--link--default` |

**Label colors** used in tables/badges: `blue`, `teal`, `green`, `orange`, `purple`, `red`, `orangered`, `grey`, `yellow`.

---

## Light theme palette (resolved hex)

Use CSS variables when PF is loaded; hex values are PF 6.3.1 light-theme defaults.

### Backgrounds

| Usage | Token | Hex |
|-------|-------|-----|
| Main content / cards / tables | `--pf-t--global--background--color--primary--default` | `#ffffff` |
| Masthead, sidebar, striped rows | `--pf-t--global--background--color--secondary--default` | `#f2f2f2` |
| Row/cell hover | `--pf-t--global--background--color--primary--hover` | `#f2f2f2` |
| Form inputs | `--pf-t--global--background--color--control--default` | `#ffffff` |
| Code blocks / dark panels | `--pf-t--color--gray--95` | `#151515` |

### Text

| Role | Token | Hex |
|------|-------|-----|
| Primary body | `--pf-t--global--text--color--regular` | `#151515` |
| Secondary / nav / subtle | `--pf-t--global--text--color--subtle` | `#4d4d4d` |
| Disabled / muted | `--pf-t--global--text--color--disabled` | `#a3a3a3` |
| On primary buttons | `--pf-t--global--text--color--on-brand--default` | `#ffffff` |
| Page description | regular text at `opacity: 0.8` | ~`rgba(21,21,21,0.8)` |

### Brand / links / buttons

| Role | Token | Hex |
|------|-------|-----|
| Brand / primary button / links | `--pf-t--global--color--brand--default` | `#0066cc` |
| Brand hover | `--pf-t--global--color--brand--hover` | `#004d99` |
| Visited links | `--pf-t--global--text--color--link--visited` | `#5e40be` |
| Danger button bg | `--pf-t--global--color--status--danger--default` | `#b1380b` |

> **Do not use Ansible red (`#ee0000`) for primary buttons.** PF6 brand is blue.

### Status colors

Used for badges, labels, job status bars, and ANSI log output (`frontend/common/Ansi.css`).

| Status | Token | Hex | Typical use |
|--------|-------|-----|-------------|
| Success | `--pf-t--global--color--status--success--default` | `#3d7317` | OK, successful jobs |
| Danger | `--pf-t--global--color--status--danger--default` | `#b1380b` | Failed, errors |
| Warning | `--pf-t--global--color--status--warning--default` | `#ffcc17` | Changed, canceled |
| Info | `--pf-t--global--color--status--info--default` | `#5e40be` | Running, pending |

### Borders

| Usage | Token / override | Hex |
|-------|------------------|-----|
| Default (app override) | `light-dark(#ccc, #505050)` | `#cccccc` |
| Stock PF light default | `--pf-t--global--border--color--default` | `#c7c7c7` |
| Strong | `--pf-t--global--border--color--200` | `#a3a3a3` |

### Ansible-specific accents

| Color | Hex | Where |
|-------|-----|-------|
| Ansible mark red (PF palette) | `#ee0000` | Logo/mark context only |
| Status danger (actual UI) | `#b1380b` | Errors, failed states |
| Login footer link | `#b9dafc` (`--pf-t--color--blue--20`) | Login page only |
| Pre-load body | `#222222` | Splash before app loads |
| Logo | `currentColor` | Black (light) / white (dark) |

---

## Dark theme

| Item | Detail |
|------|--------|
| Toggle UI | `framework/PageMasthead/PageThemeSwitcher.tsx` |
| Activation | `document.documentElement.classList.add('pf-v6-theme-dark')` |
| Default setting | `system` (follows OS `prefers-color-scheme`) |
| Storage | `localStorage['user-preferences']` |
| Logo | `platform-logo.svg` (light) / `platform-logo-white.svg` (dark) |

### Dark palette (resolved hex)

| Usage | Hex |
|-------|-----|
| Page/content bg | `#292929` |
| Masthead/sidebar bg | `#151515` |
| Primary text | `#ffffff` |
| Subtle text | `#c7c7c7` |
| Border (app override) | `#505050` |

```html
<html class="pf-v6-theme-dark">
```

Monaco editor dark theme uses `#222222` (`framework/components/DataEditor.tsx`).

---

## Typography

| Role | Font family | Size |
|------|-------------|------|
| Body | `"Red Hat Text", Helvetica, Arial, sans-serif` | **14px** (`0.875rem`) — default |
| Headings | `"Red Hat Display", Helvetica, Arial, sans-serif` | h1 **24px** (`1.5rem`) |
| Monospace / logs | `"Red Hat Mono", Courier New, monospace` | 12–14px |

### PF font-size tokens

| Token | rem | px (@16px root) |
|-------|-----|-----------------|
| `--pf-t--global--font--size--body--default` | 0.875rem | 14px |
| `--pf-t--global--font--size--heading--h1` | 1.5rem | 24px |
| `--pf-t--global--font--size--heading--h2` | 1.25rem | 20px |
| `--pf-t--global--font--size--heading--h3` | 1.125rem | 18px |

Page header padding: `12–16px` vertical, `24px` horizontal (`framework/PageHeader.tsx`).

---

## UI region mapping

| UI region | Source file | Colors / notes |
|-----------|-------------|----------------|
| **Top masthead** | `framework/PageMasthead/PageMasthead.tsx` | Secondary bg `#f2f2f2`; logo link `light-dark(black, white)` |
| **Left sidebar** | PF `Page` sidebar | Secondary bg; nav subtle `#4d4d4d`, current `#151515` |
| **Page header** | `framework/PageHeader.tsx` | White bg; h1 title; description `opacity: 0.8` |
| **Breadcrumbs** | `PageHeader.tsx` | Link `#0066cc` |
| **Data tables** | PF `Table` + `PageFramework.css` | White bg; striped `#f2f2f2`; borders `#ccc` |
| **Primary button** | PF `Button variant="primary"` | Blue `#0066cc` |
| **Secondary button** | PF `Button variant="secondary"` | Blue outline `#0066cc` |
| **Danger button** | PF `Button variant="danger"` | `#b1380b` |
| **Status labels** | PF `Label color="..."` | Status tokens above |
| **Banners** | `platform/main/PlatformApp.tsx` | `color="red"`, `color="yellow"` |
| **Login page** | `frontend/common/AnsibleLogin/AnsibleLogin.tsx` | Dark bg image; footer white; links `#b9dafc` |
| **Job/log ANSI** | `frontend/common/Ansi.css` | Maps terminal colors to status tokens |
| **Logo SVG** | `platform/assets/platform-logo.svg` | `fill: currentColor` |

---

## Light-theme quick reference

```
#ffffff  page background
#f2f2f2  header / sidebar / striped rows
#151515  primary text
#4d4d4d  secondary text
#a3a3a3  disabled text
#cccccc  borders (app override)
#0066cc  brand / primary button / links
#004d99  brand hover
#3d7317  success
#b1380b  danger
#ffcc17  warning
#5e40be  info
#ee0000  Ansible red (logo only, not UI chrome)
```

## Dark-theme quick reference

```
#292929  page background
#151515  header / sidebar
#ffffff  primary text
#c7c7c7  subtle text
#505050  borders (app override)
```

---

## Key source files (ansible-ui-fork)

| File | Purpose |
|------|---------|
| `framework/PageFramework.css` | App-wide color overrides |
| `framework/components/pfcolors.tsx` | Status/link token mapping |
| `framework/PageHeader.tsx` | Page title and description layout |
| `framework/PageMasthead/PageMasthead.tsx` | Top bar styling |
| `framework/PageSettings/PageSettingsProvider.tsx` | Dark mode class toggle |
| `platform/main/PlatformMain.tsx` | PatternFly CSS imports |
| `frontend/common/AnsibleLogin/AnsibleLogin.tsx` | Login page colors |
| `frontend/common/Ansi.css` | Log/status terminal colors |
| `platform/assets/platform-logo.svg` | Logo (`currentColor`) |

---

## Gotchas for report builders

1. **Brand is blue, not Ansible red** — primary actions use `#0066cc`.
2. **Borders are `#ccc` in this app**, not stock PF `#c7c7c7` (override in `PageFramework.css`).
3. **Login page is a special case** — dark photographic background, not the standard white/gray shell.
4. **Prefer CSS variables** when loading PF CSS so dark mode works automatically.
5. **No `.scss` in ansible-ui** — styling is PF tokens plus a few `.css` overrides.
6. **Re-derive this doc** when `@patternfly/patternfly` version bumps in `ansible-ui-fork/package.json`.

---

## Minimal HTML skeleton

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>AAP Report</title>
  <link rel="stylesheet" href="https://unpkg.com/@patternfly/patternfly@6.3.1/patternfly-addons.css">
  <link rel="stylesheet" href="https://unpkg.com/@patternfly/patternfly@6.3.1/patternfly-base.css">
  <link rel="stylesheet" href="report-starter.css">
</head>
<body>
  <header class="report-masthead">
    <span class="brand">Ansible Automation Platform</span>
  </header>
  <header class="report-page-header">
    <h1>Report Title</h1>
    <p class="description">Optional subtitle or date range.</p>
  </header>
  <main class="report-content">
    <div class="report-card">
      <h2>Section</h2>
      <table class="report-table">
        <thead>
          <tr><th>Name</th><th>Status</th></tr>
        </thead>
        <tbody>
          <tr><td>Example</td><td class="status-success">Successful</td></tr>
        </tbody>
      </table>
    </div>
  </main>
</body>
</html>
```
