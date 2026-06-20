# vibe-stack

**Control your AI coding agents from Telegram. No VPN. No dashboard. No laptop open.**

Message a bot → it spawns Claude Code / Codex / OpenCode in your repo, drives them, surfaces real decisions back to you, and reports results. You're the brain; it's your right hand. Watch on the mobile app, steer from chat.

## Install (3 lines)

```bash
hermes skills install https://raw.githubusercontent.com/woodylin0920-bit/vibe-stack/main/agent-manager/SKILL.md
hermes skills install https://raw.githubusercontent.com/woodylin0920-bit/vibe-stack/main/claude-code/SKILL.md
hermes skills install https://raw.githubusercontent.com/woodylin0920-bit/vibe-stack/main/codex/SKILL.md
```

No daemon to run, no port to open, no binary to install — these are **Markdown playbooks** your orchestrator reads. New here? → **[GETTING-STARTED.md](GETTING-STARTED.md)**.

## Why vibe-stack (honestly)

tmux + git-worktree agent orchestration is a **crowded, well-served space** — [Agent of Empires](https://github.com/agent-of-empires/agent-of-empires), [Batty](https://github.com/battysh/batty), and [codex-orchestrator](https://github.com/kingbootoshi/codex-orchestrator) all do it well. That's **not** our differentiator, and we don't try to out-orchestrate them.

vibe-stack's moat is a combination nobody else ships:

1. **Chat-first, channel-agnostic control.** Telegram (or LINE / Slack / web / CLI) as the *primary* interface — NAT-traversing and mobile-first: no VPN, no exposed port, no dashboard to host. The agent pushes to your phone; you reply. (Others have surfaces too — AoE a web dashboard, Batty a Discord bot — but they're add-ons to a binary, not the core model.)
2. **The Hermes skill format.** The orchestration logic is **portable, version-controlled Markdown**, installed with one line, anywhere. Nothing to compile; the playbook travels as docs — so it also **composes with** the tools above instead of replacing them.

| | **vibe-stack** | Agent of Empires | Batty | codex-orchestrator |
|---|---|---|---|---|
| Orchestration | tmux + worktrees (via skills) | tmux + worktrees | tmux + worktrees (Rust daemon) | tmux (Claude→Codex) |
| Primary control surface | **chat (Telegram / any channel)** | TUI + web dashboard | Discord bot + TUI | Claude Code / CLI |
| Reach from phone | **chat — no port/VPN** | web (host a port / tunnel) | Discord | — |
| Distribution | **one-line Markdown skills** | install the binary | install the binary | install the tool |
| Composes with the others | **yes, by design** | — | — | **used *inside* vibe-stack** |

We compose, we don't compete — see **[docs/ecosystem.md](docs/ecosystem.md)** for an honest map of when to use which, and how to stack them.

## Skills

| Skill | What it does |
|-------|--------------|
| [`agent-manager`](agent-manager/SKILL.md) | The **control plane** — open agents, send tasks, drive slash commands/keys remotely, detect prompts, report status, handle auth. Scales to **multi-agent fleets** (lanes, race mode, review gate, merge captain), with model routing and the goal-driven `/loop`. **Start here.** |
| [`claude-code`](claude-code/SKILL.md) | Drive the **Claude Code CLI** — print vs interactive PTY, flags, settings, slash commands, hooks, subagents, MCP, `--remote-control`. |
| [`codex`](codex/SKILL.md) | Drive the **OpenAI Codex CLI** — `codex exec`, `--full-auto`/`--yolo`, PTY + git-repo needs, tmux driving. |
| [`coffee-time`](coffee-time/SKILL.md) | Multi-agent **brainstorm** — fan one question to N models, synthesize with Opus, present options. |

## Contents

```
agent-manager/  SKILL.md   # remote control plane for coding agents
claude-code/    SKILL.md   # Claude Code CLI orchestration guide
codex/          SKILL.md   # OpenAI Codex CLI orchestration guide
coffee-time/    SKILL.md   # multi-agent brainstorm mode
dashboard/      server.py  # real-time agent dashboard (phone-friendly)
docs/           deployment.md · ecosystem.md
GETTING-STARTED.md          # cloud-first quickstart
```

## Deployment

Runs the same on a Linux VPS (recommended — always-on, no sleep, fixed IP) or your Mac — see **[docs/deployment.md](docs/deployment.md)**.

## License

MIT
