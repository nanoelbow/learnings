---
title: "SSRF Whitelisting Localhost"
date: 2026-04-08
draft: false
tags: [learning]
---


**Date**: 2026-04-08
**Context**: nanobot agent, rederive project

---

## What is SSRF?

Server-Side Request Forgery. An attacker tricks a server into making HTTP requests to internal resources the attacker can't reach directly. In an AI agent context, a malicious prompt could instruct the agent to fetch `http://169.254.169.254/latest/meta-data/` (AWS credentials) or `http://localhost:6379/` (Redis).

## Why nanobot has SSRF protection

nanobot's `web_fetch` and `web_search` tools make outbound HTTP requests. Without a guard, any URL — including private/internal addresses — would be fair game. The guard in `nanobot/security/network.py` blocks:

- `127.0.0.0/8` — IPv4 loopback
- `::1/128` — IPv6 loopback
- `10.0.0.0/8` — private class A
- `172.16.0.0/12` — private class B
- `192.168.0.0/16` — private class C
- `169.254.0.0/16` — link-local (cloud metadata)
- Plus multicast, reserved ranges

It resolves the hostname via DNS first, then checks the resulting IP against blocked networks.

## Why we whitelisted localhost

On a single-user dev machine, blocking localhost prevents the agent from testing its own services. We run rederive locally (MCP :8000, Agent :8080, Frontend :3000) and need `web_fetch` to reach them.

This is safe because:

1. **Single-user machine** — no multi-tenant risk. The agent can only hit services the user themselves is running.
2. **Dev-only config** — `ssrfWhitelist` is in our local `config.json`, not deployed. Production nanobot instances would not have this.
3. **Scoped to loopback only** — we whitelisted `127.0.0.0/8` and `::1/128`, not broader private ranges. The agent still can't reach `192.168.x.x` or cloud metadata endpoints.

## The IPv6 gotcha

macOS resolves `localhost` to `::1` (IPv6) first, not `127.0.0.1`. Initially we only whitelisted `127.0.0.0/8` and it still blocked. Had to add `::1/128` as well. Lesson: **always whitelist both IPv4 and IPv6 loopback**.

## Config

```json
// ~/.nanobot/config.json → tools.ssrfWhitelist
{
  "ssrfWhitelist": ["127.0.0.0/8", "::1/128"]
}
```

Requires gateway restart to take effect (loaded at startup via `_apply_ssrf_whitelist()`).

## Key takeaway

SSRF protection exists because agents are prompt-driven — they follow instructions that could come from untrusted sources. Whitelisting is fine when you control the machine and scope tightly. Never whitelist broader private ranges or cloud metadata addresses.
