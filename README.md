# Sailfork agent skills

Skills that teach coding agents (Claude Code, Codex, Cursor and others that read `SKILL.md`) to work on
[Sailfork](https://sailfork.com)-hosted repositories.

| Skill | What it covers |
| --- | --- |
| [`caisson`](skills/caisson/SKILL.md) | Reading, editing, testing and shipping a change in a caisson, a server-side worktree with no local checkout |
| [`sailfork`](skills/sailfork/SKILL.md) | Issues, pull requests, CI, review and merge through the `sailfork` CLI or its MCP tools |

## Install

With [skills.sh](https://skills.sh):

```sh
npx skills add https://git.sailfork.com/sailfork/skills.git
```

With the Sailfork CLI, which ships the same skills:

```sh
sailfork skill install
```

## Source

These files are copied from `skills/` in the Sailfork repository on every change there; edit them there, not here.

## License

MIT, see [LICENSE](LICENSE).
