# Changelog

## [1.0.0] - 2026-10-28

Initial release.

- Eight-phase publishing procedure: audit, packaging metadata, formatting proof, tests, docs,
  git history, `gh` publish, and fresh-clone verification.
- Pitfalls captured from real runs: `gh auth login` hanging under a fake PTY, `github.com` timing out
  while `api.github.com` stays reachable, `workflow` scope rejection on CI files, missing credential
  helpers, npm cache permissions, and `node --test test/` resolving a directory as a module.

[1.0.0]: https://github.com/stickyfinger7/dsh-plugin-release/releases/tag/v1.0.0
