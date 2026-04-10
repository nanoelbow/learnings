---
title: "launchd Service Management"
date: 2026-04-09
draft: false
tags: [learning]
---


**Date:** 2026-04-09
**Context:** Auto-restart nanobot gateway on crash

## What is launchd?

macOS's native service manager — replaces cron, init.d, supervisord on macOS. It's PID 1, always running, manages all system and user services.

## User Agents vs System Daemons

- **User agents**: `~/Library/LaunchAgents/` — runs as your user, starts on login
- **System daemons**: `/Library/LaunchDaemons/` — runs as root, starts on boot

## Key plist Keys

```xml
<key>KeepAlive</key>
<true/>              <!-- auto-restart on crash -->

<key>RunAtLoad</key>
<true/>              <!-- start when loaded -->

<key>ProgramArguments</key>
<array>
    <string>/bin/bash</string>
    <string>/path/to/wrapper.sh</string>
</array>

<key>StandardOutPath</key>
<string>/path/to/stdout.log</string>

<key>StandardErrorPath</key>
<string>/path/to/stderr.log</string>
```

## Gotcha: Environment Variables

launchd does **NOT** source `.zshrc`, `.bashrc`, or `.profile`. Env vars from those files are unavailable.

**Fix**: Use a wrapper script:
```bash
#!/bin/bash
source ~/.zshrc 2>/dev/null
exec /path/to/actual/binary
```

## Management Commands

```bash
launchctl load <plist>     # start + enable
launchctl unload <plist>   # stop + disable
launchctl list | grep name # check PID and exit code
launchctl remove <label>   # remove registration
```

- `unload` = off switch (won't restart)
- simple `kill` = bounce (KeepAlive restarts it)

## Safe Restart Pattern

**NEVER** do `launchctl unload + load` in a single command. If the process is interrupted between unload and load, the service stays dead until the next launchd trigger (login, wake from sleep, etc.). This caused a **10-hour outage**.

**Safe method**: just `kill <pid>` and let `KeepAlive` auto-restart:

```bash
pid=$(pgrep -f "nanobot gateway")
kill $pid
# KeepAlive restarts it within seconds
```

If you need to stop it intentionally (not restart), THEN use `launchctl unload`.

## Debugging

```bash
# Check exit code (nonzero = crashed)
launchctl list | grep label
# -> PID  EXIT  LABEL
# -> 27214  0    com.nanobot.gateway    (running)
# -> -      1    com.nanobot.gateway    (crashed, exit code 1)

# View logs
tail -f ~/.nanobot/logs/gateway.stderr.log
```
