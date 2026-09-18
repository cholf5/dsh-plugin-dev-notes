---
name: dsh-plugin-dev-notes
description: Field notes for developing DeepSeek Harness (dsh) plugins, focused on dual-face plugins (Node host half + browser client half): dsh.bundle.patch one-command install, the dsh.client browser roster, /api exact routes, client HMR semantics, npm publishing, and a verify-against-installed-source methodology.
whenToUse: Load when building, modifying, or debugging DeepSeek Harness (dsh) plugins — especially web UI (browser-side) and dual-face plugins, bundle packaging/install, npm publishing, @deepseek-ai/dsh-* API usage — or when any plugin API claim needs verification against the locally installed dsh source.
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
| Client | `lib/client.js` (exports `"./client"`) | Browser, dsh client module system | bundle registration form (§5) |

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

## 2. The one-command install secret: `dsh.bundle.patch`

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
- **Bundle additions/removals do not hot-reload; restart `dsh web`** (patch-file hot reload is §6).

## 3. Distribution: npm publishing, and the README your users need

### 3.1 Publishing (the author side)

The npm form of `dsh plugin add <name>` only works if `<name>` exists on npmjs — **publish first**:

- `package.json` must ship the bundle pieces: no `"private": true`, and `"files"` includes `lib` + `cordis.patch.yml` (a tarball without the patch file installs as a plain dependency — the reconcile warning in §2 tells you exactly that).
- `npm login` uses a browser flow. Caveat (npm policy, observed 2026-09): the login token **cannot publish** — npm requires either 2FA OTP or a **granular access token** with bypass-2FA. Two workable paths:
  - enable 2FA on the account, then `npm publish --otp=<6-digit code>`;
  - or create a granular token (npmjs.com → Access Tokens → Granular: *All packages* + *Read and write*), publish against a throwaway userconfig, and delete it after:
    ```sh
    printf '//registry.npmjs.org/:_authToken=npm_xxx\n' > /tmp/publish-npmrc && chmod 600 /tmp/publish-npmrc
    npm publish --userconfig /tmp/publish-npmrc && rm /tmp/publish-npmrc
    ```
- Verify: `npm view <pkg> version`.
- No npm account? `git+https://...` direct install works (§2) — but note a git dep needs the user to allow `prepare` builds, and it pins nothing: users get whatever main is. npm remains the recommended channel.

### 3.2 The two end-user prerequisites (call them out in your README)

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

### 3.3 The install section your plugin README must include (template)

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

Restart `dsh web`, then refresh the browser page. Verify the route is live:

```sh
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3080/api/<route>   # 401 = installed (auth fence), 404 = not
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

### 3.4 Troubleshooting rows your README should carry

| Symptom | Cause & fix |
|---|---|
| `dsh: command not found` | npx-only install — prefix `npx @deepseek-ai/dsh` |
| `pnpm was not found` (exit 127) | `npm install -g pnpm`, or use the manual fallback |
| `ERR_PNPM_ADDING_TO_ROOT` | the `-w` flag was dropped |
| Installed but the UI is unchanged | restart `dsh web` (bundle layers don't hot-reload), then refresh the page |
| git install fails on a `prepare` script | add the key pnpm printed to the profile's `pnpm-workspace.yaml` under `allowBuilds`, re-run |

## 4. Host half: your own API on the shared authenticated channel

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
- Browser side: plain `fetch('/api/my-plugin/data')` (same-origin; the cookie rides along; curl without a cookie gets 401 — that's your "route registered" probe).
- Reach for Typert (`@deepseek-ai/dsh-typert-protocol`: `TypertRemoteService` + `Remote` decorators + generated `TYPERT`/`TYPERT_REMOTE` artifacts, mounted client-side via `ctx.remote.$mount()`) only for RPC/streaming/generated endpoints; exact routes cover small plugins.
- Persistence: write `$DSH_HOME/storages/<your-name>.json` (the same user-data area the workspace controller uses). Serialize concurrent writes with a promise chain + atomic temp/rename.

## 5. Client half: joining the browser roster

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

## 6. Dev hot-reload loop (where the time goes)

| Change | Takes effect |
|---|---|
| `lib/client.js` (browser half) | **Save and it hot-swaps**: `dsh-client-hmr` stat-polls every graph bundle (default 500ms); on change it `rebuilt()` → old fiber torn down, new one mounted, no page reload |
| Host half | Restart `dsh web` (host code loads with the process) |
| A bundle's `cordis.patch.yml` (content rows) | Patch files are watched — hot reload |
| `dsh.profile.bundles` (install/remove/update) | **Not hot-reloaded; restart `dsh web`** |

Dev install: `dsh plugin --profile web add link:/abs/path/to/pkg -w` (link: symlink — source edits apply directly) or `file:` (copied — reinstall after edits).
HMR caveats: the swap resets the plugin's React state (connection/session data layers survive); a failed reload leaves the entry FAILED with no rollback; a source-map-only write does not reload code.

## 7. Verification checklist

- [ ] Host half: `node --check` / actually import it; bundle: run the factory in a `vm` with a stubbed `window.__ModuleLoader__` (catches syntax/registration errors without a browser)
- [ ] `dsh --profile web --dump-config` previews the composed tree (no boot needed) — confirm your row
- [ ] After install, probe: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3080/api/<your route>` → 401 = registered (the fence rejects unauthenticated requests), 404 = not installed; 200 requires a browser cookie
- [ ] Browser: refresh the page (the boot graph ships with the index; after an HMR swap you don't refresh); check DevTools console for your `[plugin-name]` warnings and failed-entry hints
- [ ] Uninstall path: `dsh plugin --profile web remove <pkg> -w` → restart → probe 404, no UI residue
- [ ] **README install section covers the two end-user prerequisites (§3.2)**: dsh reachability (global vs npx), pnpm + its install command, the one-command install in both forms, the no-pnpm manual fallback, and the §3.4 troubleshooting rows

## 8. Pitfalls (all field-tested)

| Pitfall | Reality |
|---|---|
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

## 9. Further reading

- `references/web-ui-plugins.md` — the full dual-face walkthrough with verification records (the main course)
- `references/fact-sources.md` — the fact-source hierarchy and source-verification techniques
- [CHECKLIST.md](CHECKLIST.md) — the 15-minute re-verification procedure for each new dsh release
- green-dalii/dsh-plugin-dev-skill — Cordis mental model / tools / LLM adapters / config / publishing (not duplicated here)
- dsh-io/dsh-plugin-skill — tool API snapshot and the `@dsh-io/dsh-dev` scaffold
- Docs: https://deepseek-harness.github.io/deepseek-harness/ ; source: https://github.com/deepseek-ai/deepseek-harness
- Worked example: https://github.com/cholf5/dsh-plugin-session-emoji
