# Joplin Bullet Point Shortcut

A [Joplin](https://joplinapp.org/) plugin that adds a keyboard shortcut to toggle unordered bullet points on selected lines in the Markdown editor — just like `Ctrl+B` for bold or `Ctrl+I` for italic.

| Shortcut | Action |
|---|---|
| `Ctrl + Shift + 8` | Toggle `- ` bullet points on selected line(s) |

> `Ctrl+Shift+8` = `Ctrl+*` on a US keyboard — the star/bullet key.

---

## Features

- **Toggle behavior** — press once to add bullets, press again on the same lines to remove them
- **Multi-line support** — select any number of lines and bullet all of them at once
- **Indentation-aware** — inserts `- ` after existing leading whitespace, so nested lists stay nested
- **Blank lines skipped** — blank lines in a selection are left untouched
- **Single undo step** — the entire operation is one `Ctrl+Z` action
- Works in **Joplin 3.x** (CodeMirror 6 editor)

---

## Installation

### Option A — Install the pre-built `.jpl`

1. Download `joplin.plugin.bulletshortcut-v1.1.0.jpl` from the [Releases](../../releases) page
2. On Windows: right-click the file → **Properties** → check **Unblock** → OK
3. In Joplin: **Tools → Options → Plugins → Install from file**
4. Select the `.jpl` file and restart Joplin

### Option B — Build from source

Prerequisites: [Node.js](https://nodejs.org/) v14+

```powershell
git clone https://github.com/YOUR_USERNAME/joplin-bullet-shortcut
cd joplin-bullet-shortcut
npm install
npm run dist
```

This produces `joplin.plugin.bulletshortcut-v1.1.0.jpl` in the project root. Install it as above.

---

## Usage

1. Open a note in Joplin's **Markdown editor** (not Rich Text)
2. Click on a line, or select multiple lines
3. Press `Ctrl+Shift+8`

**Adding bullets:**
```
Before:              After:
  Line one             - Line one
  Line two             - Line two
  Line three           - Line three
```

**Removing bullets** (press again when all selected lines already have bullets):
```
Before:              After:
  - Line one             Line one
  - Line two             Line two
  - Line three           Line three
```

**Nested lists are preserved:**
```
Before:              After:
  Line one             - Line one
    Sub-item               - Sub-item
  Line two             - Line two
```

### Changing the shortcut

Edit `src/contentScript.js` and change the key string:

```js
codeMirrorOptions: {
  extraKeys: {
    'Ctrl-Shift-8': 'toggleBulletPoints',  // ← change this
  },
},
```

Common alternatives: `'Ctrl-Shift-L'` (L for List), `'Ctrl-Shift-U'` (U for Unordered).

Then rebuild: `npm run dist`.

---

## Project Structure

```
joplin-bullet-shortcut/
├── manifest.json          # Plugin metadata (id, version, app_min_version)
├── package.json
├── tsconfig.json
├── webpack.config.js
├── api/
│   ├── index.js           # Runtime stub — exposes the joplin global
│   ├── index.d.ts         # TypeScript type declarations
│   └── types.d.ts         # ContentScriptType enum types
├── scripts/
│   └── dist.js            # Packaging script — produces the .jpl tar archive
└── src/
    ├── index.ts           # Plugin entry point — registers the content script
    └── contentScript.js   # CodeMirror 6 key binding and toggle logic
```

---

## How It Works

The plugin has two parts:

**`src/index.ts`** — The plugin entry point. When Joplin starts, it calls `joplin.plugins.register({ onStart })`. Inside `onStart`, we register a CodeMirror content script:

```typescript
await joplin.contentScripts.register(
  'codeMirrorPlugin',
  'bulletPointContentScript',
  './contentScript.js'
);
```

**`src/contentScript.js`** — Injected into Joplin's CodeMirror 6 editor. It receives the CM6 module bundle and returns a keymap extension:

```js
return [
  cm.Prec.high(
    cm.keymap.of([{ key: 'Ctrl-Shift-8', run: toggleBulletPoints }])
  )
];
```

The `toggleBulletPoints` function uses CM6's `EditorState` and `view.dispatch()` to read the selected lines, decide whether to add or remove bullets based on whether all lines already have them, and apply the changes as a single transaction.

---

## Development Notes — What We Learned Building This

This plugin was trickier than expected. Here's a complete log of the non-obvious discoveries, in the order we hit them, so you don't have to:

### 1. `.jpl` files are TAR archives, not zip files

Joplin's plugin installer uses a tar parser internally. Creating the package with `zip` produces `invalid base256 encoding` error and the plugin silently fails to install. Use `tar -cf` and list files explicitly to avoid a leading `./` prefix in the archive paths.

### 2. `joplin` is a global variable, not a CommonJS module

This was the hardest one. Every Joplin plugin tutorial says `import joplin from 'api'` — and the TypeScript compiles fine. But in Joplin's plugin sandbox, `api` is not a real Node.js module. The `joplin` object is injected as a **global variable**.

The correct runtime stub is:

```js
// api/index.js
Object.defineProperty(exports, '__esModule', { value: true });
exports.default = joplin;  // 'joplin' is a global in Joplin's plugin context
```

The stub must be **bundled into** the plugin output (not marked as `externals` in webpack). Using `externals: { api: 'api' }` generates `require('api')` in the bundle, which throws silently and prevents the plugin from starting — with no error in the log.

We only discovered this by extracting a known-working plugin's `index.js` from its cache folder and comparing the bundle structure.

### 3. No `libraryTarget: 'commonjs2'` and no `module.exports`

Official Joplin plugins use a plain IIFE (`(()=>{...})()`) with no `module.exports` at the end. The plugin registers itself as a **side effect** when the IIFE runs — `joplin.plugins.register()` is called during module evaluation. Webpack's `libraryTarget: 'commonjs2'` adds `module.exports = ...` which is unnecessary and may confuse Joplin's plugin runner.

### 4. Joplin 3.x uses CodeMirror 6 — the CM5 API does nothing

Joplin 3.x migrated to CodeMirror 6. The old CM5 approach:

```js
// ❌ CM5 — does nothing in Joplin 3.x
plugin: function(CodeMirror) {
  CodeMirror.commands.myCommand = function(cm) { ... };
},
codeMirrorOptions: { extraKeys: { 'Ctrl-Shift-8': 'myCommand' } }
```

The CM6 approach:

```js
// ✅ CM6 — works in Joplin 3.x
plugin: async function(cm) {
  return [
    cm.Prec.high(cm.keymap.of([{ key: 'Ctrl-Shift-8', run: myCommand }]))
  ];
}
```

Where `cm` is the CM6 module bundle that Joplin passes to the content script.

### 5. The manifest must have no empty string fields

`"homepage_url": ""` causes silent validation failure in Joplin 3.x — the plugin loads (manifest is parsed) but never starts. Either omit the field or provide a real URL. Also, `app_min_version` must be `"2.1"` or higher for Joplin 3.x to accept the plugin.

### 6. Windows blocks downloaded files

Windows marks files downloaded from the internet as untrusted. Joplin's file dialog will open and accept the file, but then silently do nothing. Fix: right-click the `.jpl` → **Properties** → check **Unblock**.

### 7. Debugging strategy that worked

When a plugin installs but silently doesn't start, open **Help → Toggle Developer Tools → Console** in Joplin before installing. Real-time errors appear there that never make it to `log.txt`. The most useful signal was comparing our plugin's log entries against a working plugin — ours had `PluginService: Loading plugin` but was missing `joplin.plugins: Starting plugin`, which narrowed the problem to module evaluation failure.

---

## Compatibility

| Joplin version | Status |
|---|---|
| 3.x (CodeMirror 6) | ✅ Supported |
| 2.x (CodeMirror 5) | ❌ Not supported |

---

## License

MIT
