---
title: "Claude Code CLI Orchestration via --print Mode"
date: 2026-04-11
draft: false
tags: [learning]
---



Using nanobot to drive Claude Code programmatically as a coding sub-agent.

## Problem

Claude Code is a powerful coding agent, but it runs as an interactive REPL. We needed a way for nanobot (running on Telegram) to delegate coding tasks to Claude Code and get results back — without human interaction at the terminal.

## Solution: `--print` Mode

Claude Code's `-p` flag runs a single non-interactive turn:

```bash
claude -p "implement feature X" \
  --output-format text \
  --max-turns 5 \
  --allowedTools "Write,Edit,Bash"
```

Key flags:
- `--max-turns N` — how many rounds of tool use (1 for text-only, 3-5 for file ops, 10+ for complex tasks)
- `--allowedTools` — restrict which tools Claude Code can use (`Write`, `Edit`, `Bash`, `Glob`, `Grep`)
- `--output-format text` — plain text response (also supports `json`)

## Session Persistence

The breakthrough: `--name` and `--resume` enable multi-turn context:

```bash
claude -p "create calc.py with a Calculator class" --name "calc-task" --max-turns 3

claude -p "add a divide method with zero handling" --resume "calc-task" --max-turns 3

claude -p "write unit tests in test_calc.py" --resume "calc-task" --max-turns 3
```

Claude Code remembers all previous turns in the session — files created, decisions made, code written.

## The Orchestration Pattern

Nanobot acts as the orchestrator:

1. **User tells nanobot** what to build (via Telegram)
2. **Nanobot crafts prompts** with project context (from MEMORY.md, architecture docs)
3. **Runs `claude -p`** with scoped tools and appropriate turns
4. **Reviews output** — reads changed files, runs tests
5. **Iterates** via `--resume` if needed
6. **Reports back** to user with results

This gives us: human → nanobot (orchestrator) → Claude Code (coder), with review at each step.

## Node 22 SSL Fix (macOS)

Gotcha: Node v22 on macOS can't verify cert chains for some HTTPS endpoints, causing `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` on every `claude` call.

Fix: export macOS system root certs and point Node to them:

```bash
security find-certificate -a -p /System/Library/Keychains/SystemRootCertificates.keychain > /tmp/macos-system-certs.pem
security find-certificate -a -p /Library/Keychains/System.keychain >> /tmp/macos-system-certs.pem
security find-certificate -a -p ~/Library/Keychains/login.keychain-db >> /tmp/macos-system-certs.pem
```

Then add to `~/.zshrc` and `~/.bash_profile`:
```bash
export NODE_EXTRA_CA_CERTS=/tmp/macos-system-certs.pem
```

This fixed `claude login`, `claude -p`, and all other HTTPS calls from Node 22.

## Auth: Subscription vs API Credits

Two ways to authenticate:

1. **`ANTHROPIC_API_KEY` env var** — pay-per-token API credits
2. **`claude login`** — uses Claude Max/Pro subscription plan

We use option 2 so Claude Code usage counts against the subscription, not API credits. Login is a one-time interactive step (opens browser for OAuth).

## Security Considerations

- **Scope `--allowedTools`** — only grant what the task needs. A read-only task should get `Glob,Grep`, not `Bash`
- **Review before commit** — always read Claude Code's output before accepting changes
- **No credentials in prompts** — never pass API keys or secrets via `-p`
- **Unique session names** — prevents accidentally resuming wrong sessions

## ClawHub Lessons

We also explored ClawHub (third-party skill registry) and decided against installing any third-party skills. The reasoning:

- **Self-evolve skill** grants agent full authority to modify its own config without confirmation — prompt injection risk
- **Security audit** of third-party skills requires reading every script — not scalable
- **Build in-house** gives full control and code knowledge

Decision: build skills ourselves when needed. ClawHub CLI was installed then uninstalled.

## Outcome

Created a `claude-code` skill at `skills/claude-code/SKILL.md` documenting the orchestration patterns, session management, and security guidelines. Nanobot can now delegate coding tasks to Claude Code via Telegram with full review capability.
