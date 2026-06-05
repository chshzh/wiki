---
title: nordic-wifi-webdash — Memory Budget and Webserver Cost
created: 2026-06-05
updated: 2026-06-05
type: concept
tags: [ncs, memory, wifi, build, pattern]
sources: [session:2026-06-05-webdash-memory]
confidence: high
---

# nordic-wifi-webdash — Memory Budget and Webserver Cost

Measured on nRF7002DK (`nrf7002dk/nrf5340/cpuapp`, NCS v3.3.0), STA-only builds
(SoftAP disabled, no P2P snippet). Numbers from `arm-zephyr-eabi-size` on the
app-core ELF.

---

## nRF5340 cpuapp Memory Regions

| Region | Size | Notes |
|--------|------|-------|
| FLASH | 1,048,576 B (1 MB) | Full chip flash; linker `ORIGIN=0x0, LENGTH=1024*1024` (PM disabled) |
| RAM | 458,752 B (448 KB) | 512 KB total − 64 KB reserved for `hci_ipc` (network core) |

**Partition layout** (from built DTS, no Partition Manager):

| Partition | Address | Size | Contents |
|-----------|---------|------|---------|
| `boot_partition` | 0x00000 | 64 KB | MCUboot |
| `slot0_partition` | 0x10000 | 464 KB | Active app (OTA slot) |
| `slot1_partition` | 0x84000 | 464 KB | DFU update slot |
| `storage_partition` | 0xF8000 | 32 KB | NVS / settings |

The linker uses the **full 1 MB** as the FLASH region (not just the slot). MCUboot + linker-script flash image fill the 1 MB; the DFU slot is only relevant at runtime.

---

## Measured Flash and RAM: STA-only (nRF7002DK)

| Config | text (B) | data (B) | bss (B) | **Total Flash** | **Total RAM** |
|--------|----------|----------|---------|----------------|--------------|
| STA + webserver | 680,964 | 6,099 | 383,904 | **687,063 B (671 KB / 65.5%)** | **390,003 B (381 KB / 85.0%)** |
| STA, no webserver | 635,916 | 5,633 | 365,339 | **641,549 B (627 KB / 61.1%)** | **370,972 B (362 KB / 80.9%)** |
| **Delta (webserver cost)** | +45,048 | +466 | +18,565 | **+45,514 B (+44.5 KB)** | **+19,031 B (+18.6 KB)** |

- `Total Flash = text + data` (data section is stored in flash, copied to RAM at boot)
- `Total RAM = data + bss`

---

## Webserver Flash Breakdown (~44.5 KB)

From `west build -t rom_report` on the app-core inner build:

| Component | Flash (B) | Notes |
|-----------|----------|-------|
| `webserver.c` | 9,379 | Includes static assets + HTTP handlers |
| ↳ `index.html.gz` | 1,242 | Gzip-compressed rodata |
| ↳ `main.js.gz` | 3,713 | Gzip-compressed rodata |
| ↳ `styles.css.gz` | 2,064 | Gzip-compressed rodata |
| ↳ app code (handlers, listeners) | ~2,360 | |
| `http_parser.c` | 6,532 | HTTP/1.1 request parser |
| `http_parser_url.c` | 548 | |
| `http_server_core.c` | 2,166 | Thread, init, client management |
| `http_server_http1.c` | 2,756 | HTTP/1.1 request handling |
| `dns_sd.c` | 2,529 | DNS-SD service record |
| `mdns_responder.c` | 1,528 | mDNS advertisement |
| Remaining HTTP subsystem | ~20,276 | http_server_http2, content type tables, socket init, thread data |

**Static web assets cost 7,019 B = 15.8% of the total webserver flash overhead.**
The dominant cost is the Zephyr HTTP server + parser subsystem (~21 KB).

---

## Webserver RAM Breakdown (~18.6 KB)

| Source | ~Bytes | Notes |
|--------|--------|-------|
| HTTP server client contexts × 10 | ~15,360 | `CONFIG_HTTP_SERVER_MAX_CLIENTS=10` |
| HTTP server thread stack | 2,048 | `CONFIG_HTTP_SERVER_STACK_SIZE=2048` |
| `webserver.c` static buffers | 1,536 | `system_api_buf`(512) + `button_api_buf`(512) + `led_get_api_buf`(512) |
| mDNS / DNS-SD state | ~87 | Record structs |

**Quick RAM saving**: Reduce `CONFIG_HTTP_SERVER_MAX_CLIENTS` from 10 → 2 to save ~9 KB RAM (only one Wi-Fi client ever connects in any mode).

---

## CMakeLists.txt Fix: webserver.c Was Unconditionally Compiled

Before this session, `webserver.c` and the HTTP linker sections were always compiled
regardless of Kconfig. `CONFIG_WEBSERVER_MODULE` only gated log-level choices.

**Fix applied (2026-06-05):**

`src/modules/webserver/CMakeLists.txt`:
```cmake
if(CONFIG_WEBSERVER_MODULE)
  target_sources(app PRIVATE webserver.c)
  target_include_directories(app PUBLIC ${ZEPHYR_BASE}/subsys/net/ip)
endif()
```

Top-level `CMakeLists.txt` — gated HTTP linker sections and gzip asset generation:
```cmake
if(CONFIG_HTTP_SERVER)
  zephyr_linker_sources(SECTIONS sections-rom.ld)
  zephyr_linker_section_ifdef(CONFIG_HTTP_SERVER NAME ...)
  foreach(web_resource index.html main.js styles.css)
    generate_inc_file_for_target(...)
  endforeach()
endif()
```

**Now:** Setting `CONFIG_WEBSERVER_MODULE=n` + `CONFIG_HTTP_SERVER=n` fully strips the
HTTP stack, mDNS, DNS-SD, and all static web assets from the binary. Zero overhead in production.

---

## Overlay Files for Feature Comparison Builds

Two overlay confs created in `nordic-wifi-webdash/`:

### `overlay-sta-webserver.conf` — STA-only, HTTP enabled
```conf
CONFIG_NRF70_AP_MODE=n
CONFIG_WIFI_NM_WPA_SUPPLICANT_AP=n
CONFIG_NET_DHCPV4_SERVER=n
CONFIG_NET_CONFIG_SETTINGS=n
CONFIG_ZEGO_WIFI_DEFAULT_MODE_SOFTAP=n
CONFIG_ZEGO_WIFI_DEFAULT_MODE_STA=y
```

### `overlay-sta-no-webserver.conf` — STA-only, HTTP disabled
Same as above, plus:
```conf
CONFIG_WEBSERVER_MODULE=n
CONFIG_HTTP_SERVER=n
CONFIG_HTTP_PARSER=n
CONFIG_HTTP_PARSER_URL=n
CONFIG_MDNS_RESPONDER=n
CONFIG_DNS_SD=n
CONFIG_MDNS_RESPONDER_DNS_SD=n
```

Build command (single overlay):
```sh
west build -p -b nrf7002dk/nrf5340/cpuapp -d build_sta_webserver -- \
  -DEXTRA_CONF_FILE="overlay-sta-webserver.conf"
```

---

## Using webdash as a Metrics/Config GUI for Other Apps

**Verdict: not worth it on nRF7002DK; cautious yes on nRF54LM20DK for dev-only use.**

| Factor | Detail |
|--------|--------|
| nRF7002DK flash budget | After nordic-wifi-memfault (MQTT + TLS + Memfault SDK), adding HTTP server (+44.5 KB) likely overflows 1 MB |
| nRF54LM20DK | 2 MB flash + more RAM; feasible |
| RAM at 85% | Config A already at 85% of 448 KB — margin is thin for production |
| Value vs Memfault cloud | Memfault UI already shows metrics, crash traces, fleet view with auth; local HTTP adds a single-device, unauthenticated subset |
| Security | webdash HTTP server has no authentication — any LAN host can read state or POST commands |
| Architecture fit | zbus composability means you can subscribe a new webserver listener to existing channels cheaply; the framework works, the budget is the constraint |
| Best use case | Read-only dev/debug panel during bench work; gate with `CONFIG_WEBSERVER_MODULE=n` in production builds |

---

## Related Pages
- [nrf7002dk](../entities/nrf7002dk.md) — board flash/RAM budget
- [nrf54lm20dk-plus-nrf7002eb2](../entities/nrf54lm20dk-plus-nrf7002eb2.md) — more headroom, 2 MB flash
- [ncs-build-system](ncs-build-system.md) — EXTRA_CONF_FILE overlay pattern
- [memfault-workflow](memfault-workflow.md) — why Memfault cloud UI often supersedes a local webdash
