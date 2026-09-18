# 双面插件全流程：从零到跑通一个 Web 界面插件

> 本文是 `dsh-plugin-dev-notes` skill 的主参考。全部代码取自实机验证过的插件 `dsh-plugin-session-emoji`（在 dsh `0.1.5-rc.2` 上开发、安装、验证），每节末尾标注验证方式。

## 0. 目标插件长什么样

功能：Web 侧栏每个会话标题前渲染外挂 emoji；右键会话行打开选择器；映射持久化在宿主侧。

```
dsh-plugin-session-emoji/
├── package.json          # 双面声明 + bundle patch 声明
├── cordis.patch.yml      # 自带 patch（一条命令安装的关键）
├── lib/index.js          # Host 半：GET/POST /api/session-emoji + JSON 持久化
├── lib/client.js         # Client 半：行首渲染 + 右键菜单 + 选择器（自包含，零 import）
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

## 1. Host 半：精确 Fetch 路由

服务定位：Web GUI 所有浏览器↔宿主流量走 `/api` 前缀（`dsh-client-connection`，源码常量 `API_PATH = "/api"`）。该包对每个 `/api` 请求先做信任围栏（Host/Origin 校验）+ 浏览器 cookie 认证，再分发到：精确路由表 → RPC 通道。**精确路由自动享受围栏，插件零鉴权代码。**

```js
export const name = "session-emoji";
export const inject = ["connection"];        // 函数插件的 inject 导出会被 Loader 兑现

import { promises as fs } from "node:fs";
import { homedir } from "node:os";
import { dirname, join } from "node:path";

export async function apply(ctx) {
  const connection = ctx.connection;
  connection.fetch.register({
    path: "/api/session-emoji",              // 必须在 /api/ 下，否则注册时 throw
    methods: ["GET", "HEAD", "POST"],        // 路由按 path 键控：同路径只能注册一次
    requestBody: "buffered",
    fetch: async (request) => {
      if (request.method === "HEAD") return new Response(null, { status: 200 });
      if (request.method === "GET")
        return Response.json({ emojis: await loadMap() }, { headers: { "cache-control": "no-store" } });
      const body = await request.json();     // POST：校验 → 串行化写盘 → 回全量映射
      // ... 校验 sessionId / emoji ...
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

宿主侧持久化：`$DSH_HOME/storages/<名字>.json`（workspace controller 的 `workspace.json` 同款目录）。写盘模式：promise 链串行化（Node 单线程但 async fs 会交错）+ temp 文件 rename 原子替换 + 目录 `0o700`。

注册返回 disposer（内部 `owner.effect`），插件卸载/重载时路由自动注销——无需手动清理。

**验证记录**：装好后 `curl http://127.0.0.1:3080/api/session-emoji` 无 cookie → `401 unauthorized`（围栏生效、路由已注册）；404 = 没装上。带有效 cookie 的 GET 返回了种子数据全量 JSON。

## 2. Client 半：进入浏览器 roster

### 2.1 声明与加载管线

宿主侧 `dsh-client-modules` 扫描组合树里每个带 `dsh.client` 的活动行，增量组合进 `window.__DSH_BOOT__`，随 index HTML 注入页面；浏览器按图懒加载 bundle（执行 bundle 只注册 factory，副作用在首次 materialize 时才跑）。

`dsh.client` 字段（0.1.5-rc.2 `parseDshClient` 校验逻辑）：

| 字段 | 类型 | 作用 |
|---|---|---|
| `platform` | string（必填） | `"web"` |
| `inject` | string[] | 依赖的插件行（先加载），同时是 bundle 内 `require("@deepseek-ai/<pkg>")` 可解析的来源 |
| `external` | string[] | 额外非平台模块请求（`<pkg>` 或 `<pkg>/client` 都解析到该包的 client 半） |
| `immediately` | boolean | 启动即预取（否则首次使用才加载） |

### 2.2 bundle 文件形状

整个 `lib/client.js` 就是：

```js
window.__ModuleLoader__.load({
  id: "dsh-plugin-session-emoji",            // 必须等于包名
  factory: (require) => {
    var module = { exports: {} };
    var exports = module.exports;
    // ---- 全部代码；所有副作用（含 CSS 注入）都在这个闭包里 ----
    async function apply(ctx) {
      // ...注册 MutationObserver / 事件监听...
      return async () => { /* cleanup：断开 observer、移除监听、删 style tag */ };
    }
    exports.inject = [];                     // 客户端 cordis 服务注入（不需要就空数组）
    exports.apply = apply;
    return module.exports;
  }
});
```

- `require()` 解析顺序：平台种子表 → 已 materialize 模块 → graph 行 → 已注册 factory → **throw**（不在表里且没进 `dsh.client.inject` 的包直接运行时报错）。
- 平台种子表（从已构建前端 `dsh-web-frontend/dist/assets/index-*.js` 中提取的 `by()` 函数）：`react`、`react/jsx-runtime`、`react-dom`、`react-dom/client`、`@deepseek-ai/cordis`、`@deepseek-ai/dsh-client-store`、`@deepseek-ai/dsh-client-ui-slots`、`@deepseek-ai/dsh-client-ui-primitives`、`@deepseek-ai/dsh-client-ui-dockkit`。**表会随版本变，用前先验证（见 fact-sources.md）。**
- 想用 React/JSX：学 dsh-pocket 用 esbuild 包装产出上述形状（`format:'cjs', platform:'browser'`，`external: ['react', 'react/jsx-runtime', ...]`，把产物文本嵌进 `__ModuleLoader__.load` 包装，闭包里 `var React = require("react")`）。零依赖 UI（纯 DOM）则完全不需要打包器。

### 2.3 操作既有 UI：先槽位，后 DOM 增强

1. **槽位优先**：读目标包 README 与 `lib/types/client/`，找 `slots.inject` / hole / 注册点。例：ui-workspace 自己就是 `ctx.slots.inject("sidebar.workspaces", ...)` 挂进侧栏的，并声明 `conversation.hero.workspace.directoryFlow` 这类 hole 给 picker 包填。
2. **DOM 增强**（目标组件没留扩展点时，本插件的路径）：
   - **行定位**：`MutationObserver`（`document.body`，`childList+subtree`，rAF 批处理）观察 `[role="treeitem"]`。
   - **行 → 数据映射**：React 在 DOM 节点上挂 `__reactFiber$<随机>` key；`Object.keys(el)` 找到 fiber 后沿 `fiber.return` 上溯，读各层 `memoizedProps` 找行组件的数据 props（本例：`props.node` 且 `{id:string, title:string, updatedAt:number}`——`GroupNode` 无 `id`、搜索行是 `props.result`，天然区分）。上限 25 跳防呆。
   - **渲染**：绝不往 React 管理的子节点插元素。在目标 span 上 `setAttribute("data-dsh-session-emoji", emoji)`，另注入一个 `<style data-plugin="<包名>">`，用 `span[data-dsh-session-emoji]::before { content: attr(data-dsh-session-emoji); ... }` 画出来。React 重渲染不清除未知属性；style tag 带 `data-plugin` 是 HMR 卸载清理的约定（client-hmr 移除插件拥有的 style tag）。
   - **弹层**：`contextmenu` 捕获监听 + 纯 DOM 弹层挂 `document.body`（fixed 定位 + 高 z-index，天然逃出侧栏 overflow）；外点关闭用 capture 阶段 `pointerdown`。样式跟随 dsh 主题 token（`--dsw-alias-*`），带 fallback 值双主题可用。

**验证记录**：client.js 在 Node `vm` 里 stub `window.__ModuleLoader__` 跑 factory——确认注册 id 与 `exports.apply` 形状（无浏览器抓语法/注册错误）；真机验证走用户截图 + boot graph 检查（§5）。

## 3. 安装与生命周期

```sh
dsh plugin --profile web add dsh-plugin-session-emoji -w     # npm 源
# 或 git+https://github.com/<you>/<pkg>.git -w                # git 源（无构建脚本最省事）
# 或 link:/abs/path -w                                        # 开发期软链，改源码直接生效
dsh plugin --profile web update dsh-plugin-session-emoji -w  # 更新（跨大版本加 --latest）
dsh plugin --profile web remove dsh-plugin-session-emoji -w  # 卸载（自动移出 bundles）
dsh --profile web --dump-config                              # 不启动，预览组合树
```

机制：`dsh plugin` 转发 pnpm（cwd = profile 目录）→ 成功后 `reconcilePlugins` 把声明 `dsh.bundle.patch` 的依赖自动追加进 profile `package.json` 的 `dsh.profile.bundles`。bundle 的 patch 按列表顺序参与组合树（builtin bundles → bundles → profile patch → home patch → --patch）。

- `-w` 必带（profile 是 pnpm workspace）。
- git 依赖无构建脚本最省事；有 `prepare` 则需在 profile `pnpm-workspace.yaml` 加 `allowBuilds`。
- **bundle 层增删不热重载，重启 `dsh web` 生效**。装/卸后稳态探针：路由 404 = 未生效。

**验证记录**：本机实跑 remove(git)+add(npm) 全程，`dsh.profile.bundles` 自动增删；`reconcilePlugins` 行为与 dsh CLI 源码（`plugin-Ddi42qoW.js`）一致。

## 4. 宿主进程内验证（无浏览器）

boot graph 随 index HTML 注入，认证后才可见。两种途径：

1. 浏览器 DevTools：Sources 里搜自己的包名（`__DSH_BOOT__` script），Console 看加载失败与 `web boot: N entries did not activate` 提示（会列出 pending 的服务名）。
2. 程序化（本机自有凭据，无权限提升）：`$DSH_HOME/.credentials.yaml` 里 `client-connection/browser-session` grant 的 secret 即 cookie 签名密钥。cookie 名 = `dsh-auth-` + base64url(sha256(authority))，值 = `v1.<base64url(payload)>.<base64url(HMAC-SHA256(secret, body))>`，payload = `{version:1, authority:"127.0.0.1:3080", issuedAt, expiresAt}`。带上即可 `GET /`（grep 包名确认在 boot graph + combo URL）与 `GET /api/<路由>`（看业务数据）。**这只是把文件读取权换成了等价的 HTTP 视角，不越过任何本机已有的权限边界。**

## 5. 热更与刷新语义（最容易懵的部分）

| 场景 | 行为 |
|---|---|
| 改 `lib/client.js` 保存 | `dsh-client-hmr` stat 轮询（默认 500ms）发现变化 → `rebuilt()` → 浏览器内卸旧挂新，**无需刷新页面**。仅 source map 变化不触发；reload 失败进 FAILED 不回滚；React 状态丢、数据层不丢 |
| 改 bundle 的 `cordis.patch.yml` | patch 文件被 watch，热重载 |
| 装/卸/更新包（`dsh.profile.bundles` 变化） | **不热重载，重启 `dsh web`**。pnpm 重组 node_modules 过程中的路由闪现（401→404）是瞬态，以稳态为准 |
| boot graph 变了但没热更 | 浏览器**刷新页面**（graph 随 index 注入） |

**验证记录**：热下线（删 patch 条目 → boot graph rev 变化 + 路由 404，全程未重启）与热替换（client.js 改写 → 浏览器内换新）均实测。

## 6. 完整参考实现

https://github.com/cholf5/dsh-plugin-session-emoji —— 本文所有代码与验证的原始出处，可直接对照阅读。
