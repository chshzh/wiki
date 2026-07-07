---
title: NCS Version Migration — The Iterative Grep/Fix Pattern
created: 2026-05-31
updated: 2026-07-07
type: concept
tags: [ncs, migration, build, pattern]
sources: [session:299515f9, session:0ebeff51, session:2ac79fc1, session:f157d027, session:4ea9621c, session:d34b1836]
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

## Known Changes: v3.3.0 → v3.4.0

Mbed TLS was bumped to v4.1.0 (TF-PSA-Crypto backend), which removed the legacy
mbedcrypto config path entirely. Seen migrating `zego` (nrf54lm20dk, nrf7002dk,
nrf5340_audio_dk).

### CONFIG_NRF_SECURITY became a computed symbol
**Symptom:** `NRF_SECURITY (defined at ...) is assigned in a configuration file,
but is not directly user-configurable (has no prompt).`
**Fix:** Delete `CONFIG_NRF_SECURITY=y` from prj.conf — it's now `def_bool y`,
auto-enabled for Nordic SoCs. Setting it explicitly is now a hard Kconfig error,
not a warning.

### CONFIG_MBEDTLS_LEGACY_CRYPTO_C removed entirely
**Symptom:** Same class of error as above, or "unknown symbol" if referenced in
app Kconfig (e.g. a `_SILENCE_DEPRECATION` helper symbol).
**Fix:** Delete `CONFIG_MBEDTLS_LEGACY_CRYPTO_C=y` and any app-side Kconfig entry
that referenced it (e.g. `MBEDTLS_LEGACY_CRYPTO_C_SILENCE_DEPRECATION`). The
"legacy crypto config path" it used to route through no longer exists.

### MBEDTLS_MEMORY_DEBUG no longer forwarded to generated crypto headers
**Symptom:** `implicit declaration of function 'mbedtls_memory_buffer_alloc_cur_get'`
/ `_max_get'` — compiles fine at Kconfig level, fails at C compile time.
**Root cause:** nrf_security's PSA-only config generation
(`psa_crypto_want_config.cmake`) explicitly whitelists which Kconfig symbols get
forwarded into the generated PSA crypto config header. `MBEDTLS_MEMORY_DEBUG`
isn't on that whitelist, so the C macro never gets defined even though the
Kconfig symbol is set to `y`. The v3.3.0 workaround (routing through
`MBEDTLS_LEGACY_CRYPTO_C`'s legacy path, which *did* forward it) no longer
exists since that path was removed.
**Fix:** No clean fix exists yet (upstream gap). Default the dependent Kconfig
symbol to `n` and drop the feature (e.g. omit one ZView mbedTLS-heap stat) until
Nordic exposes a forwarding path.

### More promptless mbedTLS symbols: MBEDTLS_X509_LIBRARY, MBEDTLS_TLS_LIBRARY
**Symptom:** Same class of error as `NRF_SECURITY` above, but for
`MBEDTLS_X509_LIBRARY` and/or `MBEDTLS_TLS_LIBRARY` — these are `bool` with no
prompt text in `nrf/subsys/nrf_security/Kconfig.tls`, auto-computed via
`default y if ...` / `select` chains (e.g. `MBEDTLS_TLS_LIBRARY` defaults to
`y` when `NET_SOCKETS_SOCKOPT_TLS=y`, and itself selects `MBEDTLS_X509_LIBRARY`).
**Fix:** Delete both explicit `=y` assignments from `prj.conf`. Symbols that
still have a real prompt string (e.g. `MBEDTLS_X509_CRT_PARSE_C`,
`MBEDTLS_SSL_SERVER_NAME_INDICATION`) are unaffected and can stay.
**Check before deleting:** `grep -n "^config <SYMBOL>" -A3` the defining
Kconfig file — if the line after `bool`/`config` has no quoted prompt string,
it's promptless and must not be assigned directly.

### MBEDTLS_ECP_DP_SECP384R1_ENABLED renamed to PSA_WANT_ECC_SECP_R1_384
**Symptom:** Kconfig parse succeeds (no error), but the curve enablement is
silently lost, or (if `MBEDTLS_PROMPTLESS` happens to be selected elsewhere)
a promptless-assignment error. Zephyr 4.4's migration guide
(`zephyr/doc/releases/migration-guide-4.4.rst`) lists the whole
`CONFIG_MBEDTLS_ECP_DP_*_ENABLED` family as removed in the TF-PSA-Crypto
upgrade.
**Fix:** Replace with the equivalent `CONFIG_PSA_WANT_ECC_SECP_R1_384=y`
(defined in `zephyr/modules/mbedtls/Kconfig.psa.auto`, prompt gated on
`!MBEDTLS_PROMPTLESS` which nrf_security does not select by default, so it's
still directly settable). Same pattern applies to the other renamed
`MBEDTLS_ECP_DP_*_ENABLED` → `PSA_WANT_ECC_*` symbols if they show up.

### MBEDTLS_ECDSA_C now needs MBEDTLS_ECP_C=y explicitly
**Symptom:** `warning: MBEDTLS_ECDSA_C was assigned the value 'y' but got the
value 'n'. Check these unsatisfied dependencies: MBEDTLS_ECP_C (=n)`.
**Fix:** Add `CONFIG_MBEDTLS_ECP_C=y` alongside `CONFIG_MBEDTLS_ECDSA_C=y` —
`MBEDTLS_ECDSA_C` moved under `if MBEDTLS_ECP_C` in
`nrf_security/Kconfig.tf-psa-crypto.deprecated`. Note `MBEDTLS_ECP_C` itself
shows up as a "Deprecated symbol X is enabled" warning afterward — that's
expected and non-fatal (see Kconfig strict-mode note below).

### Kconfig strict mode: NOT all warnings are fatal — only specific classes
`zephyr/scripts/kconfig/kconfig.py` turns **every** Kconfig warning into a
hard build error (`error: Aborting due to Kconfig warnings`) *except* the one
matching `warning:.*set more than once.`. This looks like it means deprecated/
experimental/not-secure symbol warnings are always fatal too — **empirically
they are not**, confirmed by a v3.4.0 migration where the exact same
`Deprecated symbol NRF_PLATFORM_LUMOS/WIFI_NM_WPA_SUPPLICANT_WEP/
_LEGACY_CRYPTO is enabled` and `Experimental symbol WIFI_NM_WPA_SUPPLICANT is
enabled` warnings appeared in **both** a failing build and the final
successful build, byte-for-byte identical. What actually aborted the failing
build were three *other*, real warnings mixed into the same batch:
a value-mismatch (`MBEDTLS_ECDSA_C` unmet-dependency case above), a
"defined without a type" (see Memfault section below), and an
"attempt to assign ... to undefined symbol". **Triage rule:** when a build
aborts with a wall of Kconfig warnings, don't try to silence every deprecated/
experimental one — grep the batch specifically for `defined without a type`,
`undefined symbol`, `is not directly user-configurable`, and
`was assigned the value .* but got the value` — those four are the fatal
classes; everything else in the same batch is very likely just noise that
was already there before your change.

---

## Memfault Firmware SDK bump riding along with an NCS version bump

NCS version bumps also bump the vendored `modules/lib/memfault-firmware-sdk`,
which has its own independent breaking-change cadence. Seen in the same
v3.3.0 → v3.4.0 migration as the mbedTLS changes above — check
`modules/lib/memfault-firmware-sdk/CHANGELOG.md` for a project's Memfault
Kconfig/API breaks separately from the NCS release notes.

### CONFIG_MEMFAULT_COREDUMP_STORAGE_RRAM → CONFIG_MEMFAULT_COREDUMP_STORAGE_NRF_RRAM
**Symptom:** `attempt to assign the value 'y' to the undefined symbol
MEMFAULT_COREDUMP_STORAGE_RRAM` in a board `.conf`.
**Fix:** Straight rename — the SDK renamed it "to better reflect that it
supports nRF devices only" (see CHANGELOG.md).

### CONFIG_MEMFAULT_FOTA removed; memfault_fota_start() → memfault_zephyr_fota_start()
**Symptom:** `MEMFAULT_FOTA_CLI_CMD (defined at ...) defined without a type` /
`the value 'y' is invalid for MEMFAULT_FOTA ... which has type unknown --
assignment ignored`. This is a **local app bug surfaced by the new strict
Kconfig checker**, not a new SDK requirement: the app's own
`Kconfig.defaults` had `config MEMFAULT_FOTA_CLI_CMD` / `config MEMFAULT_FOTA`
blocks with only a `default` line and no `bool` — a valid pattern *only* if
the symbol is defined with a type elsewhere. The SDK removed
`CONFIG_MEMFAULT_FOTA` entirely (FOTA is now driven by
`CONFIG_MEMFAULT_ZEPHYR_FOTA`, auto-selected via
`CONFIG_MEMFAULT_ZEPHYR_FOTA_BACKEND_NCS` when `MEMFAULT_HTTP_ENABLE=y`), and
`MEMFAULT_FOTA_CLI_CMD` doesn't exist anywhere in the current SDK at all
(delete it — it was already a no-op if `CONFIG_MEMFAULT_SHELL=n`).
**Source-level fix required, not just Kconfig:**
```c
// Old
#include <memfault/nrfconnect_port/fota.h>   // path no longer exists
#if IS_ENABLED(CONFIG_MEMFAULT_FOTA)
int rv = memfault_fota_start();

// New
#include <memfault/ports/zephyr/fota.h>
#if IS_ENABLED(CONFIG_MEMFAULT_ZEPHYR_FOTA)
int rv = memfault_zephyr_fota_start();
```
**Rule of thumb:** any Memfault Kconfig symbol that suddenly shows
"defined without a type" or "undefined symbol" during an NCS bump — check
`modules/lib/memfault-firmware-sdk/CHANGELOG.md`'s "💥 Breaking Changes"
sections before assuming it's an NCS/mbedTLS issue.

---

## zego brick refactor gotcha: shared-module adoption can collide with a project's own module

Not NCS-version-specific, but surfaced during the same migration session:
`zego`'s startup banner moved from `zego/bricks/wifi` to a new
`zego/bricks/ux` brick (`zego_wifi_print_banner()` → `zego_ux_print_banner()`,
dated 2026-07-03). A project (`nordic-wifi-memfault`) whose `main.c` called
the old function had to adopt the new `zego/bricks/ux` brick as an
`EXTRA_ZEPHYR_MODULE` + `CONFIG_ZEGO_UX=y` to get the banner back.

**The gotcha:** the project already had its *own*, differently-scoped local
`src/modules/ux/ux.c` (an LED-only Wi-Fi state machine, unrelated to the
banner). Both files happened to use the exact same `ZBUS_LISTENER_DEFINE`
symbol name (`app_wifi_state_listener`) for their respective listeners →
`multiple definition of 'app_wifi_state_listener'` at link time. This is a
pure naming collision (different channels, different structs, both
legitimate) — not a real functional duplicate.
**Fix:** rename the local module's listener symbol to something
project-specific (e.g. `nwm_app_wifi_state_listener`); don't remove either
module. Also add the brick's `__weak` extension points where they exist
(here: `banner_compiled_app_modules()`) to keep app-specific banner content
instead of losing it.
**Rule of thumb:** when adopting a shared brick into a project that already
has its own similarly-named local module, grep the brick's `.c` file for
`ZBUS_LISTENER_DEFINE`/`SYS_INIT`/global symbol names *before* linking, not
after hitting a linker error.

### Kconfig deprecation warnings — triage by "who selects it", not by symbol name
After a clean build, several `warning: Deprecated symbol X is enabled` lines
appeared. Two very different categories showed up, and telling them apart is
the actual skill:
1. **App-controllable defaults** — e.g. `WIFI_NM_WPA_SUPPLICANT_LEGACY_CRYPTO`
   / `_WEP` in `zephyr/modules/hostap/Kconfig` default to `y` upstream but
   aren't referenced anywhere in the app's own `prj.conf`. `grep` the symbol
   name across the app tree first — if it's silent, the app is just inheriting
   a Zephyr default and can safely override it (`grep` the CMakeLists in the
   defining module too, e.g. `hostap/CMakeLists.txt`, to see which source
   files / `#ifdef CONFIG_FIPS` blocks it gates, before deciding to disable).
2. **Vendor-internal, unconditionally selected** — e.g.
   `MBEDTLS_DECLARE_PRIVATE_IDENTIFIERS`, `MBEDTLS_BIGNUM_C`, `MBEDTLS_ECP_C`
   are `select`ed unconditionally by Nordic's own
   `nrf/subsys/net/lib/hostap_crypto/Kconfig` (`HOSTAP_CRYPTO_ALT_PSA`)
   whenever Wi-Fi is used on a Nordic SoC — not something prj.conf sets, and
   not removable without disabling Wi-Fi/WPA3. `grep -rn "select <SYMBOL>"`
   across `nrf/` and `zephyr/modules/` to find the real selector before
   assuming it's fixable.
**Rule of thumb:** if the deprecated symbol doesn't appear anywhere in the
app's own `prj.conf`/Kconfig, don't try to "fix" it there — trace the
`select`/`default y` chain to its source first.

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
