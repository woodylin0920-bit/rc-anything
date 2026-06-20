# Deploying the orchestrator

vibe-stack is **platform-agnostic**. The orchestrator (e.g. Hermes) — the always-on process that receives your chat messages and drives coding agents — runs the same way on a Mac, a Linux VPS, or any host that can stay on and reach the network. This guide covers both environments neutrally and flags where they differ, so you can pick what fits.

## What the orchestrator needs (any platform)

1. **An always-on host** — the process must keep running for your phone to reach it.
2. **Network access** — it polls Telegram (`getUpdates`); no inbound port or public webhook required.
3. **The agent CLIs you'll drive** — Claude Code, Codex, OpenCode, etc.
4. **`tmux` + `git`** — the control plane for interactive sessions and worktree/lane isolation.
5. **Auth** — a logged-in session or API key for each agent (see [Auth](#auth-any-platform)).

## Platform differences to be aware of

| Concern | macOS (laptop/desktop) | Linux VPS (cloud) |
|---------|------------------------|-------------------|
| **Stays on by default?** | **No** — sleeps on lid-close / idle, which suspends the orchestrator and freezes tmux agents. Needs a keep-awake (below). | **Yes** — designed to run 24/7. |
| **Service manager** | `launchd` (user LaunchAgent plist) | `systemd` (system service unit) |
| **Survives power loss / reboot** | Auto-restarts via launchd, but only after login on some setups | Auto-starts on boot, unattended |
| **GUI agents / desktop automation** | **Yes** — Terminal.app, browser control, AppleScript, screen-watching all work | **No** — headless; no GUI session |
| **Local file access** | Your Mac's files | Only files on the VPS (clone repos there) |
| **Cost / wear** | Free hardware, but battery/heat if kept awake | Small monthly fee (~€4–12), no wear |
| **Reachable from anywhere** | Only while awake + online | Stable public host, always reachable |

**Neither is "correct" — it's a trade-off.** A Mac shines when work needs local files, GUI agents, or desktop automation. A Linux VPS shines for always-on, headless, repo-based work. You can run both and pick per task; the same skills (`agent-manager`, `claude-code`, `codex`) work on either host.

## Running always-on

Install the orchestrator and agent CLIs per their own docs, then register it as a service so it auto-starts and auto-restarts.

### Linux (systemd)
```ini
# /etc/systemd/system/hermes.service
[Unit]
Description=Orchestrator gateway
After=network-online.target
Wants=network-online.target

[Service]
User=hermes
WorkingDirectory=/home/hermes/.hermes
ExecStart=/home/hermes/.hermes/venv/bin/python -m hermes_cli.main gateway run
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now hermes
sudo systemctl status hermes      # verify
journalctl -u hermes -f           # live logs
```

### macOS (launchd)
```xml
<!-- ~/Library/LaunchAgents/ai.hermes.gateway.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0"><dict>
  <key>Label</key><string>ai.hermes.gateway</string>
  <key>ProgramArguments</key>
  <array>
    <string>/path/to/.hermes/venv/bin/python</string>
    <string>-m</string><string>hermes_cli.main</string>
    <string>gateway</string><string>run</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
</dict></plist>
```
```bash
launchctl load -w ~/Library/LaunchAgents/ai.hermes.gateway.plist
launchctl list | grep hermes      # verify
```

> **macOS only — keep it awake.** launchd keeps the *process* alive, but macOS still sleeps the *machine* on lid-close/idle, which suspends everything. Two options, your call:
> - **Persistent `caffeinate`** (lid stays open): a second LaunchAgent running `/usr/bin/caffeinate -i -m -s`. Simple, reversible, works on battery.
> - **`sudo pmset -c disablesleep 1`** (allows lid-closed on AC power): no caffeinate needed, but requires AC and is a system-wide power change.
>
> Linux VPS users can ignore this entirely — there is no sleep.

## Connecting Telegram (any platform)

1. Create a bot with [@BotFather](https://t.me/BotFather); copy the token.
2. For group use, turn **off** the bot's privacy mode in BotFather so it can read group messages.
3. Put the token + your allowed chat IDs in the orchestrator's config (e.g. `~/.hermes/.env`):
   ```
   TELEGRAM_BOT_TOKEN=123456:ABC...
   ```
4. The orchestrator long-polls Telegram — no public webhook or open port needed (smaller attack surface, works behind NAT).

## Minimal Linux VPS provisioning

If you go the cloud route, any small VPS works (Hetzner CX22 ~€4/mo, DigitalOcean 2 GB ~$12/mo, Vultr/Linode/Lightsail). 2 vCPU / 4 GB handles the orchestrator plus a few agents. Ubuntu LTS is the easy default.
```bash
ssh root@<vps-ip>
adduser hermes && usermod -aG sudo hermes
apt update && apt install -y git tmux python3 python3-venv curl
# install agent CLIs, e.g.:
npm i -g @anthropic-ai/claude-code @openai/codex
# then install the orchestrator and register the systemd service above
```

## Auth (any platform)

Run each agent's login once (`claude auth login`, `codex login`, `opencode auth login`). On a headless VPS, when an OAuth URL appears, open it on your phone and approve. For unattended re-auth later, prefer long-lived credentials so you never need a desktop browser on the host:
- Claude: `claude setup-token` or `ANTHROPIC_API_KEY`
- Codex: `OPENAI_API_KEY` / `codex login`
- OpenCode: provider key (e.g. `OPENROUTER_API_KEY`)

See each agent's skill for details. The remote re-auth flow (surface the login URL to the user via chat) is in [`agent-manager`](../agent-manager/SKILL.md).
