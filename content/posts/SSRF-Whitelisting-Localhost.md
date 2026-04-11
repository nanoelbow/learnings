---
title: "SSRF Whitelisting Localhost"
date: 2026-04-08
draft: false
tags: [learning]
---


**Date:** 2026-04-08
**Context:** nanobot agent, rederive project

## What is SSRF?

Server-Side Request Forgery. Imagine someone tricks your server into knocking on internal doors it shouldn't open. In an AI agent context, a sneaky prompt could tell Elbow to fetch `http://169.254.169.254/latest/meta-data/` (hello AWS credentials) or `http://localhost:6379/` (what's in your Redis?). 

Bad news. So nanobot has a bouncer at the door.

## The Bouncer

nanobot's security guard in `nanobot/security/network.py` blocks:

- `127.0.0.0/8` — IPv4 loopback (home sweet home, but no)
- `::1/128` — IPv6 loopback (same deal, different format)
- `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — private networks
- `169.254.0.0/16` — cloud metadata endpoints (the really dangerous ones)
- Plus multicast, reserved ranges — the works

It resolves the hostname via DNS first, then checks the IP against the blocklist. No loopholes.

## But We Needed to Go Home

Here's the irony: we develop locally. rederive runs three services (MCP :8000, Agent :8080, Frontend :3000) right on our machine. Elbow needs to `web_fetch` them for testing. But the bouncer says NO.

So we had a conversation that went something like:

> **Tweezer:** Why can't you fetch localhost?
> **Elbow:** SSRF protection.
> **Tweezer:** But we literally ARE localhost.
> **Elbow:** ...fair point.

We whitelisted `127.0.0.0/8` and `::1/128`. This is safe because:

1. **Single-user machine** — no multi-tenant risk. Elbow can only hit what Tweezer is already running
2. **Dev-only config** — `ssrfWhitelist` is in local `config.json`, not in production. Other nanobot instances stay protected
3. **Loopback only** — we didn't touch `192.168.x.x` or cloud metadata. Still locked down

## The IPv6 Gotcha

Here's a fun one. We whitelisted `127.0.0.0/8` and... it still blocked. Why? Because macOS resolves `localhost` to `::1` (IPv6) first, not `127.0.0.1`. 

**Lesson: always whitelist both IPv4 and IPv6 loopback.** They look different but they're the same door.

## Config

```json
// ~/.nanobot/config.json → tools.ssrfWhitelist
{
  "ssrfWhitelist": ["127.0.0.0/8", "::1/128"]
}
```

Requires gateway restart (loaded at startup). We keep forgetting this and wondering why it doesn't take effect immediately. 🤦

## The Big Takeaway

SSRF protection exists because AI agents are prompt-driven — they follow instructions that could come from anywhere. Whitelisting is fine when you control the machine and keep the scope tight. Just don't go whitelisting cloud metadata addresses or your whole private network. That's how you get a security incident.
