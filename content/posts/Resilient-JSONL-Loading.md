---
title: "Resilient JSONL Loading"
date: 2026-04-09
draft: false
tags: [learning]
---


**Date:** 2026-04-09
**Context:** nanobot session recovery after corruption

## Problem

When loading a JSONL (JSON Lines) file, a single corrupted line causes `json.loads()` to throw and the entire load fails — returning `None` / empty state, losing all valid data.

## Solution: Skip Bad Lines

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
        continue  # skip, don't abort
    messages.append(data)

if bad_lines:
    logger.warning("Recovered {} messages, {} corrupted lines", len(messages), bad_lines)
```

## Key Points

- JSONL is append-only by nature — early lines are older, later lines are newer
- If corruption happens at the end (most common — killed mid-write), all older data is still valid
- Even if corruption is in the middle, surrounding data is recoverable
- Return `None` only if there's zero recoverable data (no messages AND no metadata)
- Log corruption count for monitoring — frequent corruption signals a deeper write bug

## Pattern: Defense in Depth

Pair with atomic writes for full protection:
1. **Atomic writes** prevent corruption in the first place
2. **Resilient loading** recovers data if corruption somehow still occurs
