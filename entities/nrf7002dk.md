---
title: nRF7002DK
created: 2026-05-31
updated: 2026-05-31
type: entity
entity_type: board
tags: [hardware, ncs, wifi, build]
sources: [session:d1cdeb42, session:b51223d2, session:df6d96ae]
confidence: high
---

# nRF7002DK

**PCA number:** PCA10143  
**SoC:** nRF5340 (dual-core: app + net)  
**WiFi:** nRF7002 (on-board, connected via QSPI)  
**Primary use:** Development, testing, Memfault fleet

---

## Board Target String

```sh
-b nrf7002dk/nrf5340/cpuapp
```

No shield required — nRF7002 is integrated on-board.

---

## Typical Build Command

```sh
west build -p -b nrf7002dk/nrf5340/cpuapp nordic-wifi-memfault \
  -d build_nrf7002dk -- \
  -DEXTRA_CONF_FILE=overlay-app-memfault-project-info.conf
```

---

## USB Serial Port

Pattern on macOS: `/dev/tty.usbmodem<id>`

Flash port and debug port are separate USB connections. Typically use VCOM0 for
application log output (115200 baud).

---

## Behavior Notes

- **WiFi provisioner:** Works reliably; no reset issues when pairing from nRF Wi-Fi
  Provisioner app (session bf04a4eb). Use as baseline when comparing with nRF54LM20DK.
- **WiFi stability:** Solid for reconnect testing. Stack overflow / reboot-on-timeout
  bug in nordic-wifi-memfault (session d1cdeb42) was a firmware logic issue, not
  board-specific.
- **Throughput:** Connected to nRF7002 via QSPI. Throughput benchmarks in
  [[nrf7002dk-vs-nrf54lm20dk]].

---

## Project Support

| Project | Status |
|---------|--------|
| nordic-wifi-memfault | Primary target |
| nordic-wifi-webdash | Primary target |
| nordic-wifi-audio | Not directly (uses nRF5340 Audio DK for audio) |

---

## Related Pages
- [[nrf54lm20dk-plus-nrf7002eb2]] — Newer platform for comparison
- [[nrf7002dk-vs-nrf54lm20dk]] — Side-by-side comparison
- [[ncs-build-system]] — Build commands and flags
- [[wifi-debugging-patterns]] — WiFi behavior on this board
