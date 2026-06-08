# Playbook → Document Editor Layout

## Goal

Redesign the playbook detail/edit page to feel like a document editor (Google Docs):

- Instructions body = main full-height/width column (the "document")
- Right side panel holds all other info (trigger, description, status, agents, metadata)
- Right side also surfaces **available actions** in more detail

## Acceptance criteria

- Two-pane layout: large document canvas (left) + sticky right sidebar
- Big editable title at top of the document canvas
- Instructions editor fills the available height
- Right sidebar: Details (trigger/description/status), Agents, Available Actions, Metadata + delete
- Save / Publish / History live in a top toolbar
- All existing behavior preserved (auto-save, versioning, unsaved-changes, create vs edit)
- Typecheck passes

## Tasks

- [x] Explore current page + components
- [x] Add `minHeight` + `bare` props to InstructionsTiptap (let body grow)
- [x] New `PlaybookActionsPanel.vue` — lists MCP tools grouped by plugin
- [x] Restructure `pages/playbooks/[...id].vue` into two panes + toolbar
- [x] Add i18n keys (en + pt-BR)
- [x] Verify: typecheck passes (exit 0)

## Results

- `dashboard/pages/playbooks/[...id].vue` — full-width two-pane layout: document
  canvas (big editable title + body editor) left, sticky sidebar right; Save/Publish/
  History/auto-save moved into the top toolbar. Back button now runs the unsaved-changes
  guard. Added scoped layout styles (stacks under 1024px).
- `dashboard/components/InstructionsTiptap.vue` — new `minHeight` + `bare` props so the
  body fills the canvas and blends into the "paper".
- `dashboard/components/playbooks/PlaybookActionsPanel.vue` — new: browsable list of
  available actions (MCP tools) grouped by plugin, with labels via useToolLabel.
- i18n: new `editor.*` keys in en + pt-BR playbooks.json.

## Working notes

- Page uses `<Page width=...>`; switching to `width="full"` for the editor.
- Actions today only exist as @-mentions inside the editor; the new panel makes them browsable.
- First pass — paper styling to be refined with the user.
