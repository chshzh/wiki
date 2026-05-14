---
title: How west update Works
created: 2026-05-12
updated: 2026-05-12
type: concept
tags: [ncs, zephyr, west, embedded, tooling]
sources: []
confidence: high
---

# How `west update` Works

West is the multi-repo workspace manager for Zephyr and NCS. `west update` is the
command that synchronises every project in the workspace to the revisions declared
in the active manifest. Understanding its internals prevents common pitfalls around
detached HEAD state, floating vs pinned revisions, and manifest resolution.

## Workspace Discovery

West does **not** care about your current working directory for determining which
workspace to use. When you run `west update` from any subdirectory, West walks up
the directory tree until it finds a `.west/` folder. That folder's location is the
**workspace root**, and `.west/config` inside it is the single source of truth for
which manifest is active.

```
/opt/nordic/ncs/v3.3.0/
├── .west/
│   └── config          ← manifest.path = zego (active manifest)
├── zego/
│   └── west.yml        ← West reads THIS regardless of CWD
├── nordic-wifi-memfault/
│   └── west.yml        ← NOT read unless zego/west.yml imports it
└── nrf/ zephyr/ ...
```

Running `west update` from `/opt/nordic/ncs/v3.3.0/nordic-wifi-memfault` or
`/opt/nordic/ncs/v3.3.0` produces **identical results** — the same manifest,
the same repos updated.

## Active Manifest Resolution

`.west/config` stores `manifest.path` and `manifest.file`:

```ini
[manifest]
path = zego
file = west.yml
```

West reads `<workspace_root>/<manifest.path>/<manifest.file>` — in this case
`zego/west.yml`. Every other `west.yml` in the workspace (e.g.
`nordic-wifi-memfault/west.yml`) is **ignored** unless explicitly imported via
`import: true` in the active manifest.

### Changing the Active Manifest

```bash
west config manifest.path nordic-wifi-memfault
west update   # now uses nordic-wifi-memfault/west.yml
```

This is a persistent change stored in `.west/config`.

## What `west update` Does Per Project

For each project listed in the manifest, West executes roughly:

```bash
# 1. Fetch the declared revision from the remote
git -C <project_path> fetch <remote> <revision>

# 2. Force-reset the manifest-rev branch to the fetched commit
git -C <project_path> update-ref refs/heads/manifest-rev FETCH_HEAD

# 3. Check out manifest-rev (detaches HEAD at that commit)
git -C <project_path> checkout manifest-rev
```

The `manifest-rev` branch is **West-owned**. It is force-reset on every
`west update`. You should never commit to it directly.

## Floating vs Pinned Revisions

The `revision:` field in a manifest project entry controls whether a project
is reproducible:

| `revision:` value | Type | Behaviour on `west update` |
|-------------------|------|---------------------------|
| `main` | Floating branch | Fetches remote `main` tip — changes every run |
| `v3.3.0` | Tag | Always resolves to the same commit |
| `ba167d9f3d` | Pinned SHA | Immutable — identical every run |

```yaml
- name: nordic-wifi-memfault
  revision: main          # ← moves on every west update
  remote: chshzh

- name: sdk-nrf
  revision: v3.3.0        # ← pinned tag, stable
  import: true
```

For reproducible builds (CI, release, team sharing), **pin to a SHA or tag**.
For active development of an app repo, `main` is convenient but means two
developers running `west update` on different days may get different code.

## The `manifest-rev` Branch

Every project managed by West has a local branch called `manifest-rev`. It is:

- **Created or force-updated** by `west update` to point to the pinned commit
- **Checked out** as a detached HEAD after each update
- **Not for committing** — West will overwrite it on the next `west update`
- The only branch West creates in dependency repos

```bash
cd /opt/nordic/ncs/v3.3.0/nrf
git branch
# * (HEAD detached at ba167d9f3d)
#   manifest-rev            ← West manages this
```

## Detached HEAD State — "from" vs "at"

After `west update`, every dependency repo is in **detached HEAD** state.
Git uses two variants in `git status`:

| Message | Meaning |
|---------|---------|
| `HEAD detached at 1bcfaeb` | HEAD is exactly at this commit (just checked out) |
| `HEAD detached from 6999f26` | HEAD originated at `6999f26` and has since advanced |

The "from" form appears when West has updated HEAD forward from a previous
detach point. **The current HEAD is NOT at `6999f26`** — it is at whatever
commit `manifest-rev` now points to. Always use `git rev-parse HEAD` (not
`git status`) as the authoritative source of the current commit:

```bash
git rev-parse HEAD          # actual commit SHA
git rev-parse manifest-rev  # should match HEAD after west update
git log --oneline -3        # see where HEAD sits in the history
```

## The Self Project — Manifest Repo is Not Updated

The `self:` entry in a `west.yml` designates the manifest repository itself.
West does **not** run `git checkout` on the self project during `west update` —
you manage it with normal git commands. This prevents West from overwriting
local edits to the manifest file mid-update.

```yaml
self:
  path: nordic-wifi-memfault   # West leaves this repo alone during update
```

In `zego/west.yml`, `self.path` is implicitly `zego`. So `west update` will
update `nordic-wifi-memfault` as a regular project (it's listed under `projects:`),
but will NOT touch `zego` itself.

## Common Pitfalls

**Running `west update` from inside a managed project:**
West still finds the correct workspace root and updates all projects. However,
if your shell CWD is inside a repo that West is checking out, the `git status`
output in that terminal may show a stale detach label. The actual `.git/HEAD`
on disk is correct. Use `git rev-parse HEAD` to verify.

**Dirty working tree in a managed project:**
If a project has locally modified files when `west update` runs, West prints
`M <filename>` during the fetch but still advances `manifest-rev` and checks it
out. Files that differ from the new HEAD appear as unstaged changes.

**Floating `revision: main` in CI:**
Using branch names in CI manifests breaks reproducibility. Two CI runs on
different days may fetch different commits. For NCS dependency projects, always
use the version tag (e.g. `v3.3.0`). For your own app repos, either pin the SHA
at release time or accept the floating behaviour intentionally.

**`west update` does not apply patches:**
If your build requires `git am` patches on top of upstream repos (e.g. the
`nordic-wifi-shell-sqspi` pattern), `west update` resets those repos and your
patches are gone. Re-apply them after every `west update`.

## Related Pages

- [github-actions-ncs-ci](github-actions-ncs-ci.md) — CI patterns including
  `west update -o=--depth=1 -n` cache optimisation and idempotent patch application
- [ncs-app-versioning](ncs-app-versioning.md) — how app version fields relate to
  the NCS manifest revision pinning strategy
- [embedded-system-general-debugging](embedded-system-general-debugging.md) — debugging
  patterns in NCS/Zephyr projects that depend on correct west workspace state
