---
title: Claude Code — Configuration File Locations
created: 2026-06-02
updated: 2026-06-02
type: concept
tags: [claude-code, mcp, config, guide, reference]
sources: []
confidence: high
---

# Claude Code — Configuration File Locations

Six files control Claude Code's behavior at global and project scope. Understanding
where each piece of config lives matters for backup, dotfiles tracking, and team setup.

## File Map

| File | Scope | VCS? | What it controls |
|------|-------|------|-----------------|
| `~/.claude.json` | Global + per-project | Backup only | MCPs, project-scoped tool permissions, auth, UI state |
| `~/.claude/settings.json` | Global | Commit | Model, theme, permissions, hooks, verbose |
| `~/.claude/CLAUDE.md` | Global | Commit | Behavioral instructions for every session |
| `<project>/.claude/settings.json` | Project | Commit | Project-level permissions, hooks, env vars |
| `<project>/.claude/settings.local.json` | Project | Gitignored | Local-only permission grants |
| `<project>/CLAUDE.md` | Project | Commit | Per-repo instructions, build commands, architecture |

---

## `~/.claude.json` — The Most Important File to Back Up

Large JSON blob (~750 lines). **Not** human-edited — the app manages it entirely.

Top-level `mcpServers`: global MCPs active for all projects.

`projects["<abs-path>"]` entries: per-project data keyed by the workspace root path.
Each entry carries its own `mcpServers` (project-scoped MCPs added via the UI),
`allowedTools` (tool permission grants), and session statistics.

```
~/.claude.json
  .mcpServers              ← global MCPs (e.g. "gitnexus", "codegraph")
  .projects["/path/to/ws"]
    .mcpServers            ← project-scoped MCPs (e.g. "serena" for v3.0.2)
    .allowedTools          ← granted tool permissions for that workspace
```

**Restore strategy:** Keep a versioned backup (e.g. `cp ~/.claude.json ~/.dotfiles/claude.json`).
On a new machine, copy it in before first launch. MCP server definitions in particular
are stored *only* here — they cannot be recovered from any other file.

---

## `~/.claude/settings.json` — Global Behavioral Settings

Small, human-readable, version-control friendly. Safe to commit to a dotfiles repo.

```json
{
  "permissions": { "defaultMode": "default" },
  "model": "sonnet",
  "theme": "auto",
  "verbose": true
}
```

Controls defaults that apply to every project: model selection, permission mode, hooks.

---

## `~/.claude/CLAUDE.md` — Global Instructions

Loaded into every Claude Code session regardless of project. Use it for:
- Behavioral guidelines that apply everywhere
- Personal coding style preferences
- Cross-project skill/wiki references

---

## `<project>/.claude/settings.json` — Project Settings (Committed)

Per-project overrides for permissions, hooks, and env vars. Committed to the repo,
so teammates get the same defaults. Typical use: adding `allow` rules for project-specific
tools, configuring hooks that run on file save, or setting env vars for the dev container.

---

## `<project>/.claude/settings.local.json` — Project Local Settings (Gitignored)

Same structure as `settings.json`. Excluded from git by convention (`settings.local.json`
is in Claude Code's default gitignore). Use it for:
- `allow` grants you don't want to impose on teammates
- Local path overrides
- Credentials or tokens that must not be committed

Example from this workspace:
```json
{
  "permissions": {
    "allow": [
      "Read(//Users/chsh/.claude/**)",
      "Read(//Users/chsh/**)",
      "Bash(python3 *)"
    ]
  }
}
```

---

## `<project>/CLAUDE.md` — Project Instructions (Committed)

Loaded for sessions in that workspace. Use it for:
- Build and flash commands
- Board targets and toolchain notes
- Architecture overview, key files, module map

For this workspace the file is at `/opt/nordic/ncs/v3.3.0/CLAUDE.md`.

---

## Restore Checklist (New Machine)

1. Copy `~/.claude.json` — restores all MCPs and project-scoped permissions.
2. Copy `~/.claude/settings.json` — restores model/theme/permissions defaults.
3. Copy `~/.claude/CLAUDE.md` — restores global instructions.
4. `<project>/.claude/settings.json` and `<project>/CLAUDE.md` come from git clone.
5. Recreate `<project>/.claude/settings.local.json` manually (gitignored, machine-specific).

---

## Related Pages

- [mcp-nordic-mcp-tools](mcp-nordic-mcp-tools.md) — MCP server tools and configuration patterns
- [cursor-skills-and-agents](cursor-skills-and-agents.md) — Skills vs agents: design reference
