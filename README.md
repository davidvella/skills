# skills

Agent skills I use across machines and coding agents. Each skill is a folder with
a `SKILL.md` (name and description in YAML front matter, then instructions) and
optional `references/` files the agent reads on demand. The format is plain
Markdown, so any harness that supports skills can use it.

| Skill | What it does |
| --- | --- |
| [go-style](skills/go-style/SKILL.md) | Write, review and refactor Go that is easy to read and still fast: small low-complexity functions, performance designed into the API (after Dave Cheney's "API Design as a Performance Primitive"), and changes proven with benchmarks, allocation counts and escape analysis. |

## Install

Clone once, then link each skill into the folders your agents read, so a
`git pull` updates every agent at once.

```shell
git clone https://github.com/davidvella/skills ~/Documents/GitHub/skills

# Shared location used by agents that follow the Agent Skills layout.
mkdir -p ~/.agents/skills
ln -s ~/Documents/GitHub/skills/skills/go-style ~/.agents/skills/go-style

# Claude Code.
mkdir -p ~/.claude/skills
ln -s ~/Documents/GitHub/skills/skills/go-style ~/.claude/skills/go-style
```

For a harness without skill support, point its global instructions file (for
example `~/.codex/AGENTS.md` or `~/.gemini/GEMINI.md`) at the skill:

```markdown
For any Go writing, review or refactoring, read and follow
~/Documents/GitHub/skills/skills/go-style/SKILL.md.
```

Skills load when an agent decides the description matches the task. To make one
apply every time, add the same line to the harness's always-loaded instructions.
