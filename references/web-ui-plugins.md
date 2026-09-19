# The dual-face plugin, end to end: from zero to a working web-UI plugin

> The main reference of the `dsh-plugin-dev-notes` skill. All code comes from the field-verified plugin `dsh-plugin-session-emoji` (developed, installed, and verified on dsh `0.1.5-rc.2`); each section ends with how it was verified.

## 0. What the target plugin looks like

Feature: render an externally attached emoji in front of every session title in the web sidebar; right-click a session row to open a picker; the mapping persists host-side.

```
dsh-plugin-session-emoji/
├── package.json          # dual-face declaration + bundle patch declaration
├── cordis.patch.yml      # the shipped patch (the one-command-install key)
├── lib/index.js          # Host half: GET/POST /api/session-emoji + JSON persistence
├── lib/client.js         # Client half: row decoration + context menu + picker (self-contained, zero imports)
└── assets/preview.png
```

```json
{
  "name": "dsh-plugin-session-emoji",
  "version": "0.1.0",
  "type": "module",
  "main": "lib/index.js",
  "exports": { ".": "./lib/index.js", "./client": "./lib/client.js", "./package.json": "./package.json" },
  "files": ["lib", "cordis.patch.yml", "README.md", "LICENSE", "assets"],
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": { "platform": "web", "inject": [] }
  }
}
```

## 1. Host half: exact Fetch routes

Service map: all browser↔host traffic of the web GUI runs under the `/api` prefix (`dsh-client-connection`, source constant `API_PATH = "/api"`). That package fences every `/api` request (Host/Origin checks + browser cookie auth) before dispatching to: the exact-route table → the RPC channel. **An exact route inherits the fence automatically — zero auth code in the plugin.**

```js
export const name = "session-emoji";
export const inject = ["connection"];        // a function plugin's inject export is honored by the Loader

import { promises as fs } from "node:fs";
import { homedir } from "node:os";
import { dirname, join } from "node:path";

export async function apply(ctx) {
  const connection = ctx.connection;
  connection.fetch.register({
    path: "/api/session-emoji",              // must be under /api/, or registration throws
    methods: ["GET", "HEAD", "POST"],        // routes are keyed by path: one registration, many methods
    requestBody: "buffered",
    fetch: async (request) => {
      if (request.method === "HEAD") return new Response(null, { status: 200 });
      if (request.method === "GET")
        return Response.json({ emojis: await loadMap() }, { headers: { "cache-control": "no-store" } });
      const body = await request.json();     // POST: validate → serialized write → reply with the whole map
      // ... validate sessionId / emoji ...
      const map = await enqueueMutation(async () => {
        const m = await loadMap();
        if (body.emoji === null) delete m[body.sessionId];
        else m[body.sessionId] = body.emoji;
        await saveMap(m);
        return m;
      });
      return Response.json({ ok: true, emojis: map });
    },
  });
}
```

Host-side persistence: `$DSH_HOME/storages/<name>.json` (the same user-data area as the workspace controller's `workspace.json`). Write pattern: a promise chain serializes mutations (Node is single-threaded but async fs interleaves) + atomic temp/rename + directory `0o700`.

Registration returns a disposer (an internal `owner.effect`), so the route deregisters automatically on plugin unload/reload — no manual cleanup.

**Verification record**: after install, `curl http://127.0.0.1:3080/api/session-emoji` without a cookie → `401 unauthorized` (fence live). With a valid cookie, GET returned the full seeded map — that, not the 401, is the registration proof; with a cookie, an unregistered /api path returns `404 not found`. **Corrected (0.1.5-rc.2, control-tested)**: the fence rejects *before route matching*, so the unauthenticated 401 happens for **every** /api path, registered or not — the earlier "401 = registered, 404 = not installed" reading was wrong (and contradicted the fence-dispatches-first fact, F3); only the cookie probe distinguishes them.

## 2. Client half: joining the browser roster

### 2.1 Declaration and the loading pipeline

The host-side `dsh-client-modules` scans every active row carrying `dsh.client` in the composed tree, incrementally folds them into `window.__DSH_BOOT__`, injects the graph via the index HTML, and the browser lazy-loads bundles along it.

`dsh.client` fields (0.1.5-rc.2 `parseDshClient` validation):

| Field | Type | Meaning |
|---|---|---|
| `platform` | string (required) | `"web"` |
| `inject` | string[] | dependent plugin rows (load first); also what makes `require("@deepseek-ai/<pkg>")` resolvable inside the bundle |
| `external` | string[] | extra non-platform module requests (`<pkg>` or `<pkg>/client` resolve to that package's client half) |
| `immediately` | boolean | prefetch at boot instead of first use |

### 2.2 The bundle file shape

The whole `lib/client.js` is:

```js
window.__ModuleLoader__.load({
  id: "dsh-plugin-session-emoji",            // must equal the package name
  factory: (require) => {
    var module = { exports: {} };
    var exports = module.exports;
    // ---- all code; every side effect (CSS injection included) in this closure ----
    async function apply(ctx) {
      // ... MutationObserver / event listeners ...
      return async () => { /* cleanup: disconnect observer, remove listeners, remove style tag */ };
    }
    exports.inject = [];                     // client-side cordis service injection (empty if unneeded)
    exports.apply = apply;
    return module.exports;
  }
});
```

- `require()` resolution: platform seed → materialized modules → graph rows → registered factories → **throw** (a package missing from the table and from `dsh.client.inject` fails loudly at runtime).
- Platform seed table (extracted from the built frontend `dsh-web-frontend/dist/assets/index-*.js`, the seed function): `react`, `react/jsx-runtime`, `react-dom`, `react-dom/client`, `@deepseek-ai/cordis`, `@deepseek-ai/dsh-client-store`, `@deepseek-ai/dsh-client-ui-slots`, `@deepseek-ai/dsh-client-ui-primitives`, `@deepseek-ai/dsh-client-ui-dockkit`. **The table moves between versions — verify before relying on it (see fact-sources.md).**
- Needing React/JSX: wrap esbuild output like dsh-pocket does (`format:'cjs', platform:'browser'`, `external: ['react', 'react/jsx-runtime', ...]`, embed the output text into the `__ModuleLoader__.load` wrapper, `var React = require("react")` inside the closure). A zero-dependency DOM-only UI needs no bundler at all.

### 2.3 Touching existing UI: slots first, DOM augmentation second

1. **Slots first**: read the target package's README and `lib/types/client/` for `slots.inject` / holes / registration points. Example: ui-workspace itself hangs off the sidebar via `ctx.slots.inject("sidebar.workspaces", ...)` and declares holes like `conversation.hero.workspace.directoryFlow` for picker packages.
2. **DOM augmentation** (when the component exposes no extension point — this plugin's path):
   - **Row location**: a `MutationObserver` on `document.body` (`childList+subtree`, rAF-batched) watching `[role="treeitem"]`.
   - **Row → data mapping**: React attaches a `__reactFiber$<random>` key to DOM nodes; find the fiber via `Object.keys(el)`, walk `fiber.return` upward reading each level's `memoizedProps` for the row component's data props (here: `props.node` shaped `{id:string, title:string, updatedAt:number}` — `GroupNode` has no `id`, search rows are `props.result`, so they disambiguate naturally). Cap at ~25 hops.
   - **Rendering**: never insert elements among React-managed children. `setAttribute("data-dsh-session-emoji", emoji)` on the target span, plus one injected `<style data-plugin="<package>">` with `span[data-dsh-session-emoji]::before { content: attr(data-dsh-session-emoji); ... }`. React re-renders don't clear unknown attributes; the `data-plugin` marker on style tags is the convention client-hmr teardown uses to remove a plugin's styles.
   - **Popups**: a capture-phase `contextmenu` listener + plain-DOM popups appended to `document.body` (fixed positioning + high z-index escapes sidebar overflow); outside-close via capture-phase `pointerdown`. Follow dsh theme tokens (`--dsw-alias-*`) with fallbacks so both themes look right.

**Verification record**: client.js ran in a Node `vm` with a stubbed `window.__ModuleLoader__` — confirming the registration id and the `exports.apply` shape (catches syntax/registration errors without a browser); live verification came from the user's screenshot plus boot-graph checks (§5).

## 3. Install and lifecycle

```sh
dsh plugin --profile web add dsh-plugin-session-emoji -w     # npm source
# or git+https://github.com/<you>/<pkg>.git -w                # git source (build-script-free is the smooth case)
# or link:/abs/path -w                                        # dev symlink; source edits apply directly
dsh plugin --profile web update dsh-plugin-session-emoji -w  # update (add --latest across major versions)
dsh plugin --profile web remove dsh-plugin-session-emoji -w  # uninstall (auto-removed from bundles)
dsh --profile web --dump-config                              # preview the composed tree without booting
```

Mechanism: `dsh plugin` forwards to pnpm (cwd = the profile directory) → on success reconcile scans each dependency's manifest and **auto-appends packages declaring `dsh.bundle.patch` to the profile's `dsh.profile.bundles`**. Bundle patches join the composed tree in listed order (builtin bundles → bundles → profile patch → home patch → --patch).

- `-w` is required (the profile is a pnpm workspace).
- Git deps: build-script-free is the smooth case; with `prepare`, add `allowBuilds` to the profile's `pnpm-workspace.yaml`.
- **Bundle add/remove does not hot-reload; restart `dsh web`.** Steady-state probe after install/remove: route 404 = not in effect.

**Verification record**: ran remove(git)+add(npm) live; `dsh.profile.bundles` updated automatically; behavior matches the dsh CLI source (`plugin-Ddi42qoW.js`).

## 4. In-process verification (no browser)

The boot graph ships with the index HTML and is only visible authenticated. Two routes:

1. Browser DevTools: search your package name in Sources (the `__DSH_BOOT__` script); the Console shows load failures and `web boot: N entries did not activate` hints (listing pending service names).
2. Programmatic (own-machine credentials, no privilege escalation): the secret in `$DSH_HOME/.credentials.yaml`'s `client-connection/browser-session` grant signs the auth cookie (HMAC key = the secret's **raw decoded bytes**, not the base64url string — `canonicalSecret` decodes first; missing this makes every request 401). Cookie name = `dsh-auth-` + base64url(sha256(authority)); value = `v1.<base64url(payload)>.<base64url(HMAC-SHA256(secret, body))>`; payload = `{version:1, authority:"127.0.0.1:3080", issuedAt, expiresAt}`. With it you can `GET /` (grep your package name to confirm the boot graph + combo URL) and `GET /api/<route>` (see business data). **This only trades file-read access for an equivalent HTTP view — it crosses no permission boundary the file access didn't already grant.**

## 5. Hot-reload vs refresh semantics (the confusing part)

| Scenario | Behavior |
|---|---|
| Editing `lib/client.js` | `dsh-client-hmr` stat-polls (default 500ms), detects the change, `rebuilt()` → old fiber torn down, new one mounted **without a page reload**. Source-map-only writes don't trigger; failed reloads land FAILED with no rollback; React state resets, data layers survive |
| Editing a bundle's `cordis.patch.yml` | Patch files are watched — hot reload |
| Install/remove/update (changes to `dsh.profile.bundles`) | **Not hot-reloaded; restart `dsh web`**. A transient blip where even unauthenticated probes fall to 404 means the /api channel itself was briefly unmounted during pnpm's node_modules reorganization — it is transient; trust the steady state |
| Boot graph changed without a hot swap | **Refresh the browser** (the graph ships with the index) |

**Verification record**: hot removal (deleting a patch row changed the boot graph rev and dropped the route to 404 as seen from the cookie-authenticated browser side, with no restart — an unauthenticated curl cannot show this, it 401s every /api path; see §1's corrected record) and hot swap (rewriting client.js swapped the plugin in the browser) were both observed live.

## 6. The full reference implementation

https://github.com/cholf5/dsh-plugin-session-emoji — the origin of every code sample and verification in this document; read them side by side.
