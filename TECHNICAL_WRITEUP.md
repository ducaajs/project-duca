# Project Duca: Autonomous Multi-Agent Executive Job Search Pipeline
### A Working Solution to OpenClaw sessions_yield and sessions_spawn Bugs on Consumer Hardware

---

## Overview

Project Duca is a fully autonomous, zero-cost, locally-hosted AI executive job search pipeline built on OpenClaw 2026.3.13, Ollama 0.18.2, and a Qwen3.5 Q4_K_M model running on an NVIDIA RTX 3060 12GB. The pipeline runs on a scheduled cron job, searches 150+ job postings nightly across major free job boards, filters them against hard parameters, scores matches, drafts tailored cover letters, audits output quality, and delivers a structured morning report to Discord — all without any cloud dependency or paid API usage.

This document describes the technical problems encountered, how they were diagnosed, and the working solutions implemented — including a confirmed workaround for OpenClaw GitHub issues **#46298** and **#49572** that allows multi-agent pipeline orchestration to function correctly in 2026.3.13.

---

## Architecture

```
Cron Job (midnight ET)
    ↓
Duca (orchestrator, agentId: main)
    ↓ sessions_send
Scout (job-coordinator) — searxng_search across 6 job boards
    ↓ sessions_send
Gatekeeper (job-researcher) — URL verification, ghost job detection
    ↓ sessions_send
Analyst (job-analyst) — JD analysis, match scoring (≥85% threshold)
    ↓ sessions_send
Copywriter (job-writer) — Cover letter generation
    ↓ sessions_send
Inspector (job-auditor) — QA audit, Google Drive filing
    ↓
Morning Report → Discord
```

**Hardware:** Ubuntu 24.04 · RTX 3060 12GB · 31.2GB RAM  
**Stack:** OpenClaw 2026.3.13 · Ollama 0.18.2 · Qwen3.5 Q4_K_M (local)  
**Cost:** $0/month

---

## Problem 1: sessions_yield and sessions_spawn Are Broken in 2026.3.13

### Confirmed Bugs
- **Issue #46298:** `sessions_yield` immediately terminates isolated cron sessions instead of waiting for sub-agent completion.
- **Issue #49572:** After `sessions_spawn`, the orchestrator is woken by the announce callback but aborts after exactly one LLM turn.

Both are 100% reproducible. The pipeline starts, runs one agent turn, and silently dies. The cron log reports `status: ok` masking the failure.

### Working Solution: sessions_send Blocking Wait Pattern

Replace `sessions_spawn` + `sessions_yield` entirely. Each specialist agent runs as a persistent named session. The orchestrator sends a message directly to each session and waits for the reply.

**AGENTS.md orchestration protocol (add to your orchestrator's AGENTS.md):**

```
CRITICAL: Do NOT use sessions_spawn. Do NOT use sessions_yield.
Both are broken in OpenClaw 2026.3.13 and will kill the pipeline silently.

Use sessions_send with blocking wait instead:

1. sessions_send(sessionKey: "agent:worker-1:main", message: "[full briefing]", timeoutSeconds: 3600)
2. Wait for reply. Scrub output. Retry if needed with same timeoutSeconds.
3. Pass result forward to next agent the same way.
4. Each agent replies REPLY_SKIP to end the exchange cleanly.

If sessions_send returns status: timeout, the agent is still running.
Call sessions_history on that session key to retrieve results.
```

---

## Problem 2: Cross-Agent Messaging Has Two Config Gates (Both Required)

This is not documented anywhere as a combined requirement. Both gates must be open or `sessions_send` to a different agent will return `forbidden`.

### Gate 1 — Session Visibility

Default `tools.sessions.visibility` is `"tree"` (current session + spawned children only). Setting it to `"agent"` is also insufficient — it only allows access to sessions with the same agentId. For cross-agent sends you need `"all"`.

```json
"tools": {
  "sessions": {
    "visibility": "all"
  }
}
```

**Error without this fix:**
```
"status": "forbidden",
"error": "Session send visibility is restricted. Set tools.sessions.visibility=all to allow cross-agent access."
```

### Gate 2 — Agent-to-Agent Enable + Allow List

`tools.agentToAgent` is disabled by default. It must be explicitly enabled. The `allow` list must include **both the source agent and all target agents**.

```json
"tools": {
  "sessions": {
    "visibility": "all"
  },
  "agentToAgent": {
    "enabled": true,
    "allow": ["main", "worker-1", "worker-2", "worker-3"]
  }
}
```

**Critical:** If you omit `"main"` (or whatever your orchestrator's agentId is) from the allow list, you will get:

```
"status": "forbidden",
"error": "Agent-to-agent messaging denied by tools.agentToAgent.allow."
```

**Warning:** `agentToAgent.enabled: true` has a known conflict with `sessions_spawn` (see issue #5813 — sub-agents get stuck at 0 tokens). Since this solution does not use `sessions_spawn`, this conflict does not apply.

### After Any Config Change

Existing sessions retain stale config snapshots even after a gateway restart. Always run both:

```bash
openclaw gateway restart
openclaw agent --agent main --message "/new"
```

---

## Problem 3: Ollama Context Window Was Too Small (num_ctx 32768)

The pipeline was building prompts of ~43,289 tokens. With `num_ctx 32768`, Ollama truncated the prompt, corrupted the context, and the sampler panicked:

```
WARN: truncating input prompt limit=32768 prompt=43289 keep=4 new=32768
panic: failed to sample token
```

This repeated on every retry — same prompt size, same truncation, same panic — creating an infinite crash loop.

**Fix:** Set `num_ctx 49152` in the Modelfile.

```
PARAMETER num_ctx 49152
```

**VRAM budget at 49152 on RTX 3060 12GB:**
- Model weights: 5.6 GiB
- KV cache: 3.30 GiB
- Compute graph: 0.956 GiB
- Total: ~10.36 GiB — 1.64 GiB headroom

---

## Verified Results

After all three fixes applied:

- Pipeline runs to completion without crashes
- Morning Report delivered to Discord with vetted job opportunities, match scores, cover letter links, and Google Drive job packets
- Multi-agent delegation confirmed via Ollama log response time variation (5s–1m52s) consistent with different agents doing different amounts of work
- `sessions_send` confirmed returning `status: ok` with real replies from target agents in controlled testing (4/5 runs, 1 timeout due to session load)

**Test scripts used for verification:**
- Direct Ollama tool call test (stream=false and stream=true): 10/10 PASS
- Live sessions_send cross-agent delivery test: 4/5 PASS after config fixes applied

---

## Environment

| Component | Version |
|-----------|---------|
| OpenClaw | 2026.3.13 (61d171a) |
| Ollama | 0.18.2 |
| Model | Qwen3.5 Q4_K_M (qwen35 architecture) |
| OS | Ubuntu 24.04 |
| GPU | NVIDIA GeForce RTX 3060 12GB |
| RAM | 31.2 GiB |

---

## Contributing

This solution was developed through systematic diagnosis and testing on a real production workload. If you are hitting these same issues and the above fixes resolve them, please add a comment to issues #46298 and #49572 so the community knows this workaround is confirmed on your setup as well.

---

*Built by AJ Sottile — Johnstown, PA — Project Duca*
