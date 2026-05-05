# ALZ Foundations

A single-page, offline-friendly designer for Azure Landing Zones (ALZ) Management Group hierarchies. Drag, drop, and shape your tenant's MG tree in the browser, then export it to the format your deployment tooling actually speaks.

No build step. No framework. No CDN. Just open the file.

## Quick start

```bash
open index.html
```

That's it. The app runs entirely in the browser. Your work is saved to `localStorage` automatically, so closing the tab won't lose your design.

If you want a starting point instead of a blank canvas, click **Load reference architecture** in the toolbar — you'll get the canonical ALZ tree (Platform / Landing zones / Decommissioned / Sandbox) with subscriptions already placed.

## What it does

- **Design** an MG hierarchy visually with drag-and-drop. Subscriptions sit in a side pool until you place them under an MG.
- **Pick archetypes** for each MG from the official [Azure Landing Zones Library](https://github.com/Azure/Azure-Landing-Zones-Library), fetched live from GitHub. Choose the library tag once per architecture (e.g. `platform/alz/2026.04.2`) and the archetype list updates accordingly.
- **Import** existing designs from either the `avm-ptn-alz` Terraform module's `*.alz_architecture_definition.json` format or this app's native JSON. The dialog auto-detects which one you fed it.
- **Export** the same tree to whichever format you need:
  - **SVG / PNG** — visual diagram for slides and docs.
  - **Native JSON** — round-trippable, includes the subscription pool.
  - **Bicep** — `Microsoft.Management/managementGroups` resources at tenant scope.
  - **Terraform** — `azurerm_management_group` + subscription association resources.
  - **Mermaid** — `graph TD` plus an indented text outline.
  - **ALZ architecture definition** — flat `management_groups[]` JSON for the `avm-ptn-alz` module.

## Keyboard

| Shortcut | Action |
|---|---|
| Click node | Select + open side panel |
| Click empty canvas | Deselect |
| <kbd>Ctrl/Cmd</kbd>+<kbd>C</kbd> | Copy selected node (with subtree) |
| <kbd>Ctrl/Cmd</kbd>+<kbd>V</kbd> | Paste under currently selected MG (fresh IDs) |
| <kbd>Delete</kbd> / <kbd>Backspace</kbd> | Remove selected node + subtree |
| <kbd>Ctrl/Cmd</kbd>+<kbd>Z</kbd> | Undo |
| <kbd>Esc</kbd> | Close panels + deselect |

Shortcuts are disabled while you're typing in an input field.

## How it's built

One file: `index.html` with embedded `<style>` and `<script type="module">`. Vanilla JavaScript, hand-rolled CSS that imitates Microsoft Fluent / the Azure portal. No bundler, no React, no Tailwind. Works fully offline — the only network call is fetching the ALZ Library from GitHub when you change the tag, and that result is cached in `localStorage`.

The tree is laid out with a simplified Reingold–Tilford tidy-tree algorithm. Subscriptions are forced to a single bottom row regardless of where they live in the tree, which keeps the yellow "Subscriptions" band from overlapping leaf MGs.

State lives in a single in-memory tree. Every mutation is snapshotted, so undo covers every action — drag-drop, paste, delete, edit, import.

## Notes & limitations

- **GitHub rate limit.** Unauthenticated GitHub API calls are capped at 60 per hour per IP. If you hit it, the side panel falls back to a free-text archetype field. Tag lists are cached for 24 hours; archetype lists are cached indefinitely (tags are immutable).
- **The library tag is not part of the exported ALZ JSON.** The schema doesn't define it, so it lives only in `localStorage`. Pick it again on a fresh machine if you need to.
- **One archetype per MG** in the UI, even though the schema allows an array. This matches how real ALZ designs are written and keeps the picker simple.
- **Subscriptions are leaves only.** You can't nest an MG inside a subscription, and you can't drop an MG into its own descendant.
- **Out of scope (for now):** Azure policies, role assignments, blueprints, real authentication, deployment, multi-tenant or multi-root designs.

## License

Add one before sharing publicly.
