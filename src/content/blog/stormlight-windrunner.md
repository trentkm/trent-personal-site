---
title: "Deleting tmux: Building Stormlight and the Terminal Engine Underneath It"
date: 2026-08-17
description: "I built a dashboard for running many coding agents at once on top of tmux, spent two weeks fighting it, then wrote my own terminal session engine and swapped it in — twenty minutes after the first commit."
tags: ["go", "terminal", "pty", "coding-agents", "side-project"]
---

I run a lot of coding agents at once. Claude here, Codex there, three of them in different worktrees of the same repository, one of them stuck waiting on a question I asked twenty minutes ago and never went back to answer.

The problem isn't running them. The problem is that once you're past two, you lose track. Which one finished? Which one is asking me something? Which one has been sitting idle since lunch because it hit a permission prompt? The answer to all of those was buried in some terminal tab I'd have to go find.

So on July 31st I started building [Stormlight](https://stormlight.sh) — a dashboard that keeps every agent in one place, shows me which ones need attention, and lets me step into any of their terminals without hunting for a tab.

The interesting part isn't the dashboard. It's that finishing it meant writing a terminal emulator.

## The Obvious Choice

Agents live in terminals. Something has to own those terminals and keep them alive when the dashboard isn't running — closing my dashboard should not kill an agent that's mid-refactor.

tmux does exactly this, and it's been doing it for twenty years. The first commit hosted agents in tmux, and by day two I was running tmux as a *private appliance* — my own tmux server on my own socket, invisible to whatever tmux the user already had, with server options asserted at boot so a stray `~/.tmux.conf` couldn't break the dashboard.

This worked. For about a week and a half, this worked genuinely well.

## Where It Stopped Working

Stormlight's central pane is called Spanreed — named for the long-distance writing instrument in the books the project is named after. It shows the selected agent's live terminal, and pressing Enter hands your keyboard to that agent byte for byte.

To draw a terminal you don't own, you have to ask tmux what's on the screen. That means `capture-pane`, which gives you back rendered text. It's a screen scrape. Every feature I wanted ran into the same wall:

- **Selecting an agent should be instant.** With a scrape, switching agents means capturing a fresh screen and repainting from text. The terminal's actual state — cursor shape, alternate screen, color modes, mouse tracking — isn't in a scrape, so you approximate it.
- **Resizing should reflow.** tmux owns the reflow, so I'm asking for a new scrape and hoping it matches what the agent thinks it drew.
- **Clickable links, precise selection, exact scrollback.** Each of these was a negotiation with a program that had its own opinions about all three.

The commit log reads like a slow escalation. `Scroll Spanreed into tmux scrollback for live panes`. `Enable allow-passthrough on the appliance server from boot`. `Assert server options on running appliance servers at launch`. `Carry the tally and the waiting queue into the tmux bar`.

Then, on August 9th: **`Let agents outlive the tmux server that holds them`.**

That commit is where the design broke. I was writing code to work around the lifetime of the thing that was supposed to be managing lifetime for me. tmux is a terminal multiplexer for *humans* — panes, layouts, copy mode, keybindings, a status bar. I wanted about fifteen percent of it, and the other eighty-five percent was in my way. I was building a product on top of a UI.

What I actually needed was the engine inside tmux, without tmux.

## Writing the Engine

On August 12th at 3:00 AM I made the first commit to [Windrunner](https://github.com/trentkm/windrunner): *Engine core: sessions on owned PTYs with authoritative emulation.*

The word doing the work there is **authoritative**. Instead of asking another program what's on a screen, Windrunner spawns the process on a PTY it owns and feeds every byte through a real VT emulator ([`charmbracelet/x/vt`](https://github.com/charmbracelet/x), pure Go — no cgo, no WASM). The terminal state isn't an approximation of some other program's screen. It *is* the screen.

That single change collapsed every problem from the previous section:

- A reattach snapshot is an exact serialization of emulator state — screen, scrollback, cursor — not a lossy scrape and not a replay log.
- Programs that query their terminal (cursor position, color support, capabilities) get real answers, because there's a real emulator to answer them.
- Attaching is a snapshot followed by the raw byte stream. Switching agents in the dashboard is switching which stream you're looking at.

The daemon keeps sessions alive across every client. Detach, close the dashboard, come back tomorrow — the agent is still running and its terminal is intact.

Twenty minutes after that first commit, Stormlight was running on it: `Host agents on windrunner: owned PTYs behind the same seams`. The seams had been there the whole time, which is the only reason a swap that large took twenty minutes. The next day, `internal/tmux` was deleted.

## What Falls Out of Owning the Bytes

The parts I'm happiest with are the ones I couldn't have built at all through a scrape.

**Messages go in as bracketed paste.** When you reply to an agent from the dashboard, the text is delivered into its PTY as a bracketed paste — never interpolated into a shell command string. The agent's own CLI sees a paste, exactly as if you'd typed into it.

**Metadata rides in the session.** Windrunner stores an opaque JSON document per session and never reads it. Stormlight puts the task, workspace, and provider in there. Liveness and exit status stay the daemon's own facts and always override whatever the document claims — the roster can't drift from reality, because the sessions *are* the roster.

**Conversations outlive their agents.** Claude and Codex both name every session with an id their own `resume` command accepts. Stormlight records it in an append-only log at `$XDG_STATE_HOME/stormlight/sessions.jsonl`, so deleting an agent hands its session to history rather than erasing it. `H` opens the browser, Enter reopens a months-old conversation as a live agent in the workspace it left.

**Sessions can talk to each other.** `windrunner ls`, `peek <id>`, and `send <id> text...` are a control plane any program with a shell can drive — an agent can discover its peers, read their screens, and prompt them. Speaking is opt-in per session, because writing into a process's stdin is code execution as far as that process is concerned, and every send is audit-logged and attributed. There's also an event stream for lifecycle and idle/busy transitions, so a peer can wait for a session to go quiet instead of polling its screen.

## Two Projects, Not One

I could have kept all of this inside Stormlight. I split it because the two halves have genuinely different jobs, and the split is what keeps either of them honest.

Windrunner doesn't know what an "agent" is. Or a "task", or a "workspace". It's sessions, processes, PTYs, and a metadata bag you define. It never draws anything. Every time I was tempted to teach the engine something about agents, that was the signal it belonged in Stormlight instead.

Stormlight is the opinionated half — the dashboard, the workspace grouping, the attention model that decides an agent is asking you a question versus merely finished, the provider adapters that turn "run Claude in auto mode" into an argv.

The clean version of the lesson: I spent two weeks discovering that the layer I needed didn't exist as a library, only as a feature of a product. So I wrote the library, and now the product on top of it is the easy part.

## Where It Is

Stormlight is 261 commits in, live at [stormlight.sh](https://stormlight.sh) with a demo video, and installable:

```bash
brew install trentkm/stormlight/stormlight
```

Windrunner is early 0.x and deliberately unstable — the API is settling against its first real consumer and will break while it does. Unix only for now, macOS and Linux.

Both are Go, both are MIT, and the daemon ships inside the same `stormlight` binary that runs the dashboard, started on demand over a unix socket. There's nothing else to install and nothing to configure.
