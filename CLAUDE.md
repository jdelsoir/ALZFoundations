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

Tidy-tree (Reingold–Tilford simplified). Y-coordinate forced for all subscriptions to a single bottom row (`maxMgDepth + 1`) regardless of where they live in the tree — so the yellow "Subscriptions" band never overlaps leaf MGs.

Z-stack inside `.canvas`: bands (`z-index: 1`) → connectors SVG (2) → node cards (3).

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

**Import** is one dialog (Browse, paste, or drop file) with auto-detection:
- `management_groups[]` present → ALZ architecture definition (avm-ptn-alz). Builds tree from flat list using `parent_id` links. Tag/archetype/exists fields preserved on MGs. Subscriptions are not part of this format.
- `root` present → native JSON model with full tree + pool round-trip.

**Exports** all read from the same tree:
- Visual: SVG (re-rendered from layout, not DOM-serialized) and PNG (rasterized via canvas).
- Native JSON (round-trippable).
- Bicep — `Microsoft.Management/managementGroups@2021-04-01` + `…/subscriptions@2020-05-01`. Targets `tenant` scope.
- Terraform — `azurerm_management_group` + `azurerm_management_group_subscription_association`.
- Mermaid `graph TD` + indented outline.
- ALZ architecture definition — flat `management_groups[]`. Skips subscriptions (format has none) with a toast.

## Decisions worth knowing

- **Library tag is global**, stored in `state.meta.alzTag`. One ALZ Library version per architecture. Per-MG tag was considered and rejected.
- **Single archetype per MG** in the UI (stored as a one-element array because the schema requires array). Picker is strict to the fetched list, with a free-text fallback shown only when offline / fetch failed / stored value is not in current list.
- **Tag is NOT emitted in the exported ALZ JSON** — schema doesn't define it. Tag lives only in localStorage.
- **Reference architecture loader** uses PascalCase IDs (e.g. `Platform`, `LandingZones`) to match avm-ptn-alz examples, and pre-populates archetypes (`platform`, `landing_zones_override`, `connectivity_override`, `corp`, etc.).
- **Subscriptions are tree leaves only**; an MG can never be dropped into its own descendant (cycle check in `onDragOverMg`).

## Keyboard

- Click node: select + open side panel. Click empty canvas: deselect.
- **Ctrl/Cmd+C** copy selected node (deep, with subtree).
- **Ctrl/Cmd+V** paste clone with fresh IDs under currently selected MG.
- **Delete / Backspace** remove selected node + subtree (undoable).
- **Ctrl/Cmd+Z** undo (covers every mutation).
- **Esc** close panels + deselect.

All shortcuts gated by `!inField` so they don't fire while typing in inputs.

## Out of scope (confirm before adding)

- Policies, role assignments, blueprints.
- Real Azure auth or deployment — designs only.
- Multi-tenant or multi-root.
