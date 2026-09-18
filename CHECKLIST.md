# 升级复核清单（CHECKLIST）

> dsh 处于 0.x rc 快速演进期，本笔记每条事实都有版本半衰期。本清单把「dsh 升级后复核一次」压成 15 分钟的可执行流程——**这是本仓库长期价值的核心维护动作**。
>
> 验证方法论细节见 `references/fact-sources.md`；本文是其调度层。

## 何时触发

任一条件满足即跑一轮：

```sh
# 本机安装版本
dsh --version   # 或看 ~/.npm/_npx/*/node_modules/@deepseek-ai/dsh/package.json
# npm 最新版
npm view @deepseek-ai/dsh version
```

- npm `latest` 版本号 > 笔记基线版本（见底部复核记录表）
- 用户报告「skill 说的和实际行为对不上」

## 复核步骤

1. **取新源码，不动本机安装**（运行中的 dsh 永远不要碰）：
   ```sh
   mkdir -p /tmp/dsh-recheck && cd /tmp/dsh-recheck
   npm pack @deepseek-ai/<pkg>@<new> --pack-destination .
   tar -xzf deepseek-ai-<pkg>-<new>.tgz -C <pkg> --strip-components=1
   ```
   需要复核的包（见下表）。注意 0.1.6-alpha.2 起 `dsh plugin` 的 reconcile 逻辑搬进了新包 `@deepseek-ai/dsh-plugin-manager`。
2. **逐条 grep 裁决**（关键事实 → 验证命令，当前全部针对事实表行）：

| # | 事实 | 验证命令（在解包目录内） |
|---|---|---|
| F1 | `dsh.bundle.patch` 依赖自动挂进 `dsh.profile.bundles` | `grep -n "dsh?.bundle" <dsh-plugin-manager 或 dsh>/lib/*.js` 找 `reconcile` |
| F2 | patch 后层按行胜出、整 config 替换 | `grep -o "replaces the targeted row" dsh-web-app/cordis.patch.yml` |
| F3 | 精确路由按 path 键控；`/api` 围栏先于分发 | `grep -o 'API_PATH = "[^"]*"' dsh-client-connection/lib/index.js` + `grep fetchRoutes.has` |
| F4 | `dsh.client` 字段校验（platform/inject/external/immediately） | `grep -o "parseDshClient" dsh-client-modules/lib/index.js` 后读上下文 |
| F5 | bundle 形状 `window.__ModuleLoader__.load({id, factory})` 懒 CJS | `grep -c "__ModuleLoader__.load" dsh-client-modules/lib/client.js` |
| F6 | skill 名 pattern | `grep -o "SKILL_NAME = [^;]*" dsh-skill/lib/index.js` |
| F7 | skill 根表 / frontmatter 键 / symlink 跟随 | `grep -o "followSymlinks\|nodeEntryKind\|disable-model-invocation" dsh-skill-filesystem/lib/index.js` |
| F8 | HMR stat 轮询（默认 500ms）、map-only 不重载、FAILED 不回滚 | `grep -o "pollIntervalMs" dsh-client-hmr/README.md` + 读 README 变更 |
| F9 | 会话行 `[role=treeitem]`、fiber `props.node`、`SessionNode{id,title,updatedAt}` | `grep -c 'role: "treeitem"' dsh-client-ui-workspace/lib/client.js` + 读 `lib/types/client/tree.d.ts` |
| F10 | 平台模块种子表 | `grep -o "staticModules" dsh-web-frontend/dist/assets/index-*.js` 后提取该函数全键列表 |
| F11 | `defineTool` output 必填、`required` per-property | `grep -o "required?: true" dsh-tools/lib/types/schema.d.ts` + `grep "Mandatory canonical output" dsh-tools/lib/types/index.d.ts` |

3. **裁决记录**：漂移的事实 → 改 SKILL.md / references 对应行，并在 `references/fact-sources.md` 的出处表更新出处；无漂移 → 只在底部记录表加一行。
4. **发布**：提交（`recheck: against dsh <new>`）→ 打 tag `v0.1.<n>`（复核轮次递增）→ push（含 `--tags`）。
5. **清理**：`rm -rf /tmp/dsh-recheck`。

## 复核记录

| 日期 | 笔记 commit | dsh 版本 | 结论 |
|---|---|---|---|
| 2026-09-18 | d226319（首次发布基线） | 0.1.5-rc.2 | 全部 11 条事实在该版本上验证通过 |
| 2026-09-18 | （本轮） | 0.1.6-alpha.2 | F1–F11 全部无行为漂移；唯一出处变化：F1 的 reconcile 逻辑从 dsh CLI 的 `lib/plugin-*.js` 移入新包 `@deepseek-ai/dsh-plugin-manager/lib/index.js`（`reconcile()`，语义不变：声明 `dsh.bundle.patch` 的新依赖自动 push 进 bundles，无声明则告警装为普通依赖）。fact-sources.md 出处已更新。 |

> 基线：本笔记所有内容最初在 `@deepseek-ai/dsh@0.1.5-rc.2` 上验证（2026-09-18）。
