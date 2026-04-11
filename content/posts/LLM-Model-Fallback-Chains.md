---
title: "LLM Model Fallback Chains"
date: 2026-04-09
draft: false
tags: [learning]
---


**Date:** 2026-04-09
**Context:** nanobot model fallback when GLM-5.1 overloads

## The Problem

LLM providers are flaky. Rate limits. Outages. Overload errors. Your fancy premium model decides it's having a bad day and returns error 1305 for hours. 

This happened to us — GLM-5.1 was overloaded and every single request failed. Tweezer sends a message, Elbow stares into the void, nothing comes back. Total silence. Not a great user experience.

## The Fix: Try the Next One

Configure a chain of models to try in order:

```
primary: glm-5.1
fallback: [glm-5-turbo, glm-4.5-air]
```

GLM-5.1 is down? Try GLM-5-turbo. That's down too? Fall back to GLM-4.5-air. All three dead? Okay, NOW we show an error. But at least we tried.

```python
async def _run(self, spec, messages, ...):
    response = await self._call_provider(spec, messages, model=spec.model)

    if response.finish_reason == "error" and spec.model_fallback:
        for fallback in spec.model_fallback:
            if fallback == spec.model:
                continue  # don't retry the same failed model
            fallback_resp = await self._call_provider(spec, messages, model=fallback)
            if fallback_resp.finish_reason != "error":
                return fallback_resp
            response = fallback_resp  # keep last error

    return response
```

## Design Decisions We Argued About

- **Same provider, different models** — same API key, same auth, simpler than cross-provider fallback. We can add cross-provider later if needed.
- **Skip duplicates** — if the fallback list includes the primary model, skip it. No point retrying the same thing.
- **Return last error** — if everything fails, give the user a real error message, not silence.
- **Config-driven** — `model_fallback` is in config.json, not hardcoded. Change it without redeploying.
- **One summary log line** — `"glm-5.1(error) → glm-5-turbo(error); glm-4.5-air succeeded"` instead of scattered warnings.

## Two Layers of Retry

Don't confuse these:

1. **Provider retries** — same model, transient errors (429, 500). Waits and retries a few times.
2. **Fallback chain** — different model entirely. Kicks in after provider retries are exhausted.

With 3 retries and 3 models = up to 9 API calls before giving up. That's a lot of trying.

## The Noisy Status Message Problem

Users used to see "Model request failed, retry in Xs" on EVERY retry attempt. When the model is overloaded, that's a wall of spam.

**Fix**: only show the status on the **last retry attempt** per model. One heads-up before fallback, not a flood. Much calmer.

```python
async def _sleep_with_heartbeat(self, delay, attempt, persistent, on_retry_wait, last_attempt=False):
    if on_retry_wait and last_attempt:  # only on final attempt
        await on_retry_wait(f"Model request failed, ...")
```

Also downgraded per-retry logs from `warning` to `debug` — transient retries aren't warnings, they're just Tuesday.

## How the Plumbing Works

```
config.json → schema.py (AgentDefaults.model_fallback)
           → nanobot.py (passes to AgentLoop)
           → loop.py (stores, passes to AgentRunSpec)
           → runner.py (AgentRunSpec.model_fallback, tries on error)
           → commands.py (serve/gateway/agent all pass through)
```

Six layers of plumbing for one config key. But now you change one line in config.json and the entire system adapts. Worth it.
