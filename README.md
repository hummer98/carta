# carta

A Claude Code Plugin that maintains a single source of truth for **development principles (the constitution)** shared across all your projects, and syncs it into each repository's `CLAUDE.md`.

> "carta" is Latin/Italian for "charter" (as in *Magna Carta*).

[日本語版 README](./README.ja.md)

## Why

If you maintain multiple repositories, you probably copy-paste the same set of rules into every `CLAUDE.md`: write docs in one language, don't add gratuitous comments, run tests, don't `--no-verify`, don't write end-of-session summaries, etc. This duplication causes three problems:

1. **Drift** — improving a rule in one repo doesn't propagate to the others
2. **Friction** — adding the rules to a new project means another paste
3. **No verification** — there is no way to confirm every repo has the latest version

carta solves this by keeping the constitution as a single Markdown file (`assets/constitution.md`) and syncing it into each repository's `CLAUDE.md` inside a marker block. Project-specific rules live outside the markers and are never touched.

## How it works

```
assets/constitution.md  ── /carta:apply ──>  <project>/CLAUDE.md
                                              ┌───────────────────┐
                                              │ <!-- carta:begin --> │
                                              │ (constitution body)  │
                                              │ <!-- carta:end -->   │
                                              ├───────────────────┤
                                              │ Project-specific    │
                                              │ rules (preserved)   │
                                              └───────────────────┘
```

- The block between `<!-- carta:begin v0.1.0 sha=abc1234 -->` and `<!-- carta:end -->` is managed by carta
- Everything outside the markers is **never modified**
- Re-applying the same version is a no-op (idempotent)
- The `sha` in the begin marker pins the body content to the version, so manual edits are detectable

## Installation

```
/plugin marketplace add hummer98/carta
/plugin install carta@hummer98-carta
```

## Commands

| Command | What it does |
|---|---|
| `/carta:apply [<path>] [--force]` | Apply the latest constitution to the target `CLAUDE.md`. Shows a diff first and waits for approval. |
| `/carta:show [--meta]` | Print the constitution body bundled with this plugin. `--meta` adds a one-line header with version / last-updated / maintainer. |
| `/carta:diff [<path>]` | Dry run of `/carta:apply` — show the diff but don't write. |

By default, the target is the current repository's `CLAUDE.md` (resolved via `git rev-parse --show-toplevel`). Pass a path to target a different file.

## Example

Before (`./CLAUDE.md`):

```markdown
# my-project

This project uses Bun and ships to Cloudflare Workers. Run `bun test` before pushing.
```

After `/carta:apply`:

```markdown
<!-- carta:begin v0.1.0 sha=abc1234 -->
# 開発憲法

このセクションは [carta](https://github.com/hummer98/carta) によって管理されている。
直接編集せず、リポジトリ固有のルールはマーカー外に記述すること。

## 1. コミュニケーション
- ドキュメント・コメント・ユーザー向けテキストは日本語で書く。
- ...
<!-- carta:end -->

# my-project

This project uses Bun and ships to Cloudflare Workers. Run `bun test` before pushing.
```

The original content is preserved verbatim below the marker block.

## Customizing the constitution

The constitution itself is just `assets/constitution.md` in this repository. To change it for your fleet of projects:

1. Fork carta or maintain your own marketplace entry
2. Edit `assets/constitution.md`
3. Bump the version (`./.claude-plugin/plugin.json` and `marketplace.json`)
4. Tag and release (`/release`)
5. Run `/carta:apply` in each project to pick up the new version

## Versioning

- [SemVer](https://semver.org/): major = breaking change to constitution structure or marker format, minor = new sections, patch = wording fixes.
- The applied version is recorded in the begin marker, so each project knows whether it is up to date.

## Status

v0.1.0 — MVP. Includes `/carta:apply`, `/carta:show`, `/carta:diff`, and the marker-based sync logic.

Out of scope for v0.1: multi-`CLAUDE.md` mono-repo support, per-project override layers, CI checks. See `docs/seed.md` §10 for the full scope policy.

## License

[MIT](LICENSE)
