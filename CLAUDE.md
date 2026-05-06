# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single self-contained HTML file that renders Terraform plan JSON output as an interactive viewer. No build step, no dependencies fetched at runtime — everything (HTML, CSS, JS) lives in one file. Open in browser, drag-drop or paste a plan JSON, explore.

Inspiration:
- https://github.com/braedonsaunders/codeflow — visual style reference (but simpler scope)
- https://github.com/im2nguyen/rover — resource graph and change-type UX patterns

**External deps (acceptable):**
- `https://fonts.googleapis.com` — JetBrains Mono font (loaded in `<head>`)
- Tailwind via CDN `play.tailwindcss.com` is acceptable if needed for styling

## Terraform Plan JSON Shape

Key fields in a `terraform show -json` plan output:

```
resource_changes[].change.actions       // ["create"] | ["delete"] | ["update"] | ["no-op"] | ["delete","create"]
resource_changes[].address              // "module.foo.aws_instance.bar"
resource_changes[].type                 // "aws_instance"
resource_changes[].name
resource_changes[].module_address       // present if inside a module, else absent/""
resource_changes[].provider_name        // e.g. "registry.terraform.io/hashicorp/aws"
resource_changes[].action_reason        // human-readable reason string, optional
resource_changes[].change.before        // null on create
resource_changes[].change.after         // null on delete
resource_changes[].change.after_unknown // same shape as after, `true` at leaves not yet known
resource_changes[].change.replace_paths // array of paths that triggered replace, optional
```

Action combinations to handle: `create`, `read`, `update`, `delete`, `replace` (delete+create or create+delete), `no-op`.

`canonicalAction(actions[])` maps raw action arrays → one of those six strings.

## Architecture

Single `index.html`. Internal structure (order in file):

1. **CSS** — `<style>` block with all design tokens and component styles
2. **HTML** — three-pane layout: sidebar | resource list | detail panel; plus overlay modals
3. **JS** — one IIFE `<script>` at bottom; strict mode; no modules

### JS Module Structure

All logic lives inside one `(() => { 'use strict'; ... })()` IIFE. Key sections (annotated with `// =====` banners):

| Section | Purpose |
|---------|---------|
| **Constants & state** | `ACTIONS`, `ACTION_CLASS` maps, module-level `plan`, `filters`, `selectedAddress`, `detailTab`, `showUnchanged`, `derived` |
| **ingest** | `ingest(text)` — parse JSON, validate, set `plan`, call `derive()` + `render()` |
| **filter** | `visibleResources()` — apply all active filters, return filtered array |
| **diff** | `diffWalk()`, `compareLeaf()`, `deepEqual()`, `fmtVal()` — recursive attribute diff |
| **render** | `render()` + six sub-renderers (summary, actions, modules, types, list, detail) |
| **events** | DOM event listeners wired at bottom of IIFE; `render()` called once at end |

### State Model

```js
plan            // raw parsed JSON object, null until loaded
filters = {
  actions: Set, // active action types
  search: '',   // address substring filter
  modules: Set, // active module_address values (empty = all)
  types: Set,   // active resource type values (empty = all)
}
selectedAddress // string | null — currently selected resource address
detailTab       // 'diff' | 'before' | 'after' | 'raw'
showUnchanged   // bool — whether to show unchanged attrs in diff view
derived = {
  actionCounts: {},   // { create: N, update: N, ... }
  modules: [[addr,n]],
  types: [[type,n]],
  total: N,
}
```

`derive()` recomputes `derived` from `plan`. Called once after ingest.

### DOM Helper

`el(tag, attrs, children)` — minimal createElement helper:
- `attrs.class` → `node.className`
- `attrs.on` → `{ eventName: handler }` object
- `attrs.data` → dataset values
- `attrs.html` → innerHTML (avoid for user content)
- Other attrs → `setAttribute`
- Children: string | Element | null (null skipped)

### Render Pattern

Each `render*()` function clears its root element with `innerHTML = ''` then rebuilds from state. No virtual DOM, no diffing — full re-render on any state change. `render()` calls all six sub-renderers. `renderList()` and `renderDetail()` are also called independently for performance (e.g., filter changes don't need to re-render the sidebar).

## Layout

Three-pane grid:

```
.app  (grid: 48px topbar / 1fr)
  .topbar  — logo, summary stats, Load/Paste buttons
  .panes   (grid: 280px sidebar | 1fr resource list | 440px detail)
    .pane.sidebar   — search, action checkboxes, module list, type list
    .pane (main)    — .reslist header + scrollable .res-row list
    .pane.detail    — tabs (Diff/Before/After/Raw JSON) + content
```

Overlays (z-index > main panes):
- `.dropzone-overlay` (z:100) — initial load screen; also shown on file drag
- `.modal-overlay` (z:200) — paste JSON modal

## Design System

Dark theme, monospace font throughout. CSS custom properties in `:root`:

```css
--bg0: #0a0a0c      /* page background */
--bg1: #0f0f12      /* panel/sidebar background */
--bg2: #15151a      /* input/code background */
--bg3: #1c1c21      /* unused, available */
--border: #2d2d35
--border-soft: #1f1f25
--text: #f0f0f2
--text-dim: #8b8b95
--text-faint: #5a5a63
--accent: #00ff9d   /* green highlight, focus rings, active state */
--accent-dim: rgba(0,255,157,0.12)

/* action colours */
--act-create:  #22c55e
--act-update:  #f59e0b
--act-delete:  #ef4444
--act-replace: #a855f7
--act-noop:    #6b7280
--act-read:    #3b82f6
```

Color conventions: green = create, red = delete, yellow/amber = update, purple = replace, grey = no-op, blue = read.

Badge classes: `.badge.create|update|delete|replace|noop|read`

## Style Constraints

- No external CDN links except the font and optional Tailwind (see above).
- Vanilla JS only — no frameworks, no bundler.
- All CSS inline in `<style>`. All JS inline in `<script>`.
- New UI components: follow existing `el()` + CSS class pattern. No inline style for layout — add a CSS class.

## Dev Workflow

Open `index.html` directly in browser. No server needed.

To generate a test plan JSON:
```bash
# In any Terraform project
terraform plan -out=tfplan
terraform show -json tfplan > plan.json
```

Test fixture lives at `fixtures/sample.json`. Use it when verifying rendering behaviour.

## Key Patterns for New Features

**Adding a new filter dimension:**
1. Add field to `filters` object
2. Update `visibleResources()` to check it
3. Add `render*()` function for the sidebar section
4. Call new renderer from `render()` and wherever filters change

**Adding a new detail tab:**
1. Add tab label to the `for` loop in `renderDetail()`
2. Add `else if (detailTab === 'newtab')` branch in content section
3. Write a `render*Tab(rc)` function returning an Element or DocumentFragment

**Adding a new action type:**
1. Add to `ACTIONS` array and `ACTION_CLASS` map
2. Add `--act-newtype` CSS variable
3. Add `.badge.newtype` CSS rule
4. Update `canonicalAction()` if needed

**Modifying the diff:**
- `diffWalk(before, after, unknown, path, out)` pushes `{ path, kind, before, after }` objects
- `kind` values: `'added' | 'removed' | 'changed' | 'unchanged' | 'unknown'`
- `renderDiff(rc)` consumes the `out` array — modify there for display changes

## Files

```
index.html          — entire application
fixtures/
  sample.json       — minimal Terraform plan for manual testing
CLAUDE.md           — this file
```

Screenshots (`*.png`) in root are dev artefacts, not part of the app.
