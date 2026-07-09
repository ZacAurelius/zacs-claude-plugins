# repo-wiki Plugin — Design

**Date:** 2026-07-08
**Status:** Approved (brainstorming complete)
**Deliverable:** New plugin `repo-wiki` in this marketplace (`plugins/repo-wiki/`), registered in `.claude-plugin/marketplace.json`.

## Purpose

Replicate the useful behavior of [OpenWiki](https://github.com/langchain-ai/openwiki) — automated repository documentation generation and incremental maintenance — as a Claude Code plugin that runs entirely inside the invoking Claude Code session. No separate LLM provider, API key, or billing/rate-limit surface.

The plugin is generic and marketplace-distributed: installable into any target repo, not tied to any one project's structure.

## Architecture comparison (carried forward, settled)

DeepAgents (OpenWiki's framework) is functionally a planning/todo loop + subagent dispatch + filesystem tools. Claude Code natively provides equivalents: TodoWrite (planning), Agent tool (subagent dispatch), Read/Grep/Glob/Write (filesystem). Nothing architectural is lost by going CC-native.

What OpenWiki has that this plugin deliberately handles differently:

| OpenWiki capability | This plugin's decision |
|---|---|
| Per-run model/provider choice | Dropped by design. Runs on the invoking session's model. This is the accepted tradeoff motivating the whole project. |
| LangSmith tracing | Replaced by a lightweight run log inside the manifest (timestamp, mode, commit, pages touched, subagent count). The CC transcript already covers token-level tracing. |
| Tuned crawl/draft prompts | Designed here: adaptive crawl strategy + structured subagent summary format (see Generate flow and `crawl-playbook.md`). |
| CI auto-refresh templates | Out of v1. Refresh is manual (`/repo-wiki:update`) or session-scheduled via loop/schedule skills. The manifest schema is versioned and machine-readable so CI can consume it later. A copy-paste headless snippet (`claude -p "/repo-wiki:update"`) is documented as future work — note it reintroduces an API-key surface in CI. |
| `--update` diff-based refresh | Manifest stores last-generated commit SHA + per-page source map; update regenerates only pages whose sources changed. |
| Generic portability | Detects and respects existing docs conventions in the target repo instead of imposing one layout. |

## Decisions (from brainstorming Q&A)

1. **Output location:** Detect existing docs convention on first run (docs index files, `KB_*.md` patterns, mkdocs/docusaurus configs). If found, ask user: adopt it or use default. Otherwise default to `docs/wiki/`. Choice recorded in the manifest; never re-asked.
2. **Invocation surface:** Two thin slash commands — `/repo-wiki:generate`, `/repo-wiki:update` — plus one skill carrying all methodology. Natural language ("document this repo") triggers the skill directly.
3. **CI:** None in v1 (see table above).
4. **Crawl architecture:** Adaptive. Small repos: main session reads directly. Large repos: parallel subagent fan-out per area, main session drafts from returned summaries.
5. **Staleness:** Manifest with commit SHA + per-page `sources` globs; update maps `git diff` output to stale pages.
6. **Observability:** Run log entries in the manifest, capped at last 20.
7. **Name:** `repo-wiki`.
8. **Anatomy:** One skill + references (progressive disclosure); all per-repo state in one manifest file inside the wiki dir; zero extra dotfiles in the target repo.

## Component structure

```
plugins/repo-wiki/
  .claude-plugin/plugin.json        # name "repo-wiki", version 0.1.0, author, license MIT, keywords
  commands/
    generate.md                     # thin: invoke repo-wiki skill, mode=generate; optional args: path scope, output-dir override
    update.md                       # thin: invoke repo-wiki skill, mode=update
  skills/repo-wiki/
    SKILL.md                        # trigger description, core workflow, mode dispatch; points at references
    references/
      crawl-playbook.md             # repo sizing thresholds, subagent dispatch prompts, required summary format
      page-templates.md             # page structures (index, architecture, area, getting-started, conventions) + sources: frontmatter spec
      manifest-schema.md            # manifest JSON schema, examples, versioning rules
```

- Commands contain zero methodology; the skill is the single source of truth.
- SKILL.md stays small; references load on demand (progressive disclosure).
- Registered in `.claude-plugin/marketplace.json` per this repo's convention (README documents: create `plugins/<name>/.claude-plugin/plugin.json`, add components, register).

## Generate flow (`/repo-wiki:generate`)

1. **Preflight:** Verify target is a git repo. If not: warn, proceed anyway, manifest gets `"commit": null`, and all future updates fall back to full regeneration.
2. **Existing-wiki check:** Look for an existing manifest (glob `**/.repo-wiki-manifest.json` + marker block in CLAUDE.md/AGENTS.md). If found, suggest `/repo-wiki:update` and require explicit confirmation before a full overwrite.
3. **Output-dir resolution (first run only):** Scan for existing docs conventions. Found → ask adopt vs `docs/wiki/`. Not found → `docs/wiki/`. Record in manifest.
4. **Repo sizing:** `git ls-files` file count + rough LOC. At or under threshold (~150 source files, guidance detailed in `crawl-playbook.md`) → single-pass: main session reads entry points, configs, key modules directly. Over threshold → partition repo by top-level area and dispatch parallel Explore-type subagents; each returns a structured summary: purpose, key files, public interfaces, data flow, gotchas, `file:line` references. Main session drafts from summaries and never bulk-reads large directories itself.
5. **Draft:** Plan page set from `page-templates.md`: `index.md` (always), architecture overview, one page per major area, getting-started/dev-workflow, conventions. Write pages. Every page's frontmatter lists the `sources:` paths/globs it covers.
6. **Persist:** Write manifest with `commit` = HEAD SHA. Append/replace the reference block in CLAUDE.md (and AGENTS.md if present) between markers. Append a run-log entry. Report a summary of pages written.

## Update flow (`/repo-wiki:update`)

1. **Locate manifest** (same discovery as above). None found → tell the user to run `/repo-wiki:generate`.
2. **Diff:** `git diff --name-status <manifest.commit>..HEAD`. Empty diff → no-op; say so and stop.
3. **Stale mapping:** Map each changed file to pages via per-page `sources` globs. A changed file matching no page is a coverage gap. Rule: compare the file's path against the directory roots of each page's `sources` globs; extend the page with the longest matching glob root, or create a new page when no root matches or the file only shares the repo root / a container directory (e.g. `src/`, `packages/`). The full rule lives in `crawl-playbook.md`.
4. **Regenerate stale pages only.** Re-crawl only affected areas (subagents if the area is large). Deleted sources → prune or edit affected pages. If the page set changed, rewrite `index.md`.
5. **Persist:** Update manifest `commit`, source maps, run log. Refresh the reference block via markers (idempotent).
6. **Edge — unreachable SHA** (rebase/squash rewrote history): fall back to `git merge-base` with HEAD; if that fails too, confirm with the user, then full regeneration.

## Reference block (context frugality)

The point of OpenWiki's reference block is that future agent sessions discover the wiki without loading it all into context. The CC-native equivalent, appended to CLAUDE.md (and AGENTS.md if present), delimited by HTML comment markers so re-runs replace rather than duplicate:

```markdown
<!-- repo-wiki:begin -->
## Repository Wiki
Generated documentation lives in `docs/wiki/`. Read `docs/wiki/index.md` first,
then load only pages relevant to the current task. Do not bulk-load the wiki.
Refresh with /repo-wiki:update. Build state: docs/wiki/.repo-wiki-manifest.json
<!-- repo-wiki:end -->
```

(Path reflects the resolved output dir.) Kept to ~5 lines; instructs lazy loading explicitly.

## Manifest schema (`<outputDir>/.repo-wiki-manifest.json`)

All per-repo state lives here — output-dir choice, convention decision, source maps, build SHA, run log. Discovery is via the CLAUDE.md marker block plus glob; no extra dotfiles in the target repo.

```json
{
  "version": 1,
  "generatedAt": "2026-07-08T12:00:00Z",
  "commit": "abc1234",
  "outputDir": "docs/wiki",
  "convention": "default",
  "pages": [
    { "file": "index.md", "title": "Wiki Index", "sources": ["*"] },
    { "file": "auth.md", "title": "Authentication", "sources": ["src/auth/**"] }
  ],
  "runs": [
    { "at": "2026-07-08T12:00:00Z", "mode": "generate", "commit": "abc1234", "pagesTouched": ["*"], "subagents": 0 }
  ]
}
```

- `version` is a schema version for forward compatibility (future CI consumers).
- `convention` is either `"default"` or `"detected:<short description>"` when an existing docs layout was adopted.
- `runs` capped at the 20 most recent entries.

## Error handling

- **Not a git repo:** warn, generate anyway, `commit: null`, updates always full-regen.
- **Dirty working tree at generate:** proceed; record HEAD and note in the final report that uncommitted files were included.
- **Corrupt or invalid manifest:** offer full regeneration.
- **Pre-existing marker block from an older run:** replace between markers; never duplicate.
- **Pre-existing docs at the target dir not created by this plugin:** never overwrite without explicit confirmation.
- **Subagent failure on an area:** retry once; on second failure, draft the page from a main-session skim and note the coverage gap in the page.

## Testing / verification

- Structure validation via the `plugin-dev:plugin-validator` agent.
- Skill quality review via the `plugin-dev:skill-reviewer` agent.
- Manual end-to-end: install this marketplace locally, run `/repo-wiki:generate` on a small fixture repo and on this repo; make a commit touching one area; run `/repo-wiki:update`; verify only mapped pages regenerated, manifest updated, and marker block idempotent across runs.

## Out of scope for v1

- CI workflow templates (manifest is designed to be CI-consumable later).
- Per-run model choice (dropped by design — session model only).
- LangSmith-style tracing (replaced by the manifest run log).

## Next step

Implementation plan via the writing-plans skill, then scaffold with this repo's plugin-dev tooling (`plugin-dev:create-plugin` / `plugin-dev:plugin-structure` conventions).
