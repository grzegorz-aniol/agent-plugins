# agent-plugins

A plugin marketplace of reusable skills and agentic workflows for coding
assistants. One skills tree, consumed by Claude Code, Codex, and any tool that
reads Agent Skills.

## Available plugins

| Plugin | What it gives you |
|---|---|
| `forge-fleet` | Skill `ffleet` — driving [Forge Fleet](https://ffleet.app) isolated, containerized coding-agent environments: starting work on a GitHub issue, checking on a running environment, and tearing one down safely. |

## Installing

Installing is always two steps: **register the marketplace once**, then **install
the plugins you want**. Substitute any plugin name from the table above for
`<plugin>`.

### Claude Code

Register the marketplace. Once per machine, not once per repo:

```bash
claude plugin marketplace add grzegorz-aniol/agent-plugins
```

Then install a plugin, choosing where it should apply:

```bash
claude plugin install <plugin>@agent-plugins --scope user      # every project on this machine
claude plugin install <plugin>@agent-plugins --scope project   # this repo, shared with the team
claude plugin install <plugin>@agent-plugins --scope local     # this repo, just you
```

| Scope | Written to | Use it when |
|---|---|---|
| `user` | `~/.claude/settings.json` | it is personal tooling you want everywhere |
| `project` | `.claude/settings.json`, committed | the repo's workflow depends on it |
| `local` | `.claude/settings.local.json`, gitignored | you are trying it out |

**`--scope project` registers, it does not install.** The committed settings tell a
teammate's Claude Code where the marketplace is and that the plugin should be on,
but each of them still has to run `claude plugin install` themselves. Worth saying
out loud in that repo's own README.

Managing what you have:

```bash
claude plugin list                              # what is installed, and its scope
claude plugin details <plugin>@agent-plugins    # component inventory and token cost
claude plugin update <plugin>@agent-plugins     # pull a newer version
claude plugin disable <plugin>@agent-plugins    # keep it installed, stop loading it
claude plugin uninstall <plugin>@agent-plugins
```

### Codex

Same two steps:

```bash
codex plugin marketplace add grzegorz-aniol/agent-plugins
codex plugin add <plugin>@agent-plugins
```

The `@agent-plugins` suffix is required; `codex plugin add <plugin>` alone will not
resolve. Managing what you have:

```bash
codex plugin list
codex plugin marketplace upgrade agent-plugins   # refresh the git snapshot first
codex plugin add <plugin>@agent-plugins          # then re-add to pick up the new version
codex plugin remove <plugin>@agent-plugins
```

Codex installs **skills only**. Plugins here that also ship agents or slash
commands will contribute just their skills; see [AGENTS.md](AGENTS.md) for what
each ecosystem supports.

### Any other tool

Cursor, Zed, Windsurf, and anything else that reads Agent Skills:

```bash
npx skills add grzegorz-aniol/agent-plugins --all              # every skill here
npx skills add grzegorz-aniol/agent-plugins -s <skill> -a cursor,zed
npx skills add grzegorz-aniol/agent-plugins --all -g           # user-level, not per-repo
```

Skills land in `.agents/skills/`. This path installs skills only, never agents or
commands.

### After installing

**Start a new session.** Neither Claude Code nor Codex picks up a freshly installed
skill in a session that is already running.

## Updating

Versions are per plugin, and Claude Code treats the version as a cache key: if the
version did not change, `claude plugin update` reports *"already at the latest
version"* and does nothing, even when the skill's contents did change. So an update
that appears to do nothing usually means the release did not bump its version.

To refresh the catalog itself after new plugins are published here:

```bash
claude plugin marketplace update agent-plugins
codex plugin marketplace upgrade agent-plugins
```

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Plugin installs but the skill never fires | Session was already running. Restart it. |
| `claude plugin details` shows `Skills (0)` | Layout problem in the plugin, not on your end. Open an issue. |
| `claude plugin update` says "already at latest" | The plugin's version was not bumped. |
| Skill seems to load twice, or edits have no effect | An older hand-installed copy still sits in `~/.agents/skills/<skill>` or `.claude/skills/<skill>` beside the plugin. Both load. Delete the hand-installed one. |
| `codex plugin add` errors on the selector | Missing `@agent-plugins`. |

## Contributing to this repo

[AGENTS.md](AGENTS.md) documents the layout, the manifest rules that are easy to get
wrong, and what to verify before pushing.

## License

MIT — see [LICENSE](LICENSE).
