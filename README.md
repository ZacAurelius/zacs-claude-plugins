# Zac's Claude Plugin Marketplace

Personal marketplace of Claude Code plugins.

## Add this marketplace

```
/plugin marketplace add ZacAurelius/zacs-claude-plugins
```

## Structure

```
.claude-plugin/marketplace.json   # marketplace manifest, lists plugins
plugins/
  example-plugin/                 # one plugin per directory
    .claude-plugin/plugin.json
    commands/
```

## Adding a new plugin

1. Create `plugins/<plugin-name>/.claude-plugin/plugin.json`
2. Add commands/agents/skills/hooks as needed
3. Register it in `.claude-plugin/marketplace.json` under `plugins`
