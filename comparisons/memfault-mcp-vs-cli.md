---
title: Memfault MCP vs Memfault CLI — Capabilities Comparison
created: 2026-05-29
updated: 2026-05-29
type: comparison
tags: [memfault, mcp, tools, ota, debugging, ncs]
sources: [https://app.memfault.com/mcp]
confidence: high
---

# Memfault MCP vs Memfault CLI — Capabilities Comparison

Two interfaces exist for Memfault from an AI agent perspective: the **Memfault MCP server**
(`app.memfault.com/mcp`, `mcp_memfault_*` tools) and the **Memfault CLI** (Python
`memfault` package, orchestrated by the `chsh-ag-memfault` subagent). They are
complementary — not interchangeable.

**TL;DR:** MCP = read-only observability (query fleet, decode crashes). CLI = write
operations (upload symbols, create releases, deploy OTA). A full release workflow
requires both.

See also: [mcp-nordic-mcp-tools](../concepts/mcp-nordic-mcp-tools.md) ·
[memfault-version-requirements](../concepts/memfault-version-requirements.md) ·
[cursor-skills-and-agents](../concepts/cursor-skills-and-agents.md)

---

## Capability Matrix

| Capability | MCP (`mcp_memfault_*`) | CLI (`chsh-ag-memfault`) |
|------------|------------------------|--------------------------|
| List projects | ✅ `projects_list` | ✅ |
| Query device state | ✅ `device_get` | ✅ |
| Search fleet (SQL filter) | ✅ `device_search` | ❌ (no SQL fleet query) |
| Get device attributes | ✅ `device_getAttributes` | ❌ |
| List device reboots | ✅ `device_listReboots` | ❌ |
| Get crash trace (stacktrace + logs) | ✅ `trace_get` | ❌ |
| Get issue details | ✅ `issue_get` | ❌ |
| List metric keys | ✅ `metrics_list` | ❌ |
| **Upload `.elf` symbols** | ❌ | ✅ Workflow A |
| **Create software version** | ❌ | ✅ Workflow B |
| **Upload OTA payload (`.signed.bin`)** | ❌ | ✅ Workflow B |
| **Deploy release to cohort** | ❌ | ✅ Workflow B |
| **Disable/abort active deployment** | ❌ | ✅ Workflow C |

---

## Memfault MCP Tool Reference

All tools require `project` (slug, e.g. `nrf-test`). Default project: `nrf-test`.

### `mcp_memfault_projects_list`
Lists all projects accessible to the authenticated user. Use to confirm the correct
project slug before any operation.

### `mcp_memfault_device_get`
Fetches full device state: cohort assignment, current software version, hardware
version, last seen timestamp, developer mode flag.

### `mcp_memfault_device_search`
Fleet-wide SQL `WHERE` clause query. The most powerful MCP tool — supports:
- Filter by `software_version`, `cohort`, `hardware_version`, `last_seen`, `first_seen`
- Filter by custom attribute: `attribute.'key' = 'value'`
- Filter by timeseries aggregate: `max(timeseries.'uptime/uptime') > 3600`
- `LIKE` on `device_serial` and `nickname`

**Example queries:**
```sql
-- Devices still on old version after OTA deploy
software_version != '3.3.0.2' AND cohort = 'default'

-- Devices seen in last 24h
last_seen > '2026-05-28T00:00:00Z'

-- Specific device serial prefix
device_serial LIKE 'MFLT%'
```

> **Important:** Call `mcp_memfault_metrics_list` first to discover exact attribute/
> timeseries key names — they are project-specific and not guessable.

### `mcp_memfault_device_getAttributes`
Returns the most recent attribute metric values for a device (e.g. firmware tag,
board type, custom properties set by the firmware).

### `mcp_memfault_device_listReboots`
Returns paginated reboot history for a device: timestamp, reason (human-readable),
software version at time of reboot, reboot type (memfault vs android). Supports
time-range filtering and sort direction.

**Useful for:** post-OTA verification (unexpected reboots?), debugging boot loops.

### `mcp_memfault_trace_get`
Fetches a crash trace with:
- Faulting thread stacktrace
- Fault analysis (fault reason, registers, memory context)
- `include_logs: true` — log entries leading up to the crash
- `include_all_threads: true` — all thread stacktraces

This tool can decode a crash directly in the conversation once symbols are uploaded
via CLI Workflow A — no separate symbol server lookup needed.

### `mcp_memfault_issue_get`
Fetches a crash issue by ID: title, status, occurrences, affected devices, linked
traces, software versions affected.

### `mcp_memfault_metrics_list`
Lists all metric definitions for a project. Returns both:
- **Attribute** metrics: most-recent device state (searchable, filterable)
- **Timeseries** metrics: time-series heartbeat values (aggregatable in `device_search`)

Always call this before using `attribute.'key'` or `timeseries.'key'` in `device_search`.

---

## CLI Workflows (via `chsh-ag-memfault`)

Managed by the `chsh-sk-memfault` skill and `chsh-ag-memfault` subagent.

| Workflow | What it does |
|----------|-------------|
| A — Symbol upload | Uploads `.elf` to Memfault for crash decoding; optionally ties to a software version |
| B — OTA release | Creates software version + uploads OTA payload + deploys to cohort |
| C — Abort deployment | Disables an active OTA deployment to stop rollout |

The CLI uses the `memfault` Python package and requires API key credentials configured
in `overlay-app-memfault-project-info.conf`.

---

## Combined Workflow: OTA Release + Verification

The MCP and CLI complement each other in a release cycle:

```
1. CLI Workflow B   →  upload symbols + OTA payload, deploy to cohort
2. mcp device_search  →  `software_version != '<new>' AND cohort = 'default'`
                          (how many devices are still pending update?)
3. wait / re-check
4. mcp device_search  →  all devices now on new version
5. mcp device_listReboots  →  any unexpected reboots on updated devices?
6. mcp trace_get (if issues)  →  decode crash with freshly uploaded symbols
```

This loop replaces the manual "open Memfault UI → check fleet → drill into device"
cycle, keeping the full verification within the agent conversation.

---

## Crash Debug Workflow: MCP-first

When a crash issue ID is known:

```
1. mcp issue_get(issue_id)         →  overview, affected versions
2. mcp trace_get(trace_id,         →  stacktrace + pre-crash logs
      include_logs=True)
3. Decode in conversation          →  symbols already uploaded via CLI Workflow A
4. CLI Workflow A (if needed)      →  re-upload symbols for current build
```

This is faster than the Memfault UI for crash triage during active development.

---

## Limitations of the MCP Server

- **Read-only:** No write operations. Cannot modify device state, create resources,
  or trigger deployments.
- **No upload endpoint:** Cannot upload `.elf` symbols, OTA chunks, or coredumps.
- **No cohort management:** Cannot move devices between cohorts.
- **Project-scoped:** Each tool call requires the project slug. The `terr-project`
  credentials differ from `nord-project` — verify the slug before querying.
- **Pagination:** `device_search` and `device_listReboots` paginate via cursor. For
  large fleets, loop with `cursor=nextCursor` until no cursor is returned.
