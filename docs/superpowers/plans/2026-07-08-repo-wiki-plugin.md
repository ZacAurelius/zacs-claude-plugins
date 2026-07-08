# repo-wiki Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `repo-wiki` plugin (skill + two commands) that generates and incrementally maintains a documentation wiki for any repo entirely inside a Claude Code session, and register it in this marketplace.

**Architecture:** One skill (`skills/repo-wiki/SKILL.md`) holds all methodology, with three on-demand references (manifest schema, crawl playbook, page templates). Two thin commands (`/repo-wiki:generate`, `/repo-wiki:update`) delegate to the skill. All per-target-repo state lives in a single `.repo-wiki-manifest.json` inside the generated wiki directory. Spec: `docs/superpowers/specs/2026-07-08-repo-wiki-plugin-design.md`.

**Tech Stack:** Claude Code plugin conventions (plugin.json manifest, commands/*.md, skills/<name>/SKILL.md + references/). Markdown + JSON only — no executable code ships in the plugin. Node.js one-liners for verification (Python is NOT installed on this machine).

## Global Constraints

- No separate LLM API key anywhere; everything runs in the invoking Claude Code session.
- Plugin manifest MUST be at `plugins/repo-wiki/.claude-plugin/plugin.json`; component dirs (`commands/`, `skills/`) at plugin root, NOT inside `.claude-plugin/`.
- Kebab-case for all file and directory names.
- Skill frontmatter: `name: repo-wiki` (must match skill directory name), `description` ≤ 1024 characters, written in third person with trigger phrases.
- SKILL.md must stay under 500 lines.
- The state file in target repos is exactly `.repo-wiki-manifest.json` (inside the resolved output dir). No other dotfiles may be written to target repos.
- Reference-block markers in target repos are exactly `<!-- repo-wiki:begin -->` and `<!-- repo-wiki:end -->`.
- Default output dir in target repos: `docs/wiki/`.
- All JSON files must parse (verify with Node one-liners via the Bash tool).
- Verification commands in this plan are written for the **Bash tool** (Git Bash), not PowerShell — backticks inside single quotes are literal in bash.
- Conventional-commit messages; commit at the end of every task.
- Register the plugin in `.claude-plugin/marketplace.json` (repo convention per `README.md`).

---

### Task 1: Scaffold plugin + marketplace registration

**Files:**
- Create: `plugins/repo-wiki/.claude-plugin/plugin.json`
- Create: `plugins/repo-wiki/README.md`
- Modify: `.claude-plugin/marketplace.json`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: plugin root `plugins/repo-wiki/` and plugin name `repo-wiki` — every later task creates files under this root; commands will be namespaced `/repo-wiki:generate` and `/repo-wiki:update`.

- [ ] **Step 1: Create `plugins/repo-wiki/.claude-plugin/plugin.json`** with exactly:

```json
{
  "name": "repo-wiki",
  "version": "0.1.0",
  "description": "Generate and incrementally maintain a documentation wiki for any repository, entirely inside your Claude Code session (OpenWiki-style, no extra API key)",
  "author": {
    "name": "Zac Aurelius",
    "email": "zacaurelius@gmail.com"
  },
  "license": "MIT",
  "keywords": ["documentation", "wiki", "docs", "codebase", "openwiki"]
}
```

- [ ] **Step 2: Create `plugins/repo-wiki/README.md`** with exactly:

```markdown
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
- `update` diffs `HEAD` against the manifest commit and regenerates only stale
  pages.

## State

Everything lives in the manifest; no other dotfiles are written to your repo.
To fully remove the output, delete the wiki directory and the marker-delimited
block in `CLAUDE.md`/`AGENTS.md`.
```

- [ ] **Step 3: Replace `.claude-plugin/marketplace.json`** with exactly:

```json
{
  "name": "zacs-claude-plugins",
  "owner": {
    "name": "Zac Aurelius",
    "email": "zacaurelius@gmail.com"
  },
  "plugins": [
    {
      "name": "example-plugin",
      "source": "./plugins/example-plugin",
      "description": "Starter example plugin demonstrating a basic command"
    },
    {
      "name": "repo-wiki",
      "source": "./plugins/repo-wiki",
      "description": "Generate and incrementally maintain a repo documentation wiki inside your Claude Code session (OpenWiki-style, no extra API key)"
    }
  ]
}
```

- [ ] **Step 4: Verify both JSON files parse**

Run (Bash tool):

```bash
node -e 'JSON.parse(require("fs").readFileSync("plugins/repo-wiki/.claude-plugin/plugin.json","utf8")); JSON.parse(require("fs").readFileSync(".claude-plugin/marketplace.json","utf8")); console.log("OK")'
```

Expected output: `OK`

- [ ] **Step 5: Commit**

```bash
git add plugins/repo-wiki/.claude-plugin/plugin.json plugins/repo-wiki/README.md .claude-plugin/marketplace.json
git commit -m "feat(repo-wiki): scaffold plugin manifest and marketplace registration"
```

---

### Task 2: Manifest schema reference

**Files:**
- Create: `plugins/repo-wiki/skills/repo-wiki/references/manifest-schema.md`

**Interfaces:**
- Consumes: plugin root from Task 1.
- Produces: the manifest contract every other file relies on — filename `.repo-wiki-manifest.json`; fields `version`, `generatedAt`, `commit`, `outputDir`, `convention`, `pages[]` (`file`, `title`, `sources`), `runs[]` (`at`, `mode`, `commit`, `pagesTouched`, `subagents`); `runs` cap 20; `"*"` sources semantics for `index.md`.

- [ ] **Step 1: Create `plugins/repo-wiki/skills/repo-wiki/references/manifest-schema.md`** with exactly:

````markdown
# Manifest Schema — `.repo-wiki-manifest.json`

Single source of truth for all repo-wiki state in a target repository. Lives at
`<outputDir>/.repo-wiki-manifest.json`. Machine-readable so external tooling
(e.g. future CI) can consume it. Read this file before writing or modifying any
manifest.

## Fields

| Field | Type | Meaning |
|---|---|---|
| `version` | integer | Schema version. Currently `1`. Bump only on breaking schema change. |
| `generatedAt` | string | ISO-8601 UTC timestamp of the last successful run. |
| `commit` | string \| null | Full HEAD SHA at the last successful run. `null` if the target is not a git repo. |
| `outputDir` | string | Wiki directory, relative to repo root, forward slashes, no trailing slash (e.g. `"docs/wiki"`). |
| `convention` | string | `"default"` when using `docs/wiki/`, or `"detected:<short description>"` when an existing docs layout was adopted. |
| `pages` | array | One entry per wiki page. |
| `pages[].file` | string | Filename relative to `outputDir` (e.g. `"auth.md"`). |
| `pages[].title` | string | Page title (matches the page's frontmatter `title`). |
| `pages[].sources` | array of strings | Repo-root-relative glob patterns (minimatch style) this page documents. Used by update mode to detect staleness. |
| `runs` | array | Run log, newest last, capped at the 20 most recent entries (drop oldest first). |
| `runs[].at` | string | ISO-8601 UTC timestamp of the run. |
| `runs[].mode` | string | `"generate"` or `"update"`. |
| `runs[].commit` | string \| null | HEAD SHA the run was built against. |
| `runs[].pagesTouched` | array of strings | Page filenames written this run, or `["*"]` for a full generation. |
| `runs[].subagents` | integer | Number of subagents dispatched during the run (`0` for single-pass). |

## Special semantics: `index.md`

`index.md` always has `"sources": ["*"]`. This does NOT mean "stale on any file
change". It marks a structural page: regenerate `index.md` only when the page
set changes (a page added, removed, or retitled). Never mark `index.md` stale
from a source-glob match.

## Rules

- JSON, UTF-8, 2-space indent, LF line endings.
- The manifest is authoritative. Page frontmatter `sources` is an informational
  copy for humans reading a page standalone; on conflict, the manifest wins.
- Every page written must have a manifest entry, and vice versa.
- Never store absolute paths.
- Uncommitted working-tree changes are not tracked; runs record HEAD only.

## Example

```json
{
  "version": 1,
  "generatedAt": "2026-07-08T12:00:00Z",
  "commit": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
  "outputDir": "docs/wiki",
  "convention": "default",
  "pages": [
    { "file": "index.md", "title": "Wiki Index", "sources": ["*"] },
    { "file": "architecture.md", "title": "Architecture Overview", "sources": ["src/**", "package.json"] },
    { "file": "auth.md", "title": "Authentication", "sources": ["src/auth/**"] },
    { "file": "getting-started.md", "title": "Getting Started", "sources": ["package.json", "README.md", ".env.example"] }
  ],
  "runs": [
    { "at": "2026-07-08T12:00:00Z", "mode": "generate", "commit": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2", "pagesTouched": ["*"], "subagents": 4 }
  ]
}
```
````

- [ ] **Step 2: Verify the embedded example JSON parses**

Run (Bash tool):

```bash
node -e '
const s = require("fs").readFileSync("plugins/repo-wiki/skills/repo-wiki/references/manifest-schema.md","utf8");
const m = s.match(/```json\n([\s\S]*?)```/);
if (!m) throw new Error("no json fence found");
const j = JSON.parse(m[1]);
if (j.version !== 1) throw new Error("version must be 1");
if (!Array.isArray(j.pages) || !Array.isArray(j.runs)) throw new Error("pages/runs must be arrays");
console.log("OK");
'
```

Expected output: `OK`

- [ ] **Step 3: Commit**

```bash
git add plugins/repo-wiki/skills/repo-wiki/references/manifest-schema.md
git commit -m "docs(repo-wiki): add manifest schema reference"
```

---

### Task 3: Crawl playbook reference

**Files:**
- Create: `plugins/repo-wiki/skills/repo-wiki/references/crawl-playbook.md`

**Interfaces:**
- Consumes: manifest field names from Task 2 (`pagesTouched`, `subagents`, `pages[].sources`).
- Produces: sizing threshold (150 source files), area-summary format, subagent prompt template, coverage-gap rule — SKILL.md (Task 5) refers to this file as `references/crawl-playbook.md`.

- [ ] **Step 1: Create `plugins/repo-wiki/skills/repo-wiki/references/crawl-playbook.md`** with exactly:

````markdown
# Crawl Playbook

How to read a repository for wiki generation without wasting context. Read this
before any crawl (generate mode) or re-crawl (update mode).

## 1. Size the repo

Count tracked source files: run `git ls-files` and count lines, excluding
vendored/generated content: `node_modules/`, `dist/`, `build/`, `out/`,
`vendor/`, `target/`, lockfiles, minified assets (`*.min.*`), binaries, and
generated code. For non-git repos, use Glob with the same exclusions.

- **≤ 150 source files → single-pass** (section 5): the main session reads
  directly.
- **> 150 source files → subagent fan-out** (sections 2–4).
- Exception regardless of total count: if summarizing any single area would
  require reading more than ~40 files, delegate that area to a subagent.

## 2. Partition into areas (fan-out path)

- Partition by top-level directory of the source tree (e.g. `src/auth/`,
  `src/api/`, or top-level `packages/*` in monorepos).
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

A changed file that matches no page's `sources` is a coverage gap:

- If the file shares a top-level directory with an existing page's sources →
  extend that page and widen its `sources` globs to include the file.
- Otherwise → create a new page using the area-page template, with `sources`
  covering the file's directory.

## 7. Failure handling

If a subagent fails or returns output missing required sections: retry once
with the same prompt. On second failure: skim the area yourself (README, entry
points, directory listing), write the page from that skim, and add an explicit
"**Coverage gap:** this area was summarized from a shallow scan" note at the
top of the page body.
````

- [ ] **Step 2: Verify required sections present**

Run (Bash tool):

```bash
node -e '
const s = require("fs").readFileSync("plugins/repo-wiki/skills/repo-wiki/references/crawl-playbook.md","utf8");
for (const h of ["Size the repo","Partition into areas","Dispatch subagents","Required area-summary format","Single-pass path","Coverage-gap rule","Failure handling"]) {
  if (!s.includes(h)) throw new Error("missing section: " + h);
}
if (!s.includes("150 source files")) throw new Error("missing threshold");
console.log("OK");
'
```

Expected output: `OK`

- [ ] **Step 3: Commit**

```bash
git add plugins/repo-wiki/skills/repo-wiki/references/crawl-playbook.md
git commit -m "docs(repo-wiki): add crawl playbook reference"
```

---

### Task 4: Page templates reference

**Files:**
- Create: `plugins/repo-wiki/skills/repo-wiki/references/page-templates.md`

**Interfaces:**
- Consumes: area-summary section names from Task 3; manifest `pages[].title`/`sources` from Task 2.
- Produces: page frontmatter spec (`title`, `sources`, `lastGenerated`) and the five page templates — SKILL.md (Task 5) refers to this file as `references/page-templates.md`.

- [ ] **Step 1: Create `plugins/repo-wiki/skills/repo-wiki/references/page-templates.md`** with exactly:

````markdown
# Page Templates

Structures for every wiki page. Read this before writing any page. All pages
are plain GitHub-flavored markdown with relative links; no HTML.

## Frontmatter (every page)

```yaml
---
title: <Page Title>
sources:
  - <repo-root-relative glob>
lastGenerated: <YYYY-MM-DD>
---
```

`sources` here is an informational copy for humans reading the page standalone;
the manifest is authoritative (see `manifest-schema.md`). Keep the two in sync
whenever you write a page.

## Page set

Always write `index.md`. Write the others when the repo has substance for them:

| Page | When |
|---|---|
| `index.md` | Always. |
| `architecture.md` | Repo has 2+ interacting areas. |
| One page per area (`<area>.md`) | One per area summary from the crawl. |
| `getting-started.md` | Repo has a build/run/test workflow. |
| `conventions.md` | Repo shows consistent non-obvious conventions. |

Name area pages after the area, kebab-case (e.g. `auth.md`, `api-server.md`).

## `index.md` template

```markdown
---
title: Wiki Index
sources:
  - "*"
lastGenerated: <date>
---

# <Repo Name> Wiki

<1-2 sentence repo description.>

## Pages

| Page | Covers | Read when |
|---|---|---|
| [Architecture](architecture.md) | System overview, how areas connect | You need the big picture |
| [<Area>](<area>.md) | <one-liner> | Working in `<area path>` |
| [Getting Started](getting-started.md) | Setup, run, test | First time in this repo |
| [Conventions](conventions.md) | Code style, patterns | Before writing code |

Generated by the repo-wiki plugin. Refresh with `/repo-wiki:update`.
```

## `architecture.md` template

```markdown
---
title: Architecture Overview
sources:
  - <globs covering all major areas>
lastGenerated: <date>
---

# Architecture Overview

## System summary
<2-4 sentences: what the system does and its major moving parts.>

## Components
<One short subsection per area: name, responsibility, key entry points with
file:line references.>

## Key flows
<Bulleted walkthroughs of 1-3 primary flows (e.g. request lifecycle, build
pipeline), naming files at each step.>

## Tech stack
<Languages, frameworks, notable dependencies — one line each.>
```

## Area page template (one per area)

```markdown
---
title: <Area Title>
sources:
  - <area globs>
lastGenerated: <date>
---

# <Area Title>

## Purpose
<1-2 sentences.>

## Key files
| File | Role |
|---|---|
| `<path>` | <one-liner> |

## Public interfaces
<Exact names with file:line — functions, classes, routes, CLI entry points
other areas or users call.>

## Data flow
<Bullets: inputs, transformations, outputs, dependencies on other areas, with
links to those areas' pages.>

## Conventions & gotchas
<Bullets; non-obvious things only. Omit the section if empty.>
```

## `getting-started.md` template

```markdown
---
title: Getting Started
sources:
  - <build/config file globs>
lastGenerated: <date>
---

# Getting Started

## Prerequisites
<Tools + versions, from config files — do not invent.>

## Setup
<Exact install/bootstrap commands found in the repo.>

## Run
<Exact run/dev commands.>

## Test
<Exact test commands.>
```

## `conventions.md` template

```markdown
---
title: Conventions
sources:
  - <globs for lint configs and representative source>
lastGenerated: <date>
---

# Conventions

<Bullets grouped by topic: code style, naming, file organization, testing
patterns, error handling, commit style. Only conventions actually observed in
the code or enforced by configs — cite a file for each.>
```

## Writing rules

- Every factual claim must come from the crawl (files read or area summaries);
  never invent commands, versions, or behavior.
- Cite `file:line` for interfaces and entry points.
- Keep each page under ~200 lines; split an oversized area into two pages
  instead.
- Cross-link related pages with relative links.
````

- [ ] **Step 2: Verify all five templates present**

Run (Bash tool):

```bash
node -e '
const s = require("fs").readFileSync("plugins/repo-wiki/skills/repo-wiki/references/page-templates.md","utf8");
for (const h of ["`index.md` template","`architecture.md` template","Area page template","`getting-started.md` template","`conventions.md` template","Frontmatter (every page)","Writing rules"]) {
  if (!s.includes(h)) throw new Error("missing: " + h);
}
console.log("OK");
'
```

Expected output: `OK`

- [ ] **Step 3: Commit**

```bash
git add plugins/repo-wiki/skills/repo-wiki/references/page-templates.md
git commit -m "docs(repo-wiki): add page templates reference"
```

---

### Task 5: Core skill (SKILL.md)

**Files:**
- Create: `plugins/repo-wiki/skills/repo-wiki/SKILL.md`

**Interfaces:**
- Consumes: reference filenames from Tasks 2–4 (`references/manifest-schema.md`, `references/crawl-playbook.md`, `references/page-templates.md`); manifest filename from Task 2.
- Produces: skill name `repo-wiki`; marker strings `<!-- repo-wiki:begin -->` / `<!-- repo-wiki:end -->`; generate/update workflows the commands (Task 6) delegate to.

- [ ] **Step 1: Create `plugins/repo-wiki/skills/repo-wiki/SKILL.md`** with exactly:

````markdown
---
name: repo-wiki
description: Generate and maintain a documentation wiki for the current repository. Use when the user asks to "generate a wiki", "document this repo", "create repo documentation", "update the wiki", "refresh the docs", invokes /repo-wiki:generate or /repo-wiki:update, or wants OpenWiki-style automated codebase documentation with incremental git-diff-based updates.
---

# repo-wiki: Repository Documentation Wiki

Generate a documentation wiki for the current repository and keep it fresh with
diff-based incremental updates. All state lives in one file inside the wiki
directory: `.repo-wiki-manifest.json`. Runs entirely in this session — never
call external APIs or require keys.

## Mode selection

| Signal | Mode |
|---|---|
| `/repo-wiki:generate`, "document this repo", "generate a wiki" | generate |
| `/repo-wiki:update`, "refresh the wiki", "update the docs" | update |
| Ambiguous request | manifest exists → update; none → generate |

## References (load on demand, not upfront)

- `references/manifest-schema.md` — manifest format and rules. Read before
  writing or modifying a manifest.
- `references/crawl-playbook.md` — repo sizing, subagent dispatch, area-summary
  format, coverage-gap rule. Read before any crawl or re-crawl.
- `references/page-templates.md` — page structures and frontmatter. Read before
  writing any wiki page.

## Generate workflow

1. **Preflight.** Run `git rev-parse HEAD`. If it fails (not a git repo): warn
   the user that incremental updates will be unavailable (manifest `commit`
   will be `null`; every future update becomes a full regeneration), then
   continue. If `git status --porcelain` shows uncommitted changes, note in the
   final report that the wiki reflects the working tree but the manifest
   records HEAD.
2. **Existing-wiki check.** Glob `**/.repo-wiki-manifest.json` and grep
   CLAUDE.md / AGENTS.md for `<!-- repo-wiki:begin -->`. If either exists:
   recommend update mode instead, and require explicit user confirmation before
   proceeding with a full overwrite.
3. **Resolve output directory** (recorded in the manifest; never re-asked):
   - `--output-dir <dir>` argument → use it.
   - Else detect an existing docs convention: any of `docs/docs-index.md`,
     `docs/**/KB_*.md`, `mkdocs.yml`, `docusaurus.config.*`, or a `docs/`
     directory containing 3+ markdown files. If detected, ask the user whether
     to place the wiki inside that layout or use the default.
   - Else default to `docs/wiki/`.
   - If the chosen directory already contains files not generated by
     repo-wiki, never overwrite them without explicit user confirmation.
4. **Crawl.** Follow `references/crawl-playbook.md` exactly (sizing decides
   single-pass vs. parallel subagents). If the user passed a path scope,
   restrict areas to that subtree. Output: one area summary per area.
5. **Write pages.** Follow `references/page-templates.md`. Always write
   `index.md`. Every page's frontmatter lists the `sources` globs it covers.
6. **Write the manifest** to `<outputDir>/.repo-wiki-manifest.json` per
   `references/manifest-schema.md`, including a run-log entry (`mode:
   "generate"`, `pagesTouched: ["*"]`, actual subagent count).
7. **Write the reference block** (below) into CLAUDE.md, and into AGENTS.md
   only if AGENTS.md already exists. If the repo has no CLAUDE.md, create it
   containing only the block.
8. **Report.** List pages written, output dir, subagent count, and any
   coverage-gap notes.

## Update workflow

1. **Locate the manifest** (same search as generate step 2). None found → tell
   the user to run `/repo-wiki:generate` and stop. Manifest found but
   unparseable or missing required fields → offer a full regeneration; do not
   guess at repairs.
2. **Diff.** Run `git diff --name-status <manifest.commit>..HEAD`.
   - Empty output → report "wiki is up to date as of <commit>" and stop.
   - `manifest.commit` is `null`, or unreachable (`git cat-file -e <sha>`
     fails): try `git merge-base HEAD <sha>` and diff against that; if that
     also fails, confirm with the user, then run the generate workflow as a
     full regeneration, preserving the manifest's `outputDir` and `convention`.
3. **Map stale pages.** A page is stale if any changed file matches any of its
   `sources` globs. Skip `index.md` here — its `["*"]` is structural, per the
   manifest schema. Changed files matching no page → apply the coverage-gap
   rule in `references/crawl-playbook.md`.
4. **Regenerate only stale pages.** Re-read only affected areas (subagents per
   the crawl playbook if an area is large). For deleted source files, edit or
   remove the affected pages. If any page was added, removed, or retitled,
   rewrite `index.md`.
5. **Persist.** Update manifest `commit`, `generatedAt`, page `sources`, and
   append a run-log entry (`mode: "update"`, actual `pagesTouched`). Refresh
   the reference block (idempotent via markers).
6. **Report.** Changed-file count, pages regenerated, coverage gaps filled.

## Reference block

Purpose: future agent sessions discover the wiki WITHOUT loading it into
context. If the markers already exist in the file, replace everything between
them; otherwise append the whole block at the end. Substitute `<outputDir>`;
keep the block at most 6 lines between the markers.

```markdown
<!-- repo-wiki:begin -->
## Repository Wiki
Generated documentation lives in `<outputDir>/`. Read `<outputDir>/index.md`
first, then load only pages relevant to the current task. Do not bulk-load the
wiki. Refresh with /repo-wiki:update. State: <outputDir>/.repo-wiki-manifest.json
<!-- repo-wiki:end -->
```

## Hard rules

- Never overwrite documentation this plugin did not generate without explicit
  user confirmation.
- Never duplicate the reference block; the markers are the idempotency
  contract.
- Never bulk-load the whole repo or the whole wiki into context; follow the
  crawl playbook.
- All state goes in the manifest — write no other dotfiles to the target repo.
- Wiki pages are plain GitHub-flavored markdown with relative links; the only
  HTML this plugin ever writes is the two marker comments.
- Every factual claim in a page must trace to files read or area summaries;
  never invent commands, versions, or behavior.
````

- [ ] **Step 2: Verify frontmatter, length, and reference paths**

Run (Bash tool):

```bash
node -e '
const fs = require("fs");
const s = fs.readFileSync("plugins/repo-wiki/skills/repo-wiki/SKILL.md","utf8");
const fm = s.match(/^---\r?\n([\s\S]*?)\r?\n---/);
if (!fm) throw new Error("no frontmatter");
const name = fm[1].match(/^name:\s*(.+)$/m);
const desc = fm[1].match(/^description:\s*(.+)$/m);
if (!name || name[1].trim() !== "repo-wiki") throw new Error("frontmatter name must be repo-wiki");
if (!desc) throw new Error("missing description");
if (desc[1].length > 1024) throw new Error("description over 1024 chars: " + desc[1].length);
const lines = s.split("\n").length;
if (lines > 500) throw new Error("SKILL.md too long: " + lines + " lines");
for (const r of ["manifest-schema.md","crawl-playbook.md","page-templates.md"]) {
  if (!s.includes("references/" + r)) throw new Error("SKILL.md never mentions references/" + r);
  fs.accessSync("plugins/repo-wiki/skills/repo-wiki/references/" + r);
}
for (const marker of ["<!-- repo-wiki:begin -->","<!-- repo-wiki:end -->"]) {
  if (!s.includes(marker)) throw new Error("missing marker: " + marker);
}
console.log("OK");
'
```

Expected output: `OK`

- [ ] **Step 3: Commit**

```bash
git add plugins/repo-wiki/skills/repo-wiki/SKILL.md
git commit -m "feat(repo-wiki): add core skill with generate/update workflows"
```

---

### Task 6: Slash commands

**Files:**
- Create: `plugins/repo-wiki/commands/generate.md`
- Create: `plugins/repo-wiki/commands/update.md`

**Interfaces:**
- Consumes: skill name `repo-wiki` and its mode names from Task 5.
- Produces: `/repo-wiki:generate` and `/repo-wiki:update` commands.

- [ ] **Step 1: Create `plugins/repo-wiki/commands/generate.md`** with exactly:

```markdown
---
description: Generate a documentation wiki for this repository
argument-hint: "[path-scope] [--output-dir <dir>]"
---

Invoke the repo-wiki skill in **generate** mode and follow its generate
workflow exactly — do not improvise a documentation structure; the skill and
its references define the page set, manifest, and CLAUDE.md reference block.

Arguments (optional): $ARGUMENTS

- A bare path limits generation to that subtree (path scope).
- `--output-dir <dir>` overrides output-directory detection.
```

- [ ] **Step 2: Create `plugins/repo-wiki/commands/update.md`** with exactly:

```markdown
---
description: Incrementally refresh the repository wiki from git changes since the last generation
---

Invoke the repo-wiki skill in **update** mode and follow its update workflow
exactly: locate the manifest, diff HEAD against its recorded commit, regenerate
only stale pages, then update the manifest and reference block.
```

- [ ] **Step 3: Verify frontmatter present in both**

Run (Bash tool):

```bash
node -e '
const fs = require("fs");
for (const f of ["generate","update"]) {
  const s = fs.readFileSync("plugins/repo-wiki/commands/" + f + ".md","utf8");
  if (!/^---\r?\n[\s\S]*?description:\s*.+[\s\S]*?\r?\n---/.test(s)) throw new Error(f + ".md missing description frontmatter");
  if (!s.includes("repo-wiki skill")) throw new Error(f + ".md does not delegate to the skill");
}
console.log("OK");
'
```

Expected output: `OK`

- [ ] **Step 4: Commit**

```bash
git add plugins/repo-wiki/commands/generate.md plugins/repo-wiki/commands/update.md
git commit -m "feat(repo-wiki): add generate and update commands"
```

---

### Task 7: Automated validation

**Files:**
- Modify (only if findings require): any file under `plugins/repo-wiki/`

**Interfaces:**
- Consumes: the complete plugin from Tasks 1–6.
- Produces: validated plugin; no structural or skill-quality findings outstanding.

- [ ] **Step 1: Run the plugin validator**

Dispatch the Agent tool with `subagent_type: "plugin-dev:plugin-validator"`, `run_in_background: false`, prompt:

```text
Validate the plugin at plugins/repo-wiki in this repository (marketplace root
is the repo root, manifest at .claude-plugin/marketplace.json). Check
plugin.json validity, directory structure, command frontmatter, and skill
structure. Report every finding with severity.
```

- [ ] **Step 2: Run the skill reviewer**

Dispatch the Agent tool with `subagent_type: "plugin-dev:skill-reviewer"`, `run_in_background: false`, prompt:

```text
Review the skill at plugins/repo-wiki/skills/repo-wiki/SKILL.md and its
references directory for description triggering quality, progressive
disclosure, structure, and best practices. Report every finding with severity.
```

- [ ] **Step 3: Fix all critical/major findings; re-run the failing validator until clean**

Apply fixes directly to the flagged files. Re-run the corresponding agent from Step 1/2 after fixing. Cosmetic/nitpick findings may be skipped with a one-line justification in the commit body.

- [ ] **Step 4: Re-run all Task 1–6 verification commands** (the five Node one-liners). Expected: `OK` from each.

- [ ] **Step 5: Commit (only if files changed)**

```bash
git add plugins/repo-wiki
git commit -m "fix(repo-wiki): address plugin-validator and skill-reviewer findings"
```

---

### Task 8: Fixture dry-run (end-to-end sanity check)

**Files:**
- Modify (only if gaps found): `plugins/repo-wiki/skills/repo-wiki/SKILL.md`, files under `plugins/repo-wiki/skills/repo-wiki/references/`

**Interfaces:**
- Consumes: complete validated plugin from Task 7.
- Produces: evidence the written workflows are executable as written; fixes for any instruction gap discovered.

This task does NOT install the plugin. The implementer acts as the skill's executor: read SKILL.md and its references, then follow the generate and update workflows literally against a fixture repo. Any point where the instructions are ambiguous or wrong is a bug in the plugin text — fix the text.

- [ ] **Step 1: Create the fixture repo** in the session scratchpad directory (`FIX="<scratchpad>/wiki-fixture"`):

```bash
mkdir -p "$FIX/src/auth" "$FIX/src/api"
cd "$FIX" && git init -b main
cat > package.json <<'EOF'
{ "name": "wiki-fixture", "version": "1.0.0", "scripts": { "start": "node src/api/server.js", "test": "node --test" } }
EOF
cat > README.md <<'EOF'
# wiki-fixture
Tiny two-area app: token auth + HTTP API.
EOF
cat > src/auth/login.js <<'EOF'
// Issues a session token for a username.
function login(username) { return { token: "tok-" + username, expires: 3600 }; }
module.exports = { login };
EOF
cat > src/api/server.js <<'EOF'
// Minimal HTTP API that authenticates via src/auth.
const { login } = require("../auth/login");
function handle(req) { return req.path === "/login" ? login(req.user) : { status: 404 }; }
module.exports = { handle };
EOF
git add -A && git commit -m "fixture: initial app"
```

- [ ] **Step 2: Execute the generate workflow** from `plugins/repo-wiki/skills/repo-wiki/SKILL.md` against `$FIX`, following every numbered step literally (this repo is far under 150 files → single-pass path; no subagents needed).

- [ ] **Step 3: Verify generate output**

```bash
cd "$FIX"
node -e '
const fs = require("fs");
const m = JSON.parse(fs.readFileSync("docs/wiki/.repo-wiki-manifest.json","utf8"));
const head = require("child_process").execSync("git rev-parse HEAD").toString().trim();
if (m.commit !== head) throw new Error("manifest commit != HEAD");
if (m.version !== 1 || m.outputDir !== "docs/wiki") throw new Error("bad manifest fields");
if (!m.pages.some(p => p.file === "index.md")) throw new Error("no index page entry");
fs.accessSync("docs/wiki/index.md");
const c = fs.readFileSync("CLAUDE.md","utf8");
if ((c.match(/<!-- repo-wiki:begin -->/g) || []).length !== 1) throw new Error("marker count != 1");
console.log("OK");
'
```

Expected output: `OK`

- [ ] **Step 4: Verify reference-block idempotency** — re-execute only workflow step 7 (write the reference block) a second time, then:

```bash
cd "$FIX" && node -e '
const c = require("fs").readFileSync("CLAUDE.md","utf8");
if ((c.match(/<!-- repo-wiki:begin -->/g) || []).length !== 1) throw new Error("block duplicated");
console.log("OK");
'
```

Expected output: `OK`

- [ ] **Step 5: Execute the update workflow** — first make a change touching only the auth area:

```bash
cd "$FIX"
cat >> src/auth/login.js <<'EOF'
function logout(token) { return { revoked: token }; }
module.exports.logout = logout;
EOF
git add -A && git commit -m "fixture: add logout"
```

Then follow the update workflow from SKILL.md literally.

- [ ] **Step 6: Verify update precision**

```bash
cd "$FIX" && node -e '
const m = JSON.parse(require("fs").readFileSync("docs/wiki/.repo-wiki-manifest.json","utf8"));
const head = require("child_process").execSync("git rev-parse HEAD").toString().trim();
if (m.commit !== head) throw new Error("manifest commit not advanced");
const last = m.runs[m.runs.length - 1];
if (last.mode !== "update") throw new Error("last run not update");
if (last.pagesTouched.includes("*")) throw new Error("update did a full regen");
if (!last.pagesTouched.some(p => /auth/.test(p))) throw new Error("auth page not touched");
if (last.pagesTouched.some(p => /api/.test(p))) throw new Error("api page touched but its sources did not change");
console.log("OK");
'
```

Expected output: `OK`

- [ ] **Step 7: Fix any instruction gaps found** in SKILL.md or references (ambiguity, wrong ordering, missing detail an executor needed), re-run the affected dry-run step, then commit in the plugin repo:

```bash
cd /c/Dev/ZacsPlugins
git add plugins/repo-wiki
git commit -m "fix(repo-wiki): close instruction gaps found in fixture dry-run"
```

(Skip the commit if the dry-run required no changes; state that explicitly.)

- [ ] **Step 8: Final check — clean tree, all verifications green**

```bash
cd /c/Dev/ZacsPlugins && git status --porcelain
```

Expected output: empty.
