---
title: "Atomic File Writes"
date: 2026-04-09
draft: false
tags: [learning]
---


**Date:** 2026-04-09
**Context:** nanobot session persistence bug

## Problem

Writing files with `open(path, "w")` is **not atomic**. The OS truncates the file first, then writes content. If the process is killed mid-write (e.g. SIGKILL, crash), the file is left truncated — all data lost.

This caused nanobot's `SessionManager.save()` to corrupt session JSONL files on gateway kill, losing all conversation history.

## Solution: Temp File + Atomic Rename

```python
import os, tempfile

fd, tmp_path = tempfile.mkstemp(dir=target_dir, suffix=".tmp")
try:
    with os.fdopen(fd, "w") as f:
        f.write(data)
        f.flush()
        os.fsync(f.fileno())  # ensure data hits disk
    os.replace(tmp_path, target_path)  # atomic on POSIX
except BaseException:
    os.unlink(tmp_path)  # cleanup on failure
    raise
```

## Key Points

- `os.replace()` is atomic on the same filesystem — either the old file exists OR the new file exists, never a half-written state
- `tempfile.mkstemp(dir=...)` — use same directory to guarantee same filesystem
- `os.fsync()` ensures data is flushed to disk before rename
- On crash: old file stays intact, temp file is orphaned garbage
- On success: old file is replaced atomically

## When to Use

- Session/data files that are written periodically
- Any file where partial writes = data loss
- Config files that might be written during signal handling
