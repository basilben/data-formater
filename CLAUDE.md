# Data Formatter — Project Guide

## What this is

A single-file PWA (`index.html`) for parsing, visualising, and exploring XML and JSON data. No build system, no dependencies, no npm. Everything — HTML, CSS, JavaScript — lives in `index.html`.

## Files

| File            | Purpose                                   |
| --------------- | ----------------------------------------- |
| `index.html`    | The entire app (HTML + CSS + JS)          |
| `sw.js`         | Service Worker — cache-first PWA strategy |
| `manifest.json` | PWA manifest (name, icons, theme colour)  |
| `icon.svg`      | App icon                                  |

## Running locally

Serve with any static file server — the Service Worker requires `https://` or `http://localhost`:

```bash
npx serve .          # or
python3 -m http.server 3000
```

Then open `http://localhost:3000`.

## Architecture — `index.html`

### Key globals (declared at top of `<script>`)

| Variable                       | What it is                          |
| ------------------------------ | ----------------------------------- |
| `dataInput`                    | The `<textarea>`                    |
| `treeContainer`                | Right-panel tree mount point        |
| `typeBadge`                    | XML / JSON / — badge element        |
| `statsBar`, `statSummary`      | Bottom stats strip                  |
| `tabsData`                     | `Map<id, TabState>` — all tab state |
| `activeTabId`                  | Currently active tab id             |
| `modalHistory`, `modalCurrent` | Tile modal navigation stack         |

### Two badge setters (both do the same thing — keep both)

- `setBadge(label, cls)` — used in `parseXML`, `parseJSON`, `updateTypeBadge`
- `setbadge(label, cls)` — used in sample buttons, `btnClear`

### Parse flow

```
btnParse → parse() → parseXML() | parseJSON()
                         ↓
              buildXmlNode() / buildJsonNode()   (recursive, builds DOM tree)
                         ↓
              treeContainer.appendChild(frag)
                         ↓
              updateActiveTabLabel()
```

### Tile modal flow

```
.tile-btn click → openTileModal(node, label, type)
                         ↓
                   renderModal()
                         ↓
              buildXmlTileGrid() / buildJsonTileGrid()
                         ↓
              buildXmlTileCard() / buildJsonTileCard()

Navigating deeper: navigateModal() — pushes to modalHistory, re-renders
Back button:        modalHistory.pop(), re-renders
```

### Tab system

Each tab stores `{ id, label, input, badgeLabel, badgeClass, inputHint, statsHtml, statsVisible, treeContent }`.
`treeContent` is a **detached `<div>`** that holds live DOM nodes (with all event listeners). On tab switch, nodes are physically moved in/out of `treeContainer` — never cloned or serialised.

```
switchTab(id)
  clearSearch() + closeTileModal()
  saveCurrentTabState()   → moves DOM nodes from treeContainer → tab.treeContent
  restoreTabState(id)     → moves DOM nodes from tab.treeContent → treeContainer
```

### Tile card kv-list rules (XML)

A card shows an inline key-value list when **all direct children are leaves** (`childElements.every(c => c.children.length === 0)`).
Each kv row shows:

1. **Attributes** of the child element (amber name, red value — matching tree syntax colours)
2. **Text content** of the child element (if present)

### CSS colour tokens

All colours are CSS custom properties on `:root` (dark) and `body.light` (light mode). Syntax colours:

- `--c-tag` — element/key names (blue)
- `--c-attr` — attribute names (amber)
- `--c-val` — attribute values (red)
- `--c-str / --c-num / --c-bool / --c-null` — JSON value types

## Critical gotchas

### Browser cache (PWA Service Worker)

The SW serves `index.html` from cache. After editing, **always force-reload** in the preview:

```javascript
location.href = location.href.split("?")[0] + "?v=" + Date.now();
```

Or use DevTools → Application → Service Workers → "Update on reload".

### Never use `innerHTML = ''` on `treeContainer` between tab operations

The tab system moves live DOM nodes. Calling `treeContainer.innerHTML = ''` destroys all event listeners on those nodes. Only `restoreTabState` and `saveCurrentTabState` should touch `treeContainer` children directly.

### `showEmpty()` writes to `treeContainer`

After `showEmpty()` the treeContainer contains a fresh `empty-state` div (not the tab's saved nodes). Only call it when truly resetting a tab (e.g. Clear button, new empty tab).

## Verification workflow (after any edit)

1. Reload with cache-bust: `location.href = ... + '?v=' + Date.now()`
2. Check browser console for errors
3. Load XML Sample and/or JSON Sample → Format
4. Spot-check the affected feature
5. Confirm no console errors

## Context upudate

1. After making each change update the CLAUDE.md file with the changes made and the verification workflow (if any).
