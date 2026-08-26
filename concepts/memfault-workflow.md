---
title: Memfault Workflow — Build, Flash, Upload, Release, Deploy
created: 2026-05-31
updated: 2026-05-31
type: concept
tags: [memfault, ota, build, pattern, debug]
sources: [session:d1cdeb42, session:54fb87e8, session:b51223d2, session:0ebeff51]
confidence: high
---

# Memfault Workflow

## The Full Loop (from scratch)

```
1. Build firmware with Memfault overlay
   west build -p -b <board> nordic-wifi-memfault -d build_<board> -- \
     -DEXTRA_CONF_FILE=overlay-app-memfault-project-info.conf \
     [-DSHIELD=nrf7002eb2]

2. Flash to hardware
   west flash -d build_<board>

3. Verify device connects to Memfault cloud
   → Watch UART logs for "Connected to Memfault"
   → Check Memfault web console for device heartbeat

4. Upload ELF symbol file (enables crash decode)
   → Use chsh-sk-memfault skill or chsh-ag-memfault agent

5. Create OTA release
   → Bump version in VERSION or prj.conf
   → Build with new version
   → Upload release artifact + symbols

6. Deploy release to cohort
   → Requires explicit approval gate
   → Can abort if issues emerge
```

---

## Build Commands by Board

```sh
# nRF54LM20DK + nRF7002EB2
west build -p -b nrf54lm20dk/nrf54lm20a/cpuapp nordic-wifi-memfault \
  -d build_nrf54lm20dk -- \
  -DSHIELD=nrf7002eb2 \
  -DEXTRA_CONF_FILE=overlay-app-memfault-project-info.conf

# nRF7002DK
west build -p -b nrf7002dk/nrf5340/cpuapp nordic-wifi-memfault \
  -d build_nrf7002dk -- \
  -DEXTRA_CONF_FILE=overlay-app-memfault-project-info.conf
```

`overlay-app-memfault-project-info.conf` must be populated from the `.template` file:
```sh
cp overlay-app-memfault-project-info.conf.template overlay-app-memfault-project-info.conf
# Edit to add project key and device ID
```

---

## Memfault API Evolution

### mflt_services_init → mflt_services_sys_init

In a Memfault SDK update (encountered in session b51223d2, 2025-10):

```c
// Old — causes linker error in newer SDK
mflt_services_init();

// New
mflt_services_sys_init();
```

**Pattern:** If you see an undefined reference to `mflt_services_init` in a linker error,
this is the rename. grep your code: `grep -r "mflt_services_init" .`

---

## Release Naming Convention

From session 54fb87e8 (2026-05-16), release `3.3.0.2` was deployed as a "quick test"
Workflow B run. The convention is:
```
<NCS_VERSION>.<RELEASE_ITERATION>
e.g., 3.3.0.2 = NCS v3.3.0, second release build
```

---

## MQTT Debugging: Memfault OTA Topic

Seen in session 54fb87e8: investigated MQTT subscription to topic  
`Memfault/F4CE3600AF12/Count`

This is the Memfault OTA chunk count topic. If the device isn't pulling OTA updates,
check:
1. MQTT connection is established (UART log: "MQTT connected")
2. Device is subscribed to the topic
3. Cohort deployment is active in Memfault console
4. Device version matches the deployment filter

---

## Memfault + WiFi Reconnect: Double Jeopardy

**Critical interaction found in session d1cdeb42 (2026-05-21):**

If WiFi disconnects and the device reboots (even intentionally), any crash data in the
RAM coredump buffer is lost. Memfault only uploads on the next successful connection.

**Implication:** For debugging WiFi reconnect crashes, capture UART logs *in addition*
to relying on Memfault crash reports. The crash may never reach Memfault if it happens
during the reconnect window.

---

## Incident: Device Freezes Forever in the Memfault SDK 1.6.0 HTTP Port (2026-08-18)

**Symptom (nordic-wifi-memfault v2.6.4.5, nRF7002DK):** after a random uptime — ~29 min in
the reported case — UART log output stopped mid-stream and all Memfault metrics/logs stopped
arriving. No crash, no reboot, no coredump. Only a power cycle recovered the device.

**Root cause — `prv_read_socket_data()` in `ports/zephyr/common/memfault_platform_http.c`:**

```c
const int len = recv(sock_fd, buf, *buf_len, MSG_DONTWAIT);
if (len <= 0) {
  if ((errno == EAGAIN) || (errno == EWOULDBLOCK)) { *buf_len = 0; return true; }
```

`recv()` returns **0** on an orderly peer shutdown and **leaves `errno` untouched**. `errno`
is thread-local and never cleared, so a stale `EAGAIN` — routinely left behind by
`prv_try_send()`'s non-blocking `send()` and by the TLS handshake (Zephyr's `tls_tx`/`tls_rx`
pass `ZSOCK_MSG_DONTWAIT` unconditionally) — makes end-of-stream look retryable.
`prv_wait_for_http_response()` then loops forever on zero-byte reads.

The loop never throttles: after a peer close on a **stream** socket Zephyr keeps `POLLIN`
asserted and adds `POLLHUP` (`ztls_poll_update_pollin()`), and deliberately does *not* reset
the mbedTLS context so consecutive `recv()` calls keep returning 0 (`recv_tls()`). So
`poll()` returns immediately every iteration instead of blocking for its timeout.

**Why total silence, not just a stalled upload — two amplifiers:**
1. The SDK starts its periodic-upload work queue at `K_HIGHEST_APPLICATION_THREAD_PRIO`
   (priority 0). Zephyr's log processing thread runs at `K_LOWEST_APPLICATION_THREAD_PRIO`
   (14) unless `CONFIG_LOG_PROCESS_THREAD_CUSTOM_PRIORITY` is set. A never-sleeping
   priority-0 thread starves it → **UART logging dies**.
2. The project's `tls_heap_lock` (needed because `CONFIG_MBEDTLS_THREADING_C` is unavailable
   without CryptoCell) was acquired `K_FOREVER` at every TLS site, so the spinning thread
   froze MQTT, HTTPS, and the app's own upload thread with it.

**Distinguishing detail:** an **abrupt** reset is handled correctly (`POLLERR` → `recv()`
returns −1 with `ECONNRESET` → clean failure). Only a **graceful** FIN/close_notify hits the
infinite loop. That's what makes the failure time look random.

**Fixes** (`patches/memfault-firmware-sdk/0003`, `0004`): clear `errno` before `recv()`,
treat `len == 0` as end-of-stream, bound the total response wait; drop the upload queue to
priority 7; replace every `K_FOREVER` lock wait with `TLS_HEAP_LOCK_TIMEOUT` (180 s) and skip
the cycle on timeout. Also moved the app's forced-heartbeat upload off the **system work
queue**, where a blocking TLS post stalls DHCP renewal, the heap monitor, and the L3 watchdog.

**Cleared during the same review:** the "always close socket on failed upload" fix
(`0001`) cannot block — `ztls_close_ctx()` sends close_notify through `tls_tx()`, which is
unconditionally non-blocking, and the result is discarded.

**RAM constraint worth remembering:** this project links at **99.53 % of the 448 KB app RAM
region** (2164 B free). Any stack bump has to be justified by a thread-analyzer measurement;
raising the 2048 B `mflt_http` stack to 4096 B left only 116 B of headroom and was reverted.

---

## Artifacts Location

```
nordic-wifi-memfault/
├── artifacts/                  # Pre-built binaries (for release)
├── build_nrf7002dk/            # Build output (gitignored)
├── build_nrf54lm20dk/          # Build output (gitignored)
├── overlay-app-memfault-project-info.conf          # Populated (gitignored)
└── overlay-app-memfault-project-info.conf.template # Template (committed)
```

---

## Incident: Wrong-Workspace Binary Uploaded as OTA Payload (2026-07-27)

**Symptom:** OTA release `v2.6.4.1` (nord-project, nrf7002dk-fw) downloaded cleanly on
the device (100%, no network/DNS errors), device self-rebooted, but came back up as the
exact same old build (same version string, same `__DATE__`/`__TIME__`). No MCUboot error
visible (its console/log is compiled out on this project to save flash), so it looked
like a silent MCUboot swap failure.

**Root cause:** `chsh-ag-memfault` had a **hardcoded** `Project root:
/opt/nordic/ncs/v3.3.0/nordic-wifi-memfault` baked into its instructions, with all
build/artifact paths (`build_nrf7002dk/.../zephyr.signed.bin`) relative to that one
fixed root. `nordic-wifi-memfault` is checked out under *multiple* NCS-version
workspaces simultaneously (v2.6.4, v3.3.0, v3.4.0, ...) — same filenames, same board
target string (`nrf7002dk-fw`/`nrf7002dk`), completely different actual firmware per
workspace. When delegated a task for the v2.6.4 workspace (even with explicit
`/tmp/fw/...` artifact paths given), the agent still resolved its own hardcoded
project root and uploaded the wrong workspace's leftover `build_nrf7002dk` binary
(a v3.4.0.x build) as the OTA payload for release `v2.6.4.1`.

**How it was actually found:** Server-side and cohort/deployment checks all looked
correct (`memfault-reason: cohort_settings`, HTTP 200, valid signed URL, deployment
`status: done`). The only thing that revealed the real problem was diffing the
**downloaded OTA CDN payload's bytes** against the known-correct GitHub release
artifact — different size, different SHA256, and `strings -a <file> | grep -E
'^v[0-9]+\.[0-9]+\.[0-9]+'` showed the CDN payload embedded `v3.4.0.1`/`v3.4.0.2`
instead of `v2.6.4.1`. **Lesson: when OTA "succeeds" at every layer but the device
never actually updates, verify the actual bytes served by the CDN URL, not just
that a release/deployment exists.**

**Fix applied** (both `chsh-ag-memfault.md` and `chsh-sk-memfault` SKILL.md):
1. Removed the hardcoded project root entirely — the agent now *requires* an
   explicit absolute `PROJECT_ROOT` (or explicit absolute artifact paths) from
   every delegating task, and echoes it back before using it.
2. Added a mandatory pre-upload check: `strings -a <exact file about to be
   uploaded> | grep -E '^v?[0-9]+\.[0-9]+\.[0-9]+(\.[0-9]+)?$'` must match
   `$FW_VERSION` before any `upload-mcu-symbols`/`upload-ota-payload` call —
   abort on mismatch instead of uploading.
3. Added a post-upload verification: re-download the just-uploaded OTA payload
   from Memfault's CDN and re-check its embedded version string, so the check
   covers what the cloud actually serves, not just the local file.
4. `chsh-sk-memfault` SKILL.md now requires the delegating prompt to always
   paste a literal resolved absolute project root (or artifact paths) into the
   `chsh-ag-memfault` task text — never left implicit.

### Part 2 — even after fixing the artifact, OTA still didn't apply

Re-uploading the correct `v2.6.4.1` binary (verified correct both locally and via
a fresh CDN re-download) was **not sufficient** — the device still received the
same wrong `v3.4.0.x` bytes on the very next OTA check. Root cause: this Memfault
project (`nrf-test`) hosts releases for **both** the v2.6.4 branch and the
unrelated v3.3.0/v3.4.0 branch, and cohort `default` had `count_active_releases: 4`
simultaneously "done" for the identical `(cohort, hardware_version=nrf7002dk,
software_type=nrf7002dk-fw)` target: `v3.4.0.0`, `v3.4.0.1`, `v3.4.0.2`, and the
newly-fixed `v2.6.4.1`. **Memfault's "latest release" resolution for a device
picks the numerically/lexicographically highest active version across ALL
"done" deployments sharing that exact target combo — not the most-recently-
deployed one, and not `cohort.last_deployment` either.** Since `3.4.0.2 >
2.6.4.1`, the `v3.4.0.2` release always won, no matter what got (re)deployed as
`v2.6.4.1`.

**Confirmed fix:** pulling (disabling) the `v3.4.0.0`/`.1`/`.2` deployments from
cohort `default` immediately made the correct `v2.6.4.1` payload resolve (empirically
re-verified: CDN hash changed to the correct binary, embedded strings `v2.6.4.1`).
**Caveat:** this also stops cohort `default` from serving `v3.4.0.x` OTA updates to
any *other* real devices in that cohort (their current version doesn't regress,
they just won't get further `v3.4.0.x`-line updates until those deployments are
re-enabled).

**Structural lesson:** never share one Memfault cohort (or more precisely, one
`(cohort, hardware_version, software_type)` triple) across two unrelated
branches/projects with independent version-numbering schemes — a lower-numbered
branch's releases can never be selected as "latest" while a higher-numbered
branch's release is also active there, and pulling the higher one to make room
is a shared/destructive action affecting the other branch's real fleet. Prefer a
dedicated cohort per branch/project, or at minimum check `count_active_releases`
and enumerate `GET /deployments?cohort=<slug>` for the target hardware_version+
software_type *before* trusting a "successful" symbol/OTA upload+deploy to mean
the device will actually receive it.

---

## Related Pages
- [[ncs-build-system]] — Build command details and flag reference
- [[nrf7002dk]] — nRF7002DK device in Memfault fleet
- [[nrf54lm20dk-plus-nrf7002eb2]] — nRF54LM20DK device in Memfault fleet
- [[wifi-debugging-patterns]] — WiFi failures that affect Memfault telemetry
