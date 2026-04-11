---
title: "launchd Service Management"
date: 2026-04-09
draft: false
tags: [learning]
---


**Date:** 2026-04-09
**Context:** Auto-restart nanobot gateway on crash

## What is launchd?

It's macOS's service manager — PID 1, always running, manages everything from your WiFi to your cloud sync. Think of it as macOS's built-in systemd but actually sane.

## The Setup

We registered nanobot gateway as a **User Agent** (`~/Library/LaunchAgents/`) with `KeepAlive: true`. This means: if the gateway crashes, launchd automatically restarts it. Beautiful.

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
```

## The Environment Variable Trap

launchd does **NOT** source `.zshrc`, `.bashrc`, or `.profile`. None of your carefully configured PATH entries, API keys, or aliases. Nothing.

This bit us hard — the gateway would start via launchd but couldn't find Python, Node, or any env vars. 

**Fix**: a wrapper script that sources everything:
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
```

Important distinction:
- `kill <pid>` = bounce (KeepAlive restarts it within seconds)
- `launchctl unload` = off switch (stays dead until you load again)

## The 10-Hour Outage (How to Do It Wrong)

Here's our cautionary tale. We wanted to restart the gateway, so we ran:

```bash
launchctl unload ~/Library/LaunchAgents/com.nanobot.gateway.plist && \
launchctl load ~/Library/LaunchAgents/com.nanobot.gateway.plist
```

Seems fine, right? WRONG. The laptop went to sleep between the `unload` and the `load`. The gateway was unloaded but never reloaded. It stayed dead. For 10 hours. Through the entire night. Tweezer woke up to a cold, lifeless bot. 💀

## The Safe Way

Just kill the process. KeepAlive handles the rest:

```bash
pid=$(pgrep -f "nanobot gateway")
kill $pid
```

Only use `launchctl unload` when you intentionally want it OFF (like during maintenance). Never combine unload+load in one command — that gap between them is a reliability hazard.

## Debugging

```bash
launchctl list | grep nanobot

tail -f ~/.nanobot/logs/gateway.stderr.log
```

If EXIT is nonzero, it crashed and launchd is either restarting it or has given up. The logs will tell you why.
