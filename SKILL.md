---
name: dsh-plugin-dev-notes
description: Field notes for developing DeepSeek Harness (dsh) plugins, focused on dual-face plugins (Node host half + browser client half): requirement scoping and decomposition, dsh.bundle.patch one-command install, the dsh.client browser roster, /api exact routes, client HMR semantics, npm publishing, and a verify-against-installed-source methodology.
whenToUse: Load when building, modifying, or debugging DeepSeek Harness (dsh) plugins — especially when scoping/decomposing a plugin requirement, building web UI (browser-side) or dual-face plugins, bundle packaging/install, npm publishing, @deepseek-ai/dsh-* API usage — or when any plugin API claim needs verification against the locally installed dsh source.
---

# dsh Plugin Development Field Notes (dual-face perspective)

> This is a field-verified operating manual focused on one underserved perspective: **a plugin = a Host half (Node/Cordis) + a Client half (browser)**. Compared with the community skills — [green-dalii/dsh-plugin-dev-skill](https://github.com/green-dalii/dsh-plugin-dev-skill) (Cordis mental model, tools, LLM adapters, config, publishing) and [dsh-io/dsh-plugin-skill](https://github.com/dsh-io/dsh-plugin-skill) (tool API snapshot + scaffold CLI) — this skill covers the web-UI/plugin territory they leave out, and treats **the locally installed dsh source as the single source of truth**.
>
> **Verification baseline**: every template here ran for real on dsh `0.1.5-rc.2`, and the fact table was re-verified against `0.1.6-alpha.2` — see [CHECKLIST.md](CHECKLIST.md) for the recheck log and the per-release procedure.

## 0. Three layers of truth (read first)

Before writing any dsh code, source material in this priority order:

1. **Official installed source = the single final source of truth.** The dsh install running on this machine is authoritative: package READMEs (exceptionally good), built `lib/*.js`, and `lib/types/*.d.ts`. Location: `node_modules/@deepseek-ai/dsh*` under the global npm root (or the npx cache).
2. **Current API snapshot**: dsh-io/dsh-plugin-skill (tool API, scaffold CLI `npx @dsh-io/dsh-dev scaffold`).
3. **First systematic reference**: green-dalii/dsh-plugin-dev-skill.

⚠️ Reference repos drift from your dsh version (APIs move between rcs). **Grep the local source to confirm every signature before using it** — the technique is in `references/fact-sources.md`.

## 1. Mental model: one package, two halves

The dsh web GUI itself is assembled from "dual-face" plugins (ui-workspace, ui-sidebar, …). Yours can be one too:

| Half | File | Runs in | Exports |
|---|---|---|---|
| Host | `lib/index.js` (`"main"`) | Node, the Cordis Loader | `apply(ctx, config)`, optional `name`/`inject`/`Config` |
| Client | `lib/client.js` (exports `"./client"`) | Browser, dsh client module system | bundle registration form (§6) |

- Both halves are declared in one `package.json` `dsh` field (either half alone is fine):
  ```json
  {
    "main": "lib/index.js",
    "exports": { ".": "./lib/index.js", "./client": "./lib/client.js" },
    "dsh": {
      "bundle": { "patch": "./cordis.patch.yml" },
      "client": { "platform": "web", "inject": [], "external": [] }
    }
  }
  ```
- The Host half is a plain Cordis plugin (function form first; a `Service` subclass only when you provide a service). The Cordis iron rules live in the green-dalii skill §1–§9. One verified extra fact here: **a function plugin's `export const inject = ['connection']` is honored by the Loader** (services resolve before `apply` runs).
- The Client half is not a normal module: the whole file is `window.__ModuleLoader__.load({ id: "<package name>", factory: (require) => { ...; return module.exports; } })`. Official packages produce this shape with tsdown/esbuild; hand-write or wrap with esbuild (dsh-pocket wraps, see `references/web-ui-plugins.md` §6).
- Don't pull React if you don't need it: a self-contained bundle (zero imports) is the lowest-friction option. The platform module table currently holds react, react/jsx-runtime, react-dom, react-dom/client, @deepseek-ai/cordis, dsh-client-store, dsh-client-ui-slots, dsh-client-ui-primitives, dsh-client-ui-dockkit (extracted from the built frontend; may change between versions — verify first, see `references/fact-sources.md`).

### The five-minute skeleton

A complete, runnable dual-face plugin. Both halves are independent — ship only one if that's all you need.

```
dsh-plugin-minimal/
├── package.json
├── cordis.patch.yml
└── lib/
    ├── index.js     # Host half
    └── client.js    # Client half
```

```json
{
  "name": "dsh-plugin-minimal",
  "version": "0.0.1",
  "type": "module",
  "main": "lib/index.js",
  "exports": { ".": "./lib/index.js", "./client": "./lib/client.js" },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": { "platform": "web" }
  }
}
```

```yaml
# cordis.patch.yml
- insert:
    - id: minimal
      name: dsh-plugin-minimal
```

```js
// lib/index.js — Host half: one exact route on the authenticated channel
export const inject = ["connection"];

export async function apply(ctx) {
  ctx.connection.fetch.register({
    path: "/api/minimal/ping",
    methods: ["GET"],
    requestBody: "buffered",
    fetch: async () => Response.json({ pong: Date.now() }),
  });
}
```

```js
// lib/client.js — Client half: the registration form; side effects run at materialization
window.__ModuleLoader__.load({
  id: "dsh-plugin-minimal",
  factory: (require) => {
    var module = { exports: {} };
    var exports = module.exports;
    async function apply(ctx) {
      console.log("[minimal] hello from the browser half");
      return async () => {};
    }
    exports.inject = [];
    exports.apply = apply;
    return module.exports;
  }
});
```

Install and verify (prerequisites in §4.2):

```sh
npx @deepseek-ai/dsh plugin --profile web add link:/abs/path/to/dsh-plugin-minimal -w
# restart dsh web, refresh the browser, then mint a session cookie from the launch URL dsh web printed:
curl -s -c /tmp/dsh-cookies.txt "http://127.0.0.1:3080/?token=<token>" -o /dev/null   # 303, cookie saved
curl -s -b /tmp/dsh-cookies.txt http://127.0.0.1:3080/api/minimal/ping   # {"pong":…} = route live; 404 "not found" = not registered
# (an unauthenticated curl gets 401 for EVERY /api path — the fence rejects before route matching — so 401 proves nothing about registration)
# DevTools console shows: [minimal] hello from the browser half
```

## 2. Scope before code: the decomposition assessment

dsh's composition machinery — bundles as layers, patch rows as toggleable units, slots/holes and services as seams — rewards plugins that are **shaped for combination**. The classic failure is the mega-plugin: one package, one row, five unrelated features, so nobody after you can toggle, replace, or recombine any part. Before writing code, run the assessment below and **present it to the user** — the assessment is a deliverable, not a silent choice.

### 2.1 The three questions

1. **What varies independently?** (the axis test) — different backends for one interface? UI without the logic, or logic without the UI? Policy a deployment should toggle? Each axis that varies independently is a boundary candidate.
2. **Who must be able to combine or replace what?** — if another deployment should swap your backend, drop your UI, or disable one feature while keeping the rest, that part needs its own boundary (a package, a row, or a slot).
3. **What must change together?** — the cohesion core: the minimum that makes no sense apart. That is your package.

### 2.2 Mapping answers onto dsh structure (the decision table)

| Finding | Structure | Verified example |
|---|---|---|
| One interface, interchangeable implementations | one definition + separate provider packages; a one-line patch switches the provider | directory picker: the `-auto` row mounts `-native` or `-browse`; "Mount -native or -browse directly in an overlay to pin the interaction" (`dsh-web-app/cordis.patch.yml`) |
| UI vs data/logic with different audiences | two packages sharing a seam (host pkg + client UI pkg) | `dsh-message-feedback` (host, log-backed) + `dsh-client-ui-message-feedback` (browser UI) |
| Features toggled independently, same audience | one package, **one row per feature** (stable ids; `disabled: true` per row) | the web app patch ships dozens of rows; e.g. `ui-schedule` arrives `disabled: true` and an overlay enables it |
| One feature, no independent axis | one package, one row, both halves | session-emoji (dual-face in a single bundle) |
| Policy vs mechanism | mechanism in your plugin; policy as a separate hook plugin on the pipeline | permission presets vs the tool pipeline |

Key vocabulary the assessment speaks: a **package** is a distribution unit; a **row** in a patch is a composition unit (deployers enable/disable/override per row); **slots/holes** and **services/events** are the seams others plug into. Design so the toggles people will want are rows, and the substitutions they will want are seams.

### 2.3 The deliverable: an assessment table, not a silent choice

Before coding, present: a table of units (package / half / row / seam) with one responsibility line each, the **named axis of variation** every split serves, and — equally important — the **non-splits** with reasons. Guardrail: no preemptive splitting. A boundary that cannot name its axis of variation is noise; green-dalii's three-role capability layering (definition / provider / consumer) is the deep dive for provider seams, and its rule stands: don't split until a real consumer or axis exists.

### 2.4 Red flags

- "the plugin that also does X" — two features share one row with no independent toggle
- config keys that switch unrelated behaviors inside one row
- another plugin needs to disable *part* of you (they can only disable rows or everything — if part-toggling is foreseeable, it's a row today)
- you must reach into another plugin's internals because no seam exists (coupling debt: ui-workspace's session rows expose no decoration slot — DOM augmentation is the workaround *and* the warning)
- a README install section that says "features A, B, C are all on; there is no switch"

## 3. The one-command install secret: `dsh.bundle.patch`

Why `dsh plugin --profile web add <pkg> -w` is one step: the command forwards to pnpm inside the profile, then reconcile scans each installed dependency's manifest — **a package declaring `dsh.bundle.patch` is appended to the profile's `dsh.profile.bundles` layer list automatically**. The package's own `cordis.patch.yml` joins the composed tree as one patch layer.

```yaml
# the package's cordis.patch.yml
- insert:
    - id: session-emoji          # stable id
      name: dsh-plugin-session-emoji   # package name, resolved from profile node_modules
```

- `-w` is required (the profile is a pnpm workspace; without it: `ERR_PNPM_ADDING_TO_ROOT`).
- Git deps install source; **build-script-free packages are the smooth case**. With a `prepare` script, pnpm blocks it until `allowBuilds` is added to the profile's `pnpm-workspace.yaml` (tell the user honestly: that authorizes code execution at install time).
- Layer order: bundles in listed order → profile `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml` → `--patch` overlays. **Later layers win per row, and a patch replaces the target row's whole `config` (no deep merge)** — when overriding another layer's row, restate every key it needs.
- Update/remove: `dsh plugin --profile web update|remove <pkg> -w`.
- **Bundle additions/removals do not hot-reload; restart `dsh web`** (patch-file hot reload is §7).

## 4. Distribution: npm publishing, and the README your users need

### 4.1 Publishing (the author side)

The npm form of `dsh plugin add <name>` only works if `<name>` exists on npmjs — **publish first**:

- `package.json` must ship the bundle pieces: no `"private": true`, and `"files"` includes `lib` + `cordis.patch.yml` (a tarball without the patch file installs as a plain dependency — the reconcile warning in §3 tells you exactly that).
- `npm login` uses a browser flow. Caveat (npm policy, observed 2026-09): the login token **cannot publish** — npm requires either 2FA OTP or a **granular access token** with bypass-2FA. Two workable paths:
  - enable 2FA on the account, then `npm publish --otp=<6-digit code>`;
  - or create a granular token (npmjs.com → Access Tokens → Granular: *All packages* + *Read and write*), publish against a throwaway userconfig, and delete it after:
    ```sh
    printf '//registry.npmjs.org/:_authToken=npm_xxx\n' > /tmp/publish-npmrc && chmod 600 /tmp/publish-npmrc
    npm publish --userconfig /tmp/publish-npmrc && rm /tmp/publish-npmrc
    ```
- Verify: `npm view <pkg> version`.
- No npm account? `git+https://...` direct install works (§3) — but note a git dep needs the user to allow `prepare` builds, and it pins nothing: users get whatever main is. npm remains the recommended channel.

### 4.2 The two end-user prerequisites (call them out in your README)

Two things silently block "one-command" installs for non-developer users — **your plugin README must spell both out**:

1. **`dsh` itself may not be a command.** Most users run dsh via `npx @deepseek-ai/dsh web` without a global install, so bare `dsh ...` commands fail with "command not found". Every command in your README needs the npx form alongside:
   ```sh
   dsh --version                  # global install
   npx @deepseek-ai/dsh --version # npx-only install — equally valid
   ```
2. **pnpm is required by the plugin manager.** `dsh plugin` shells out to pnpm; without it the command exits 127 with `pnpm was not found; install pnpm and make it available on PATH` (verified on 0.1.5-rc.2 and 0.1.6-alpha.2). The fix is one line, but only if your README says so:
   ```sh
   npm install -g pnpm    # or: brew install pnpm, or corepack enable pnpm
   ```

### 4.3 The install section your plugin README must include (template)

Copy, adapt the `<placeholders>`, and keep the fallback — it is what makes the README work for people without pnpm:

````markdown
## Install

Prerequisites:

- **dsh** reachable — `dsh --version`, or use `npx @deepseek-ai/dsh` everywhere below
- **pnpm** on PATH (the dsh plugin manager calls it): `npm install -g pnpm`

```sh
# install (npm)
npx @deepseek-ai/dsh plugin --profile web add <pkg> -w
# install (direct from GitHub)
npx @deepseek-ai/dsh plugin --profile web add git+https://github.com/<you>/<pkg>.git -w
```

Restart `dsh web`, then refresh the browser page. Verify the route is live (with a cookie — an unauthenticated 401 is not a registration signal, it happens for every /api path):

```sh
curl -s -c /tmp/dsh-cookies.txt "http://127.0.0.1:3080/?token=<token-from-launch-url>" -o /dev/null   # mint session cookie (303)
curl -s -b /tmp/dsh-cookies.txt http://127.0.0.1:3080/api/<route>   # expected body = registered; 404 "not found" = not
```

<details>
<summary>No pnpm, and don't want it? Manual fallback</summary>

```sh
git clone https://github.com/<you>/<pkg>.git ~/.dsh/profiles/web/node_modules/<pkg>
```

Then edit `~/.dsh/profiles/web/cordis.patch.yml` so the top-level list contains (this is the file's final state — do not blindly append after a `[]` line):

```yaml
- insert:
    - id: <plugin-id>
      name: <pkg>
```

The running dsh hot-loads this row (patch file watch); refresh the browser afterwards.

</details>

<details>
<summary>Update / remove</summary>

```sh
npx @deepseek-ai/dsh plugin --profile web update <pkg> -w    # or remove <pkg> -w
```

Restart `dsh web` afterwards.

</details>
````

### 4.4 Troubleshooting rows your README should carry

| Symptom | Cause & fix |
|---|---|
| `dsh: command not found` | npx-only install — prefix `npx @deepseek-ai/dsh` |
| `pnpm was not found` (exit 127) | `npm install -g pnpm`, or use the manual fallback |
| `ERR_PNPM_ADDING_TO_ROOT` | the `-w` flag was dropped |
| Installed but the UI is unchanged | restart `dsh web` (bundle layers don't hot-reload), then refresh the page |
| git install fails on a `prepare` script | add the key pnpm printed to the profile's `pnpm-workspace.yaml` under `allowBuilds`, re-run |

## 5. Host half: your own API on the shared authenticated channel

All browser↔host traffic of the web GUI runs under the `/api` prefix (`dsh-client-connection`, constant `API_PATH = "/api"`), fenced by that package (Host/Origin checks + browser cookie auth). Register an **exact Fetch route** and inherit the whole security model with **zero auth code**:

```js
export const inject = ['connection'];

export async function apply(ctx) {
  const connection = ctx.connection;
  connection.fetch.register({
    path: '/api/my-plugin/data',          // must be under /api/
    methods: ['GET', 'POST'],             // one route per path; fold methods into one registration
    requestBody: 'buffered',              // or streaming
    fetch: async (request) => {           // standard Fetch Request → Response
      if (request.method === 'GET') return Response.json({ /* ... */ });
      const body = await request.json();
      return Response.json({ ok: true });
    },
  });
}
```

- Routes are keyed by path: registering the same path twice throws; register once and dispatch internally.
- Browser side: plain `fetch('/api/my-plugin/data')` (same-origin; the cookie rides along). An unauthenticated curl gets 401 for **every** `/api` path — the fence rejects *before* route matching — so 401 means "fence up", not "route registered"; the registration probe is the cookie probe in §8 (registered → your body, unregistered → 404 "not found").
- Reach for Typert (`@deepseek-ai/dsh-typert-protocol`: `TypertRemoteService` + `Remote` decorators + generated `TYPERT`/`TYPERT_REMOTE` artifacts, mounted client-side via `ctx.remote.$mount()`) only for RPC/streaming/generated endpoints; exact routes cover small plugins.
- Persistence: write `$DSH_HOME/storages/<your-name>.json` (the same user-data area the workspace controller uses). Serialize concurrent writes with a promise chain + atomic temp/rename.

**Security boundary — what the fence does and does not do.** The fence authenticates the browser session and checks Host/Origin *before* your handler; it does **not** scope data inside your route. Two consequences to design for:

- **Every authenticated browser session can read and mutate everything your routes expose.** dsh web has no per-user identity model — one deployment, one shared cookie identity. Return the minimum your UI needs; avoid whole-store GETs for sensitive data; make destructive mutations explicit and deliberate.
- **Your route's exposure equals the GUI's exposure.** If the deployment reaches beyond loopback — LAN, or a [dsh-pocket](https://github.com/shaobeichen/dsh-pocket) public tunnel behind its password — your routes sit behind that same password. dsh-pocket is a common add-on; assume it can appear, and don't build a route you wouldn't expose to a phone on a cellular network.

## 6. Client half: joining the browser roster

The host scans every active row carrying `dsh.client` into `window.__DSH_BOOT__` (the boot graph), injected into the page via the index HTML; the browser lazy-loads bundles along it.

- **`dsh.client` fields** (0.1.5-rc.2 `parseDshClient`): `platform` (required string, "web"), `inject` (string[]: which plugin rows must load first — also what makes `require("@deepseek-ai/<pkg>")` resolvable inside your bundle), `external` (extra non-platform module requests), `immediately` (boolean: prefetch at boot instead of first use).
- **Bundle shape** (browser CJS; executing only registers the factory, side effects run at materialization):
  ```js
  window.__ModuleLoader__.load({
    id: "my-plugin",                    // must equal the package name
    factory: (require) => {
      var module = { exports: {} };
      var exports = module.exports;
      // ---- all code; every side effect (CSS injection included) in this closure ----
      async function apply(ctx) { /* ... */ return async () => { /* cleanup */ }; }
      exports.inject = [];              // client-side cordis service injection
      exports.apply = apply;
      return module.exports;
    }
  });
  ```
- `require()` resolution: platform seed → materialized modules → graph rows → registered factories → **throw** (a package missing from the table and from `dsh.client.inject` fails loudly at runtime).
- Needing React/JSX: wrap esbuild output like dsh-pocket does (`format:'cjs', platform:'browser'`, `external: ['react', 'react/jsx-runtime', ...]`, embed the text into the `__ModuleLoader__.load` wrapper, `var React = require("react")` inside). A zero-dependency DOM-only UI needs no bundler at all.
- **Two ways to touch existing UI**:
  1. **Slot injection (the official way)**: check the target package for seats — `ctx.slots.inject("sidebar.workspaces", ...)` / holes / declaration-merged registration points. The ui-workspace and ui-sidebar READMEs enumerate theirs.
  2. **DOM augmentation** (when the component exposes no extension point — this plugin's path): MutationObserver + structural selectors (`[role="treeitem"]`) + React fiber reading (`__reactFiber$`-prefixed key → walk `fiber.return` for the row component's data props, e.g. `node.id`) + **render via a `data-*` attribute and one injected CSS rule `::before content: attr()`** — never insert elements among React-managed children. Attribute writes and style tags survive React re-renders; tag style tags with `data-plugin="<your package>"` (HMR teardown removes them by that convention).
  - Decision rule: slots first; DOM augmentation only when no slot exists — and record your selectors/props shape in comments for the next dsh upgrade.

## 7. Dev hot-reload loop (where the time goes)

| Change | Takes effect |
|---|---|
| `lib/client.js` (browser half) | **Save and it hot-swaps**: `dsh-client-hmr` stat-polls every graph bundle (default 500ms); on change it `rebuilt()` → old fiber torn down, new one mounted, no page reload |
| Host half | Restart `dsh web` (host code loads with the process) |
| A bundle's `cordis.patch.yml` (content rows) | Patch files are watched — hot reload |
| `dsh.profile.bundles` (install/remove/update) | **Not hot-reloaded; restart `dsh web`** |

Dev install: `dsh plugin --profile web add link:/abs/path/to/pkg -w` (link: symlink — source edits apply directly) or `file:` (copied — reinstall after edits).
HMR caveats: the swap resets the plugin's React state (connection/session data layers survive); a failed reload leaves the entry FAILED with no rollback; a source-map-only write does not reload code.

## 8. Verification checklist

- [ ] **The decomposition assessment (§2) was presented to the user before coding** — package/half/row boundaries with named axes of variation, and the deliberate non-splits
- [ ] Host half: `node --check` / actually import it; bundle: run the factory in a `vm` with a stubbed `window.__ModuleLoader__` (catches syntax/registration errors without a browser)
- [ ] `dsh --profile web --dump-config` previews the composed tree (no boot needed) — confirm your row
- [ ] After install, probe with a cookie: mint one (`curl -s -c /tmp/dsh-cookies.txt "http://127.0.0.1:3080/?token=<token-from-launch-url>" -o /dev/null`) then `curl -s -b /tmp/dsh-cookies.txt http://127.0.0.1:3080/api/<your route>` → expected body = registered, 404 "not found" = not; an unauthenticated curl returns 401 for any /api path (fence before routing — fence-alive signal only)
- [ ] Browser: refresh the page (the boot graph ships with the index; after an HMR swap you don't refresh); check DevTools console for your `[plugin-name]` warnings and failed-entry hints
- [ ] Uninstall path: `dsh plugin --profile web remove <pkg> -w` → restart → probe 404, no UI residue
- [ ] **README install section covers the two end-user prerequisites (§4.2)**: dsh reachability (global vs npx), pnpm + its install command, the one-command install in both forms, the no-pnpm manual fallback, and the §4.4 troubleshooting rows

### Automate what can be automated

Node 22 ships `node --test` — zero test dependencies (dsh-pocket ships 109 tests this way). Three patterns cover most of a dual-face plugin:

```js
// test/host.test.mjs — route handlers, with $DSH_HOME pointed at a temp dir
import { test } from "node:test";
import assert from "node:assert/strict";
import { apply } from "../lib/index.js";

test("GET /api/... returns the map", async () => {
  process.env.DSH_HOME = await fs.mkdtemp(join(tmpdir(), "dsh-test-")); // store files land there
  let registered;
  await apply({ connection: { fetch: { register: (r) => { registered = r; } } } });
  const res = await registered.fetch(new Request("https://x/api/my-plugin/data", { method: "GET" }));
  assert.equal(res.status, 200);
});
```

The trick: my host half resolves the store path via `resolveDshHome()` **at call time**, so pointing `$DSH_HOME` at a temp directory in the test isolates the filesystem — a design-for-testability habit worth copying (resolve env/config inside handlers, not at module top level).

```js
// test/client.test.mjs — the bundle factory, no browser needed
import { test } from "node:test";
import assert from "node:assert/strict";
import { readFileSync } from "node:fs";
import vm from "node:vm";

test("bundle registers under the package id with an apply export", () => {
  let registered;
  const sandbox = {
    window: { __ModuleLoader__: { load: (def) => { registered = def; } } },
    require: () => { throw new Error("unexpected require"); },
  };
  vm.createContext(sandbox);
  vm.runInContext(readFileSync("lib/client.js", "utf8"), sandbox);
  assert.equal(registered.id, "dsh-plugin-minimal");
  assert.equal(typeof registered.factory(sandbox.require).apply, "function");
});
```

What's left for manual verification (§8 checklist): composed-tree shape, the live fence probe, actual DOM behavior — the parts that need a running dsh and a browser.

## 9. Pitfalls (all field-tested)

| Pitfall | Reality |
|---|---|
| One package, one row, many unrelated features | deployers can only toggle whole rows — split features into rows (or packages) so composition stays possible (§2) |
| Model-facing tool never appears in web sessions | the web surface **disables agent-plane rows** (tool-bash, tool-fs, …) and lets each session mount a preset instead (verified in `dsh-web-app/cordis.patch.yml`, F13) — model-facing rows belong in an agent preset (`~/.agent-presets` is user-authored); verify with `--dump-config` + a live session |
| A `/api` route that returns everything | the fence authenticates, it does not scope data — and your route's exposure equals the GUI's (dsh-pocket tunnels included); see the security boundary in §5 |
| Unauthenticated 401 taken as proof of route registration | the fence rejects **before route matching** — every /api path 401s without a cookie (control-tested on 0.1.5-rc.2); registration proof = cookie probe + expected body vs 404 "not found" (§8) |
| Hand-editing the profile's cordis.patch.yml instead of shipping a bundle | Works, but not the convention; `dsh.bundle.patch` is the one-command path |
| Registering the same exact route path twice | `fetchRoutes` is path-keyed; the second registration throws — fold methods into one registration |
| `require("react")` in a bundle | react is in the platform table; **anything not in the table must be listed in `dsh.client.inject` (external)** or it throws "missed the module table" |
| Inserting DOM among React-managed children | Reconciled away or corrupted; use `data-*` + CSS `::before` or a slot |
| Forgetting `-w` | pnpm fails with `ERR_PNPM_ADDING_TO_ROOT` |
| Assuming the user has pnpm | `dsh plugin` exits 127 without it ("pnpm was not found") — READMEs must carry the prerequisite or the manual fallback |
| Assuming the user has a global `dsh` | Most run npx-only; bare `dsh ...` in a README is a dead end for them |
| README shows the npm install for an unpublished package | `dsh plugin add <npm-name>` only resolves names that exist on npmjs — publish first, or point at git+https |
| Installed but "nothing happened" | bundle layers don't hot-reload — restart `dsh web`; only client.js edits hot-swap |
| `pnpm peers check` complaints | Check whose peer is missing — often another plugin's (e.g. dsh-pocket's), not yours |
| Importing another package's client half | It must actually export `./client` (official types live in `lib/types/client/`), and it goes in your `dsh.client.inject` |

## 10. Further reading

- `references/web-ui-plugins.md` — the full dual-face walkthrough with verification records (the main course)
- `references/fact-sources.md` — the fact-source hierarchy and source-verification techniques
- [CHECKLIST.md](CHECKLIST.md) — the 15-minute re-verification procedure for each new dsh release
- green-dalii/dsh-plugin-dev-skill — Cordis mental model / tools / LLM adapters / config / publishing, plus the three-role capability layering that §2's provider splits build on (not duplicated here)
- dsh-io/dsh-plugin-skill — tool API snapshot and the `@dsh-io/dsh-dev` scaffold
- Docs: https://deepseek-harness.github.io/deepseek-harness/ ; source: https://github.com/deepseek-ai/deepseek-harness
- Worked example: https://github.com/cholf5/dsh-plugin-session-emoji
