# The upgrade recheck checklist

> dsh moves fast through 0.x rcs, and every fact in these notes has a version half-life. This checklist compresses "re-verify after a dsh upgrade" into a 15-minute executable procedure — **it is the core maintenance motion that keeps this repo valuable over time.**
>
> The verification methodology lives in `references/fact-sources.md`; this file is its scheduler.

## When to run

Any of these triggers a round:

```sh
# the locally installed version
dsh --version   # or read ~/.npm/_npx/*/node_modules/@deepseek-ai/dsh/package.json
# the npm latest
npm view @deepseek-ai/dsh version
```

- npm `latest` is newer than the notes' baseline version (see the recheck log at the bottom)
- A user reports "the skill says X but the behavior is Y"

## Procedure

1. **Fetch the new source without touching the local install** (never disturb a running dsh):
   ```sh
   mkdir -p /tmp/dsh-recheck && cd /tmp/dsh-recheck
   npm pack @deepseek-ai/<pkg>@<new> --pack-destination .
   tar -xzf deepseek-ai-<pkg>-<new>.tgz -C <pkg> --strip-components=1
   ```
   Packages to recheck (see the table). Note: since 0.1.6-alpha.2, the `dsh plugin` reconcile logic lives in the new `@deepseek-ai/dsh-plugin-manager` package.
2. **Grep-verdict fact by fact** (key facts → commands, run inside the extracted dirs):

| # | Fact | Verification command |
|---|---|---|
| F1 | `dsh.bundle.patch` deps auto-append to `dsh.profile.bundles` | `grep -n "dsh?.bundle" <dsh-plugin-manager or dsh>/lib/*.js` then find `reconcile` |
| F2 | Later patch layers win per row; whole-config replacement | `grep -o "replaces the targeted row" dsh-web-app/cordis.patch.yml` |
| F3 | Exact routes are path-keyed; the `/api` fence dispatches first | `grep -o 'API_PATH = "[^"]*"' dsh-client-connection/lib/index.js` + `grep fetchRoutes.has` |
| F4 | `dsh.client` field validation (platform/inject/external/immediately) | `grep -o "parseDshClient" dsh-client-modules/lib/index.js` then read around it |
| F5 | bundle shape `window.__ModuleLoader__.load({id, factory})`, lazy CJS | `grep -c "__ModuleLoader__.load" dsh-client-modules/lib/client.js` |
| F6 | skill-name pattern | `grep -o "SKILL_NAME = [^;]*" dsh-skill/lib/index.js` |
| F7 | skill roots / frontmatter keys / symlink following | `grep -o "followSymlinks\|nodeEntryKind\|disable-model-invocation" dsh-skill-filesystem/lib/index.js` |
| F8 | HMR stat-poll (default 500ms), map-only no reload, FAILED no rollback | `grep -o "pollIntervalMs" dsh-client-hmr/README.md` + read the README diff |
| F9 | session row `[role=treeitem]`, fiber `props.node`, `SessionNode{id,title,updatedAt}` | `grep -c 'role: "treeitem"' dsh-client-ui-workspace/lib/client.js` + read `lib/types/client/tree.d.ts` |
| F10 | platform module seed table | `grep -o "staticModules" dsh-web-frontend/dist/assets/index-*.js` then extract the seed function's full key list |
| F11 | `defineTool` output mandatory, `required` per-property | `grep -o "required?: true" dsh-tools/lib/types/schema.d.ts` + `grep "Mandatory canonical output" dsh-tools/lib/types/index.d.ts` |
| F12 | `dsh plugin` requires pnpm on PATH (exit 127 otherwise) | `grep -o "pnpm was not found\|pnpm not found on PATH" dsh/lib/plugin-*.js` |
| F13 | web surface disables agent-plane rows; sessions mount presets | `grep -o "lets each session mount a preset instead" dsh-web-app/cordis.patch.yml` + `grep -A2 "id: tool-bash" dsh-web-app/cordis.patch.yml` |

3. **Record the verdicts**: drifted facts → fix the corresponding lines in SKILL.md / references and update the source citations in `references/fact-sources.md`; no drift → just append a row to the log below.
4. **Ship**: commit (`recheck: against dsh <new>`) → set `META/dsh-baseline.txt` to `<new>` (the scheduled watcher compares against this file and auto-closes its tracking issue) → tag (next version) → push (including `--tags`).
5. **Clean up**: `rm -rf /tmp/dsh-recheck`.

> The release watcher (`.github/workflows/dsh-release-watcher.yml`) runs weekly and on demand: npm `latest` ≠ baseline file → it opens/updates one tracking issue with this procedure; baseline catches up → it closes the issue. Nobody has to remember to look.

## Recheck log

| Date | Notes commit | dsh version | Verdict |
|---|---|---|---|
| 2026-09-18 | d226319 (first-release baseline) | 0.1.5-rc.2 | all 11 facts verified on that version |
| 2026-09-18 | (this round) | 0.1.6-alpha.2 | F1–F11 no behavioral drift; the only source move: F1's reconcile logic relocated from the dsh CLI's `lib/plugin-*.js` into the new package `@deepseek-ai/dsh-plugin-manager/lib/index.js` (`reconcile()`, same semantics: new deps declaring `dsh.bundle.patch` are auto-pushed into bundles; deps without one are installed as plain dependencies with a warning). fact-sources.md citations updated. |

> Baseline: everything in these notes was first verified on `@deepseek-ai/dsh@0.1.5-rc.2` (2026-09-18).
