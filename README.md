# hiboute-skills

Personal Claude Code skills, installed via symlinks with periodic auto-update checks.
Pattern inspired by [garrytan/gstack](https://github.com/garrytan/gstack).

## Skills included

- **ideation** — Audit a codebase for concrete improvements (security, performance,
  docs, code-quality, UI/UX, code-revealed feature opportunities) and publish
  findings to GitHub as labeled issues.
- **roadmap** — Generate a strategic product roadmap and publish it to a GitHub
  Project (v2). Re-runnable: each call adds one new sprint, optionally a new phase.
- **autopilot** — Autonomously build a feature end-to-end via a three-phase
  pipeline: plan → implement (in an isolated git worktree) → AI review with a
  bounded fix loop. Each phase runs as a fresh-context subagent. Pipeline pattern
  inspired by [AndyMik90/Aperant](https://github.com/AndyMik90/Aperant).

## Install

```bash
git clone https://github.com/hiboute/skills.git ~/.claude/skills/hiboute-skills
~/.claude/skills/hiboute-skills/setup
```

`setup` does four things:
1. For every skill folder in this repo, creates a real directory at
   `~/.claude/skills/<skill>/` containing a single symlink
   `SKILL.md` → `~/.claude/skills/hiboute-skills/<skill>/SKILL.md`. This is the same
   pattern used by [gstack](https://github.com/garrytan/gstack) — it lets Claude
   discover the skill at the top level of `~/.claude/skills/` (so it's reachable
   as `/ideation`, not `/hiboute-skills/ideation`) while the actual content stays in
   the cloned repo. Any legacy whole-directory symlink left by an earlier
   install is migrated in place. Skill name comes from the SKILL.md `name:`
   frontmatter field, with the directory name as fallback.
2. Marks scripts in `bin/` executable.
3. Installs a `SessionStart` hook in `~/.claude/settings.json` that, on each
   Claude Code session start, forks a background job which `git pull`s the repo,
   re-runs `setup -q` if `HEAD` moved, and logs to
   `~/.hiboute-skills/analytics/session-update.log`. Pattern modeled on gstack's
   `gstack-session-update`: exits fast, never blocks the session, 1h throttle,
   lockfile with stale-PID detection.
4. Writes a marker-delimited block to `~/.claude/CLAUDE.md` listing the installed
   skills and the install/update commands, so Claude knows they exist. The block
   is regenerated on every run; any content outside the `<!-- BEGIN/END hiboute-skills -->`
   markers is preserved.

Because only `SKILL.md` is symlinked, skill files that reference sibling content
(like `categories/…` or `references/…`) must use absolute paths inside the
cloned repo: `~/.claude/skills/hiboute-skills/<skill>/<path>`. Same convention as
gstack's SKILL.md files.

Re-run `setup` any time to repair links, refresh the CLAUDE.md block, or pick up
newly added skills.

## Update

Updates happen automatically on the next Claude Code session start (forked to
the background — zero latency). The `SessionStart` hook runs
`bin/hiboute-skills-session-update`, which:

- Exits 0 immediately (fork to background; never blocks session startup)
- Throttles to at most one check per hour
- Acquires a lockfile (with stale-PID detection) to avoid concurrent upgrades
- Runs `git pull --ff-only` and, if `HEAD` moved, re-runs `./setup -q`
- Writes `~/.hiboute-skills/just-upgraded-from` so the next session can surface a note
- Logs every run to `~/.hiboute-skills/analytics/session-update.log`

To force an update now (outside of a session):

```bash
~/.claude/skills/hiboute-skills/bin/hiboute-skills-upgrade
```

To see whether a newer version is available without pulling:

```bash
~/.claude/skills/hiboute-skills/bin/hiboute-skills-update-check
```

## State directory

State lives in `~/.hiboute-skills/`:
- `last-session-update` — epoch of the last auto-update run (1h throttle)
- `last-update-check` — epoch of the last manual version check (1h cooldown)
- `update-snoozed` — `<version> <level> <epoch>` snooze record (manual check only)
- `just-upgraded-from` — marker for "you just upgraded from version X"
- `.setup-lock/` — lockfile directory used during background auto-updates
- `analytics/session-update.log` — per-run log of the auto-update hook

Delete this dir to fully reset update state.

## Disable the auto-update hook

Edit `~/.claude/settings.json` and remove the entry whose `command` references
`hiboute-skills-session-update`. The skills themselves keep working — only the
periodic auto-update stops.

## Uninstall

```bash
~/.claude/skills/hiboute-skills/setup --uninstall
```

Removes the per-skill directories we own (those whose `SKILL.md` is a symlink
pointing into this repo), any leftover legacy symlinks, the SessionStart hook
entry, and the CLAUDE.md block. Leaves the repo dir and `~/.hiboute-skills/` alone
in case you want to reinstall.

## License

MIT.
