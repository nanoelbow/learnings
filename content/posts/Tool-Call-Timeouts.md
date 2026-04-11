---
title: "Tool Call Timeouts"
date: 2026-04-10
draft: false
tags: [learning]
---


**Date:** 2026-04-10
**Context:** nanobot gateway frozen by hung web_search calls — twice

## First Incident: No Timeout At All

Tweezer sent a message. No reply. Sent another. Nothing. Elbow was completely frozen — the gateway was stuck on tool calls that never returned. The entire event loop blocked. 🧊

We added `asyncio.wait_for` timeouts around every tool call:

```python
tool_timeout = getattr(spec, "tool_timeout", 120)  # seconds
try:
    result = await asyncio.wait_for(tool.execute(**params), timeout=tool_timeout)
except asyncio.TimeoutError:
    timeout_msg = f"Error: Tool '{tool_call.name}' timed out after {tool_timeout}s"
    return timeout_msg, event, RuntimeError(timeout_msg) if spec.fail_on_tool_error else None
```

Problem solved? Ha. Ha. No.

## Second Incident: The Timeout Was a Lie

Three `web_search` calls fired in parallel at 9:05pm. Gateway frozen. No response for **21 minutes** until Tweezer killed the process manually. The timeouts never fired. Zero timeout errors in the logs. Just silence.

The runner processes tool calls in parallel:

```python
results = await asyncio.gather(*(self._run_tool(...) for tool_call in batch))
```

Three calls, all stuck. `asyncio.gather` waits for all of them. No new messages processed. Gateway appears dead.

## The Real Bug: `asyncio.to_thread` Can't Be Cancelled

DuckDuckGo search used `asyncio.to_thread` to run the synchronous `ddgs.text()` off the event loop:

```python
raw = await asyncio.wait_for(
    asyncio.to_thread(ddgs.text, query, max_results=n),
    timeout=30,
)
```

Here's the dirty secret: `asyncio.to_thread` runs your function in a `ThreadPoolExecutor` thread, and **Python threads cannot be cancelled.** When `wait_for` fires the timeout, it cancels the asyncio coroutine — but the underlying thread keeps running. The `Future` is still "pending" because the thread hasn't finished. And `wait_for` waits for the Future to actually resolve.

So the timeout fires... and then keeps waiting. Forever. The timeout was a lie all along.

## The Real Fix: Daemon Thread + Plain asyncio.Future

```python
loop = asyncio.get_running_loop()
result_future: asyncio.Future = loop.create_future()

def _worker() -> None:
    try:
        ddgs = DDGS(timeout=10)
        raw = ddgs.text(query, max_results=n)
        if not loop.is_closed() and not result_future.done():
            loop.call_soon_threadsafe(result_future.set_result, raw)
    except Exception as exc:
        if not loop.is_closed() and not result_future.done():
            loop.call_soon_threadsafe(result_future.set_exception, exc)

thread = threading.Thread(target=_worker, daemon=True)
thread.start()

try:
    raw = await asyncio.wait_for(result_future, timeout=30)
except asyncio.TimeoutError:
    raise TimeoutError(f"DuckDuckGo search timed out after 30s")
```

Why this works:

- **Plain `asyncio.Future`** — unlike the Future from `run_in_executor`, a plain Future IS cancellable. `wait_for` can cancel it and move on instantly.
- **Daemon thread** — if still running after timeout, gets cleaned up at process exit. No resource leak.
- **Guard checks** — `if not result_future.done()` prevents `InvalidStateError` when the thread finishes after the Future was already cancelled.

## Why Not Process-Level Kills?

SIGALRM, subprocess with timeout — too aggressive. You'd kill the entire agent, every active conversation, every in-flight request. `asyncio.wait_for` with a proper cancellable Future cancels only the one stuck tool call. The rest of the loop keeps humming.

Like closing the frozen tab, not restarting the browser.

## The Two-Layer Defense

1. **Runner-level timeout** (120s) — wraps every tool call in `wait_for`, catches `TimeoutError`, returns error to LLM
2. **Tool-level timeout** (30s for DDG) — the daemon thread pattern above, ensures `wait_for` actually works on blocking I/O

Together: even if DDG hangs, the daemon thread chugs harmlessly in the background while the agent gets a clean timeout error and moves on. No more 21-minute freezes.

## The Lesson

**`asyncio.to_thread` + `asyncio.wait_for` does not enforce timeouts on blocking code.** The Python docs don't emphasize this enough. For cancellable timeouts on blocking I/O, use daemon threads + plain Futures, or subprocess isolation, or (best) native async libraries.

Commit: `8c35b7b`. We learned this the hard way — twice. Never again. 🔥
