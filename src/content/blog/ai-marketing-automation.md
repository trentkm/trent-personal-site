---
title: "Building an AI Marketing Machine on a Mac Mini"
date: 2026-03-01
description: "How I automated Pinterest marketing and blog writing for my side project with Claude Code, cron jobs, and a self-learning feedback loop — all running on a Mac Mini on my desk."
tags: ["claude-code", "automation", "marketing", "mac-mini", "side-project"]
---

I haven't thought about Pinterest marketing in days, and it's still happening. Pins are being generated, blog drafts are appearing as pull requests, and my phone buzzes with a preview every time something's ready to review. The whole thing runs on a Mac Mini sitting on my desk.

This is how I built it — in a single day, with Claude Code.

## The Problem

[Waymarked](https://www.waymarked.com/) turns travel photos into vintage-style map prints. I'd been manually creating Pinterest pins — picking destinations, choosing map styles, writing descriptions, exporting images, posting. It worked, but it ate 2-3 hours a week and was easy to put off. I also knew I needed a blog for SEO, but the idea of writing two posts a week on top of everything else wasn't happening.

I started by researching OpenClaw, an AI agent platform I'd seen people using for automated marketing. The deep dive was revealing. The "self-learning" feature turned out to be structured markdown files — a learnings doc that the agent reads at the start of every task and updates based on outcomes. Nothing proprietary. The security story was worse — 20% of skills in OpenClaw's public registry had been flagged as potentially malicious.

The conclusion: build it myself. The self-learning pattern is literally just files that Claude reads and writes. No framework needed.

## Iteration 1: Cron + Claude Code CLI

The simplest possible thing. Cron jobs on the Mac Mini that run `claude -p` — Claude Code's headless mode — with detailed prompts. Generate 3 pins Monday and Thursday at 9am. Pull Pinterest analytics daily. Review performance weekly.

Each script follows the same pattern: set PATH, log start time, run Claude with a prompt, extract a summary, send a notification, log end time. Nothing clever. Just shell scripts and cron.

## Iteration 2: Notifications

The first version had no way to tell me when things happened. I set up ntfy — a free, open-source push notification service. One curl command sends a push notification to my phone. I added hooks to Claude Code's settings so it notifies me when it finishes a task or needs attention.

The scripts extract a `SUMMARY:` line from Claude's output and include it in the notification. So instead of "Pin generation complete," I get "Hungary (pure-terrain-borders), Jordan (ember-atlas), Chile (nordic-frost)." Actual information, not status updates.

## Iteration 3: GitHub as the Review Surface

The original plan was to review everything through Claude Code's remote control feature on my phone. It works, but it's not great for visual content. You can't really review map images through a text interface.

The breakthrough was using GitHub as the review surface. Pins become GitHub Issues with images embedded inline — the PNGs get compressed from ~13MB to ~300KB JPEGs, uploaded to Cloudflare R2, and rendered directly in the issue body. Blog posts become pull requests. The post is the diff. Merge to publish, close to reject.

This works perfectly on the GitHub mobile app. Tap the ntfy notification, see the content, make a decision.

## Iteration 4: Instant Processing with Cloudflare Tunnel

Initially, review processing ran on a daily 5pm cron job. But why wait? I set up a Cloudflare Tunnel from the Mac Mini to `webhook.waymarked.com` and configured a GitHub webhook to POST when issues close.

The flow: close an issue, GitHub POSTs to the tunnel, a small Node.js server verifies the webhook signature, writes a trigger file, macOS launchd detects the file change, runs the processing script in a proper login shell, Claude reads my comments, posts approved pins to Pinterest, logs rejections.

The launchd WatchPaths trick was born from necessity. Claude Code's CLI needs a proper TTY environment that you don't get when spawning from a Node.js process. Using launchd as an intermediary with `bash -l` gives it the login shell it needs. Instant event-driven automation without building a queue system.

## Iteration 5: Self-Learning

The learning system is embarrassingly simple. There's a `learnings.md` file with three sections:

- **Promoted Rules** — hard constraints. "Always use gift_for angle" because 5 of 9 posted pins used it.
- **Observations** — soft patterns. "Watercolor markers + poster bar combo appears in 5 successful posts."
- **Rejection Log** — table of what was rejected and why.

Claude reads this file at the start of every generation. When the same rejection reason appears 3 times, it gets promoted to a rule. A weekly review cron job analyzes Pinterest analytics and updates the insights. Over time, the system converges on what works.

The "AI" part is just that Claude can read natural language instructions and follow them. No vector databases, no fine-tuning, no inference infrastructure. A markdown file.

## What Runs Today

```
         8am              9am              10am
Mon  [analytics]   [3 pinterest pins]   [weekly review]
Tue  [analytics]   [blog draft]
Wed  [analytics]
Thu  [analytics]   [3 pinterest pins]
Fri  [analytics]   [blog draft]
Sat  [analytics]
Sun  [analytics]

Pin review processing happens instantly via webhook.
```

Four always-on launchd services: Next.js dev server for Playwright map rendering, Cloudflare Tunnel, webhook server, and a WatchPaths trigger for review processing.

## The Stack

Everything runs on a single Mac Mini on my desk:

- **Claude Code** — `claude -p` for headless, `--dangerously-skip-permissions` for unattended file writes
- **Cron** for scheduling
- **launchd** for always-on services
- **ntfy** for push notifications
- **GitHub Issues/PRs** for visual review
- **Cloudflare Tunnel** for webhooks without exposing ports
- **Cloudflare R2** for hosting pin preview images
- **Playwright** for headless map screenshot capture

Total monthly cost: included in the Claude Max plan I already pay for. The R2 storage for preview images is pennies.

## What I Learned

**Start with the dumbest thing that works.** The first version was literally cron + `claude -p`. No webhook, no GitHub integration, no image uploads. Each iteration solved a real pain point, not a hypothetical one.

**Claude Code's headless mode is underrated.** `claude -p "your prompt here"` turns Claude into a unix tool. Pipe it, cron it, chain it. The `--dangerously-skip-permissions` flag lets it write files without prompts — essential for unattended operation.

**launchd WatchPaths is the macOS secret weapon.** Instead of polling, you tell macOS "run this script when this file changes." Combined with a webhook server that writes trigger files, you get instant event-driven automation without building a queue system.

**GitHub is a surprisingly good review interface.** Issues render images inline. PRs render markdown. The mobile app works great. Comments become a natural approve/reject mechanism. And you get a complete audit trail for free.

**The self-learning loop doesn't need AI infrastructure.** It's a markdown file. Claude reads it, Claude writes it. Rejections become observations, observations become rules.

## What's Next

The system has been running for a few weeks now. Pins are being generated and posted, blog drafts are appearing as PRs, and the learnings file is growing. The weekly review has started analyzing actual Pinterest analytics data.

The immediate improvements I'm thinking about — companion Pinterest pins for each blog post, A/B testing different description angles for the same destination, and automated scheduling of approved pins across the week instead of batch posting.

But the best part is that marketing went from a task I dreaded to something that just happens.
