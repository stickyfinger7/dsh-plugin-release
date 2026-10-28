# dsh-plugin-release

A **DeepSeek Harness (DSH) skill** that takes a local plugin from "works on my machine" to a public,
installable GitHub repository.

It is a procedure, not a script: the agent audits the repo, fills in packaging metadata, writes docs
and tests, proves a format pass did not change behavior, builds a clean first commit, tags it,
publishes with `gh`, and then verifies the result from a fresh clone.

> This repository is a **skill bundle**, not a plugin. `dsh plugin add` does not install it — clone
> it into a skill root (see below).

## Install

DSH discovers skills from these roots (highest priority first):

| Rank | Root |
|---|---|
| 100 | `<project>/.dsh/skills` |
| 200 | `<project>/.agents/skills` |
| 300 | `customSkillDirs` from config |
| 400 | `~/.dsh/skills` |
| 500 | `~/.agents/skills` |

User-level — available in every project:

```sh
git clone https://github.com/stickyfinger7/dsh-plugin-release.git ~/.dsh/skills/dsh-plugin-release
```

Project-level — travels with the repository:

```sh
git clone https://github.com/stickyfinger7/dsh-plugin-release.git .dsh/skills/dsh-plugin-release
```

No restart needed: DSH watches the roots and refreshes the catalog for the next model step.

## Use

- **Automatic.** Ask for the outcome — "把这个插件清理后发布到 GitHub", "release this plugin" — and the
  skill is loaded when the task matches its description.
- **Manual.** Type `/dsh-plugin-release` in the composer; the slash menu lists every user-invocable
  skill.

The frontmatter omits both invocation switches, so both surfaces are open. To narrow it:

| Goal | Frontmatter |
|---|---|
| Model-only (hide from `/`) | `user-invocable: false` |
| User-only (never auto-load) | `disable-model-invocation: true` |

## What it covers

| Phase | Content |
|---|---|
| 0 | Settle owner, visibility, commit identity, license holder, whether CI is wanted |
| 1 | Audit the repo and delete leftovers before anything else |
| 2 | `package.json` / `dsh.plugin.json` field-by-field, `npm pack --dry-run` check |
| 3 | One formatting pass, plus a minified before/after comparison proving semantics did not change |
| 4 | `node:test` coverage list for host stores, prompt rendering, injection and HTTP guards |
| 5 | README / README.zh / CHANGELOG / LICENSE / AGENTS requirements |
| 6 | Clean git history: one initial commit, annotated tag, re-tag after an amend |
| 7 | `gh` auth (device flow, proxy), `gh repo create`, topics, release, push without a credential helper |
| 8 | Fresh-clone verification and a final report |
| Appendix | Pitfalls table, a "never" list, and a Chinese quick reference |

The pitfalls come from real runs: `gh auth login` hanging under a fake PTY, `github.com` timing out
while `api.github.com` works, `workflow` scope rejection on `.github/workflows/*`, missing credential
helpers, `npm` cache permission errors, and `node --test test/` resolving a directory as a module.

## Requirements

- DSH `0.2.x` (skills support).
- `gh`, `git`, Node.js `>=22.19.0`, and a POSIX shell.

## License

MIT — see [LICENSE](LICENSE).

---

## 中文

把本地插件从"能跑"整理成"可公开安装"的 DSH **技能**。它不是脚本，而是流程：审计 → 元数据 →
文档 → 测试 → 格式统一（并用压缩前后逐字节比对证明语义未变）→ 干净的首个提交 → 打 tag →
`gh` 发布 → 从全新 clone 验证。

安装（用户级，所有项目可用）：

```sh
git clone https://github.com/stickyfinger7/dsh-plugin-release.git ~/.dsh/skills/dsh-plugin-release
```

用法：直接说"把这个插件清理后发布到 GitHub"会自动加载；也可以输入 `/dsh-plugin-release`。
DSH 监听技能目录，装完不用重启。

它刻意做成技能而不是插件：判断（文档齐不齐、测试该覆盖什么、什么时候该确认可见性）留在可读的
Markdown 里，机械步骤用经过验证的命令；不引入需要信任的第三方代码。
