# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is `claude-plugins-official`, the curated marketplace directory of plugins for Claude Code. It is almost entirely *metadata and pointers*, not application code:

- `.claude-plugin/marketplace.json` — the single source of truth: a JSON list of every plugin entry (name, description, author, category, `source`, homepage, optional `skills`/`tags`/`keywords`/`renames`). Most plugins live in external repos and are referenced via `source: {source: "url"|"git-subdir", url, ref, sha}` (pinned to a commit SHA). A minority ("internal" plugins) are vendored directly in this repo under `source: "./plugins/<name>"`.
- `plugins/` — internal plugins maintained by Anthropic in-tree (e.g. `example-plugin`, `agent-sdk-dev`, `claude-code-setup`, `code-review`, `commit-commands`, `clangd-lsp`/`csharp-lsp` LSP plugins). Each follows the standard plugin layout: `.claude-plugin/plugin.json`, optional `commands/`, `agents/`, `skills/`, `README.md`.
- `external_plugins/` — a small set of third-party plugins vendored directly in-repo (Asana, GitHub, GitLab, Linear, Playwright, Terraform, etc.), same layout as `plugins/`.
- `.github/workflows/` and `.github/scripts/` — CI that validates and maintains the marketplace (see below).

There is no build, package, or app to run here — "development" means editing `marketplace.json` entries and/or the files inside `plugins/<name>` or `external_plugins/<name>`.

## Marketplace entry conventions

- `name` is an **immutable slug** once published — installs (`{plugin-name}@claude-plugins-official`) depend on it. Never rename a `name`; instead set/change `displayName`, or if a rename is unavoidable, add an `old-name: new-name` mapping to the top-level `renames` map so existing installs auto-migrate.
- External-repo sources must be pinned with a commit `sha` (this is a hard-enforced invariant in CI, referred to as I5).
- `source: "git-subdir"` pulls a `path` within a repo at a given `ref`/`sha`; plain `source: "url"` clones the whole repo.
- Skill-bundle plugins (repos that ship `SKILL.md` files without a `plugin.json` manifest) use `strict: false` plus an explicit `skills` array of paths relative to `source.path`; each is registered as `<plugin-name>:<skill-name>`.
- Standard plugin directory layout (see `plugins/example-plugin` for a reference implementation):
  ```
  plugin-name/
  ├── .claude-plugin/plugin.json   # required metadata
  ├── .mcp.json                    # optional MCP server config
  ├── commands/                    # optional slash commands
  ├── agents/                      # optional agent definitions
  ├── skills/                      # optional skill definitions
  └── README.md
  ```

## CI / validation workflows

These run automatically on PRs touching plugin/marketplace paths and are the closest thing this repo has to "tests" — reproduce failures by reading the referenced scripts/actions rather than trying to run a local build:

- **`validate-plugins.yml`** (`validate`, required status check) — runs `anthropics/claude-plugins-community/.github/actions/validate-plugins` against `.claude-plugin/marketplace.json`. Enforces marketplace schema invariants; SHA-pinning (I5) is a hard error, invariants I1/I3/I8/I11 are currently warn-only.
- **`validate-frontmatter.yml`** — runs `.github/scripts/validate-frontmatter.ts` (via `bun`) against changed `agents/*.md`, `skills/*/SKILL.md`, and `commands/*.md` files in the diff.
- **`validate-licenses.yml`** — checks that vendored plugins carry the expected license file/content.
- **`scan-plugins.yml`** (`scan`, required status check) — Claude-driven policy scan of changed *external* marketplace entries, per `.github/policy/prompt.md` / `.github/policy/schema.json`; caches verdicts per `(plugin, sha)` keyed on the policy hash.
- **`check-mcp-urls.yml`** — validates MCP server URLs referenced by plugins.
- **`bump-plugin-shas.yml`** / **`discover_bumps.py`** — nightly automation that discovers new upstream commits for `git-subdir`/`url` sources and opens PRs bumping their pinned `sha`s (tracked in `.github/bump-tracking.json`).
- **`revert-failed-bumps.yml`**, **`external-pr-scope-guard.yml`**, **`close-external-prs.yml`** — repo hygiene automation (revert bumps that fail validation, restrict external PRs to their own plugin's paths, close stale external PRs).

Because `validate` and `scan` are required status checks that only trigger on specific path globs, several workflows deliberately widen their `paths:` triggers (workflow files, policy files, plugin READMEs/assets) so that PRs touching only those files aren't stuck "Expected — Waiting for status" forever. Keep this in mind if you add new top-level files that a PR might touch in isolation.

## Making changes

- **Adding/updating a plugin entry**: edit `.claude-plugin/marketplace.json`. For an externally-hosted plugin, pin `source.sha` to a real commit on the given `ref`. For an in-tree plugin, add/update files under `plugins/<name>/` (or `external_plugins/<name>/`) following the standard layout above, then add the marketplace entry with `source: "./plugins/<name>"`.
- **Renaming a plugin**: never change an existing `name`; use `displayName` or the `renames` map.
- Submissions from third parties go through the [plugin directory submission form](https://clau.de/plugin-directory-submission), not direct edits to internal plugins.
