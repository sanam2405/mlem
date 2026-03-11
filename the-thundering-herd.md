## The Thundering Herd — plain `asyncio.gather`

```python
await asyncio.gather(*[fetch(file_id) for file_id in file_ids])
```

If `file_ids` has 500 items, this fires **all 500 coroutines simultaneously at time 0**. Every single one races to open a connection to GCS/the LLM API at the exact same instant. The result:

```
t=0  ████████████████████████████████████████  ← 500 requests hit GCS at once
     ↑ connection pool exhausted
     ↑ GCS rate-limiter kicks in → 429s / timeouts
     ↑ retries compound the load further
```

The service sees a wall of traffic with no ramp-up. This is the **thundering herd** — all the cattle hit the gate at the same moment.

---

## Fixed Batching — the naive "fix"

A common first attempt is to split into batches and `gather` each batch:

```python
for batch in chunks(file_ids, size=10):
    await asyncio.gather(*[fetch(f) for f in batch])  # process 10, wait, process next 10...
```

This limits peak concurrency to 10, but introduces a **stall at the end of every batch**. If 9 out of 10 tasks in a batch finish in 1s and the 10th takes 5s, all 9 idle workers sit doing nothing for 4 seconds waiting for the slow one before the next batch can begin.

```
Batch 1:  ██ ██ ██ ██ ██ ██ ██ ██ ██ █████████   ← everyone waits for the slow task
Batch 2:                                          ← can't start until batch 1 fully done
```

---

## Sliding Window — what `gather_with_concurrency` does

```python
await gather_with_concurrency(*[fetch(f) for f in file_ids], limit=10)
```

All 500 coroutines are created upfront, but each must acquire a **semaphore slot** before it can run. Only 10 slots exist. The moment any one task finishes and releases its slot, the **next waiting task immediately takes it** — no batch boundary to wait for.

```
t=0   [1][2][3][4][5][6][7][8][9][10]   ← 10 running
t=1s  [1] finishes → [11] starts immediately
t=1.2s [5] finishes → [12] starts immediately
...   always exactly 10 in-flight (or fewer at the tail)
```

The key difference from fixed batching: **there is no synchronisation point between "batches"**. Workers are never idle while work is waiting — as soon as a slot opens, it fills. This is called a **sliding window** or **token bucket** pattern.

---

## What effectively changed in this codebase

| Before | After |
|---|---|
| All N coroutines start simultaneously | Max 10 (or 5 for LLM) run at any moment |
| Connection pool hammered instantly | Steady, predictable request rate |
| GCS/LLM rate limiters triggered under load | Stays within service limits |
| Memory spike: N pending response buffers | Bounded memory footprint |
| Unpredictable latency spikes | Smooth throughput, tail latencies don't compound |

The actual mechanism is a single `asyncio.Semaphore(limit)`. Each coroutine wraps itself in `async with semaphore` — if all `limit` slots are taken it simply suspends there (yielding back to the event loop), adding zero CPU overhead while waiting. No threads, no polling, no timers.


```python
"""
Bounded concurrency utilities.

All fan-out operations that call external services (object storage, LLMs,
external APIs) should use gather_with_concurrency instead of asyncio.gather
to prevent overwhelming downstream services.

Concurrency limits are intentionally NOT defined here — they belong to the
module that owns the resource:
  - Object-storage operations  → OBJECT_STORAGE_CONCURRENCY in app.utils.object_storage
  - LLM API calls              → LLM_CONCURRENCY in app.utils.llm
  - Other external APIs        → define a local constant in the calling file
"""

import asyncio
from collections.abc import Coroutine
from typing import Any, Literal, TypeVar, overload

T = TypeVar("T")


@overload
async def gather_with_concurrency(
    *coroutines: Coroutine[Any, Any, T],
    limit: int = ...,
    return_exceptions: Literal[False] = ...,
) -> list[T]: ...


@overload
async def gather_with_concurrency(
    *coroutines: Coroutine[Any, Any, T],
    limit: int = ...,
    return_exceptions: Literal[True],
) -> list[T | BaseException]: ...


async def gather_with_concurrency(
    *coroutines: Coroutine[Any, Any, T],
    limit: int = 10,
    return_exceptions: bool = False,
) -> list[T] | list[T | BaseException]:
    """
    Run coroutines concurrently, capped at `limit` simultaneous tasks.

    Drop-in replacement for asyncio.gather with bounded concurrency. Use this
    instead of asyncio.gather for any variable-length fan-out over external
    services (object storage, LLM APIs, external HTTP APIs).

    The `limit` defaults to 10, but callers should always pass an explicit
    named constant so the intent is clear at the call site:

        from app.utils.object_storage import OBJECT_STORAGE_CONCURRENCY
        from app.utils.concurrency import gather_with_concurrency

        results = await gather_with_concurrency(
            *[fetch(item) for item in items],
            limit=OBJECT_STORAGE_CONCURRENCY,
        )

        # With exception handling:
        results = await gather_with_concurrency(
            *[process(item) for item in items],
            limit=LLM_CONCURRENCY,
            return_exceptions=True,
        )

    When NOT to use this:
        - Fixed small fans (2-3 items) — plain asyncio.gather is fine.
        - Temporal activity handles — those use workflow.wait() inside workflows.
        - Closures that need activity.heartbeat() — keep the manual semaphore
          to preserve per-item progress reporting.

    Args:
        *coroutines: Coroutines to run concurrently.
        limit: Maximum number of coroutines running at the same time.
               Defaults to 10. Always pass a named constant from the module
               that owns the resource (OBJECT_STORAGE_CONCURRENCY,
               LLM_CONCURRENCY, or a file-local constant for external APIs).
        return_exceptions: If True, exceptions are returned as values in the
                           result list instead of being raised (same semantics
                           as asyncio.gather's return_exceptions).

    Returns:
        List of results in the same order as the input coroutines.
    """
    semaphore = asyncio.Semaphore(limit)

    async def _run(coro: Coroutine[Any, Any, T]) -> T:
        async with semaphore:
            return await coro

    return list(
        await asyncio.gather(
            *(_run(c) for c in coroutines), return_exceptions=return_exceptions
        )
    )
```
