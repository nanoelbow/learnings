---
title: "LLM Model Fallback Chains"
date: 2026-04-09
draft: false
tags: [learning]
---


**Date:** 2026-04-09
**Context:** nanobot model fallback when GLM-5.1 overloads

## Problem

LLM providers have rate limits, outages, and overload errors. If your primary model is unavailable, the entire request fails — user gets nothing.

## Solution: Ordered Fallback List

Configure a list of models to try in order when the primary fails:

```
primary: glm-5.1
fallback: [glm-5-turbo, glm-4.5-air]
```

On error from primary → try glm-5-turbo → try glm-4.5-air → return last error if all fail.

## Implementation Pattern

```python
async def _run(self, spec, messages, ...):
    response = await self._call_provider(spec, messages, model=spec.model)

    if response.finish_reason == "error" and spec.model_fallback:
        for fallback in spec.model_fallback:
            if fallback == spec.model:
                continue  # skip duplicates
            fallback_resp = await self._call_provider(spec, messages, model=fallback)
            if fallback_resp.finish_reason != "error":
                return fallback_resp
            response = fallback_resp  # keep last error

    return response
```

## Design Decisions

- **Same provider, different models** — simpler than cross-provider fallback (same API key, same auth)
- **Skip primary if duplicated** in fallback list (avoid retrying same failed model)
- **Return last error** if all models fail — gives user a real error message, not silence
- **Plumb through all layers** — config schema → app → loop → runner → CLI, so config drives behavior everywhere
- **Config-driven** — `model_fallback: ["glm-5-turbo", "glm-4.5-air"]` in config, not hardcoded

## Layer Plumbing

```
config.json → schema.py (AgentDefaults.model_fallback)
           → nanobot.py (passes to AgentLoop)
           → loop.py (stores, passes to AgentRunSpec)
           → runner.py (AgentRunSpec.model_fallback, tries on error)
           → commands.py (serve/gateway/agent all pass through)
```
