# my-skills — CLAUDE.md block

The `setup` script manages a marker-delimited block inside
`~/.claude/CLAUDE.md`. The block tells Claude which `my-skills` are installed
and how to install/update them. It's regenerated on every `./setup` run so
newly added skills show up automatically.

The installed block looks like this:

```
<!-- BEGIN my-skills -->
Available my-skills skills:

/ideation
/roadmap

Install with: git clone https://github.com/hiboute/my-skills.git ~/.claude/skills/my-skills && ~/.claude/skills/my-skills/setup
Update with:  ~/.claude/skills/my-skills/bin/my-skills-upgrade
<!-- END my-skills -->
```

To remove the block, run `./setup --uninstall`.
