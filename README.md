<div align="center">

# 📝 dsh-plugin-dev-notes

**Field-verified notes for developing DeepSeek Harness (dsh) plugins — packaged as an agent skill**

English | [简体中文](README.zh-CN.md)

Focuses on the territory the community skills leave out: **dual-face plugins (Node host half + browser client half)**, the `dsh.bundle.patch` one-command install, the `dsh.client` browser roster, `/api` exact routes, the client HMR loop, and a **"the locally installed dsh source is the single source of truth"** API-verification methodology.

[![license](https://img.shields.io/badge/license-MIT-2da44e)](LICENSE)
[![dsh](https://img.shields.io/badge/verified%20on-dsh%200.1.5--rc.2%20%2B%200.1.6--alpha.2-1f6feb)](#-relation-to-versions)
[![format](https://img.shields.io/badge/format-Agent%20Skills-8250df)](https://agentskills.io)

</div>

---

## Why this skill exists

Two excellent community skills already cover dsh plugin development; these notes stand on their shoulders and fill their shared blind spot:

| Reference | Positioning | How these notes use it |
|---|---|---|
| [green-dalii/dsh-plugin-dev-skill](https://github.com/green-dalii/dsh-plugin-dev-skill) | First systematic reference: the Cordis mental model, tools / LLM adapters / config / publishing | Not duplicated — referenced |
| [dsh-io/dsh-plugin-skill](https://github.com/dsh-io/dsh-plugin-skill) | Current API snapshot: the `defineTool` contract, scaffold CLI | Referenced as the tool-API quick index |
| **Official deepseek-harness source** | **The single final source of truth** | Every fact in these notes was verified against the locally installed `@deepseek-ai/dsh@0.1.5-rc.2`, with file-level citations |

The unique content (all field-verified): the dual-face plugin structure and browser bundle shape, `dsh.client` roster field semantics, the platform module seed table, host `/api` exact routes + reusing the auth fence, the externally-attached DOM rendering technique for React-managed UIs (`data-*` + CSS `::before` + fiber reading), exact HMR/restart/refresh semantics, and pitfalls like "bundle layers don't hot-reload".

## Contents

```
dsh-plugin-dev-notes/
├── SKILL.md                        # The operating manual (canonical): the dual-face mental model,
│                                   #   bundle mechanics, exact routes, the client roster,
│                                   #   a verification checklist, and the pitfall table
├── CHECKLIST.md                    # The upgrade recheck procedure: a 15-minute grep checklist
│                                   #   per dsh release, with a running recheck log
└── references/
    ├── web-ui-plugins.md           # Main reference: the full dual-face walkthrough with
    │                               #   per-section verification records
    └── fact-sources.md             # The methodology: three truth layers + 7 source-verification
                                    #   techniques + the fact table (file-level citations)
```

Companion worked example: [cholf5/dsh-plugin-session-emoji](https://github.com/cholf5/dsh-plugin-session-emoji) — the origin of every code sample (an externally attached emoji on sidebar sessions; npm: [`dsh-plugin-session-emoji`](https://www.npmjs.com/package/dsh-plugin-session-emoji)).

## 📦 Install

The skill is standard Agent Skills format (`SKILL.md` + YAML frontmatter); any host that speaks the format can load it.

### One command (recommended): `npx skills`

[skills](https://github.com/vercel-labs/skills) is the package manager for the open Agent Skills ecosystem — it installs straight from this git repo, no registration:

```sh
# dsh, user-level — available in every project
npx skills add cholf5/dsh-plugin-dev-notes -g -a zed

# dsh, project-level — only in the repo you run it from (run at the repo root)
npx skills add cholf5/dsh-plugin-dev-notes -a zed
```

Why `-a zed`: dsh is not yet in the CLI's agent list, but it reads the shared Agent Skills directories — `~/.agents/skills/` (user) and `<git-root>/.agents/skills/` (project). The `zed` alias (same as `cline`, `warp`, `kimi-code-cli`, `loaf`, `sarvam-code`, `dexto`) installs exactly there:

| Scope | Flags | Lands in | dsh discovery root |
|---|---|---|---|
| user-level (every project) | `-g -a zed` | `~/.agents/skills/` | ✅ user-agents (rank 500) |
| project-level (this repo only) | `-a zed` | `./.agents/skills/` | ✅ project-agents (rank 200) — run at the repo root; the CLI does not walk up to the git root |

⚠️ Traps: `-g -a universal` lands in `~/.config/agents/skills/` and `-a claude-code` lands in `.claude/skills/` — dsh scans neither (the latter is fine for Claude Code itself).

Verify and update:

```sh
npx skills ls -g                            # confirm installed
npx skills update dsh-plugin-dev-notes -g   # pull the latest version
```

dsh watches its skill roots live — new sessions (and even the running session's catalog) pick the skill up without a restart.

**Other agents**: run `npx skills add cholf5/dsh-plugin-dev-notes` without `-a` and the CLI auto-detects installed agents (Claude Code, Codex, Cursor, …); or target one explicitly, e.g. `-g -a claude-code` → `~/.claude/skills/`.

### Manual fallback (no npx)

```sh
# user-level: available in every project
git clone https://github.com/cholf5/dsh-plugin-dev-notes.git ~/.dsh/skills/dsh-plugin-dev-notes

# or project-level: only in that repo
git clone https://github.com/cholf5/dsh-plugin-dev-notes.git <project>/.dsh/skills/dsh-plugin-dev-notes

# or symlink (git pull = update)
ln -s /path/to/dsh-plugin-dev-notes ~/.dsh/skills/dsh-plugin-dev-notes
```

DSH follows directory symlinks (the skill-filesystem stats through symlinks and treats them as directory bundles) and watches skill roots live — installs and edits need no restart.

For other agents without the CLI, copy or symlink the repo into their skills dir:

| Agent | Path |
|---|---|
| Claude Code | `~/.claude/skills/dsh-plugin-dev-notes/` |
| Codex / generic | `~/.agents/skills/dsh-plugin-dev-notes/` |

## 🕹️ Usage

- **Automatic**: the agent loads the skill when a task involves dsh plugin development
- **By name**: tell the agent "use dsh-plugin-dev-notes"

## ⚠️ Relation to versions

All facts were verified on `@deepseek-ai/dsh@0.1.5-rc.2` and re-verified against `0.1.6-alpha.2`. dsh moves fast through 0.x rcs — **before relying on an API claim, re-verify it against your local source using `references/fact-sources.md`**. The methodology outlives any individual fact; the local source never lies.

**Maintenance is automated**: a scheduled workflow ([dsh-release-watcher](./.github/workflows/dsh-release-watcher.yml)) compares npm's `latest` against the fact-table baseline ([META/dsh-baseline.txt](./META/dsh-baseline.txt)) and opens a tracking issue when dsh releases. The 15-minute recheck procedure in [CHECKLIST.md](./CHECKLIST.md) is self-contained — any agent (or human) can execute it from the issue alone, no session history required.

## 🙏 Credits

- [green-dalii/dsh-plugin-dev-skill](https://github.com/green-dalii/dsh-plugin-dev-skill) (MIT) — the first systematic reference
- [dsh-io/dsh-plugin-skill](https://github.com/dsh-io/dsh-plugin-skill) (MIT) — the current API snapshot reference
- [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) — the official source, the single source of truth; the quality of its in-package READMEs is why this verification methodology works at all

## 📄 License

[MIT](LICENSE) © cholf5
