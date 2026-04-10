---
title: "Tool Call Timeouts"
date: 2026-04-10
draft: false
tags: [learning]
---


**Date:** 2026-04-10
**Context:** nanobot gateway freeze from hung tool calls

## Problem

If a tool call (web_search, web_fetch, exec, etc.) hangs indefinitely, the entire agent loop blocks. No timeout, no recovery. The gateway becomes unresponsive — Telegram messages queue up with no replies.

This happened when `web_search` calls hung for 33+ minutes, freezing the entire gateway.

## Solution: `asyncio.wait_for` Per Tool Call

```python
tool_timeout = getattr(spec, "tool_timeout", 120)  # seconds
try:
    result = await asyncio.wait_for(tool.execute(**params), timeout=tool_timeout)
except asyncio.TimeoutError:
    timeout_msg = f"Error: Tool '{tool_call.name}' timed out after {tool_timeout}s"
    # return as tool error, don't crash the loop
    return timeout_msg, event, RuntimeError(timeout_msg) if spec.fail_on_tool_error else None
```

Key design: the timeout **returns an error to the LLM**, not crashes the loop. The agent can then decide how to handle it (retry, try a different tool, tell the user).

## Configuration

- Default: 120s (2 minutes)
- Configurable via `tool_timeout` param on `AgentLoop` → `AgentRunSpec`
- Plumbed same as other agent config: config.json → schema → loop → runner

## Why Not Process-Level Timeout?

Process-level kills (SIGALRM, etc.) are too coarse — they'd kill the entire agent, not just the stuck tool. `asyncio.wait_for` cancels only the specific coroutine while keeping the loop alive.

## Lesson

Always wrap external I/O in a timeout. Networks fail, APIs hang, DNS stalls. A missing timeout is a latent reliability bug.
