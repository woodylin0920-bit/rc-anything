---
name: overnight
description: "Unattended, goal-driven autonomy loop — give the orchestrator a goal before bed; it decomposes the work into isolated lanes, spawns coding agents with `vibe new`, polls every 60s, gates each finished lane through `vibe-review`, surfaces every real decision, and reports results to your channel. Built on agent-manager + vibe + vibe-poller."
version: 1.0.0
author: Hermes Agent + Teknium
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Autonomy, Overnight, Orchestration, Multi-Agent, Worktree, Review-Gate, Telegram, Automation]
    related_skills: [agent-manager, vibe-poller]
---

# overnight — ship while you sleep

Hand the orchestrator a **goal** at night and wake up to **reviewed, ready-to-merge work** — or a precise question if it hit a real decision. This is the long-running, unattended arm of the [agent-manager](../../agent-manager/SKILL.md) control plane: it drives the [drive loop](../../agent-manager/SKILL.md) across multiple [lanes](../../agent-manager/SKILL.md) until each is **done**, **blocked on you**, or **failed** — never going silent ([Prime directive](../../agent-manager/SKILL.md)).

It does **not** replace your judgment. It executes the routine path autonomously and **escalates every real decision** (architecture, scope, merge) to you. You are the brain; this is the right hand that works the night shift.

## Prerequisites
- `vibe` and `vibe-review` on `PATH` (the [vibe-stack CLI](../../bin/vibe)).
- A channel configured (Telegram / LINE / Slack / web / CLI) — overnight is channel-agnostic.
- [vibe-poller](../vibe-poller/SKILL.md) for the event-notification half (state-change pings).

## What you give it
A one-line **goal**, and (optionally) a pre-decomposition into lanes. If you don't decompose it, the orchestrator does — that's its job.
```
overnight "Add JWT refresh + rotate, with tests, on the api repo"
```

## The loop (what runs each night)

1. **Decompose the goal into lanes.** Split into independent chunks of **≤3 deliverables each** (the agent-manager [decomposition rule](../../agent-manager/SKILL.md) — one loaded agent is slow and unfocused; small specs review cleanly). Each chunk becomes one lane = **worktree + branch + agent session**, isolated so agents never clobber each other's working tree.
2. **Spawn each lane.** One `vibe new` per lane, into its own git worktree:
   ```bash
   git -C /repo worktree add ../lanes/jwt-refresh -b lane/jwt-refresh
   vibe new claude claude-jwt-refresh /repo/../lanes/jwt-refresh "<task spec: goal + ≤3 done_criteria + test_command>"
   ```
   Vary agent/model per lane when you want genuinely different attempts (see agent-manager Race mode).
3. **Poll every 60s.** `vibe status` snapshots the whole fleet in one call. Relay **change events** (idle↔working↔waiting-input) the instant they happen — this is exactly [vibe-poller](../vibe-poller/SKILL.md). Report changes, not heartbeats (the periodic "still alive" ping is the [heartbeat](#heartbeat--still-alive-ping-every-30-min), below).
4. **Self-heal the mechanical, no asking.** Idle-but-incomplete → `vibe send <session> "continue toward: <goal>"`. Recoverable error / test failure → feed it back to the agent. These are not decisions.
5. **Gate every finished lane.** The moment a lane goes **working → idle** (task likely done), run the review gate against its branch:
   ```bash
   vibe-review lane/<slug>      # diffstat + bash -n / py_compile on changed files
   ```
   - **PASS** → tell the user the lane is **ready to merge**, with the diffstat. **Do not merge** — that's a real decision (see below).
   - **FAIL** → feed the failures back to the lane's agent (`vibe send`) and keep driving; re-gate when it finishes again.
6. **Escalate every real decision — never auto-merge.** Anything irreversible or a matter of taste — which option to pick, delete vs keep, and **merging to `main`** — goes to the user via the [Decision Gate](../../agent-manager/SKILL.md). Overnight surfaces "lane X passed the gate, merge it?" and **waits**; the [merge captain](../../agent-manager/SKILL.md) integrates only on your go-ahead, one lane at a time.
7. **Morning report.** When every lane is terminal (merged-ready / blocked / failed), send a roll-up: per-lane state, gate result, what's waiting on you.

## Message shapes
```
🌙 overnight started — goal: "JWT refresh + rotate" → 2 lanes spawned
🔵 claude-jwt-refresh — working
🟢 claude-jwt-refresh — finished; gate PASS → ready to merge (api: +３ files, 120/120 tests)  ❓ merge it?
🟡 claude-jwt-tests — needs you (waiting-input)
   > Overwrite existing auth_test.py? (y/N)
🔴 claude-jwt-rotate — gate FAIL → fed errors back, still working
🌅 morning report — 1 lane ready to merge · 1 waiting on you · 1 retrying
```
Emoji follow `vibe` convention: 🟢 idle · 🔵 working · 🟡 waiting-input · 🔴 problem/gone.

## Reference watch loop (the orchestrator runs this each tick; it prints events to relay)
Pure shell, bash 3.2. Deps: `vibe`, `vibe-review`, `python3` (JSON parsing, already used across the repo). Run it from inside a worktree of the target repo so `vibe-review` can resolve the lane branches. Prints one line per event; the orchestrator relays each line (and any captured logs) to the channel.

```bash
#!/usr/bin/env bash
# overnight-watch — poll lanes; gate each through vibe-review the moment it
# finishes; emit events for the orchestrator to relay. Stays silent when quiet.
set -u
POLL="${OVERNIGHT_POLL:-60}"                         # seconds between polls
STATE="${VIBE_STATE_DIR:-$HOME/.vibe-stack}/overnight-state.tsv"
mkdir -p "$(dirname "$STATE")"; [ -f "$STATE" ] || : > "$STATE"
prev() { awk -F'\t' -v s="$1" '$1==s{print $2; exit}' "$STATE"; }

while :; do
  now=$(mktemp)
  vibe status | python3 -c 'import sys,json
for a in json.load(sys.stdin): print(a["session"]+"\t"+a["state"])' > "$now" 2>/dev/null

  # No agent sessions left → the run is over.
  if [ ! -s "$now" ]; then
    echo "🌅 overnight: all lanes finished"; vibe summary; rm -f "$now"; break
  fi

  while IFS=$'\t' read -r s st; do
    [ -n "$s" ] || continue
    was=$(prev "$s")
    [ "$was" = "$st" ] && continue                   # only act on transitions
    case "$st" in
      working)
        printf '🔵 %s — working (%s → working)\n' "$s" "${was:-new}";;
      waiting-input)
        printf '🟡 %s — needs you (waiting-input)\n' "$s"
        vibe logs "$s" --tail 10 | sed 's/^/   > /';;
      idle)
        # working → idle = the lane likely just finished → run the review gate.
        if [ "$was" = "working" ]; then
          slug=${s#*-}; branch="lane/$slug"          # session claude-<slug> → lane/<slug>
          gate=$(mktemp)
          if vibe-review "$branch" > "$gate" 2>&1; then
            printf '🟢 %s — finished; gate PASS → ready to merge ❓ (ask the user before merging)\n' "$s"
            grep -A4 '^diffstat:' "$gate" | sed 's/^/   /'
          else
            printf '🔴 %s — gate FAIL; feeding failures back\n' "$s"
            grep -E '^\s+FAIL|RESULT:' "$gate" | sed 's/^/   /'
            # Self-heal: hand control back to the agent (single line — paste-buffer
            # turns newlines into Enter presses, which would submit prematurely).
            vibe send "$s" "vibe-review failed on $branch — fix the reported syntax errors, re-run the tests, and continue toward the goal."
          fi
          rm -f "$gate"
        else
          printf '🟢 %s — idle (%s → idle)\n' "$s" "${was:-?}"
        fi;;
      *)
        printf '🔴 %s — %s\n' "$s" "$st";;
    esac
  done < "$now"

  mv "$now" "$STATE"
  sleep "$POLL"
done
```

## Rules (inherited from agent-manager)
- **Never go silent.** Poll every 60s; surface prompts and gate results unprompted ([Prime directive](../../agent-manager/SKILL.md)).
- **Drive to the goal, escalate every real decision.** Self-heal the mechanical; the user decides anything irreversible or taste-driven — **especially merging**.
- **One lane = one git worktree.** Never run parallel lanes in a shared tree; they clobber each other.
- **≤3 deliverables per lane.** More work = more lanes, not a bigger prompt.
- **Agents never self-merge `main`.** Lanes pass the gate, then the merge captain integrates on your go-ahead, one at a time.
