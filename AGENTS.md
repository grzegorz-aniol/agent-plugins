# Repository guide

This repo is a plugin marketplace serving three installers from one skills tree:
Claude Code, Codex, and `npx skills`.

## Layout

```
agent-plugins/
├── .claude-plugin/marketplace.json      # Claude catalog
├── .agents/plugins/marketplace.json     # Codex catalog
└── plugins/<plugin>/
    ├── .claude-plugin/plugin.json
    ├── .codex-plugin/plugin.json
    ├── skills/<skill>/SKILL.md          # shared by all three installers
    ├── agents/<agent>.md                # Claude only
    └── commands/<cmd>.md                # Claude only
```

## Rules

- Only manifests belong in `.claude-plugin/` and `.codex-plugin/`. `skills/`,
  `agents/`, `commands/` sit at the plugin root. Nesting them inside a dot-dir
  yields a plugin with zero components and no error.
- A skill is a directory whose name matches its frontmatter `name`, containing
  `SKILL.md` (uppercase, case-sensitive). `skill.md` is never discovered.
- With `metadata.pluginRoot` set, `source` in the Claude marketplace must have
  no `./` prefix. `claude plugin validate` does not catch this; only an install does.
- The per-entry `skills` array is what lets `npx skills` find skills. Claude
  ignores it. Always include it.
- Keep `version` in sync between `.claude-plugin/plugin.json` and
  `.codex-plugin/plugin.json`, and bump it on every change: it is Claude's cache
  key, and `claude plugin update` does nothing without a bump.

## Before pushing

```bash
claude plugin validate . --strict
claude plugin marketplace add "$PWD" && claude plugin install <plugin>@agent-plugins -y
```
