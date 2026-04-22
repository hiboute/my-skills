# my-skills

Personal Claude Code skills, installed via symlinks with periodic auto-update checks.
Pattern inspired by [garrytan/gstack](https://github.com/garrytan/gstack).

## Skills included

- **ideation** — Audit a codebase for concrete improvements (security, performance,
  docs, code-quality, UI/UX, code-revealed feature opportunities) and publish
  findings to GitHub as labeled issues.
- **roadmap** — Generate a strategic product roadmap and publish it to a GitHub
  Project (v2). Re-runnable: each call adds one new sprint, optionally a new phase.

## Install

```bash
git clone https://github.com/hiboute/my-skills.git ~/.claude/skills/my-skills
~/.claude/skills/my-skills/setup
```

`setup` does three things:
1. Creates symlinks `~/.claude/skills/<skill>` → `~/.claude/skills/my-skills/<skill>`
   for every skill folder in this repo.
2. Marks scripts in `bin/` executable.
3. Installs a `SessionStart` hook in `~/.claude/settings.json` that silently checks
   for new versions on each Claude Code session start.

Re-run `setup` any time to repair links or pick up newly added skills.

## Update

The `SessionStart` hook runs `bin/my-skills-update-check` quietly in the background.
It compares the local `VERSION` file against the remote one on `main` and writes a
marker to `~/.my-skills/just-upgraded-from` when there's something newer.

To actually pull updates:

```bash
~/.claude/skills/my-skills/bin/my-skills-upgrade
```

This runs `git pull --ff-only` and re-runs `setup` to relink any new skills.

## State directory

Update-check state lives in `~/.my-skills/`:
- `last-update-check` — epoch of the last remote check (1h cooldown)
- `update-snoozed` — `<version> <level> <epoch>` snooze record
- `just-upgraded-from` — marker for "you just upgraded from version X"

Delete this dir to fully reset update state.

## Disable the auto-update hook

Edit `~/.claude/settings.json` and remove the entry whose `command` references
`my-skills-update-check`. The skills themselves keep working — only the periodic
check stops.

## Uninstall

```bash
~/.claude/skills/my-skills/setup --uninstall
```

Removes symlinks and the hook entry. Leaves the repo dir and `~/.my-skills/` alone
in case you want to reinstall.

## License

MIT.
