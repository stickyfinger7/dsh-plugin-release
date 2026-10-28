---
name: dsh-plugin-release
description: Turn a local DSH plugin (or small Node repo) into a clean, versioned, publicly installable GitHub repository — metadata, docs, tests, formatting, git history, tag, gh publish, release, and post-publish verification.
whenToUse: Use when the user asks to clean up, formalize, version, release, or publish a plugin or repository to GitHub, especially a DeepSeek Harness (DSH) plugin.
---

# Publish a DSH plugin to GitHub

Goal: a public repo at `https://github.com/<owner>/<repo>` with a clean initial commit, an annotated
`vX.Y.Z` tag, a GitHub Release, and a fresh clone that passes `npm run check` and `npm test`.

Work in this order. Ask only for facts you cannot derive: owner, visibility, auth, license holder.
Publishing is a public, hard-to-reverse action — confirm visibility before creating the repo.

## 0. Facts to settle first

- Owner and repo name (`gh api user --jq .login` when already authenticated).
- Public or private. Plugins are normally public; confirm anyway.
- Commit identity: `<login>` + `<id>+<login>@users.noreply.github.com`. Get the numeric id from
  `gh api users/<login> --jq .id`. Never invent a real email address.
- License: keep the existing one; change the copyright holder only to the user's name or handle.
- Whether CI is wanted. A workflow file requires the `workflow` OAuth scope to push (see Pitfalls).

## 1. Audit the repo

```sh
git status --short; git log --oneline | head; git remote -v
find . -maxdepth 2 -type f -not -path './.git/*' | sort
```

A plugin that was never committed looks like this: branch `main` with no commits, every file staged,
no remote. That is fine — you are about to create the first real commit.

Remove leftovers before doing anything else: scratch/preview directories, duplicated docs
(`INSTALL.md` when the README already has install steps), debug logs, vendored samples. Verify a path
is inside the repo before deleting it.

## 2. Metadata checklist

`package.json` is what DSH installs from. Required/expected:

| Field | Value |
|---|---|
| `name` | kebab-case; match the repo name |
| `version` | `X.Y.Z`, start at `1.0.0` for a first public release |
| `description` | one sentence, same as the GitHub repo description |
| `type` | `module` |
| `main` / `exports` | host entry (`lib/host.js`) plus `./client`, `./cordis.patch.yml`, `./package.json` |
| `files` | `lib`, `cordis.patch.yml`, `dsh.plugin.json`, docs, `LICENSE` — never tests or scratch |
| `scripts` | `check` (`node --check` both sources) and `test` (`node --test`) |
| `engines` | `node: >=22.19.0`, `dsh: >=0.2.0` |
| `dsh` | `bundle.patch: ./cordis.patch.yml`, `client.platform: web` |
| `license`, `author`, `homepage`, `repository`, `bugs`, `keywords` | fill all of them; empty fields read as abandoned |

`dsh.plugin.json` is optional release metadata (id, entry, bundle, client, contributes). Mirror the
ecosystem convention if the user wants it; DSH 0.2.x installs from `package.json`, so it must stay
consistent with it.

Run `npm pack --dry-run` and confirm the tarball contains exactly the intended files.

## 3. Normalize the code

Settle on one style and let a formatter apply it; do not hand-edit mixed formatting. A good default
matching the DSH plugin ecosystem:

```json
{ "semi": false, "singleQuote": true, "printWidth": 120, "trailingComma": "all" }
```

```sh
npx --yes prettier@3 --write lib test
```

Prove the rewrite is semantically identical by comparing minified output before and after:

```sh
npx --yes esbuild@0.25.10 /tmp/before.js --minify --charset=utf8 --log-level=error > /tmp/a.min.js
npx --yes esbuild@0.25.10 lib/host.js --minify --charset=utf8 --log-level=error > /tmp/b.min.js
cmp /tmp/a.min.js /tmp/b.min.js && echo "identical"
```

Then `node --check` every entry file.

## 4. Tests

Use the built-in runner: no dependency, no install step. `package.json` script is `node --test`.

Cover the surface that can actually regress:

- config normalization and clamping
- input cleaning, dedupe, size caps, per-session isolation
- prompt rendering and escaping of the wrapper tag
- hidden-context injection, draft-token stripping and fallback
- HTTP routes: success paths, missing header/content-type, cross-site and non-loopback rejection,
  invalid JSON, and the error path

Keep tests in `test/` and out of `files`.

## 5. Documentation

- `README.md`: badges that really exist (license; a CI badge only after a workflow is pushed),
  one-paragraph pitch, features, how it works, requirements, install (GitHub, pinned tag, local),
  verify steps, configuration table, compatibility, limitations, development, license.
- `README.zh.md`: same structure, not a literal machine translation.
- `CHANGELOG.md`: Keep a Changelog, newest first, with the link reference for the tag.
- `LICENSE`: MIT by default; holder is the user, not "contributors".
- `AGENTS.md`: runtime model and invariants for maintainers, plus the release steps. Keep it short.

Every install command must use the real owner; no `<owner>` placeholders survive a release.

## 6. Git history

```sh
git init -b main                      # only when there is no history yet
git config user.name "<login>"
git config user.email "<id>+<login>@users.noreply.github.com"
git add -A
git commit -m "feat: initial release"
git tag -a v1.0.0 -m "v1.0.0"
```

One intentional initial commit beats an invented history. If you amend after tagging, delete and
recreate the tag (`git tag -d v1.0.0`) so the tag points at the pushed commit.

## 7. Authenticate and publish

Preflight:

```sh
gh auth status                                     # who am I, which scopes
curl -sS -m 10 -o /dev/null -w '%{http_code}\n' https://api.github.com/rate_limit
```

- `api.github.com` may work while `github.com` is blocked. Test both; if the web host times out and a
  local proxy is listening (`7897` is the common Clash port), export `HTTPS_PROXY`/`HTTP_PROXY` for
  `gh`, `git`, and `curl`.
- Device flow must not run under a fake PTY. `script -q /dev/null gh auth login --web` hangs on the
  survey cursor-position query. Use piped input with prompts disabled:

  ```sh
  printf 'Y\n\n' | GH_PROMPT_DISABLED=1 HTTPS_PROXY=http://127.0.0.1:7897 \
    gh auth login --hostname github.com --git-protocol https --web
  ```

  It prints a one-time code. Give the user the code and `https://github.com/login/device`, then wait
  for the background job to report `Logged in as <login>`.

Create and push:

```sh
gh repo create <owner>/<repo> --public --source=. --remote=origin --push \
  --description "<one sentence>"
```

If git has no credential helper, `git push` fails with `could not read Username for 'https://github.com'`.
Push with a header instead of writing the token anywhere:

```sh
auth=$(printf 'x-access-token:%s' "$(gh auth token)" | base64 | tr -d '\n')
git -c http.extraheader="Authorization: Basic $auth" push -u origin main --tags
```

Then finish the repo:

```sh
gh repo edit <owner>/<repo> --add-topic dsh-plugin --add-topic deepseek-harness ...
gh release create v1.0.0 --title "v1.0.0" --notes-file /tmp/release-notes.md
```

If the repo keeps a local proxy setting, `git config --local http.proxy ...` is enough; leave the
user's global config alone.

## 8. Verify from the outside

```sh
git -c http.proxy=$HTTPS_PROXY clone --depth 1 --branch v1.0.0 \
  https://github.com/<owner>/<repo>.git /tmp/verify
cd /tmp/verify && npm run check && npm test
gh release view v1.0.0 --repo <owner>/<repo> --json isDraft,isPrerelease,tagName
gh api repos/<owner>/<repo>/git/trees/main?recursive=1 --jq '[.tree[].path] | length'
```

Report the repo URL, release URL, the exact install command, and anything deliberately left out
(for example "no CI: the token has no workflow scope").

## Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `could not read Username for 'https://github.com'` | no git credential helper | `-c http.extraheader=...` push, or ask before running `gh auth setup-git` |
| `refusing to allow an OAuth App to create or update workflow ... without workflow scope` | token lacks `workflow` | `gh auth refresh -s workflow`, or drop `.github/workflows/` from the commit and say so |
| `gh auth login --web` hangs at `(Y/n)` | fake PTY + survey cursor query | `GH_PROMPT_DISABLED=1` + piped newlines, no `script` |
| `dial tcp ...: i/o timeout` on `login/oauth/access_token` | `github.com` blocked, `api.github.com` fine | route through the local proxy |
| Homebrew `Failure writing output to destination` | sandbox denies `~/Library/Caches` or `/opt/homebrew` | ask for wider access, or avoid a system install |
| npm `EACCES ... ~/.npm` | cache owned by another user | `npm_config_cache=/tmp/npm-cache` |
| `Cannot find module .../test` | `node --test test/` treated as a file | run `node --test` (auto-discovery) |
| README badge 404 | badge added before the workflow/release existed | only add badges for artifacts that exist |

## Never

- Publish without an explicit user request; confirm public vs private first.
- Push to an account or org the user did not name.
- Put a token in a remote URL, a file, or a commit — it leaks into config, reflog, and logs.
- Change global git config or install packages without telling the user.
- Rewrite published history; bump the version and cut a new tag instead.
- Claim CI, tests, or verification that did not run.

## 中文速览

1. 先定：账号、仓库名、公开/私有、提交邮箱、许可证署名、要不要 CI。
2. 清理仓库：去掉临时目录和重复文档（如 README 已含安装步骤就删 `INSTALL.md`）。
3. 补元数据：`package.json`（含 `dsh` 字段）、`dsh.plugin.json`、`files`、`npm pack --dry-run` 核对。
4. 统一格式：Prettier；用 esbuild 压缩前后 `cmp` 证明语义未变。
5. 补 `node:test` 测试，`npm test` 跑 `node --test`。
6. 写 README/README.zh/CHANGELOG/LICENSE/AGENTS，安装命令用真实 owner。
7. 建干净历史：单条初始提交 + annotated tag。
8. 授权与发布：`gh auth status` → 设备码登录（代理、`GH_PROMPT_DISABLED=1`）→ `gh repo create` → topics → release。
9. 验证：clone tag → `npm run check` + `npm test`；确认 Release 非草稿。
10. 汇报仓库地址、Release 地址、安装命令，以及故意没做的事。
