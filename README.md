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

- The block between `<!-- carta:begin v0.2.0 sha=abc1234 -->` and `<!-- carta:end -->` is managed by carta
- Everything outside the markers is **never modified**
- Re-applying the same version is a no-op (idempotent)
- The `sha` in the begin marker pins the body content to the version, so manual edits are detectable

## Profiles

Bundle-only optional add-ons that get concatenated to the default constitution at apply time.

- Available profiles live under `assets/profiles/*.md`. v0.2.0 ships `typescript` and `flutter` as minimal placeholders.
- Pick one or more with `--profile`:

  ```
  /carta:apply --profile typescript
  /carta:apply --profile typescript,flutter
  /carta:apply --profile=                # explicitly drop all profiles (recommended form)
  ```

- Running `/carta:apply` without `--profile` inherits the profiles already recorded in the existing marker.
- The marker records the chosen set: `<!-- carta:begin v0.2.0 sha=abc1234 profiles=flutter,typescript -->`
- `sha` is computed over the **concatenated** body, so changing the profile set is detected as a real update.
- Markers with no `profiles=` attribute (v0.1.0 style) are treated as "no profiles" and remain compatible — the resulting `sha` is byte-for-byte identical to v0.1.0.
- Note: `/carta:show --profile=` is **not** the way to view the default — it errors with "name required". Use `/carta:show` (no args) instead.

## Installation

```
/plugin marketplace add hummer98/carta
/plugin install carta@hummer98-carta
```

## Commands

| Command | What it does |
|---|---|
| `/carta:apply [<path>] [--profile <names>] [--force]` | Apply the latest constitution (+ chosen profiles) to the target `CLAUDE.md`. Shows a diff first and waits for approval. |
| `/carta:show [--meta] [--profile <name>]` | Print the constitution body (or a single profile body) bundled with this plugin. `--meta` adds a one-line header. |
| `/carta:list` | List bundled profiles (name + one-line summary). Display only. |
| `/carta:diff [<path>] [--profile <names>]` | Dry run of `/carta:apply` — show the diff but don't write. |
| `/carta:help` | Print the command list, common usage examples, and marker spec summary. Display only. |

By default, the target is the current repository's `CLAUDE.md` (resolved via `git rev-parse --show-toplevel`). Pass a path to target a different file.

## Example

Before (`./CLAUDE.md`):

```markdown
# my-project

This project uses Bun and ships to Cloudflare Workers. Run `bun test` before pushing.
```

After `/carta:apply`:

```markdown
<!-- carta:begin v0.2.0 sha=abc1234 -->
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

Add a new profile by placing a Markdown file (with a single trailing newline) under `assets/profiles/` and bumping the plugin version. The first line should be `<!-- carta-profile: <name> -->` — this comment is a frozen constant; do not change it after release (changing it breaks every existing `sha`).

## Versioning

- [SemVer](https://semver.org/): major = breaking change to constitution structure or marker format, minor = new sections, patch = wording fixes.
- The applied version is recorded in the begin marker, so each project knows whether it is up to date.

## Status

v0.2.0 — Profiles. Default constitution + opt-in language/framework profiles (typescript, flutter shipped as minimal placeholders). v0.1.0 markers remain fully compatible; the `sha` is preserved when re-applied with no profiles.

Out of scope for v0.x: multi-`CLAUDE.md` mono-repo support, per-project override layers, CI checks. See `docs/seed.md` §10 for the full scope policy.

## License

[MIT](LICENSE)
