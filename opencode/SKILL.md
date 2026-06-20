---
name: opencode
description: "Orchestrate the OpenCode CLI through Hermes using tmux: install, auth, launch the TUI, send prompts, and detect idle/working/waiting state."
version: 1.0.0
author: Hermes Agent + SST
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Coding-Agent, OpenCode, SST, tmux, PTY, Automation]
    related_skills: [agent-manager, claude-code, codex]
---

# OpenCode CLI Hermes Orchestration

Procedural memory for delegating coding work to [OpenCode](https://github.com/sst/opencode) from a Hermes agent. OpenCode is a provider-agnostic terminal coding agent (a full-screen TUI, plus a `run` one-shot and a headless `serve` mode). Keep this operational: drive the TUI under tmux, capture the pane to monitor, and report results after OpenCode finishes.

All commands below were checked against **opencode 1.17.8** (`opencode --version`).

## 1. Install & Auth

```bash
curl -fsSL https://opencode.ai/install | bash   # or:
npm i -g opencode-ai@latest                      # npm / bun / pnpm / yarn
brew install anomalyco/tap/opencode             # macOS / Linux
opencode --version
```

OpenCode talks to **any** provider (Anthropic, OpenAI, OpenRouter, local, OpenCode Zen, …) — so unlike Claude Code / Codex, it has **no built-in account**; you must add a provider before it can run a model:

```bash
opencode auth login         # interactive provider + key/OAuth picker (alias: opencode providers)
opencode auth list          # show configured providers (alias: opencode auth ls)
opencode auth logout
```

Credentials live in `~/.local/share/opencode/auth.json` — treat it as a secret; never paste it into chat, logs, or commits. **A fresh install has 0 credentials**: the TUI shows `Run /connect to add an AI provider and start coding`, and any prompt you send returns a "Connect provider" tip until you log in. Check `opencode auth list` before delegating.

## 2. Launch pattern: interactive TUI under tmux

Use the TUI for multi-turn work. tmux is the control plane — persistent PTY, `send-keys`, `capture-pane`. With vibe-stack, prefer the wrapper (it handles the CJK-safe paste and waits for the TUI to paint):

```bash
vibe new opencode opencode-api /path/to/repo "Fix the failing auth tests; keep the change minimal and report files changed."
```

Equivalent raw tmux:

```bash
tmux new-session -d -s opencode-api -x 200 -y 50
tmux send-keys -t opencode-api 'cd /path/to/repo && opencode' Enter   # [project] defaults to cwd
sleep 4                                                                # TUI paints in ~3 s; allow margin on first run
# Inject the task CJK-safely (paste buffer, then Enter as its own keystroke):
printf '%s' 'Fix the failing auth tests. Report files changed.' | tmux load-buffer -b oc -
tmux paste-buffer -t opencode-api -b oc -d
tmux send-keys -t opencode-api C-m
# Monitor:
sleep 5; tmux capture-pane -t opencode-api -p -S -80
```

`opencode` opens a **full-screen (alt-screen) TUI**, so `capture-pane` shows the *current* frame, not scrollback — capture a generous window (`-S -80`) to grab the footer and any dialog. Submit = `Enter`; multiline = `shift+Enter` / `ctrl+j`.

### Non-interactive (one-shot)

For scripted, single-turn work without a TUI, use `run` — it prints the result and exits:

```bash
opencode run "Review the current diff for regressions and list prioritized findings"
opencode run -m anthropic/claude-opus-4-8 --format json "Summarize the repo structure"
opencode run -c "Continue: now add a regression test"      # -c continues the last session
```

Headless server (for the dashboard / `--attach`): `opencode serve --port 4096` (set `OPENCODE_SERVER_PASSWORD`).

## 3. Detecting state (idle vs working vs waiting)

This is the contract `vibe status` relies on. OpenCode has **two built-in agents** — `build` (default, full access) and `plan` (read-only) — switched with `Tab` / `shift+Tab`; the active one shows in the input footer (`Build · <model>`).

| State | What the pane shows | How to detect |
|-------|---------------------|---------------|
| **idle** | The input box `┃ Ask anything…`, footer `tab agents  ctrl+p commands`, and the bottom status bar (`<cwd>  …  <version>`). The frame is **static**. | Two captures ~0.4 s apart are **identical** and no run/permission signature is present. |
| **working** | A status label — `Working…`, `Thinking`, or `Generating` — plus a spinner and an **`esc` to `Interrupt`** hint; tool output / diffs stream in. | The pane **changes** between two captures, **or** the tail matches `Working`/`Thinking`/`Generating`/`Interrupt`. |
| **waiting-input** | A tool-permission dialog with selectable options: **`Allow once`**, **`Always allow`**, **`Deny`** (also `Approve` / `Reject` / `Grant` / `Permission …`). | The tail matches `Allow once` / `Always allow` / a `Permission` request. Surface it to the user — this is a Decision Gate. |

Notes that make detection robust:
- **Motion is the primary "working" signal** and is provider-agnostic — when a model streams, the frame changes every capture. The text labels above are the backstop for a momentary pause (e.g. mid tool call).
- The idle TUI does **not** animate (no cursor-blink churn), so a static frame reliably means idle — not a false "working".
- The process name is **`opencode.exe`** (a bun-compiled binary) even on macOS/Linux — `vibe`'s command-based detection matches on the `opencode` substring, so a session needn't be named `opencode-*`.
- `escape` interrupts a running turn (`session_interrupt`); `ctrl+c` clears the input box (or exits when empty).

## 4. Model selection

OpenCode addresses models as **`provider/model`**:

```bash
opencode -m anthropic/claude-opus-4-8        # set model for the TUI session
opencode run -m openai/gpt-5.5 "..."         # set model for a one-shot
opencode models                              # list every available model (opencode models <provider> to filter)
```

Inside the TUI: `f2` picks a recent model, `ctrl+p` → `/models` opens the full picker, `ctrl+t` cycles agent variants.

## 5. Keys & slash commands (drive via send-keys)

| Action | Key / command |
|--------|---------------|
| Submit message | `Enter` (multiline: `shift+Enter`, `ctrl+j`) |
| Interrupt the running turn | `escape` |
| Switch agent (build ↔ plan) | `Tab` / `shift+Tab` |
| Command palette | `ctrl+p` |
| Add a provider | `/connect` |
| New session / models / help | `/new` · `/models` · `/help` |
| Reference a file / run bash inline | type `@file` / start a line with `!` |
| Exit | `ctrl+c`, `ctrl+d`, or leader `ctrl+x` then `q` |

## 6. Hermes recipe & never-go-silent

After **every** `send-keys` (task, slash command, or permission choice), `capture-pane` again in **3–5 s** to confirm the input landed and OpenCode advanced — a dropped Enter or a fresh permission dialog is otherwise invisible. Keep polling until OpenCode is **idle/done, errored, or waiting on a permission prompt**, then report; surface any `Allow once / Always allow / Deny` dialog to the user the instant it appears, unprompted. "Sent" is not "done." Full rule: [`agent-manager` Prime directive](../agent-manager/SKILL.md#️-prime-directive--never-go-silent).

Through vibe: `vibe status` reports each OpenCode session as `idle` / `working` / `waiting-input`; `vibe send opencode-api "<follow-up>"` injects a follow-up CJK-safely; `vibe kill opencode-api` tears the session down.

## 7. OpenCode-specific pitfalls

- **No provider = no work.** A clean install has zero credentials; run `opencode auth login` (or `/connect` in the TUI) first, or every prompt just returns the connect tip. Check `opencode auth list`.
- **It's an alt-screen TUI.** `capture-pane` shows the live frame, not scrollback — capture a wide window and re-capture to see motion; don't expect a scrolling transcript.
- **`opencode` (TUI) vs `opencode run` (one-shot)** are different surfaces. Use the TUI under tmux for watchable, multi-turn delegation; use `run` for scripted single-shot jobs.
- **Model ids are `provider/model`** — a bare model name is rejected. Run `opencode models` to see valid ids for the configured providers.
- **`plan` agent is read-only.** If edits silently don't happen, you may be on `plan` — press `Tab` to return to `build`.
- **CJK `send-keys` + Enter in one call drops the Enter** (macOS, IME active). Inject via the paste buffer and send `C-m` separately (the launch snippet above), or just use `vibe send` / `vibe new`, which do this for you.
- **Don't report "sent" as "done"** — `capture-pane` 3–5 s after every injection and confirm OpenCode actually advanced (see §6).
