# 事实来源分层与源码验证手法

> dsh 处于快速演进期（0.x rc），任何二手资料（包括本 skill 和两个社区 skill）都可能过时。本文规定取材优先级，并给出「30 秒内在本机源码里验证一条 API」的具体手法。**这条方法论比任何具体事实都保值。**

## 1. 分层

| 层 | 角色 | 用法 |
|---|---|---|
| ① 本机安装的官方源码 | **唯一最终事实来源** | 一切 API 签名、行为语义、配置字段的裁决者 |
| ② dsh-io/dsh-plugin-skill | 当前 API 快照参考 | tool API / 脚手架 CLI（`npx @dsh-io/dsh-dev scaffold`）的快速索引 |
| ③ green-dalii/dsh-plugin-dev-skill | 第一版体系化参考 | Cordis 心智模型、tool/LLM adapter/配置/发布的系统讲义 |
| ④ 官方文档站 | 背景 | https://deepseek-harness.github.io/deepseek-harness/ |

规则：②③用于**快速形成假设**，①用于**裁决**。冲突时信 ①；②③里每条要用的 API 都回到 ① 确认签名后再写代码。本仓库（dsh-io）与社区生态的 GitHub topic `dsh-plugin` 可用于发现同类插件源码（如 dsh-pocket 的 esbuild 包装姿势）。

## 2. 找到本机源码

```sh
npm root -g                       # 全局安装：node_modules/@deepseek-ai/
# npx 用户：~/.npm/_npx/<hash>/node_modules/@deepseek-ai/
ls <root>/@deepseek-ai/ | grep dsh # 全家桶：dsh、dsh-base、dsh-web-app、dsh-client-*、dsh-tools、dsh-skill...
```

运行中的 profile（`~/.dsh/profiles/web/`）里 `dsh.profile.bundles` + `cordis.patch.yml` 是**当前真实组合**的组合清单；`~/.dsh/storages/` 是宿主侧用户数据区。探针：`curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3080/api/<路由>` → 401=已注册，404=没有。

## 3. 验证手法（按性价比排序）

1. **包 README 先行**。`@deepseek-ai` 包的 README.md 质量极高（行为契约、配置表、源码地图、设计边界），多数问题读 README 就能裁决。中文版 README.zh.md 同步存在。
2. **类型定义裁决签名**：`<pkg>/lib/types/*.d.ts`（构建时生成，与运行时同步）。例：tool 的 `ToolDefinition.output` 是否必填、`ParameterPropertySpec.required?: true`（per-property required 铁律的出处）、`ctx.connection.fetch` 的路由形状。
3. **构建产物裁决行为**：`<pkg>/lib/*.js` 是未混淆的构建输出，可读、可 grep。例：`dsh-client-modules/lib/index.js` 的 `parseDshClient`（`dsh.client` 字段校验）、`orderByModuleGraph`（external 解析与环检测）、`dsh` CLI 的 `plugin-Ddi42qoW.js`（`reconcilePlugins` 的 bundle 自动挂载）。
4. **浏览器侧事实从产物里挖**：平台模块种子表不在任何 README 里——从 `dsh-web-frontend/dist/assets/index-*.js` 里 grep `staticModules` 提取（react、react/jsx-runtime、react-dom、react-dom/client、cordis、dsh-client-store、dsh-client-ui-slots、dsh-client-ui-primitives、dsh-client-ui-dockkit @ 0.1.5-rc.2）。
5. **上游 README 里的相对链接是免费的地图**：包 README 的 "Further Exploration" 指向同仓库其它包与 `.agents/notes/**` 设计决策记录——按图索骥比搜索快。
6. **运行中进程**：`--dump-config` 预览组合树；DevTools console 看 `web boot: N entries did not activate`（会列出 pending 的服务名）；`/plugins/events` SSE 有 graph/rebuilt 帧。
7. **动手验证一条链路**：`node --check`/`import` 过 host 半；`vm` stub `window.__ModuleLoader__` 跑 client factory；`curl` 探路由；最后真机装（`dsh plugin add link:... -w`）。

## 4. 本次会话裁决过的事实（含出处，供交叉核对）

| 事实 | 出处（0.1.5-rc.2） |
|---|---|
| `dsh.bundle.patch` → `dsh plugin add` 自动挂进 `dsh.profile.bundles` | dsh CLI `lib/plugin-Ddi42qoW.js` `reconcilePlugins()` |
| `- insert:` 行格式；patch 后层按行胜出、整 config 替换 | `dsh-web-app/cordis.patch.yml` 头注 + `dsh` README |
| 精确路由按 path 键控、`/api` 围栏先于分发 | `dsh-client-connection/lib/index.js` `registerFetchRoute` / L767 `/api` 路由注册 |
| `dsh.client` 字段校验：platform 必填、inject/external/immediately | `dsh-client-modules/lib/index.js` `parseDshClient` |
| bundle = `window.__ModuleLoader__.load({id, factory})`，懒 CJS | `dsh-client-modules/lib/client.js` 头注 + 实际包产物 |
| skill 名 pattern `^[a-z0-9]+(?:-[a-z0-9]+)*$` | `dsh-skill/lib/index.js` L17 `SKILL_NAME` |
| skill 根：`.dsh/skills`(100)/`.agents/skills`(200)/custom(300)/`~/.dsh/skills`(400)/`~/.agents/skills`(500)；frontmatter `disable-model-invocation`/`user-invocable`（拒绝 legacy 驼峰键） | `dsh-skill-filesystem/README.md` + `lib/index.js` `parseInvocationPolicy` |
| HMR：stat 轮询 500ms、仅真实 revision 变化触发、map-only 不重载、FAILED 不回滚、`data-plugin` style tag 清理 | `dsh-client-hmr/README.md` |
| 会话行结构 `[role=treeitem]`、fiber props `node` | `dsh-client-ui-workspace/lib/client.js` `SessionNodeItem` + `lib/types/client/tree.d.ts` |

## 5. 引用纪律

- 写进代码/文档的每条 API 事实，标注「验证过」还是「据资料」。验证过的给出处；没验证的明确写「未验证，用前确认」。
- 你的插件里留一句「本插件在 dsh <版本> 验证」的注释，升级出问题时第一个排查点就是它。
