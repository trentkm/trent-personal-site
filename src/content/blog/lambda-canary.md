---
title: "Catching Failures Before Users Do: A Canary for Lambda WebGL Rendering"
date: 2026-03-01
description: "How I built a canary that exercises my headless Chrome + WebGL Lambda renderer every 6 hours, rotating through all 27 map styles."
tags: ["aws", "lambda", "cloudwatch", "monitoring", "side-project"]
---

In my [first post](/blog/building-waymarked), I described the moment mobile broke everything. Mobile browsers crashed trying to capture high-resolution map canvases, so I moved rendering to a Lambda running headless Chrome with SwiftShader for software WebGL. A 10GB Docker container that loads a full React app, renders a MapLibre map, and captures an 8K canvas. It works. But it's fragile in ways that normal Lambdas aren't.

The problem: I only found out it was broken when a real user's render failed and hit the dead letter queue. By then, someone had already paid for a map and was waiting for an email that wasn't coming.

## Why This Lambda Is Different

Most Lambdas are straightforward — take input, call some APIs, return output. This one boots a full headless browser, initializes a WebGL context through software emulation, loads a Next.js page on Vercel, fetches vector tiles from Cloudflare R2 via PMTiles range requests, renders a MapLibre GL map, waits for custom marker and QR code overlays via Konva, then captures a canvas. Any one of those steps can fail silently.

And the failure modes aren't obvious. Chrome might start but SwiftShader might not initialize. The page might load but tiles might timeout. Tiles might load but the `__RENDER_READY__` signal might never fire. Everything might work for 26 of 27 styles but one style loads 16 external GeoJSON files from GitHub and takes three minutes.

I needed something that would catch these failures before a customer did.

## The Canary

The idea is simple. Every 6 hours, an EventBridge rule drops a message onto the SQS queue with `{ canary: true }`. The Lambda picks it up and, instead of rendering a customer's map, it renders a test map — two markers in Paris and Rome, on whichever style the rotation selects.

The style rotation is a single line:

```js
const styleIndex = Math.floor(Date.now() / (6 * 60 * 60 * 1000)) % 27;
```

Each 6-hour window gets a different style. Full cycle completes in about 7 days. This matters because the styles aren't homogeneous — roughly half use only PMTiles and render in 10-20 seconds, while the other half load external GeoJSON sources (bathymetry, ocean labels, graticule lines) and can take 30-60+ seconds.

On success, it emits a `WaymarkedCanary/RenderSuccess` CloudWatch metric. On failure, the Lambda throws, SQS retries, and the message lands in the dead letter queue — which already has an alarm. A separate canary-specific alarm fires if the success metric drops below 1 over a 12-hour window, with missing data treated as breaching. So even if the EventBridge rule itself breaks and nothing runs at all, I still get alerted.

The canary reuses the existing `/dev/generate` page — the same page the real renderer hits, with the same `__RENDER_READY__` and `__captureMap__` signals. It just skips the R2 upload and database write since there's no customer image to store. This means the canary tests the exact same code path as a real render.

## What It Costs

An EventBridge rule. One CloudWatch metric and one alarm. A few seconds of Lambda compute every 6 hours. The canary runs for about a minute on heavy styles, which at 10GB Lambda pricing is roughly $0.01 per run. Four runs a day — $0.04. About $1.20 a month to know my renderer is healthy before my customers tell me it isn't.

## Closing the Mobile Loop

The Lambda renderer exists because mobile browsers can't handle print-quality map rendering. It's the most critical piece of infrastructure in the entire product — without it, mobile users can't get their maps. And mobile users are the majority.

The canary closes a gap that's been there since launch. Before, a Chrome update could break SwiftShader, a Vercel deployment could regress the render page, a tile source could go down, and I wouldn't know until someone's order sat in the queue unanswered. Now I know within 12 hours, before it matters.

The whole thing — EventBridge rule, CloudWatch metric, alarm, style rotation, production access gate — was about 200 lines of code across 6 files. Built in a day with Claude Code, running reliably since.
