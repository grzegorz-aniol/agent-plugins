---
name: ffleet
description: >
  How I run Forge Fleet (`ffleet`) — isolated, containerized coding-agent
  environments per task. Use when: spinning up work on a GitHub issue, checking
  on a running agent environment, or tearing one down — but only in a repo
  already configured for ffleet; if it isn't, say so and stop. Trigger keywords:
  "ffleet", "forge fleet", "spin up an environment", "worktree environment",
  "sandbox for issue", "remove the environment", "ffleet up", "ffleet remove".
user-invocable: true
---

# ffleet — Forge Fleet

## Precondition — is this project configured?

Everything below applies **only to a repo already configured for ffleet**. Check
first; it is read-only and costs nothing:

```bash
ffleet ls          # exit 0 = configured (lists envs, possibly none)
                   # exit 1 = "no ffleet config found for this project"
```

**If it exits 1, stop.** Don't run `ffleet init` and don't fall back to plain
`git worktree` — setting a project up is the user's call. Tell them the project
isn't configured and that `ffleet init` (`-y` for defaults) would set it up, then
carry on with whatever they actually asked, without ffleet.

Each environment = a **git worktree on its own branch** + a **Docker container**
+ a **coding-agent session**, named by a short **slug**.

**All commands run from inside the project repo** — any subdirectory or any of
its worktrees works; `ffleet` resolves the project from the current directory
(git root + origin remote). Per-project config lives by default at
`~/.forge-fleet/<project>-<hash>/ffleet.toml`, outside the repo.

`ffleet up` is **state-driven**: absent env → create; live session → attach;
stopped/dead → rebuild from current config and resume. Same command every time.

> **Two commands attach to an interactive tmux/shell session and will block a
> non-interactive caller: `ffleet up` (without `--no-attach`) and `ffleet goto`.**
> Agents: always add `--no-attach` to `up`, and read a worktree with
> `git -C "$(ffleet path <slug>)" …` instead of `goto`. Humans detach with
> `Ctrl-B` then `D` — the agent keeps running.

## Starting work

**From a template** (the usual path). `-t <template> <issue-id>`: fetches the
GitHub issue via `gh`, derives the slug from its title, seeds the prompt, and
picks the subagent configured for that template.

```bash
ffleet up -t task 355     # implement issue #355  (dev-orchestrator)
ffleet up -t plan 355     # harden the ticket first (plan-orchestrator)
```

Templates exist per project — check `[templates.*]` in that project's
`ffleet.toml` before assuming `task`/`plan` are available (`ffleet project ls`
prints each project's id; its config is `~/.forge-fleet/<id>/ffleet.toml`).

**Ad-hoc**, no issue behind it — invent a unique slug and pass the prompt:

```bash
ffleet up --no-attach issue-slug-unique-name -p "…what to do…"
```

Useful `up` flags: `--no-attach` (run in background), `-p` / `--prompt-file`,
`--from <branch>` (base a first-time worktree on an existing branch),
`--peek` (side-effect-free: attach only if the agent is already live).

## Checking on it

```bash
ffleet ls [--json]       # all envs: container state, last activity, blocked reason
ffleet status <slug>     # container + agent state, branch, worktree path
ffleet tail <slug> [--lines N]
ffleet path <slug>       # worktree path, for cd or git -C
```

`BLOCKED_REASON: waiting for input` in `ffleet ls` = the agent is asking a
question; re-attach with `ffleet up <slug>` to answer.

## Removing safely

`remove` stops the container, **deletes the worktree**, and deletes the branch
if ffleet created it (a `--from` branch is left alone). Irreversible.

**Land the work first — ffleet never pushes or opens a PR for you.** Check
`ffleet status <slug>` for the branch, make sure the PR is merged or the commits
are pushed, *then* remove.

```bash
ffleet stop <slug>       # just park it: container down, worktree/branch kept
ffleet remove <slug>     # prompts, and warns about uncommitted/unpushed work
```

## `-y` — unattended mode

`-y` / `--yes` skips the confirmation prompt on `remove`, `project rm`,
`project prune`, and `init`. Use it in scripts and non-interactive runs.
(`up` has no `-y`; its unattended switch is `--no-attach`.)

**`-y` also skips the unsaved-work warning** on `remove` — the dirty-tree /
unpushed-commits guard is only rendered into the prompt that `-y` bypasses. So
before any `ffleet remove -y`, run that guard yourself:

```bash
W=$(ffleet path <slug>)
git -C "$W" status --porcelain          # must be empty
git -C "$W" log --oneline @{u}..HEAD    # must be empty
ffleet remove <slug> -y
```

Anything other than empty output from either command — including an `@{u}`
error, which means the branch was never pushed — is work that `-y` would
silently destroy. Sort it out before removing.

## Project registry & setup

```bash
ffleet init [-y]               # one-time per repo; create-only, never overwrites
ffleet project ls [--json]
ffleet project prune [-y]      # drop stale entries + their ~/.forge-fleet dirs
ffleet project rm <id> [-y]    # delete one project's registry entry + home dir
```
