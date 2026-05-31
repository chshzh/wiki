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
# nRF7002DK
west build -p -b nrf7002dk/nrf5340/cpuapp nordic-wifi-memfault \
  -d build_nrf7002dk -- \
  -DEXTRA_CONF_FILE=overlay-app-memfault-project-info.conf

# nRF54LM20DK + nRF7002EB2
west build -p -b nrf54lm20dk/nrf54lm20a/cpuapp nordic-wifi-memfault \
  -d build_nrf54lm20dk -- \
  -DSHIELD=nrf7002eb2 \
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

## Related Pages
- [[ncs-build-system]] — Build command details and flag reference
- [[nrf7002dk]] — nRF7002DK device in Memfault fleet
- [[nrf54lm20dk-plus-nrf7002eb2]] — nRF54LM20DK device in Memfault fleet
- [[wifi-debugging-patterns]] — WiFi failures that affect Memfault telemetry
