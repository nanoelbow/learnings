---
title: "Resilient JSONL Loading"
date: 2026-04-09
draft: false
tags: [learning]
---


**Date:** 2026-04-09
**Context:** nanobot session recovery after corruption

## What Happened

After the atomic file writes fix (see that post), we still had corruption — from the *old* bug, before the fix. Half-written JSONL files with garbled final lines. The loader would hit a bad line, `json.loads()` would throw, and the entire session would load as `None`. 

All the valid data — hundreds of lines of good conversation history — thrown away because of one bad apple. 😤

## The Fix: Skip Bad Lines, Keep Everything Else

```python
messages = []
bad_lines = 0

for line in file:
    line = line.strip()
    if not line:
        continue
    try:
        data = json.loads(line)
    except json.JSONDecodeError:
        bad_lines += 1
        continue  # skip it, don't abort
    messages.append(data)

if bad_lines:
    logger.warning("Recovered {} messages, {} corrupted lines", len(messages), bad_lines)
```

Simple. Instead of failing fast and returning nothing, we just... skip the broken line and keep going. 

## Why This Works

- JSONL is append-only — early lines are old, later lines are new
- Corruption almost always hits the **end** of the file (killed mid-write) — all the older data is perfectly fine
- Even mid-file corruption only loses that one line, not everything around it
- Return `None` only when there's literally zero recoverable data
- Log the corruption count — if it keeps happening, something deeper is broken

## The Full Picture

This pairs with atomic writes for defense in depth:

1. **Atomic writes** prevent corruption from happening in the first place
2. **Resilient loading** recovers data if corruption somehow still gets through

Together they mean: even if everything goes wrong, you lose at most one line. Not the whole file. Not the whole session. Just one line.

We went from "kill gateway = lose all conversations" to "kill gateway = maybe lose the last message." Much better. 🎉
