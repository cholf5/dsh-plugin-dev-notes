<div align="center">

# 📝 dsh-plugin-dev-notes

**DeepSeek Harness (dsh) 插件开发实战笔记 —— 一个 agent skill**

[English](README.md) | 简体中文

聚焦社区资料没覆盖的领域：**双面插件（Host Node 半 + 浏览器 Client 半）**、`dsh.bundle.patch` 一条命令安装、`dsh.client` 浏览器 roster、`/api` 精确路由、client HMR 热更循环，以及一套「**以本机安装的 dsh 源码为唯一事实来源**」的 API 验证方法论。

[![license](https://img.shields.io/badge/license-MIT-2da44e)](LICENSE)
[![dsh](https://img.shields.io/badge/verified%20on-dsh%200.1.5--rc.2%20%2B%200.1.6--alpha.2-1f6feb)](#-与版本的关系)
[![format](https://img.shields.io/badge/format-Agent%20Skills-8250df)](https://agentskills.io)

</div>

---

## 为什么自己做一个

社区已有两份优秀的 dsh 插件开发 skill，本笔记站在它们肩膀上，但补上了它们的共同盲区：

| 参考 | 定位 | 本笔记如何使用 |
|---|---|---|
| [green-dalii/dsh-plugin-dev-skill](https://github.com/green-dalii/dsh-plugin-dev-skill) | 第一版体系化参考：Cordis 心智模型、tool / LLM adapter / 配置 / 发布 | Cordis 铁律与 tool 之外的内容**不重复**，直接引用 |
| [dsh-io/dsh-plugin-skill](https://github.com/dsh-io/dsh-plugin-skill) | 当前 API 快照：`defineTool` 契约、脚手架 CLI | 作为 tool API 的快速索引被引用 |
| **官方 deepseek-harness 源码** | **唯一最终事实来源** | 本笔记每条事实都在本机安装的 `@deepseek-ai/dsh@0.1.5-rc.2` 上逐一验证，附文件级出处 |

本笔记的独有内容（全部实机验证）：dual-face 插件结构与浏览器 bundle 形状、`dsh.client` roster 字段语义、平台模块种子表、宿主 `/api` 精确路由 + 认证围栏复用、对 React 管理的 DOM 做外挂渲染的手法（`data-*` + CSS `::before` + fiber 读取）、HMR/重启/刷新的准确语义、bundle 层不参与热重载等踩坑实录。

## 内容结构

```
dsh-plugin-dev-notes/
├── SKILL.md                        # 操作手册（canonical，英文）：双面心智模型、bundle 机制、
│                                   #   精确路由、client roster、验证清单、踩坑表
├── CHECKLIST.md                    # dsh 升级复核清单：15 分钟逐条 grep 复核流程 + 复核记录
└── references/
    ├── web-ui-plugins.md           # 主参考：dual-face 全流程代码 + 每节实机验证记录
    └── fact-sources.md             # 方法论：三层事实来源 + 7 种源码验证手法
                                    #   + 裁决事实表（带文件级出处 + 0.1.6-alpha.2 复核列）
```

> 说明：skill 正文为英文（单一 canonical 版本，AI 消费者双语无差别）；本中文 README 是人类读者的入口。

配套实机样例：[cholf5/dsh-plugin-session-emoji](https://github.com/cholf5/dsh-plugin-session-emoji) —— 笔记里所有代码的原始出处（侧栏会话 emoji 外挂，npm: [`dsh-plugin-session-emoji`](https://www.npmjs.com/package/dsh-plugin-session-emoji)）。

## 📦 安装

标准 Agent Skills 格式（`SKILL.md` + YAML frontmatter），任何支持该格式的 agent 都能用。

**DSH**（项目级 rank 100 / 用户级 rank 400）：

```sh
# 用户级：所有项目可用
git clone https://github.com/cholf5/dsh-plugin-dev-notes.git ~/.dsh/skills/dsh-plugin-dev-notes

# 或项目级：只在该仓库生效
git clone https://github.com/cholf5/dsh-plugin-dev-notes.git <project>/.dsh/skills/dsh-plugin-dev-notes

# 或软链（git pull 即更新）
ln -s /path/to/dsh-plugin-dev-notes ~/.dsh/skills/dsh-plugin-dev-notes
```

DSH 对目录软链完全兼容（skill-filesystem 会 stat 跟随 symlink 识别为 directory bundle），且 skill 目录被实时 watch——安装、修改无需重启。

**其它 agent**（同一份内容）：

| Agent | 路径 |
|---|---|
| Claude Code | `~/.claude/skills/dsh-plugin-dev-notes/` |
| Codex / 通用 | `~/.agents/skills/dsh-plugin-dev-notes/` |

## 🕹️ 使用

- **自动触发**：任务涉及 dsh 插件开发时，agent 按 skill 描述自动加载
- **按名加载**：对 agent 说「用 dsh-plugin-dev-notes」即可

## ⚠️ 与版本的关系

所有事实在 `@deepseek-ai/dsh@0.1.5-rc.2` 上验证，并已对照 `0.1.6-alpha.2` 复核。dsh 处于快速演进期（0.x rc）——**引用本笔记的 API 前请按 `references/fact-sources.md` 的手法对本地源码复核**。这条方法论比任何具体事实都保值；本机源码不会撒谎。

## 🙏 引用与致谢

- [green-dalii/dsh-plugin-dev-skill](https://github.com/green-dalii/dsh-plugin-dev-skill)（MIT）—— 第一版体系化参考
- [dsh-io/dsh-plugin-skill](https://github.com/dsh-io/dsh-plugin-skill)（MIT）—— 当前 API 快照参考
- [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) —— 官方源码，唯一事实来源；其 npm 包内 README 的质量是本笔记方法论的信心来源

## 📄 License

[MIT](LICENSE) © cholf5
