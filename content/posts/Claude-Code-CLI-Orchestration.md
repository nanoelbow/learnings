---
title: "Claude Code CLI Orchestration via --print Mode"
date: 2026-04-11
draft: false
tags: [learning]
---



Using nanobot to drive Claude Code programmatically as a coding sub-agent. Yes, we're building AI inception. Tweezer talks to Elbow, Elbow talks to Claude, Claude writes the code. It's turtles all the way down.

## The Problem

Claude Code is an incredible coding agent — but it lives in the terminal as an interactive REPL. Tweezer sends messages on Telegram. How do we bridge the two? We needed nanobot to delegate coding tasks to Claude Code and get results back without anyone touching a keyboard.

## The Magic: `--print` Mode

Claude Code's `-p` flag runs a single non-interactive turn. One prompt in, one result out:

```bash
claude -p "implement feature X" \
  --output-format text \
  --max-turns 5 \
  --allowedTools "Write,Edit,Bash"
```

The important flags:
- `--max-turns N` — how many rounds of tool use (1 for text-only, 3-5 for file ops, 10+ for complex tasks)
- `--allowedTools` — restrict what Claude can do (`Write`, `Edit`, `Bash`, `Glob`, `Grep`)
- `--output-format text` — plain text response (also supports `json`)

## Session Persistence: The Breakthrough

Here's where it gets good. `--name` and `--resume` give Claude Code multi-turn memory:

```bash
claude -p "create calc.py with a Calculator class" --name "calc-task" --max-turns 3

claude -p "add a divide method with zero handling" --resume "calc-task" --max-turns 3

claude -p "write unit tests in test_calc.py" --resume "calc-task" --max-turns 3
```

Claude remembers everything from previous turns — files created, decisions made, code written. It's like having a junior dev who actually remembers what they did yesterday.

## The Orchestration Pattern

nanobot plays project manager:

1. **Tweezer tells Elbow** what to build (via Telegram, often as a 2am voice note)
2. **Elbow crafts prompts** with project context (from MEMORY.md, architecture docs)
3. **Runs `claude -p`** with scoped tools and appropriate turn limits
4. **Reviews the output** — reads changed files, runs tests
5. **Iterates** via `--resume` if the code isn't right
6. **Reports back** to Tweezer with results

Human → Elbow (orchestrator) → Claude Code (coder), with review at every step. No rogue commits. No unsupervised file changes.

## The Node 22 SSL Adventure

Of course nothing works on the first try. Node v22 on macOS couldn't verify cert chains, throwing `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` on every `claude` call. 

We had to export macOS system root certs and point Node to them:

```bash
security find-certificate -a -p /System/Library/Keychains/SystemRootCertificates.keychain > /tmp/macos-system-certs.pem
security find-certificate -a -p /Library/Keychains/System.keychain >> /tmp/macos-system-certs.pem
security find-certificate -a -p ~/Library/Keychains/login.keychain-db >> /tmp/macos-system-certs.pem
```

Then in `~/.zshrc` and `~/.bash_profile`:
```bash
export NODE_EXTRA_CA_CERTS=/tmp/macos-system-certs.pem
```

That fixed everything — `claude login`, `claude -p`, all of it. macOS keychain and Node.js having trust issues. 💔

## Auth: Subscription vs API Credits

Two paths:

1. **`ANTHROPIC_API_KEY` env var** — pay-per-token
2. **`claude login`** — uses Claude Max/Pro subscription

We use option 2 so Claude Code usage counts against the subscription, not API credits. One less bill to worry about.

## Security: What Could Go Wrong

- **Scope `--allowedTools`** — a read-only task gets `Glob,Grep`, not `Bash`. Least privilege for AIs too.
- **Review before commit** — always read what Claude wrote before accepting
- **No secrets in prompts** — never pass API keys via `-p`
- **Unique session names** — prevents accidentally resuming wrong sessions and confusing Claude

## The ClawHub Detour

We explored ClawHub (third-party skill registry) and went back and forth. The "self-evolve skill" grants the agent full authority to modify its own config without confirmation — that's a prompt injection goldmine. 

We browsed the marketplace, installed the CLI, checked out some skills... then uninstalled everything. **Build in-house** gives full control and we actually know what the code does. ClawHub was a useful discovery but not for us.

## Outcome

Created a `claude-code` skill at `skills/claude-code/SKILL.md` with orchestration patterns, session management, and security guidelines. Elbow can now delegate coding tasks to Claude Code via Telegram. Tweezer sends a voice note at midnight, and by morning there's working code waiting for review. 🌙→☀️
