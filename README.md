# skills

Agent Skills for the [Kanonak Protocol](https://kanonak.org) — an open protocol
for defining, versioning, and sharing semantic ontologies across independent
publishers, with no central registry.

## Install

```bash
npx skills add kanonak-protocol/skills --skill kanonak-protocol --agent claude-code
```

`--agent` decides how the skill lands. With it, the skill is copied straight
into that agent's own directory — `.claude/skills/` for Claude Code. Without
it, the skill goes to the shared `.agents/skills/` and each agent gets a
symlink pointing at it, which is the arrangement that tends to break: symlinks
don't survive a `git clone` on Windows, and some sandboxes and container builds
won't follow them.

Swap the value for `cursor`, `github-copilot`, `gemini-cli`, or `codex`, or
pass `'*'` to install into every agent found.

## Skills

| Skill | What it teaches |
| --- | --- |
| [`kanonak-protocol`](kanonak-protocol/SKILL.md) | Read, write, and validate Kanonak ontologies (`.kan.yml`): the document format, the `publisher/package@version/name` URI scheme, the `kanonak` CLI, and where the canonical modeling and styling guides live. |

Skills here follow the [Agent Skills](https://agentskills.io) format and work
with any compatible client. Nothing in a skill is Claude-specific; only the
install command above names one.

## License

Apache-2.0
