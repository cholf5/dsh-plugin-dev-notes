---
name: dsh-plugin-dev-notes
description: 开发 DeepSeek Harness (dsh) 插件的双面（Node + 浏览器）实战笔记：dual-face 插件结构、dsh.bundle.patch 一条命令安装、dsh.client 浏览器 roster、/api 精确路由、client HMR 热更循环，以及「以本机安装的 dsh 源码为唯一事实来源」的验证方法论。
whenToUse: 当任务涉及为 DeepSeek Harness (dsh) 编写、修改或调试插件，尤其是 Web 界面（浏览器侧）插件、双面插件、bundle 打包与安装、@deepseek-ai/dsh-* API 用法，或需要针对本机安装的 dsh 版本验证插件 API 事实时加载。
---

# dsh 插件开发实战笔记（dual-face 视角）

> 本 skill 是一份经过实机验证的开发笔记，聚焦一个更完整视角：**插件 = Host 半（Node/Cordis）+ Client 半（浏览器）**。
> 相比社区已有的 tool 向资料（green-dalii/dsh-plugin-dev-skill 的 Cordis 心智模型、dsh-io/dsh-plugin-skill 的 tool API 快照），本笔记补上了 Web UI 插件这块它们没覆盖的领域，并把「一切以本机安装的官方源码为唯一事实来源」作为工作方法。
>
> 所有代码模板均在 dsh `0.1.5-rc.2` 上实机跑通（见 references/web-ui-plugins.md 的验证记录）。

## 0. 三层事实来源（先读这个）

写任何 dsh 代码前，按此优先级取材：

1. **官方源码 = 唯一最终事实来源。** 你机器上正在运行的 dsh 安装就是权威：包 README（质量极高）+ 构建产物 + 类型定义。位置：npm 全局安装的 `node_modules/@deepseek-ai/dsh*`（`npm root -g` 或 npx 缓存里找）。
2. **当前 API 快照参考**：dsh-io/dsh-plugin-skill（tool API 与脚手架 CLI `npx @dsh-io/dsh-dev scaffold`）。
3. **第一版体系化参考**：green-dalii/dsh-plugin-dev-skill（Cordis 心智模型、tool/LLM/配置/发布全套）。

⚠️ 参考仓库与你的 dsh 版本可能不一致（不同 rc 之间 API 会漂移）。**凡引用的 API，先在本地源码 grep 确认签名再用**（方法见 references/fact-sources.md）。

## 1. 心智模型：一个包，两个半身

dsh 的 Web 界面本身就是一堆「双面插件」拼出来的（ui-workspace、ui-sidebar…）。你的插件也可以是：

| 半身 | 位置 | 跑在哪 | 导出 |
|---|---|---|---|
| Host | `lib/index.js`（`"main"`） | Node，Cordis Loader | `apply(ctx, config)`、可选 `name`/`inject`/`Config` |
| Client | `lib/client.js`（exports `"./client"`） | 浏览器，dsh 客户端模块系统 | bundle 注册形式（见 §4） |

- `package.json` 的 `"dsh"` 字段同时声明两个半身（可只要其中一个）：
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
- Host 半是普通 Cordis 插件（函数形态为主；需要对外提供服务时用 `Service` 子类）。Cordis 铁律见 green-dalii skill §1-§9，此处不重复；本笔记只强调验证过的一点：**函数插件的 `export const inject = ['connection']` 会被 Loader 兑现**（服务在 apply 前就绪）。
- Client 半不是普通模块：文件内容整体是 `window.__ModuleLoader__.load({ id: "<包名>", factory: (require) => { ...; return module.exports; } })`。官方包用 tsdown/esbuild 产出这个形状；手写或 esbuild 包装都行（dsh-pocket 用 esbuild 包装，见 references/web-ui-plugins.md §6）。
- 不需要 React 就别引 React：自包含 bundle（零外部 import）最省心，平台模块表（platform seed）目前只有 react、react/jsx-runtime、react-dom、react-dom/client、@deepseek-ai/cordis、dsh-client-store、dsh-client-ui-slots、dsh-client-ui-primitives、dsh-client-ui-dockkit（从已构建前端产物中提取，随版本可能变——用前先验证，见 references/fact-sources.md）。

## 2. 一条命令安装的秘密：`dsh.bundle.patch`

`dsh plugin --profile web add <包名> -w` 之所以一步到位：命令转发 pnpm 装包，装完 `reconcilePlugins` 扫描每个依赖的 manifest，**声明了 `dsh.bundle.patch` 的包自动追加进 profile 的 `dsh.profile.bundles` 层叠列表**。包自带的 `cordis.patch.yml` 作为一层 patch 参与组合树。

```yaml
# 包根的 cordis.patch.yml
- insert:
    - id: session-emoji          # 稳定 id
      name: dsh-plugin-session-emoji   # 包名，从 profile node_modules 解析
```

要点：
- `-w` 必须带（profile 是 pnpm workspace，否则 `ERR_PNPM_ADDING_TO_ROOT`）。
- git 依赖拉的是源码；**无构建脚本的包最省事**。有 `prepare` 脚本时 pnpm 默认拒绝，需在 profile 的 `pnpm-workspace.yaml` 加 `allowBuilds`（诚实告知用户这是允许安装期执行代码）。
- patch 层顺序：bundles 依序 → profile `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml` → `--patch` overlays。**后层按行胜出，且替换目标行整个 config（非深合并）**——覆盖别层的行要重述它需要的所有键。
- 更新/卸载：`dsh plugin --profile web update|remove <包名> -w`。
- **bundle 层的增删不参与热重载，装完要重启 `dsh web`**（patch 文件本身的热重载见 §5）。

## 3. Host 半：在共享认证通道上开你自己的 API

Web GUI 的所有浏览器↔宿主通信走 `/api` 前缀，由 `dsh-client-connection` 统一加围栏（Host/Origin 校验 + 浏览器 cookie 认证）。你的插件注册「精确 Fetch 路由」即可白嫖这套安全模型，**零鉴权代码**：

```js
export const inject = ['connection'];

export async function apply(ctx) {
  ctx.connection.fetch.register({
    path: '/api/my-plugin/data',          // 必须在 /api/ 下
    methods: ['GET', 'POST'],             // 同一路径只能注册一次，多方法合一
    requestBody: 'buffered',              // 或流式
    fetch: async (request) => {           // 标准 Fetch API Request → Response
      if (request.method === 'GET') return Response.json({ /* ... */ });
      const body = await request.json();
      return Response.json({ ok: true });
    },
  });
}
```

- 路由按 path 键控：重复注册同一路径会 throw；把多个方法放进一次注册、内部分发。
- 浏览器端直接 `fetch('/api/my-plugin/data')`（同源，cookie 自动带上；curl 无 cookie 会得到 401——这就是验证路由已注册的探针）。
- 需要 RPC/流式/生成端点时才考虑 Typert（`@deepseek-ai/dsh-typert-protocol` 的 `TypertRemoteService` + `Remote` 装饰器 + 生成的 `TYPERT`/`TYPERT_REMOTE` 工件，client 侧 `ctx.remote.$mount()`）；小插件用精确路由足够。
- 持久化：写 `$DSH_HOME/storages/<你的名字>.json`（workspace controller 同款位置）。并发写用 promise 链串行化 + temp/rename 原子替换。

## 4. Client 半：进入浏览器 roster

宿主把每个带 `dsh.client` 的活动行扫描进 `window.__DSH_BOOT__`（boot graph），浏览器按图懒加载 bundle。

- **`dsh.client` 字段**（0.1.5-rc.2 校验逻辑）：`platform`（必填字符串，web）、`inject`（字符串数组：要求哪些插件行先加载，同时是模块 external 的来源）、`external`（额外的非平台模块请求）、`immediately`（boolean：启动即预取，不等首次使用）。
- **bundle 形状**（浏览器 CJS，factory 只注册不执行，副作用全在闭包里、首次 materialize 时才跑）：
  ```js
  window.__ModuleLoader__.load({
    id: "my-plugin",                    // 必须等于包名
    factory: (require) => {
      var module = { exports: {} };
      var exports = module.exports;
      // ...全部代码，副作用都在这里...
      async function apply(ctx) { /* ... */ return async () => { /* cleanup */ }; }
      exports.inject = [];              // 客户端 cordis 的服务注入
      exports.apply = apply;
      return module.exports;
    }
  });
  ```
- **操作既有 UI 的两条路**：
  1. **Slot 注入**（官方姿势）：先看目标包是否留了槽位（`ctx.slots.inject("sidebar.workspaces", ...)` / hole / 声明合并的注册点）。ui-workspace/ui-sidebar 的 README 列了各自的 seat。
  2. **DOM 增强**（目标组件未留扩展点时）：MutationObserver + 结构化选择器（`[role="treeitem"]`）+ React fiber 读取（`__reactFiber$` 前缀 key → 沿 `fiber.return` 找行组件 props 里的数据，如 `node.id`）+ **`data-*` 属性 + 注入一段 CSS `::before content: attr()` 渲染**——绝不往 React 管理的子节点里插元素。设置属性、挂 style tag 都不会被 React 重渲染清掉；style tag 加 `data-plugin="<你的包名>"`（HMR 卸载会清同名的）。
  - 判断标准：有槽位走槽位；没有才 DOM 增强，并把选择器/props 形状记进注释，dsh 升级后好排查。
- **client `inject` 的真相**：它既声明加载顺序（依赖行先上），也是你 bundle 内 `require("@deepseek-ai/<pkg>")` 能解析到的来源（external 语义：<pkg> 或 <pkg>/client 解析到该包的 client 半；平台表内的名字无需声明）。

## 5. 开发热更循环（省时间的核心）

| 改动 | 生效方式 |
|---|---|
| `lib/client.js`（浏览器半） | **保存即热替换**：`dsh-client-hmr` 对每个 graph bundle 做 stat 轮询（默认 500ms），检测到变化走 `rebuilt()` → 浏览器内卸旧挂新，无需刷新页面 |
| Host 半 | 重启 `dsh web`（Host 代码随进程加载） |
| bundle 的 `cordis.patch.yml`（内容行变更） | patch 文件被 watch，热重载 |
| `dsh.profile.bundles`（装/卸包） | **不热重载，重启** |

开发期装法：`dsh plugin --profile web add link:/abs/path/to/pkg -w`（link: 软链，改源码直接生效）；或 `file:`（复制，改完要重装）。
注意：HMR 换的是插件实例，React 状态会丢（连接/会话等数据层不丢）；reload 失败该插件进 FAILED 态，不自动回滚；仅 source map 变化不触发代码重载。

## 6. 验证清单（动手前后各过一遍）

- [ ] `node --check` / 实际 import 过 host 半；bundle 用 vm 模拟 `__ModuleLoader__.load` 跑一遍 factory（无浏览器也能抓住语法/注册错误）
- [ ] `dsh --profile web --dump-config` 预览组合树，确认你的行在（别启动）
- [ ] 装好后探针：`curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:3080/api/<你的路由>` → 401=已注册（围栏拦下未认证请求），404=没装上；带浏览器 cookie 才能看到 200
- [ ] 浏览器侧：刷新页面（boot graph 随 index 注入，热更后不用刷）；DevTools console 看 `[你的插件名]` 告警与 failed entries 提示
- [ ] 卸载路径：`dsh plugin --profile web remove <包名> -w` → 重启 → 探针 404、界面无残留

## 7. 常见坑（实机踩过的）

| 坑 | 真相 |
|---|---|
| 手动改 profile 的 cordis.patch.yml 而不是用 bundle | 能跑但不是惯例；用 `dsh.bundle.patch` 才是一条命令 |
| 同一路径注册两次精确路由 | `fetchRoutes` 按 path 键控，第二次 throw；多方法合进一次注册 |
| bundle 里 `require("react")` 但没进平台表 | react 在平台表里没问题；**不在表里的包必须进 `dsh.client.inject`（external）**，否则运行时 throw "missed the module table" |
| 往 React 管理的子节点插 DOM | 会被 reconcile 清/乱；用 `data-*` + CSS `::before` 或 slot |
| 忘了 `-w` | pnpm 报 `ERR_PNPM_ADDING_TO_ROOT` |
| 装完没反应就一直等 | bundle 层不热重载，重启 `dsh web`；client.js 改动才是热替换 |
| `pnpm peers check` 报缺 peer | 先看是谁的 peer——常见是别的插件（如 dsh-pocket）的，与你无关 |
| 依赖别的包 client 半的导出 | 对方得真的从 `./client` 导出（官方包类型在 `lib/types/client/`），且把它加进你的 `dsh.client.inject` |

## 8. 延伸阅读

- `references/web-ui-plugins.md` —— dual-face 全流程：从零到跑通一个 Web 插件的完整代码与验证记录（本 skill 的主菜）
- `references/fact-sources.md` —— 事实来源分层与「对本地源码验证 API」的 grep 手法清单
- green-dalii/dsh-plugin-dev-skill —— Cordis 心智模型 / tool / LLM adapter / 配置 / 发布（本笔记不重复的部分）
- dsh-io/dsh-plugin-skill —— tool API 快照与 `@dsh-io/dsh-dev` 脚手架
- 官方文档站：https://deepseek-harness.github.io/deepseek-harness/ ；源码：https://github.com/deepseek-ai/deepseek-harness
