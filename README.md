# rc-anything

A [Claude Code](https://code.claude.com/docs/en/cli-reference) skill: **`claude-code`** — a comprehensive orchestration guide for delegating coding tasks to the Claude Code CLI (Anthropic's autonomous coding agent).

It covers print mode vs. interactive PTY sessions, the full CLI flag reference, settings & `CLAUDE.md` hierarchy, slash commands, hooks, custom subagents, MCP integration, PR-review patterns, parallel instances, and launching with `--remote-control`.

## Contents

```
claude-code/
  SKILL.md    # the skill guide
```

## Install as a Claude Code skill

Drop it into your skills directory:

```bash
# Personal (all projects)
mkdir -p ~/.claude/skills
cp -r claude-code ~/.claude/skills/

# or Project-scoped (team-shared, git-tracked)
mkdir -p .claude/skills
cp -r claude-code .claude/skills/
```

Claude Code auto-discovers skills in `.claude/skills/` and invokes them by natural language when a task matches the skill's `description`. Just ask Claude to delegate a coding task to the Claude Code CLI and it will pull in this guide.

## Use as a reference

`claude-code/SKILL.md` also works standalone — read it as a cheat sheet for the Claude Code CLI, its flags, and automation patterns.

## License

MIT
