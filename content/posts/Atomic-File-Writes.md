---
title: "Atomic File Writes"
date: 2026-04-09
draft: false
tags: [learning]
---


**Date:** 2026-04-09
**Context:** nanobot session persistence bug

## What Happened

Tweezer killed the nanobot gateway. Normal thing — happens all the time when you're iterating. Except this time, every single conversation session file got wiped to zero bytes. All of Elbow's conversation history. Gone. Poof. 💨

We stared at the empty files for a good minute before realizing what happened.

## The Villain: `open(path, "w")`

Here's the thing about `open(path, "w")` — it's not atomic. The OS **truncates the file first** (empties it), then writes the new content. If someone pulls the plug between those two steps — SIGKILL, crash, power loss, Tweezer's itchy trigger finger — the file is left as a sad empty shell with all data lost.

nanobot's `SessionManager.save()` was doing exactly this. Every save was a game of Russian roulette.

## The Fix: Temp File + Atomic Rename

The pattern is simple and beautiful:

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

The magic is `os.replace()` — on the same filesystem, it's **atomic**. The old file exists OR the new file exists. Never a half-written state. Never an empty file. POSIX guarantees it.

## The Checklist

- `os.replace()` is atomic on the same filesystem — use `tempfile.mkstemp(dir=...)` with the same directory
- `os.fsync()` ensures data actually hits disk before the rename (OS write caches lie to you)
- On crash: old file stays intact, temp file is orphaned garbage
- On success: old file is replaced atomically, no in-between state

## When You Need This

- Session/data files written periodically (every save is a risk window)
- Any file where partial writes = data loss
- Config files that might be written during signal handling

Basically: if losing the file would make you sad, use atomic writes. We learned this the hard way. Don't be like us. 😅
