# agent-plugins

Marketplace of reusable skills and agentic workflows for coding assistants
(Claude Code, Codex, and any tool that consumes Agent Skills).

## Plugins

| Plugin | What it gives you |
|---|---|
| `forge-fleet` | Skill `ffleet` — driving [Forge Fleet](https://ffleet.app) isolated, containerized coding-agent environments: starting work on an issue, checking on a running environment, and tearing one down safely. |

## Install

**Claude Code**

```bash
claude plugin marketplace add grzegorz-aniol/agent-plugins
claude plugin install forge-fleet@agent-plugins --scope user
```

**Codex**

```bash
codex plugin marketplace add grzegorz-aniol/agent-plugins
codex plugin add forge-fleet@agent-plugins
```

**Any other tool**

```bash
npx skills add grzegorz-aniol/agent-plugins --all
```

Start a new session after installing.

## License

MIT — see [LICENSE](LICENSE).
