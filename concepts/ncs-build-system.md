---
title: NCS Build System — Commands, Pitfalls, and Evolution
created: 2026-05-31
updated: 2026-07-07
type: concept
tags: [ncs, build, zephyr, pattern, kconfig]
sources: [session:b51223d2, session:df6d96ae, session:0ebeff51, session:f157d027, session:a3f9bf71]
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

## MBEDTLS_LEGACY_CRYPTO_C: Deprecated Warning, But Sometimes Required

`CONFIG_MBEDTLS_LEGACY_CRYPTO_C` triggers a deprecation warning in NCS v3.3.0 (nrf_security
defaults to a PSA-only crypto path via `nrf_config.cmake`), but **don't remove it reflexively**
— on CRACEN/PSA-only boards (e.g. nRF54LM20DK), it may be the only way to pull in mbedTLS's
legacy heap-tracking source files.

**Concrete case:** `zego/memonitor`'s `CONFIG_MBEDTLS_MEMORY_DEBUG` feature calls
`mbedtls_memory_buffer_alloc_cur_get/_max_get`. On nRF7002DK (software mbedTLS) these link fine
by default; on nRF54LM20DK (CRACEN/PSA-only path) they silently fail to link
(`undefined reference`) unless `CONFIG_MBEDTLS_LEGACY_CRYPTO_C=y` is set explicitly, alongside
`CONFIG_NRF_SECURITY=y`. Silence the resulting deprecation warning with the app's own
`MBEDTLS_LEGACY_CRYPTO_C_SILENCE_DEPRECATION` Kconfig symbol rather than dropping the config.

**Pattern:** If board-specific mbedTLS linker errors appear only on CRACEN/PSA-only boards
(nRF54 series), check whether the feature needs `MBEDTLS_LEGACY_CRYPTO_C` before assuming the
deprecation warning means "delete this."

**Seen in:** session 0ebeff51 (2026-04-17, first hit on nordic-wifi-memfault) and session
a3f9bf71 (2026-07-02, ported the same fix to nordic-wifi-app-template).

---

## Kconfig Circular Dependency: `select` Breaks What `depends on` Can't

**Symptom:** A brick module's Kconfig default (`default y if FOO`) never fires even though
`FOO` is enabled, forcing every app to hardcode the value in `prj.conf` as a workaround.

**Root cause pattern:**
```
A (depends on B) ──▶ B (depends on C) ──▶ C
         ▲                                │
         └────── (default y if A) ────────┘
```
`C`'s conditional default can never activate because `A` can't turn on until `C` already has —
a real circular dependency, not just a Kconfig ordering quirk. `west build`'s Kconfig resolver
does not detect or report this; it just silently leaves `C` off, and the workaround (explicit
`CONFIG_C=y` in every downstream `prj.conf`) masks the loop indefinitely.

**Fix:** Replace `depends on` + conditional `default` with `select` at the point where a brick
module has a genuinely mandatory (not optional/overridable) requirement on another symbol.
`select` force-enables unconditionally instead of requiring the dependency pre-resolved, which
breaks the cycle at its root. Concretely: `A: select B` instead of `A: depends on B`, and drop
the now-redundant `default y if A` on `B`.

**When NOT to use `select`:** Only for hard, non-negotiable requirements. Keep `depends on` for
anything an app might legitimately want to override (e.g. `L2_WIFI_CONNECTIVITY_AUTO_CONNECT`),
and `select` cannot target non-bool types (int/string tunables stay as conditional defaults or
explicit app overrides).

**Verification methodology:** After restructuring, diff a full pristine-build `.config` against
a pre-change baseline for every affected board — byte-identical output proves the `select` chain
now produces the same result the hardcoded `prj.conf` overrides used to, so those overrides (and
now-dead `default y if` lines) can be safely commented out (not deleted, so intent is preserved).

**Seen in:** session a3f9bf71 (2026-07-02) — `zego/bricks/wifi` and `zego/bricks/network`
Kconfig, breaking a `ZEGO_NETWORK → ZEGO_WIFI → NETWORKING` cycle that both
nordic-wifi-app-template and nordic-wifi-memfault had been working around with hardcoded
`CONFIG_NETWORKING=y` (and 14 other symbols) in `prj.conf`.

---

## `FIXED_PARTITION_*` Macros Deprecated in Favor of `PARTITION_*` (NCS v3.4.0)

**Symptom:** Build succeeds but emits `warning: Macro is deprecated` pointing at a call site
using `FIXED_PARTITION_ID()`, `FIXED_PARTITION_OFFSET()`, or `FIXED_PARTITION_SIZE()` — e.g.
`flash_area_open(FIXED_PARTITION_ID(my_partition), ...)`.

**Root cause:** `zephyr/include/zephyr/storage/flash_map.h` renamed these to `PARTITION_ID()`,
`PARTITION_OFFSET()`, `PARTITION_SIZE()` and kept the old names only as
`__DEPRECATED_MACRO`-tagged aliases for compatibility.

**Fix:** Straight rename, no behavior change — `FIXED_PARTITION_ID(label)` →
`PARTITION_ID(label)` (same for `_OFFSET`/`_SIZE`). Safe to apply everywhere in app-owned code;
leave vendored/west-module code alone (e.g. `modules/lib/memfault-firmware-sdk/...` still uses
the old names upstream — reverted on next `west update` anyway, and it's a warning, not an
error).

**Seen in:** session d34b1836 (2026-07-07, nordic-wifi-memfault) — 3 files
(`app_memfault_log_state_restore.c`, `app_memfault_nrf70_fw_stats_cdr.c`,
`app_memfault_flash_coredump_storage.c`) plus the `pm_config.h` PM-compat shim.

---

## WPA Supplicant Legacy-Crypto/WEP/WPA3 Warnings Are Expected, Not Bugs (NCS v3.4.0)

**Symptom:** Pristine build of any Wi-Fi STA app prints a wall of Kconfig warnings:
`Deprecated symbol WIFI_NM_WPA_SUPPLICANT_LEGACY_CRYPTO/WEP is enabled`,
`Experimental symbol PSA_WANT_ALG_WPA3_SAE_H2E/FIXED/WIFI_NM_WPA_SUPPLICANT is enabled`,
`Not secure symbol ... is enabled`.

**Root cause:** `zephyr/modules/hostap/Kconfig` defaults
`WIFI_NM_WPA_SUPPLICANT_LEGACY_CRYPTO=y` (which `imply`s `_WEP`) so WPA1/TKIP interop with
legacy APs works out of the box; both `select DEPRECATED` and `select NOT_SECURE` unconditionally
when enabled. WPA3 SAE (`PSA_WANT_ALG_WPA3_SAE_*`) is marked experimental upstream but required
for WPA3 network support. None of this is set explicitly by the app — it's the hostap module's
own default.

**Do not disable reflexively** (same lesson as `MBEDTLS_LEGACY_CRYPTO_C` above): setting
`CONFIG_WIFI_NM_WPA_SUPPLICANT_LEGACY_CRYPTO=n` silences the warnings but drops interop with
older WPA-PSK/TKIP/WEP access points still seen in the field. Only disable if the target
deployment is confirmed WPA2/WPA3-only.

**Verdict:** Leave as-is for general-purpose Wi-Fi apps; these are informational upstream
defaults, not app bugs. `NRF_PLATFORM_LUMOS` (deprecated) and the `NETCORE_HCI_IPC` choice
warning are similarly SoC/sysbuild-level (nRF54LM20A is single-core with no netcore image) and
not app-controllable.

**Seen in:** session d34b1836 (2026-07-07, nordic-wifi-memfault) — investigated then left
unchanged after confirming build succeeds and warnings are benign/expected.

---

## ZView `LookupError` on `__weak extern` k_heap Symbols (chshzh/zview)

**Symptom:** `west zview live -e <elf> -r jlink -t <target> -s <serial>` loads the ELF fine
("Loading ELF... OK") then crashes:

```
LookupError: Symbol 'net_buf_mem_pool_rx_bufs' address not found.
```

**Root cause:** `zego/bricks/memonitor/src/memonitor.c` declares heap symbols as
`extern __weak struct k_heap net_buf_mem_pool_rx_bufs;` so the module compiles regardless of
which Kconfig combination is active (the comment literally says "prevents linker errors for
any Kconfig combination that lacks a particular heap"). `net_buf_mem_pool_rx_bufs`/`_tx_bufs`
are only *actually defined* (via `NET_BUF_POOL_VAR_DEFINE`'s `K_HEAP_DEFINE`) when
`CONFIG_NET_BUF_VARIABLE_DATA_SIZE=y`; with the more common `CONFIG_NET_BUF_FIXED_DATA_SIZE=y`
(`NET_BUF_POOL_FIXED_DEFINE`), the real symbols are `net_buf_fixed_rx_bufs`/`_tx_bufs` instead,
and the weak extern stays undefined (resolves to NULL, which the app code correctly guards
against with `if (&net_buf_mem_pool_rx_bufs && ...)`).

The DWARF debug info still records the bare `extern` as a `DW_TAG_variable` DIE with
`DW_AT_declaration: 1` (no `DW_AT_location`, i.e. never actually allocated). ZView's
`elf_inspector.py::_parse()` scanned every `k_heap`-typed `DW_TAG_variable` DIE for
`find_struct_variable_names()` without filtering out declaration-only ones (the sibling
`DW_TAG_structure_type` branch already had this guard; the `DW_TAG_variable` branch didn't).
`_discover_heap_addresses()` then does an unguarded dict comprehension over every discovered
name, so one missing symbol crashes the whole live view.

**Fix (applied 2026-07-07):** Added the same `if die.attributes.get("DW_AT_declaration"): continue`
guard to the `DW_TAG_variable` branch in `elf_inspector.py`, and bumped `_CACHE_SCHEMA_VERSION`
(2→3) so previously-cached (buggy) parses of the same ELF get invalidated automatically —
no manual cache-clearing needed. Verified via direct `ElfInspector` instantiation: the two
weak/undefined names now correctly drop out of `find_struct_variable_names("k_heap")`, while
the 5 real heaps (`_system_heap`, `shell_uart_history_heap`, `wifi_drv_ctrl_mem_pool`,
`wifi_drv_data_mem_pool`, `wifi_nm_wpa_supplicant_mem_pool`) still resolve.

**Pattern:** Any time app code uses `extern __weak struct <T> foo;` purely to make a
maybe-absent global's address queryable at runtime (common for Kconfig-conditional buffer
pools/heaps), tools that walk DWARF for `<T>`-typed globals must skip `DW_AT_declaration`-only
DIEs, or they'll crash on exactly the "doesn't exist in this Kconfig combination" case the
`__weak` pattern was designed to handle gracefully.

**Seen in:** session d34b1836 (2026-07-07, nordic-wifi-memfault) — fixed in
`modules/tools/zview/src/backend/elf_inspector.py` (own repo, `chshzh/zview`, not vendored).

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
- [[zephyr-assert-usage]] — CONFIG_ASSERT behavior, sub-options, and release-vs-debug recommendation
