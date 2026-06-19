---
title: "Idempotency keys in practice"
date: 2026-06-18
tags: [backend, api, systems]
excerpt: "How to make retried requests safe without double-charging anyone."
---

Networks fail. Clients retry. If your write endpoints aren't idempotent, a retry
can create two orders, send two emails, or charge a card twice. Idempotency keys
fix this with very little machinery.

<!--more-->

## The idea

The client generates a unique key per logical operation and sends it with the
request:

```http
POST /payments
Idempotency-Key: 8f14e45f- ...
```

The server stores the result of the first request under that key. Any retry with
the same key returns the stored result instead of doing the work again.

## A minimal implementation

1. On request, look up the key.
2. If found, return the saved response.
3. If not, process the request, then save `(key -> response)` atomically.

The atomic step matters: store the key and the side effect in the same
transaction, or you'll have gaps where a crash leaves the work done but the key
unsaved.

## Don't forget expiry

Keys don't need to live forever. A 24-hour TTL is usually plenty to cover client
retry windows without growing the table unbounded.

Small pattern, big payoff — it turns "did that go through?" from a support
ticket into a non-event.
