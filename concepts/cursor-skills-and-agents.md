---
title: Cursor Skills and Agents — Design Reference
created: 2026-05-13
updated: 2026-05-13
type: concept
tags: [cursor, ai-agent, meta, guide, agent-workflow, reference]
sources: []
confidence: high
---

# Cursor Skills and Agents — Design Reference

Practical reference for designing, authoring, and wiring together Cursor Agent
Skills (`SKILL.md`) and subagents (`.md` files in `~/.claude/agents/`). Synthesized
from hands-on experience building the `chsh-sk-memfault` / `chsh-ag-memfault` pair.

---

## What They Are

### Skill (`SKILL.md`)

A markdown file injected into the **parent agent's context window** as instructions.
The parent agent reads the skill and executes the workflow itself.

- Lives in `~/.claude/skills/<skill-name>/SKILL.md`
- Loaded only when the agent decides it's relevant (via description matching)
- Shares the parent's context — competes with conversation history and other skills
- Best for: lightweight orchestration, build decisions, pre-processing, reference tables

### Agent (`.md` in `~/.claude/agents/`)

A **separate execution context** with its own system prompt, own context window, and
own tool access. The parent delegates the entire task; the subagent runs it autonomously.

- Lives in `~/.claude/agents/<agent-name>.md`
- **Never invoked automatically** — requires explicit trigger (user mention or `Task` tool)
- Has its own fresh context window — does not consume parent tokens
- Best for: multi-step workflows, destructive operations, approval gates, long-running tasks

---

## Comparison Table

| Dimension | Skill | Agent |
|-----------|-------|-------|
| Execution | Parent agent reads + executes | Subagent executes independently |
| Context | Shared with parent | Own isolated context |
| Token cost to parent | Injected on every relevant message | Zero cost to parent context |
| Auto-triggered | Yes (via description match) | No (always explicit) |
| Approval gates | Parent handles them | Subagent handles them |
| State | Parent conversation state | Subagent's own session |
| Best for | Orchestration, setup, light reference | Multi-step pipelines, API workflows |
| Analogous to | A how-to card the parent reads | A specialist you hand the task to |

---

## Invocation Model

### Skills — automatic in theory, explicit in practice

Skills listed in `available_skills` are shown to the agent in every message.
The agent uses the `description` field to decide whether to apply a skill.
**You do not need to type the skill name** — it should fire automatically if
the description contains strong trigger phrases that match your phrasing.

In practice, explicit mention (e.g. `/chsh-sk-memfault`) guarantees the skill
is read regardless of description quality. Good descriptions reduce this need.

**Design rule:** Put the exact phrases you naturally say into the `description`:

```yaml
description: "... Use when uploading symbols, creating an OTA release,
              deploying to a cohort, or disabling active deployments."
```

### Agents — always explicit

Agents in `~/.claude/agents/` are never auto-invoked. Trigger paths:
1. User names the agent (`chsh-ag-memfault`)
2. A skill delegates to it (via `Task` tool)
3. Parent agent autonomously decides to spawn it

---

## Skill → Agent Delegation Pattern

A skill can act as a dispatcher: do pre-processing work, then spawn an agent.

```
User → parent agent reads skill → skill does setup → skill spawns agent → agent executes
```

**When the wrapper skill is justified:**

The skill must do meaningful work before delegating. Test:
> "Does the skill do anything beyond relaying the task to the agent?"

| Skill | Pre-work before delegating | Wrapper justified? |
|-------|---------------------------|-------------------|
| `chsh-sk-memfault` | Determines workflow (A/B/C), runs firmware build if needed | ✅ Yes |
| `chsh-sk-git` | Minimal — mostly relays context | ⚠️ Borderline |

If the skill does nothing but say "call the agent," it adds token cost with no benefit.
The agent's own `description` field serves the same discovery purpose without a wrapper.

### Delegation handoff pattern

The skill spawns the agent using the `Task` tool with a clear task description:

```
Invoke subagent: chsh-ag-memfault
Task: "Run Workflow B for version 3.3.0.1 — artifacts are already built."
```

The agent receives the task, not the conversation history. Pass all context
explicitly (version, workflow letter, pre-conditions) in the task string.

---

## Agent Design Rules (learned from `chsh-ag-memfault`)

1. **Hard rules first.** Lead with what the agent must never do (delete without approval, build without being asked). This is the first thing LLMs read and conditions the whole session.

2. **Step 0 always.** Start every agent session with credential/context loading. Never assume env vars are set.

3. **Use `AskQuestion` for all destructive ops.** Delete deployment, delete release, push to remote — all need explicit `AskQuestion` approval, not just text confirmation.

4. **Scope boundaries.** Be explicit about what the agent does NOT do (`Do not build firmware. Do not modify source files. Do not push to git.`). This prevents scope creep when the agent sees adjacent problems.

5. **API pitfall notes inline.** Document discovered API quirks in the agent itself (e.g., "GET /releases/{VER} does NOT return an `activations` field — use GET /deployments"). This prevents re-discovering the same bugs.

6. **Model selection.** Use fast/haiku models for pattern-matching work (git commits). Use mid-tier models for multi-step decision logic with branching (Memfault workflows). Heavy reasoning models are overkill for both.

---

## Skill Design Rules (from review of `chsh-sk-memfault`)

1. **≤ 500 lines in SKILL.md.** If longer, extract detail into sub-files or an agent.

2. **Self-Update Policy section required.** Without it, the skill silently rots after the first session.

3. **Related Skills table required.** Skills that interact with other skills (e.g., memfault skill needs `chsh-sk-ncs-env` for builds) must cross-reference them.

4. **`name` must match directory name.** Renaming the directory via file manager breaks discovery unless SKILL.md frontmatter is also updated.

5. **Description = trigger phrases.** Write the description as the phrases the user will actually say. "Use when uploading .elf symbols, creating an OTA release..." not "Handles Memfault operations."

---

## File Locations

```
~/.claude/
├── skills/
│   └── chsh-sk-memfault/
│       └── SKILL.md          # dispatcher skill (116 lines)
└── agents/
    ├── chsh-ag-git.md         # git commit+push specialist
    └── chsh-ag-memfault.md    # Memfault upload/deploy specialist (273 lines)
```

---

## Related Pages

- [mcp-nordic-mcp-tools](mcp-nordic-mcp-tools.md) — MCP tools pattern (complementary invocation mechanism)
- [hermes-architecture](hermes-architecture.md) — Hermes agent framework that hosts skills
- [eedp-platform](eedp-platform.md) — broader embedded AI agent platform context
