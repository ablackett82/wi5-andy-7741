# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Where In 5?** is a single-file PWA — a shared household inventory app for a family of 5. Users ask "where are the batteries?" via voice or text, and the app either answers locally (fuzzy search) or calls Claude to interpret and act on the query.

The entire application lives in `index.html`. There is no build step, no bundler, no package.json, and no test suite. Changes are made directly to `index.html` and opened in a browser.

## Running locally

```bash
# Any static file server works; Python is usually available:
python3 -m http.server 8080
# Then open http://localhost:8080
```

The service worker (`sw.js`) caches the app shell. If you change `index.html` during development and the browser serves stale content, open DevTools → Application → Service Workers → "Unregister", then hard-reload.

## Architecture

Everything is in one `<script>` block in `index.html`, structured in labelled sections:

| Section | What it does |
|---|---|
| **STATE + STORAGE** | `state` object, `localStorage` keys, `loadConfig()`, `saveLocalInventory()` |
| **JSONBIN SYNC** | `syncWithJSONBin()` — GET→merge→PUT cycle. Conflict resolution: last-write-wins by `lastModified` timestamp; deletes are tombstones (`deleted: true`), not real deletes |
| **VOICE** | Web Speech API (`SpeechRecognition` / `speechSynthesis`). `r.continuous = false` is intentional — iOS Safari requires it |
| **FUSE SEARCH** | `fuse.js` fuzzy search across item names and location strings. `rebuildFuse()` must be called after any inventory mutation |
| **INTENT CLASSIFICATION** | Regex-based `classifyIntent()` decides FIND / STORE / DELETE / UPDATE / UNCLEAR before hitting Claude |
| **LOCAL FIND** | `handleFindLocal()` answers FIND queries from the local index. Falls through to Claude only if local search is ambiguous or empty |
| **CLAUDE API** | Direct browser fetch to `https://api.anthropic.com/v1/messages` (requires `anthropic-dangerous-direct-browser-access: true` header). Uses `claude-haiku-4-5`, forced tool-use (`tool_choice: {type: 'any'}`), max 500 tokens |
| **CLAUDE TOOLS** | 5 tools: `store_item`, `delete_item`, `update_item_location`, `find_item`, `ask_clarification` |
| **QUERY HANDLING** | `handleUserQuery()` — main entry point. FIND → try local → fall to Claude. All others go straight to Claude |
| **INVENTORY TREE** | Recursive location tree built from `item.location` arrays (1–4 levels, Title Case). Top-level groups expanded by default |
| **MODALS** | Add/edit modal has cascading location pickers — each picker level rebuilds on change |
| **SETTINGS** | API keys stored in `localStorage` under `wherein5.*` keys |

## Data model

Each item:
```json
{
  "id": "1718000000000",
  "name": "batteries",
  "location": ["Kitchen", "Drawer", "Top"],
  "added": "2024-06-10T10:00:00.000Z",
  "lastModified": "2024-06-10T10:00:00.000Z",
  "deleted": false
}
```

Inventory is stored as `{ items: [...] }` in both `localStorage` and JSONBin. Deleted items are kept as tombstones with `deleted: true` so they survive a merge with a remote copy.

## Key constraints

- **No build step** — plain JS, no TypeScript, no modules. `'use strict'` at the top of the script block.
- **iOS Safari compatibility** — `SpeechRecognition.continuous` must be `false`; safe-area insets used throughout CSS.
- **API keys in browser** — the Anthropic key is stored in `localStorage` and sent directly from the browser. The `anthropic-dangerous-direct-browser-access: true` header is required and intentional.
- **Service worker cache** — bump `CACHE_NAME` in `sw.js` when changing cached assets, or old clients will serve stale files.
- **Default rooms** — `DEFAULT_ROOMS` (`Lounge`, `Kitchen`, `Landing`, `Loft`, `Joss Cupboard`, `Landing Cupboard`) appear as options at level 0 of the location picker. Update this array to change the defaults.
- **Claude model** — currently `claude-haiku-4-5`. Change in `callClaude()` if switching models.
