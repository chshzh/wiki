---
title: Porting Commits Between Diverged NCS App Forks (zego-brick Case)
created: 2026-08-26
updated: 2026-08-26
type: concept
tags: [ncs, zego, git, migration, code-review]
sources: [session:cross-repo-commit-review-2026-08-26]
confidence: high
---

# Porting Commits Between Diverged NCS App Forks

## The Scenario

Two forks of the same sample app exist, targeting different NCS SDK versions
(e.g. `nordic-wifi-memfault` on v2.6.4 vs `nordic-wifi-memfault-ncs340` on
v3.4.0). Someone makes a batch of commits on the older fork (bug fixes,
refactors, doc corrections) and asks: "review these N commits, apply to the
other branch/fork if they fit."

**The naive approach (git cherry-pick / manual diff-apply) is often wrong**
if the newer fork has undergone a **brick extraction** — i.e. modules that
used to be local `src/modules/*` files got promoted into a shared library
repo (here: `zego`, pinned via `west.yml`, physically checked out as a
sibling project in the west workspace).

## Why This Breaks Naive Porting

1. **The file may not exist anymore.** `heap_monitor.c`,
   `wifi_prov_over_ble.c`, `net_event_mgmt.c`'s SoftAP code — all moved to
   `zego/bricks/memonitor`, `zego/bricks/wifi_ble_prov`,
   `zego/bricks/network` respectively. A cherry-pick would just fail to
   apply, or worse, silently create a dead local file that shadows nothing.

2. **The shared brick evolves independently, and is often *ahead*.** Because
   `zego` is used by multiple projects, it gets its own bug reports and
   fixes from other consumers. In this case, `zego/bricks/wifi_ble_prov` had
   already fixed the *same* two reconnect-hang bugs the older fork fixed —
   using a more robust guard (`wifi_prov_state_get()` instead of
   `current_conn != NULL`) and additionally handling a third case (explicit
   retry scheduling on failed connect attempts) the older fork's fix didn't
   cover. Always check the brick's own git log / changelog table in its
   `docs/*-spec.md` before assuming a fix is missing.

3. **"Dormant scaffolding" in one fork may be a live feature in the shared
   brick.** The older fork removed SoftAP code as dead (only used STA mode).
   But `zego/bricks/network` is consumed by other apps that *do* use
   SoftAP/P2P_GO — removing it there would be a breaking change for those
   consumers, not a cleanup. Scope removal decisions to "is this dead in the
   file I'm editing", not "is this dead in the one app I'm thinking about."

4. **Kconfig-gated design supersedes hardcoded fixes.** A commit that
   hardcodes "stop logging this periodic line" may be moot if the newer
   fork already made that behavior a `CONFIG_..._LOG_PERIODIC` toggle
   (default off) — strictly better than a code-level removal.

5. **Doc/PRD commits carry project-specific facts that may not transfer.**
   A correction like "nRF7002DK only has 2 buttons/2 LEDs" is true only for
   a single-board, STA-only fork. A dual-board fork (still supporting
   nRF54LM20DK+nRF7002EB2 with real button/LED modules) has different facts
   and the "correction" doesn't apply — check the target's own PRD/specs
   for the same claim before assuming it needs the same fix.

## The Right Workflow

1. `git show --stat <sha>` (or `git log --oneline` across the commit range)
   on the **source** repo to see exactly what each commit touches — don't
   rely on GitHub's web-rendered commit page (huge, table-formatted,
   awkward to parse via web-fetch/markdown conversion; local `git show` is
   far more reliable and 100x faster when you have a local clone).
2. For each touched path, check whether that path **still exists** in the
   target fork. If not, `grep`/`find` for where the equivalent logic now
   lives (check `west.yml` for extracted shared-library projects first).
3. If the logic lives in a shared brick repo now: read the brick's current
   source for the same bug pattern, and its changelog table (zego bricks
   keep one in `docs/<brick>-spec.md`) to see if it was already
   independently fixed — often in a superior way. Don't port backwards.
4. If a shared brick genuinely needs the fix too: that's a **separate
   decision** from "apply to the target app repo" — changing a
   multi-consumer shared library has wider blast radius. Surface this to
   the user explicitly rather than doing it silently as part of an
   app-repo porting task.
5. For doc/PRD commits, diff the *claim*, not the *line numbers* — grep the
   target's own docs for the same stale fact before assuming the
   correction is needed.
6. When you find a structurally-similar-looking cleanup opportunity in the
   target repo (e.g. two near-duplicate zbus listeners), check whether the
   target's own dev-specs document an intentional design reason for the
   apparent duplication (with a stated regression-test note) before
   "fixing" it — a structural resemblance to a known bug elsewhere is not
   proof it's a bug here.

## Outcome From This Case

Evaluated 9 commits from `nordic-wifi-memfault` (`ncs264` branch) against
`nordic-wifi-memfault-ncs340` (`ncs-v3.4.x-lts` branch). Result: **0 of 9
applied.** All were either already superseded (and improved upon) in the
`zego` brick library, referenced files/facts that didn't exist in the
target's dual-board architecture, or were docs describing states already
correctly documented differently in the target. This is not a "the task
failed" outcome — it's the correct outcome for two forks that diverged via
brick extraction; forcing the port would have reintroduced already-fixed
bugs or removed live shared features.

## Related Pages
- [[ncs-version-migration]] — single-project SDK version bumps (different
  problem: same repo, newer SDK, not a diverged sibling fork)
- [[west-update-internals]] — how shared projects like `zego` get pinned/
  checked out in a west workspace
