# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-page HTML app for designing an Azure Landing Zones (ALZ) Management Group hierarchy with subscriptions placed inside it. Imports/exports the `avm-ptn-alz` Terraform module's `*.alz_architecture_definition.json` format and several other targets.

## Run

Just open `index.html` in a browser. No build, no dependencies, no server.

## Stack — committed

- One file: `index.html` with embedded `<style>` + `<script type="module">`.
- Vanilla JS, hand-rolled CSS that mimics Microsoft Fluent / Azure portal.
- No bundler, no framework, no CDN — works offline (except live GitHub fetches; see below).
- Split to `app.js` / `app.css` siblings only if file grows past ~1500 lines.

## Data model

Single in-memory tree, source of truth for every export.

```js
state = {
  root: { id: 'tenant-root', type: 'root', name: 'Tenant root group', children: [...] },
  pool: [/* unplaced subscriptions */],
  meta: { orgName, archName, alzTag },
}
```

Node types: `root` | `mg` | `subscription`. MG nodes carry `archetypes: string[]` and `exists: bool`. Subscriptions carry `subscriptionId: string|null`. All mutations go through pure functions (`addMg`, `moveNode`, `placeSubscription`, `unplaceSubscription`, `deleteNode`, `updateNode`) and call `snapshot()` first → undo stack works for every action.

Persistence: `localStorage['alz-foundations:state']`, debounced 200ms.

## Layout

Tidy-tree (Reingold–Tilford simplified) for **MG nodes only**. Subscriptions are not in the tidy-tree pass — they're placed afterwards in a third layout pass.

Three passes in `layout()`:
1. **Width** (`first`): per MG subtree, sum of MG-children widths only (subs share parent's column, don't add width).
2. **Place MGs** (`placeMgs`): assign x,y for every MG via tidy-tree.
3. **Place subs** (`placeSubs`): compute `subStartY = maxMgBottom + V_GAP` (one row below the deepest MG anywhere in the tree). Every sub stack starts at this **uniform global Y** in its parent's x column. So a sub of a depth-2 MG starts at the same Y as a sub of a depth-3 MG — it just gets a longer trunk connector. This keeps the global "Subscriptions" yellow band as a clean horizontal strip below the MG band.

Sub cards have **variable height** via `estimateSubHeight(name)` (lines = ceil(text / ~17 chars), height = lines × 17px + 24px padding, min `NODE_H`). Stacked subs accumulate y by `prev._h + SUB_VGAP` (10px), not a fixed step. `_h` is stored on each sub node so `bbox()` and the global sub-band wrap them correctly.

Connector trunk: each parent draws **one** vertical line from its bottom-mid down to the **first** sub of its stack. Subsequent subs are visually adjacent in the same column — no per-sub line (cleaner, no overdraw).

**Subscription pool** is a fixed-position panel collapsed by default, slid off-screen via `transform: translateX(100%)`. A floating `.pool-toggle` button (top-right of canvas area) shows a count badge and toggles `.pool.open`. The toggle button doubles as a drop target for un-placing subscriptions (so you can un-place without opening the pool first). The pool itself has a close-X.

Z-stack inside `.canvas`: bands (`z-index: 1`) → connectors SVG (2) → node cards (3). Pool toggle (95) and pool (90) sit above the canvas. Side panel (200) and modals (300+) sit on top.

## ALZ Library integration (live GitHub)

Toolbar has a "Library" tag selector. App fetches:

- Tags: `GET /repos/Azure/Azure-Landing-Zones-Library/tags?per_page=100` (paginated up to 3 pages), filtered by prefix `platform/alz/`.
- Archetypes for tag T: `GET /repos/Azure/Azure-Landing-Zones-Library/contents/platform/alz/archetype_definitions?ref=<T>`. File names ending `.alz_archetype_definition.json` define the available archetype set.

Caching:
- `alz-foundations:tags` — 24h TTL.
- `alz-foundations:archetypes:<tag>` — no TTL (tags immutable).
- Refresh `↻` button busts cache.

Rate limit (GitHub unauth, 60/hr/IP) handled with toast + manual-entry fallback in side panel.

## Import / export

**Import** is one dialog (Browse, paste, or drop file) with auto-detection. File picker accepts `.json/.tf/.hcl/.tfvars`.

- `subscription_placement = {` token (HCL, not valid JSON) → AVM subs placement. Each entry parsed via regex (`(\w+) = { subscription_id = "…" management_group_name = "…" }`). Each sub is created with `name = key`, `subscriptionId = subscription_id`, then placed under the MG whose `id` (case-insensitive) or `display_name` matches `management_group_name`. Unmatched ones go to the pool. Toast reports placed/unplaced counts.
- `management_groups[]` present → ALZ architecture definition (avm-ptn-alz). Builds tree from flat list using `parent_id` links. Tag/archetype/exists fields preserved on MGs. Subscriptions are not part of this format.
- `root` present → native JSON model with full tree + pool round-trip.

**Exports** all read from the same tree:
- Visual: SVG (re-rendered from layout, not DOM-serialized) and PNG (rasterized via canvas).
- Native JSON (round-trippable).
- Bicep — `Microsoft.Management/managementGroups@2021-04-01` + `…/subscriptions@2020-05-01`. Targets `tenant` scope.
- Terraform — `azurerm_management_group` + `azurerm_management_group_subscription_association`.
- Mermaid `graph TD` + indented outline.
- ALZ architecture definition — flat `management_groups[]`. Skips subscriptions (format has none) with a toast.
- AVM subs placement (HCL) — `subscription_placement = { … }` block. Key = slugified sub name (lowercase, alphanumeric+underscore, deduped with numeric suffix). `subscription_id` from sub node (placeholder UUID + `# TODO` comment if unset). `management_group_name` = parent MG's `id`. Unplaced subs are listed as comments only.

## Archetype X-ray (live policy heat map)

For every MG with an archetype assigned and a Library tag selected, the app fetches the full archetype definition from `raw.githubusercontent.com/Azure/Azure-Landing-Zones-Library/<tag>/platform/alz/archetype_definitions/<name>.alz_archetype_definition.json`. Cached under `alz-foundations:archetype-def:<tag>||<name>` (no TTL — tags immutable). Refresh button invalidates the cache for the active tag.

- **Canvas badge**: each MG with archetype shows a `Np` badge in the upper-right (N = effective policy_assignments count, summed from own + ancestor archetypes). Color heat scales from white (0) → light blue → deep blue (max in tree). Hover reveals breakdown tooltip with own + each ancestor's contribution.
- **Side panel X-ray section** (`#sp-xray`): expandable lists for `policy_assignments`, `policy_definitions`, `policy_set_definitions`, `role_assignments`, `role_definitions` with item counts per section. Below: blue effective-count card + collapsible inheritance trail.
- Loading badges show `…`; failed fetches degrade silently (no badge).

raw.githubusercontent.com is on a separate quota from the API (60/hr/IP unauth) — no extra rate-limit handling needed.

## Command palette

`Ctrl/Cmd+K` (or `⌘K` toolbar button) opens fuzzy-search palette. Three sections in results:

- **Management groups** — blue `mg` icon, breadcrumb path (`Contoso › Platform`). Selecting jumps + scrolls card into view, opens side panel.
- **Subscriptions** — yellow `sub` (key) icon, breadcrumbs (or `unplaced pool`). Selecting opens side panel for that sub.
- **Actions** — neutral icon. Includes Load reference architecture, New MG/Sub, Undo, Clear, Import, all Export targets.

Scoring: substring match first (rank by position), then subsequence with word-boundary bonus. Match characters wrapped in `<mark>`. Arrow keys navigate, Enter runs, Esc closes. Ctrl+K works even from inside form fields (palette intentionally bypasses `inField` gate).

## Decisions worth knowing

- **Library tag is global**, stored in `state.meta.alzTag`. One ALZ Library version per architecture. Per-MG tag was considered and rejected.
- **Single archetype per MG** in the UI (stored as a one-element array because the schema requires array). Picker is strict to the fetched list, with a free-text fallback shown only when offline / fetch failed / stored value is not in current list.
- **Tag is NOT emitted in the exported ALZ JSON** — schema doesn't define it. Tag lives only in localStorage.
- **Reference architecture loader** uses PascalCase IDs (e.g. `Platform`, `LandingZones`) to match avm-ptn-alz examples, and pre-populates archetypes (`platform`, `landing_zones_override`, `connectivity_override`, `corp`, etc.).
- **Subscriptions are tree leaves only**; an MG can never be dropped into its own descendant (cycle check in `onDragOverMg`).

## Keyboard

- Click node: select + open side panel. Click empty canvas: deselect **and close side panel**.
- **Ctrl/Cmd+K** open command palette (works from form fields too).
- **Ctrl/Cmd+C** copy selected node (deep, with subtree).
- **Ctrl/Cmd+V** paste clone with fresh IDs under currently selected MG.
- **Delete / Backspace** remove selected node + subtree (undoable).
- **Ctrl/Cmd+Z** undo (covers every mutation).
- **Esc** close panels, palette, modal + deselect.

All shortcuts except `Ctrl/Cmd+K` are gated by `!inField` so they don't fire while typing in inputs.

## Out of scope (confirm before adding)

- Policies, role assignments, blueprints.
- Real Azure auth or deployment — designs only.
- Multi-tenant or multi-root.
