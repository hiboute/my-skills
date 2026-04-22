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

`setup` does four things:
1. Creates symlinks `~/.claude/skills/<skill>` → `~/.claude/skills/my-skills/<skill>`
   for every skill folder in this repo.
2. Marks scripts in `bin/` executable.
3. Installs a `SessionStart` hook in `~/.claude/settings.json` that, on each
   Claude Code session start, forks a background job which `git pull`s the repo,
   re-runs `setup -q` if `HEAD` moved, and logs to
   `~/.my-skills/analytics/session-update.log`. Pattern modeled on gstack's
   `gstack-session-update`: exits fast, never blocks the session, 1h throttle,
   lockfile with stale-PID detection.
4. Writes a marker-delimited block to `~/.claude/CLAUDE.md` listing the installed
   skills and the install/update commands, so Claude knows they exist. The block
   is regenerated on every run; any content outside the `<!-- BEGIN/END my-skills -->`
   markers is preserved.

Re-run `setup` any time to repair links, refresh the CLAUDE.md block, or pick up
newly added skills.

## Update

Updates happen automatically on the next Claude Code session start (forked to
the background — zero latency). The `SessionStart` hook runs
`bin/my-skills-session-update`, which:

- Exits 0 immediately (fork to background; never blocks session startup)
- Throttles to at most one check per hour
- Acquires a lockfile (with stale-PID detection) to avoid concurrent upgrades
- Runs `git pull --ff-only` and, if `HEAD` moved, re-runs `./setup -q`
- Writes `~/.my-skills/just-upgraded-from` so the next session can surface a note
- Logs every run to `~/.my-skills/analytics/session-update.log`

To force an update now (outside of a session):

```bash
~/.claude/skills/my-skills/bin/my-skills-upgrade
```

To see whether a newer version is available without pulling:

```bash
~/.claude/skills/my-skills/bin/my-skills-update-check
```

## State directory

State lives in `~/.my-skills/`:
- `last-session-update` — epoch of the last auto-update run (1h throttle)
- `last-update-check` — epoch of the last manual version check (1h cooldown)
- `update-snoozed` — `<version> <level> <epoch>` snooze record (manual check only)
- `just-upgraded-from` — marker for "you just upgraded from version X"
- `.setup-lock/` — lockfile directory used during background auto-updates
- `analytics/session-update.log` — per-run log of the auto-update hook

Delete this dir to fully reset update state.

## Disable the auto-update hook

Edit `~/.claude/settings.json` and remove the entry whose `command` references
`my-skills-session-update`. The skills themselves keep working — only the
periodic auto-update stops.

## Uninstall

```bash
~/.claude/skills/my-skills/setup --uninstall
```

Removes symlinks, the hook entry, and the CLAUDE.md block. Leaves the repo dir
and `~/.my-skills/` alone in case you want to reinstall.

## License

MIT.
