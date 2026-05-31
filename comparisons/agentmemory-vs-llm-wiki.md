---
title: AgentMemory vs LLM Wiki — Two Complementary Memory Systems
created: 2026-05-31
updated: 2026-05-31
type: comparison
tags: [pattern]
sources: [session:agentmemory-analysis-2026-05-31]
confidence: high
---

# AgentMemory vs LLM Wiki

## The Problem This Page Solves

You have agentmemory capturing every session across Hermes, Copilot, and Cursor.
You also have this wiki. They serve fundamentally different purposes, and confusing
them leads to both being used badly.

---

## What Each System Stores

| | AgentMemory | LLM Wiki |
|--|-------------|----------|
| **Unit** | Session observation | Synthesized page |
| **Content** | "session d1cdeb42: debugged stack overflow on nRF54LM20DK" | "When WiFi times out on boot, the firmware was rebooting instead of retrying — fix the timeout handler" |
| **Lifetime** | Permanent (confidence-weighted decay) | Permanent + curated |
| **Growth** | Automatic (hooks or MCP calls) | Manual (agent + human curation) |
| **Value** | "What happened when?" | "What should I do next time?" |
| **Analogy** | Activity log / diary | Engineering notebook |

---

## The Core Insight: Sessions ≠ Experience

A session title like *"Fixing Kconfig build errors in Nordic project"* (session f157d027)
tells you nothing about:
- Which Kconfig symbols were deprecated
- Why they were deprecated and what replaced them
- How to detect the same issue in future migrations
- Whether this is NCS-version-specific or permanent

The wiki page [[ncs-build-system]] captures all of that. **The session log just tells
you it happened.**

Human experts don't remember every past incident — they remember the *pattern* extracted
from many incidents. That's what the wiki compiles.

---

## When to Use AgentMemory

- "Did we look at this before?" — use `memory_recall`
- "What was the build command we used last time?" — use `memory_recall`
- "When did we first encounter this board?" — use `memory_recall`
- Capturing in-progress context mid-session — use `memory_save`
- Correlating timing: "did this break after the v3.3.0 upgrade?" — use timeline queries

---

## When to Use the Wiki

- "How do I debug a WiFi reconnect failure?" → [[wifi-debugging-patterns]]
- "What flags does west build need for nRF54LM20DK?" → [[ncs-build-system]]
- "What changed in NCS v3.2.1 → v3.3.0?" → [[ncs-version-migration]]
- Before starting a migration: "what have we learned about NCS migration?" → [[ncs-version-migration]]
- Onboarding a new AI session to domain context: read wiki first

---

## The Flow: AgentMemory Feeds the Wiki

```
Coding sessions
     │
     ▼
AgentMemory (auto-captured)
     │
     │  periodic: "mine sessions for wiki pages"
     ▼
LLM Wiki (curated, synthesized)
     │
     │  at session start: "orient on wiki"
     ▼
AI agent has distilled experience, not just raw log
```

AgentMemory is the raw material. The wiki is the refined product.
You don't use both for the same query — you use agentmemory to find raw events,
wiki to find distilled experience.

---

## AgentMemory Limitations Observed

From analysis of the agentmemory database (2026-05-31):

1. **Auto-imported lessons are low quality.** Many lessons from consolidation are
   fragments ("do not hear back from you soon", "do not actually experience being slow")
   imported from unrelated conversations. Confidence=0.4, no context.

2. **Session summaries are thin.** Most `mem_*` records have only a one-line title and
   a short facts list. The rich debugging detail lives in the raw observation stream,
   which is harder to query.

3. **Insights are mostly about infra.** The `mcp_agentmemory_memory_insight_list` results
   were almost entirely about agentmemory configuration itself (remote vs local server)
   rather than NCS engineering insights.

4. **No cross-referencing.** AgentMemory doesn't know that session d1cdeb42 (WiFi
   reconnect) and session bf04a4eb (provisioner reset) both point to the same
   nRF54LM20DK platform issue.

The wiki provides what agentmemory cannot: synthesis, cross-references, and
"experience" — not just "events."

---

## Practical Recommendation

**Start each new session:**
1. Check wiki first (`SCHEMA.md` + `index.md` + recent `log.md`)
2. Use `memory_recall` only if wiki doesn't cover the specific question

**After resolving a non-trivial problem:**
1. Update the relevant wiki page with what you learned
2. Log the session in `log.md`
3. Let agentmemory handle session-level capture automatically

**Periodically (monthly):**
1. Mine agentmemory sessions for new patterns
2. Create new wiki pages or update existing ones
3. Run wiki lint to find orphans and stale content

---

## Related Pages
- [[ncs-build-system]] — Example of distilled build experience
- [[wifi-debugging-patterns]] — Example of distilled debugging experience
- [[ncs-version-migration]] — Example of distilled migration experience
