# The fact-source hierarchy and how to verify against the source

> dsh moves fast (0.x rcs); any secondhand material — including this skill and the two community skills — can go stale. This document fixes the source priority and gives the concrete technique for verifying an API claim against the local source in 30 seconds. **The methodology is worth more than any individual fact.**

## 1. The hierarchy

| Layer | Role | Use |
|---|---|---|
| ① The locally installed official source | **The single final source of truth** | Arbiter for every API signature, behavior semantic, and config field |
| ② dsh-io/dsh-plugin-skill | Current API snapshot reference | Quick index for the tool API / scaffold CLI (`npx @dsh-io/dsh-dev scaffold`) |
| ③ green-dalii/dsh-plugin-dev-skill | First systematic reference | Systematic lecture notes on Cordis, tools, LLM adapters, config, publishing |
| ④ Official docs site | Background | https://deepseek-harness.github.io/deepseek-harness/ |

Rule: ②③ are for **forming hypotheses fast**; ① **decides**. On conflict, trust ①; every API claim taken from ②③ goes back to ① for signature confirmation before it lands in code.

## 2. Locating the local source

```sh
npm root -g                       # global install: node_modules/@deepseek-ai/
# npx users: ~/.npm/_npx/<hash>/node_modules/@deepseek-ai/
ls <root>/@deepseek-ai/ | grep dsh # the whole family: dsh, dsh-base, dsh-web-app, dsh-client-*, dsh-tools, dsh-skill...
```

The running profile (`~/.dsh/profiles/web/`) holds `dsh.profile.bundles` + `cordis.patch.yml` — the manifest of the **currently live composition**; `~/.dsh/storages/` is the host-side user-data area. Probe: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3080/api/<route>` → 401 = registered, 404 = absent.

## 3. Verification techniques (ranked by cost-effectiveness)

1. **Package READMEs first.** The `@deepseek-ai` READMEs are exceptionally good (behavior contracts, config tables, source maps, design boundaries) — most questions end there. A synchronized `README.zh.md` usually exists.
2. **Type definitions decide signatures**: `<pkg>/lib/types/*.d.ts` (generated at build, in sync with runtime). Examples: whether `ToolDefinition.output` is mandatory, `ParameterPropertySpec.required?: true` (the source of the per-property-required rule), the route shape of `ctx.connection.fetch`.
3. **Built artifacts decide behavior**: `<pkg>/lib/*.js` is readable, non-minified build output — grep it. Examples: `dsh-client-modules/lib/index.js` `parseDshClient` (the `dsh.client` field validation), `orderByModuleGraph` (external resolution + cycle detection), the dsh CLI's plugin command (the bundle auto-mount reconcile).
4. **Browser-side facts hide in the frontend dist**: the platform module seed table appears in no README — grep `staticModules` inside `dsh-web-frontend/dist/assets/index-*.js` and extract the seed function's keys.
5. **Relative links in the upstream READMEs are a free map**: each package README's "Further Exploration" points at sibling packages and `.agents/notes/**` design-decision records — following the map beats searching.
6. **The running process**: `--dump-config` previews the composed tree; the DevTools console shows `web boot: N entries did not activate` (listing pending service names); `/plugins/events` SSE broadcasts graph/rebuilt frames.
7. **Verify a whole chain hands-on**: `node --check` / import the host half; run the client factory in a `vm` with a stubbed `window.__ModuleLoader__`; `curl` the route; finally install for real (`dsh plugin add link:... -w`).

## 4. The fact table (with sources, for cross-checking)

| Fact | Source (0.1.5-rc.2) | Recheck (0.1.6-alpha.2, 2026-09-18, see CHECKLIST.md) |
|---|---|---|
| `dsh.bundle.patch` → `dsh plugin add` auto-appends to `dsh.profile.bundles` | dsh CLI `lib/plugin-Ddi42qoW.js` `reconcilePlugins()` | ✅ no drift; the logic moved into `@deepseek-ai/dsh-plugin-manager/lib/index.js` `reconcile()` |
| `- insert:` row format; later layers win per row, whole-config replacement | `dsh-web-app/cordis.patch.yml` header notes + the dsh README | ✅ header notes unchanged |
| Exact routes are path-keyed; the `/api` fence dispatches first | `dsh-client-connection/lib/index.js` `registerFetchRoute` / the `/api` route registration | ✅ `API_PATH="/api"` and `fetchRoutes.has` still present |
| `dsh.client` field validation: platform required, inject/external/immediately | `dsh-client-modules/lib/index.js` `parseDshClient` | ✅ |
| bundle = `window.__ModuleLoader__.load({id, factory})`, lazy CJS | `dsh-client-modules/lib/client.js` header notes + shipped bundles | ✅ |
| skill-name pattern `^[a-z0-9]+(?:-[a-z0-9]+)*$` | `dsh-skill/lib/index.js` `SKILL_NAME` | ✅ unchanged |
| skill roots: `.dsh/skills`(100) / `.agents/skills`(200) / custom(300) / `~/.dsh/skills`(400) / `~/.agents/skills`(500); frontmatter `disable-model-invocation` / `user-invocable` (legacy camelCase keys rejected) | `dsh-skill-filesystem/README.md` + `lib/index.js` `parseInvocationPolicy` | ✅ followSymlinks / nodeEntryKind / key names all present |
| HMR: 500ms stat-poll, only real revision changes trigger, map-only writes don't reload, FAILED has no rollback, `data-plugin` style-tag cleanup | `dsh-client-hmr/README.md` | ✅ pollIntervalMs 500 |
| Session row `[role=treeitem]`, fiber `props.node` | `dsh-client-ui-workspace/lib/client.js` `SessionNodeItem` + `lib/types/client/tree.d.ts` | ✅ `SessionNode{id,title,updatedAt}` unchanged |
| Platform module seed table (nine keys) | `dsh-web-frontend/dist/assets/index-*.js` seed function | ✅ all nine keys unchanged |
| tools: `defineTool` output mandatory, `required` per-property | `dsh-tools/lib/types/{index,schema}.d.ts` | ✅ |
| `dsh plugin` shells out to pnpm — without it on PATH: exit 127, "pnpm was not found" | 0.1.5-rc.2: `dsh/lib/plugin-Ddi42qoW.js` `spawnSync("pnpm")` ENOENT branch; 0.1.6-alpha.2: `dsh/lib/plugin-DJ-rVHUS.js` `result.exitCode === 127` message | ✅ 0.1.5 verified live on this machine; 0.1.6 verified in source |

## 5. Citation discipline

- Every API fact that lands in code or docs is marked either "verified" (with a source) or "unverified — confirm before use".
- Leave a "verified against dsh <version>" comment in your plugin — it is the first thing to check when an upgrade misbehaves.
