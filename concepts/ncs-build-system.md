---
title: NCS Build System — Commands, Pitfalls, and Evolution
created: 2026-05-31
updated: 2026-05-31
type: concept
tags: [ncs, build, zephyr, pattern]
sources: [session:b51223d2, session:df6d96ae, session:0ebeff51, session:f157d027]
confidence: high
---

# NCS Build System

## The #1 Rookie Mistake: Missing zephyr-env.sh

**Every** west command requires the Zephyr environment to be sourced first. This is the
most common reason west commands fail mysteriously — especially in fresh terminals.

```sh
source /opt/nordic/ncs/v3.3.0/zephyr/zephyr-env.sh
```

Seen in: sessions from 2025-10 repeatedly. Even experienced users forget this when
opening a new terminal tab.

**Pattern:** If `west build` fails with a `west: command not found` or bizarre CMake
errors about missing toolchain, check if `zephyr-env.sh` was sourced.

---

## The Canonical Build Command

```sh
west build -p -b <board-target> <app-dir> -d <build-dir> -- \
  -DSHIELD=<shield> \
  -DEXTRA_CONF_FILE=<overlay.conf>
```

### Flag reference

| Flag | Meaning | When to use |
|------|---------|-------------|
| `-p` | Pristine (clean) build | Always when changing board, overlay, or after SDK update |
| `-b <target>` | Board target | Required every build |
| `-d <dir>` | Custom build directory | When building for multiple boards |
| `--` | Separator before CMake args | Required before `-D` flags |
| `-DSHIELD=nrf7002eb2` | Add nRF7002EB2 shield | nRF54LM20DK builds |
| `-DEXTRA_CONF_FILE=...` | Extra overlay config | Memfault project info, feature overlays |

### Board target strings

| Hardware | Target string |
|----------|--------------|
| nRF54LM20DK + nRF7002EB2 | `nrf54lm20dk/nrf54lm20a/cpuapp` |
| nRF7002DK | `nrf7002dk/nrf5340/cpuapp` |
| nRF5340 Audio DK | `nrf5340_audio_dk/nrf5340/cpuapp` |

### Multiple overlay files

```sh
-DEXTRA_CONF_FILE="overlay-a.conf;overlay-b.conf"
```

Use semicolon-separated list (not comma). Quotes are mandatory.

---

## OVERLAY_CONFIG → EXTRA_CONF_FILE Migration

**The change:** `OVERLAY_CONFIG` was deprecated in NCS v3.2.x. `EXTRA_CONF_FILE` is the
replacement. Old documentation and old scripts still show `OVERLAY_CONFIG`.

| Old (deprecated) | New |
|-----------------|-----|
| `-DOVERLAY_CONFIG=overlay.conf` | `-DEXTRA_CONF_FILE=overlay.conf` |

**When this bites you:** If you copy a build command from old README, NCS docs older than
v3.2.x, or from a colleague's notes pre-2025, it will use `OVERLAY_CONFIG`. The overlay
silently doesn't apply — builds succeed but without the intended config.

**Seen in:** sessions df6d96ae (2025-10-10), 0ebeff51 (2026-04-17), and the migration
of project READMEs in 2026-05.

---

## Memfault API Rename: mflt_services_init → mflt_services_sys_init

In a Memfault SDK update, the initialization function was renamed:

```c
// Old
mflt_services_init();

// New
mflt_services_sys_init();
```

**Pattern:** When building after a Memfault SDK update and seeing a linker error about
`mflt_services_init` undefined, this is the cause.

**Seen in:** session b51223d2 (2025-10-13).

---

## MBEDTLS_LEGACY_CRYPTO_C Deprecation

`MBEDTLS_LEGACY_CRYPTO_C` Kconfig symbol was deprecated, causing build warnings that
became errors in later NCS versions.

**Fix:** Remove the symbol from overlays/prj.conf, or replace with the new equivalent.
Check NCS release notes for the exact replacement config.

**Seen in:** session 0ebeff51 (2026-04-17).

---

## Flash Partitioning: Partition Manager → DTS

Nordic moved flash layout from `pm_static.yml` to DTS overlay files.

| Old | New |
|-----|-----|
| `pm_static.yml` in project root | `boards/<board>.overlay` DTS nodes |
| Nordic Partition Manager tool | Zephyr DTS flash layout |

**Seen in:** session 0ebeff51 (2026-04-17) — migrated nordic-wifi-memfault from PM to DTS.

**Pattern:** If build complains about partition manager not finding regions, check if the
project still uses `pm_static.yml` and needs to be ported to DTS.

---

## Flash Memory Overflow

**Symptom:** Build succeeds but `west flash` fails, or linker error about `.text` overflow.

**Approach:**
1. Check `west build` output for section sizes
2. Identify the largest contributor (usually Memfault or TLS)
3. Disable unused features via Kconfig (`CONFIG_xxx=n`)
4. Consider using sysbuild with separate net/app cores to share flash budget

**Seen in:** session 558f2717 (2025-12-29).

---

## Related Pages
- [[ncs-version-migration]] — API changes to expect when upgrading SDK
- [[memfault-workflow]] — Memfault-specific build commands
- [[nrf7002dk]] — nRF7002DK board target details
- [[nrf54lm20dk-plus-nrf7002eb2]] — nRF54LM20DK board target details
