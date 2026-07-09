# repo-wiki

A Claude Code plugin that writes and maintains documentation for your codebase,
built specifically for agents — and it runs entirely inside your Claude Code
session. No separate LLM provider, no second API key, no extra billing or
rate-limit surface. Inspired by [OpenWiki](https://github.com/langchain-ai/openwiki).

## Why

Agents write better code when they understand the repo they're working in:
where the key logic lives, how files connect, and what patterns the codebase
follows. That context leads to more informed code changes and fewer avoidable
mistakes — but documentation is hard to keep current, and stuffing everything
into one monolithic `CLAUDE.md` bloats every session's context.

A wiki gives humans and agents a structured way to understand a codebase.
repo-wiki generates one, keeps it current from git diffs, and leaves only a
five-line pointer in your instruction file so agents load pages on demand
instead of carrying the whole wiki in context.

## Install

```
/plugin marketplace add ZacAurelius/zacs-claude-plugins
/plugin install repo-wiki@zacs-claude-plugins
```

## Quick start

```
/repo-wiki:generate
```

First run:

1. Detects an existing docs convention (mkdocs, docusaurus, docs index files,
   `KB_*.md` patterns) and asks whether to adopt it — otherwise defaults to
   `docs/wiki/`.
2. Crawls the repo. Small repos are read directly; large ones (over ~150
   source files) are partitioned into areas and crawled by parallel read-only
   subagents (up to 6 at a time).
3. Writes the page set and a manifest, and appends a marker-delimited
   reference block to `CLAUDE.md` (and `AGENTS.md` if present).

Then, whenever the code has moved on:

```
/repo-wiki:update
```

Only pages whose source files changed since the last run are regenerated.
If nothing changed, it says so and stops.

## Commands

| Command | What it does |
|---|---|
| `/repo-wiki:generate` | Full wiki generation for the current repo |
| `/repo-wiki:generate src/api` | Limit generation to a subtree (path scope) |
| `/repo-wiki:generate --output-dir docs/kb` | Override output-directory detection |
| `/repo-wiki:update` | Incremental refresh from git changes since the last run |

Natural language works too — "document this repo", "refresh the wiki" — the
skill triggers without the slash commands.

## What gets generated

| Page | Contents |
|---|---|
| `index.md` | Always. Page table: what each page covers and when to read it |
| `architecture.md` | System overview, components, key flows, tech stack |
| One page per area | Purpose, key files, public interfaces (`file:line`), data flow, gotchas |
| `getting-started.md` | Prerequisites, setup, run, and test commands found in the repo |
| `conventions.md` | Code style and patterns actually observed, each with a citing file |

Every factual claim traces to files the crawl actually read — the skill's hard
rules forbid inventing commands, versions, or behavior. Each page's frontmatter
lists the `sources` globs it documents.

## How agents discover the wiki

generate appends this block to `CLAUDE.md` (and `AGENTS.md` if one exists),
delimited by markers so re-runs replace it instead of duplicating it:

```markdown
<!-- repo-wiki:begin -->
## Repository Wiki
Generated documentation lives in `docs/wiki/`. Read `docs/wiki/index.md`
first, then load only pages relevant to the current task. Do not bulk-load the
wiki. Refresh with /repo-wiki:update. State: docs/wiki/.repo-wiki-manifest.json
<!-- repo-wiki:end -->
```

Every future session — yours or any other coding agent that reads the
instruction file — picks up the latest wiki through that existing reference.
Nothing else enters context until a page is actually needed.

## How incremental updates work

All state lives in one file, `<outputDir>/.repo-wiki-manifest.json`:
last-generated commit SHA, a per-page map of source globs, and a run log
(last 20 runs — the built-in answer to "when was this last built and what
changed").

On `update`:

1. Diff `HEAD` against the manifest's commit. If that commit is no longer an
   ancestor (rebase/squash), fall back to the merge-base; if history is gone
   entirely, confirm and do a full regeneration.
2. Map changed files onto pages via the source globs. The wiki's own output
   and the root-level `CLAUDE.md`/`AGENTS.md` edits are excluded, so the wiki
   never documents itself.
3. Changed files matching no page are coverage gaps: the closest existing
   area page is extended, or a genuinely new area gets a new page.
4. Only stale pages are re-crawled and rewritten. `index.md` is rewritten
   only when the page set changes.

## What repo-wiki never does

- Never calls an external API or needs a key — your session's model does all
  the work.
- Never commits — generated files land in your working tree; you review and
  commit them like any other change.
- Never bulk-loads your repo or the wiki into context.
- Never overwrites documentation it didn't generate without asking.
- Never writes state anywhere except the single manifest inside the wiki dir.

## Keeping it fresh

- **Manual:** run `/repo-wiki:update` after a stretch of work.
- **Scheduled in-session:** pair with a scheduling skill (e.g. `/loop 1h
  /repo-wiki:update`, or a scheduled cloud agent) if you want hands-off
  refreshes.
- **CI:** not shipped in v1. The manifest is versioned and machine-readable,
  so a future workflow can run `claude -p "/repo-wiki:update"` headlessly and
  open a PR with the refreshed docs — note that puts an API key back into CI,
  which is exactly the billing surface this plugin exists to avoid.

## vs. OpenWiki

| | OpenWiki | repo-wiki |
|---|---|---|
| Runtime | Standalone npm CLI (DeepAgents) | Your Claude Code session |
| API key | Own provider key in `~/.openwiki/.env` | None — session model |
| Model choice | Any supported provider/model per run | Whatever the session runs |
| Output | `openwiki/` fixed | Detects existing docs convention, else `docs/wiki/` |
| Incremental update | `--update` from git diffs | `/repo-wiki:update`, manifest + per-page source globs |
| Agent discovery | Reference block in `AGENTS.md`/`CLAUDE.md` | Same idea, marker-delimited and idempotent |
| Observability | LangSmith tracing | Run log in the manifest |
| CI refresh | GitHub Actions / GitLab templates | Out of v1 (manifest is CI-ready) |

## FAQ

**Not a git repo?** Generation still works; the manifest stores `commit: null`
and every update becomes a full regeneration.

**Dirty working tree?** Fine — pages reflect the working tree; the report
notes that the manifest records HEAD.

**Existing `docs/` layout?** Detected on first run; you choose whether the
wiki lives inside it. The choice is recorded and never re-asked. Pre-existing
files are never overwritten without confirmation.

**Monorepo?** Areas partition by `packages/*`-style roots; use a path scope
(`/repo-wiki:generate packages/api`) to document one package at a time.

**Uninstall the output?** Delete the wiki directory and the marker-delimited
block in `CLAUDE.md`/`AGENTS.md`. That's everything — there are no other
dotfiles.
