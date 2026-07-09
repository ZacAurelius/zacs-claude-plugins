# repo-wiki

Generate a documentation wiki for the current repository and keep it fresh with
diff-based incremental updates — entirely inside your Claude Code session. No
separate LLM API key or billing surface. Inspired by OpenWiki.

## Commands

- `/repo-wiki:generate [path-scope] [--output-dir <dir>]` — full wiki generation
- `/repo-wiki:update` — regenerate only pages whose sources changed since the last run

Natural language works too: "document this repo", "refresh the wiki".

## How it works

- First run detects an existing docs convention (or defaults to `docs/wiki/`),
  crawls the repo (parallel subagents on large repos), and writes a page set
  with an `index.md`.
- All state lives in `<outputDir>/.repo-wiki-manifest.json`: last-generated
  commit SHA, per-page source globs, and a run log.
- A short marker-delimited reference block is appended to `CLAUDE.md` (and
  `AGENTS.md` if present) so future agent sessions discover the wiki without
  loading it into context.
- `update` diffs `HEAD` against the last-generated commit (or a merge-base
  fallback when history was rewritten) and regenerates only stale pages.

## State

Everything lives in the manifest; no other dotfiles are written to your repo.
To fully remove the output, delete the wiki directory and the marker-delimited
block in `CLAUDE.md`/`AGENTS.md`.
