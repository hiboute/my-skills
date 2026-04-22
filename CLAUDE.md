# my-skills — repo notes for Claude

## Install pattern

This repo follows [gstack](https://github.com/garrytan/gstack)'s symlink pattern
for Claude skills. The repo is cloned to `~/.claude/skills/my-skills/`, then for
each skill folder with a `SKILL.md`, `setup` creates:

- A real directory at `~/.claude/skills/<name>/`
- A `SKILL.md` symlink inside it, pointing to this repo's
  `<skill>/SKILL.md`

Skill name comes from the frontmatter `name:` field (falling back to the dir
name). Sibling content inside a skill (e.g. `categories/*.md`,
`references/*.md`) is addressed inside SKILL.md via absolute paths
(`~/.claude/skills/my-skills/<skill>/<path>`), because only `SKILL.md` is linked
into `~/.claude/skills/`. That is gstack's convention — don't rewrite sibling
references back to relative paths.

## CLAUDE.md block

The `setup` script also manages a marker-delimited block inside
`~/.claude/CLAUDE.md`. The block tells Claude which `my-skills` are installed
and how to install/update them. It's regenerated on every `./setup` run so
newly added skills show up automatically.

The installed block looks like this:

```
<!-- BEGIN my-skills -->
Available hiboute skills:

'/ideation'
'/roadmap'

Install with: 'git clone https://github.com/hiboute/my-skills.git ~/.claude/skills/my-skills && ~/.claude/skills/my-skills/setup'
Update with:  '~/.claude/skills/my-skills/bin/my-skills-upgrade'
<!-- END my-skills -->
```

To remove the block, run `./setup --uninstall`.
