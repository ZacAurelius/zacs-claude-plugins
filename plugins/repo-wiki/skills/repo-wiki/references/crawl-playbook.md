# Crawl Playbook

How to read a repository for wiki generation without wasting context. Read this
before any crawl (generate mode) or re-crawl (update mode).

## 1. Size the repo

Count tracked source files: run `git ls-files` and count lines, excluding
vendored/generated content: `node_modules/`, `dist/`, `build/`, `out/`,
`vendor/`, `target/`, lockfiles, minified assets (`*.min.*`), binaries,
generated code, and the wiki's own `outputDir` (it documents the repo — it is
never itself a documentation subject). For non-git repos, use Glob with the
same exclusions.

- **≤ 150 source files → single-pass** (section 5): the main session reads
  directly.
- **> 150 source files → subagent fan-out** (sections 2–4).
- Exception regardless of total count: if summarizing any single area would
  require reading more than ~40 files, delegate that area to a subagent.

## 2. Partition into areas (fan-out path)

- Partition by top-level directory of the source tree (e.g. `src/auth/`,
  `src/api/`, or top-level `packages/*` in monorepos), applying the same
  exclusions as section 1 (vendored/generated content and the wiki's own
  `outputDir`).
- Merge directories with fewer than 5 files into a single "misc" area.
- Aim for 3–10 areas. If a single directory dominates (> 60% of files), split
  it one level deeper.

## 3. Dispatch subagents

Dispatch one read-only exploration subagent (Explore type) per area, in
parallel, at most 6 at a time. Record how many you dispatched — the manifest
run-log field `subagents` needs the total. Use this prompt template,
substituting `<AREA_PATH>` and `<REPO_ROOT>`:

```text
You are summarizing one area of a repository so documentation can be written
from your summary. Do not propose fixes. Do not include full file contents.

Area: <AREA_PATH>
Repository root: <REPO_ROOT>

Read the files in this area (skim large files; skip generated/vendored files,
lockfiles, and binaries) and return ONLY the following structure, under 400
words:

## Area summary: <AREA_PATH>
**Purpose:** 1-2 sentences.
**Key files:**
- <path> — one-line role
**Public interfaces / entry points:**
- <name> (<file>:<line>) — what it does
**Data flow:** bullets describing how data moves through this area, including
inbound/outbound dependencies on other areas.
**Conventions & gotchas:** bullets; non-obvious things only.
**Suggested page:** a page title plus the list of repo-root-relative source
globs the page should cover.
```

## 4. Required area-summary format

Whether produced by a subagent or by the main session, every area summary must
contain: Purpose, Key files, Public interfaces / entry points (with file:line),
Data flow, Conventions & gotchas, Suggested page (title + source globs). The
"Suggested page" globs become the page's `sources` in the manifest.

## 5. Single-pass path (small repos)

Read in priority order, skimming with offset/limit rather than fully reading
big files:

1. Manifest/config files: `package.json`, `pyproject.toml`, `Cargo.toml`,
   `go.mod`, build configs.
2. `README.md` and any existing docs.
3. Entry points (main/index files, CLI entry, server bootstrap).
4. Key modules per directory — enough to describe purpose and interfaces, not
   every line.

Then write the same area-summary structure (section 4) yourself, one summary
per logical area.

## 6. Coverage-gap rule (update mode)

A changed file that matches no page's `sources` is a coverage gap. Resolve it
by comparing the file's path against the directory roots of each page's
`sources` globs (a glob's root is its path up to the first wildcard, e.g.
`src/auth/**` → `src/auth/`):

- If exactly one page's glob root is a prefix of the changed file's path,
  extend that page and widen its `sources` to include the file.
- If several pages qualify, extend the page with the longest matching glob
  root (the most specific area).
- If no glob root matches — or the only shared path is the repo root or a
  container directory that holds multiple areas (`src/`, `packages/`,
  `apps/`, `lib/`) — create a new page using the area-page template, with
  `sources` covering the file's own directory.

## 7. Failure handling

If a subagent fails or returns output missing required sections: retry once
with the same prompt. On second failure: skim the area yourself (README, entry
points, directory listing), write the page from that skim, and add an explicit
"**Coverage gap:** this area was summarized from a shallow scan" note at the
top of the page body.
