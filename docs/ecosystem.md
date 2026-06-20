# Ecosystem map — when to use vibe-stack vs the others (and how to stack them)

The AI-coding-agent orchestration space is **crowded and good**. vibe-stack does not try to be a better tmux orchestrator than the tools below — it's a **chat-first control layer + portable skill format** that **composes** with them. This page is an honest map.

## The landscape

| Tool | What it is | Strongest at | Control surface |
|------|-----------|--------------|-----------------|
| **vibe-stack** (this) | Portable Hermes **skills** for chat-driven agent control | Driving agents from your **phone** over any chat channel; one-line install; composing other tools | Telegram / LINE / Slack / web / CLI |
| [Agent of Empires](https://github.com/agent-of-empires/agent-of-empires) | Rust session manager (TUI + web) | A polished **visual dashboard** over many agents; broad agent support; Docker sandboxing | TUI + web (mobile via web) |
| [Batty](https://github.com/battysh/batty) | Rust daemon, kanban-driven, test-gated | **Unattended team runs** with roles, self-healing, closed test→merge loop | Discord bot + TUI |
| [codex-orchestrator](https://github.com/kingbootoshi/codex-orchestrator) | Tmux delegation tool | **Claude orchestrating Codex** — Claude plans, Codex does the deep coding | Claude Code / CLI |
| [claude-squad](https://github.com/smtg-ai/claude-squad) | Tmux + worktree TUI | Quick **parallel sessions** across agents in one terminal | TUI |
| [vibe-kanban](https://github.com/BloopAI/vibe-kanban) | Kanban board for agents | **Card → branch → PR** visual workflow | Web board |

(No star counts here on purpose — they go stale; follow the links for current state.)

## When to use which

- **You want to run agents from your phone, anywhere, with nothing to host** → **vibe-stack**. Chat is the interface; the agent pushes to you.
- **You want a rich visual dashboard / TUI at your desk** → **Agent of Empires** (or claude-squad for a lighter TUI). Run it *alongside* vibe-stack.
- **You want a daemon that supervises a whole team unattended with test-gated merges** → **Batty**. (vibe-stack's fleet model is the same shape, expressed as skills rather than a binary — pick the binary if you want it turnkey.)
- **You want Claude to delegate the heavy coding to Codex** → **codex-orchestrator** (see below — vibe-stack recommends it for exactly this).
- **You want a card→branch→PR board** → **vibe-kanban**.

## How they compose with vibe-stack

vibe-stack's [lanes/fleets](../agent-manager/SKILL.md#multi-agent-orchestration-fleets) are deliberately built from primitives (tmux, git worktrees), so other tools slot in:

- **Claude → Codex delegation inside a lane.** Don't hand-roll Claude-orchestrating-Codex — run [codex-orchestrator](https://github.com/kingbootoshi/codex-orchestrator) *inside* a vibe-stack lane. Claude Code plans and synthesizes; codex-orchestrator spawns/monitors the Codex workers; vibe-stack carries the whole thing over chat to your phone. See [`agent-manager` → Ecosystem integrations](../agent-manager/SKILL.md#ecosystem-integrations).
- **Visual TUI alongside chat.** Keep [Agent of Empires](https://github.com/agent-of-empires/agent-of-empires) or [claude-squad](https://github.com/smtg-ai/claude-squad) open at your desk for a live TUI of the same tmux sessions vibe-stack drives — chat for mobile, TUI when you're at the keyboard. They watch the same sessions.
- **Batty for the unattended team, vibe-stack for the phone.** If you adopt Batty's daemon for supervised team runs, vibe-stack's chat layer + skills still give you the mobile control/notification surface on top.
- **vibe-kanban for the board.** Use it as the visual board behind vibe-stack's lane states if you want cards and PRs.

## What vibe-stack is *not*

- **Not a daemon or a binary.** Nothing to compile or keep running besides your orchestrator (Hermes). The skills are Markdown.
- **Not a better tmux orchestrator.** The tools above are excellent at that; we reuse the same primitives and lean on them.
- **Not a dashboard.** We ship a tiny optional [`dashboard/`](../dashboard/), but the point is **you don't need one** — chat is the surface.

The bet: orchestration is becoming a commodity; **the interface (chat, anywhere) and the format (portable skills that compose)** are where the durable value is.
