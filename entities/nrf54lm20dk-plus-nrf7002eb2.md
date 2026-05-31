---
title: nRF54LM20DK + nRF7002EB2
created: 2026-05-31
updated: 2026-05-31
type: entity
entity_type: board
tags: [hardware, ncs, wifi, build, debug, failure]
sources: [session:d1cdeb42, session:bf04a4eb]
confidence: high
---

# nRF54LM20DK + nRF7002EB2

**DK PCA number:** PCA10184  
**SoC:** nRF54LM20A  
**WiFi:** nRF7002EB2 shield (connected via SPI — not QSPI like the DK)  
**Primary use:** Newer platform validation, Memfault fleet expansion

---

## Board Target String

```sh
-b nrf54lm20dk/nrf54lm20a/cpuapp -DSHIELD=nrf7002eb2
```

The `nrf7002eb2` shield is **always required** — the nRF7002 is on a separate board.

---

## Typical Build Command

```sh
west build -p -b nrf54lm20dk/nrf54lm20a/cpuapp nordic-wifi-memfault \
  -d build_nrf54lm20dk -- \
  -DSHIELD=nrf7002eb2 \
  -DEXTRA_CONF_FILE=overlay-app-memfault-project-info.conf
```

---

## USB Serial Port

Pattern on macOS: `/dev/tty.usbmodem0010518067141` (confirmed serial in session d1cdeb42)  
Use VCOM0 at 115200 baud for application logs.

---

## Known Issues and Behavior Notes

### ⚠️ Provisioner Reset Bug
**Status:** Confirmed (2026-05-15, session bf04a4eb)  
**Symptom:** Device resets when attempting to pair from nRF Wi-Fi Provisioner app.  
**nRF7002DK does NOT have this issue.**

Root cause is not fully isolated but believed to be initialization timing via the SPI
shield path (vs on-board QSPI on nRF7002DK). When troubleshooting provisioner issues,
always compare against nRF7002DK first to determine if it's board-specific.

### WiFi Timeout Reboot (Fixed)
**Status:** Fixed (2026-05-21, session d1cdeb42)  
**Symptom:** Device rebooted on WiFi reconnect after first boot.  
**Root cause:** Control flow in WiFi timeout handler called reboot instead of retry.  
**Fix:** Modified handler to retry with stored credentials.

This presented as a "stack overflow" in initial framing but was actually a logic bug.
See [[wifi-debugging-patterns]] for the pattern.

---

## Hardware Architecture Difference (vs nRF7002DK)

| Aspect | nRF7002DK | nRF54LM20DK + nRF7002EB2 |
|--------|-----------|--------------------------|
| nRF7002 connection | QSPI (on-board) | SPI (shield) |
| Interface speed | Higher | Lower |
| Timing | Integrated | Shield adds latency |
| Known WiFi issues | Stable | Provisioner reset bug |

The SPI-vs-QSPI difference is significant for throughput. See [[nrf7002dk-vs-nrf54lm20dk]].

---

## Project Support

| Project | Status |
|---------|--------|
| nordic-wifi-memfault | Active — both builds maintained |
| nordic-wifi-webdash | Active — both builds maintained |
| nordic-wifi-audio | Not applicable (different DK) |

---

## Related Pages
- [[nrf7002dk]] — Reference board for baseline comparison
- [[nrf7002dk-vs-nrf54lm20dk]] — Detailed comparison
- [[wifi-debugging-patterns]] — Board-specific WiFi failure patterns
- [[ncs-build-system]] — Build command with shield flag
