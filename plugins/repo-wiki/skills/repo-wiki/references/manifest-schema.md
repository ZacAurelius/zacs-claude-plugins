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
