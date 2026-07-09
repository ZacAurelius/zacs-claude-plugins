# Zac's Claude Plugin Marketplace

Personal marketplace of Claude Code plugins.

## Add this marketplace

```
/plugin marketplace add ZacAurelius/zacs-claude-plugins
```

Then install any plugin:

```
/plugin install <plugin-name>@zacs-claude-plugins
```

## Plugins

| Plugin | What it does |
|---|---|
| [repo-wiki](plugins/repo-wiki/) | Generates and maintains a documentation wiki for any repo, entirely inside your Claude Code session — OpenWiki-style, no extra API key. `/repo-wiki:generate`, `/repo-wiki:update`. |
| [example-plugin](plugins/example-plugin/) | Starter example demonstrating a basic command. |

### Featured: repo-wiki

Agents write better code when they understand the repo they're working in.
repo-wiki crawls a codebase (parallel subagents on big repos), writes a
structured wiki (`index`, architecture, per-area pages, getting-started,
conventions), and leaves a five-line reference block in `CLAUDE.md` so every
future session loads pages on demand instead of bloating context. Incremental
`update` regenerates only pages whose source files changed since the last run,
tracked through a single manifest file. Full guide:
[plugins/repo-wiki/README.md](plugins/repo-wiki/README.md).

## Structure

```
.claude-plugin/marketplace.json   # marketplace manifest, lists plugins
plugins/
  <plugin-name>/                  # one plugin per directory
    .claude-plugin/plugin.json    # plugin manifest (required)
    commands/                     # slash commands (*.md)
    skills/<skill-name>/SKILL.md  # skills + references/
    README.md                     # usage guide
```

## Adding a new plugin

1. Create `plugins/<plugin-name>/.claude-plugin/plugin.json` (kebab-case name,
   semver version).
2. Add commands/agents/skills/hooks as needed — component dirs live at the
   plugin root, not inside `.claude-plugin/`.
3. Register it in `.claude-plugin/marketplace.json` under `plugins`.
4. Validate before publishing (the `plugin-dev` plugin's validator and skill
   reviewer catch structural and triggering issues early).
