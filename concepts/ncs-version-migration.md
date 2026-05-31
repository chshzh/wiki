---
title: NCS Version Migration — The Iterative Grep/Fix Pattern
created: 2026-05-31
updated: 2026-05-31
type: concept
tags: [ncs, migration, build, pattern]
sources: [session:299515f9, session:0ebeff51, session:2ac79fc1, session:f157d027]
confidence: high
---

# NCS Version Migration

## The Core Lesson: Expect Iteration, Not a One-Shot Fix

Every NCS major version migration involves **multiple separate error cycles**. The
pattern from session 299515f9 (v3.2.1 → v3.3.0, 21 turns) shows this is normal:

> "Multiple build error cycles — iterative grep/fix on inet_nton and other API changes."

**Mental model:** A migration is not one problem. It's 5–15 independent API breaks,
config symbol renames, and infrastructure changes. Approach it as a loop:

```
while (build has errors):
    1. Run west build -p to get current error list
    2. Group errors by root cause
    3. Fix the largest cluster
    4. Repeat
```

Do NOT try to find and fix all issues at once before rebuilding. Fix one cluster,
rebuild, then see what's left. API changes sometimes mask each other.

---

## Known API Changes: v3.2.x → v3.3.0

### Networking: inet_nton rename
```c
// Old (v3.2.x)
inet_nton(AF_INET, &addr, buf, sizeof(buf));

// New (v3.3.0) — function renamed or signature changed
// grep your codebase: grep -r "inet_nton" .
```
**Seen in:** session 299515f9 (2026-05-27), nordic-wifi-audio migration.

### Build config: OVERLAY_CONFIG → EXTRA_CONF_FILE
See [[ncs-build-system]] for details.

### Memfault API: mflt_services_init → mflt_services_sys_init
See [[ncs-build-system]] for details.

---

## Known Changes: v3.1.1 → v3.2.0

- Various Kconfig symbol cleanups (deprecated symbols removed)
- NCS v3.1.1 → v3.2.0 triggered build troubleshooting in session 2ac79fc1 (2025-10-30, 17 turns)
- MBEDTLS_LEGACY_CRYPTO_C became a build error (was a warning)

---

## The MBEDTLS Legacy Crypto Migration

This affected the transition from v3.1.x to v3.2.x:

**Symptom:** Build error or warning about `MBEDTLS_LEGACY_CRYPTO_C`.

**Fix strategy:**
1. Remove `CONFIG_MBEDTLS_LEGACY_CRYPTO_C=y` from all overlay and prj.conf files
2. Check if TLS 1.0/1.1 support was needed (rarely required for IoT)
3. Consult NCS migration guide for the exact replacement symbol

**Seen in:** session 0ebeff51 (2026-04-17).

---

## Partition Manager → DTS Migration

This was a separate structural change (not tied to a single version):

**Old approach:** `pm_static.yml` file in project root describes flash partitions  
**New approach:** Flash layout in `boards/<board>.overlay` DTS file

**Migration steps:**
1. Delete or rename `pm_static.yml`
2. Create `boards/<board>.overlay` with equivalent `&flash0` DTS nodes
3. Map `PM_APP_ADDRESS` etc. to DTS `reg` properties
4. Rebuild with `-p`

**Seen in:** session 0ebeff51 (2026-04-17) — nordic-wifi-memfault ported from PM to DTS.

---

## Systematic Migration Checklist

For any NCS version upgrade, check these in order:

1. **Update west.yml** → point `revision:` to new SDK tag
2. **`west update`** → pull new dependencies
3. **Pristine build** → `west build -p` to see fresh error list
4. **Scan for deprecated Kconfig** → `grep -r "DEPRECATED\|CONFIG_MBEDTLS_LEGACY" .`
5. **Check networking APIs** → `grep -r "inet_nton\|inet_aton\|net_addr" .`
6. **Check Memfault API** → `grep -r "mflt_services_init" .`
7. **Check OVERLAY_CONFIG** → `grep -r "OVERLAY_CONFIG" .` (in CMake args, scripts, CI)
8. **Read NCS release notes** → `nrf/RELEASE_NOTES.rst` in the SDK

---

## How Long Does It Take?

From the agentmemory history:
- **v3.1.1 → v3.2.0** (session 2ac79fc1): 17 turns — non-trivial
- **v3.2.1 → v3.3.0** for nordic-wifi-audio (session 299515f9): 21 turns — significant
- **MBEDTLS + PM→DTS** (session 0ebeff51): 14 turns — two separate refactors

Budget **half a day** for a well-maintained project migrating one major version.
Budget **a full day** if the project uses Memfault, P2P, and audio features.

---

## Related Pages
- [[ncs-build-system]] — Build commands and flag reference
- [[memfault-workflow]] — Memfault-specific migration notes
- [[nrf7002dk]] — Board target strings
- [[nrf54lm20dk-plus-nrf7002eb2]] — Shield-based board migration notes
