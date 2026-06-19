---
title: "Keeping services observable"
date: 2026-06-14
tags: [systems, observability, backend]
excerpt: "A practical baseline for logs, metrics, and traces that pays off the first time something breaks."
---

You don't appreciate observability until production is on fire and you're
staring at a dashboard that tells you nothing. Here's the minimal baseline I
set up on every service.

<!--more-->

## The three signals

- **Logs** — structured, with a request/correlation ID on every line.
- **Metrics** — request rate, error rate, and latency percentiles (p50/p95/p99).
- **Traces** — so you can follow one request across service boundaries.

## Make logs structured

Plain text logs are fine until you need to search them. Emit JSON instead:

```json
{ "level": "error", "msg": "db timeout", "request_id": "abc123", "ms": 5021 }
```

Now you can filter by `request_id` and reconstruct exactly what happened during
a single request.

## Alert on symptoms, not causes

Alert on what users feel — elevated error rate or latency — not on every
internal hiccup. Cause-based alerts are noisy and train you to ignore them.

Set this up before you need it. The first incident is the worst time to
discover you're flying blind.
