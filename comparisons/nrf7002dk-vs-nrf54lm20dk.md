---
title: nRF7002DK vs nRF54LM20DK+nRF7002EB2 — Side-by-Side Comparison
created: 2026-05-31
updated: 2026-05-31
type: comparison
tags: [hardware, wifi, debug, pattern]
sources: [session:d1cdeb42, session:bf04a4eb, session:50a3219c]
confidence: high
---

# nRF7002DK vs nRF54LM20DK + nRF7002EB2

## Quick Reference

| Property | nRF7002DK | nRF54LM20DK + nRF7002EB2 |
|----------|-----------|--------------------------|
| PCA | PCA10143 | PCA10184 |
| SoC | nRF5340 | nRF54LM20A |
| WiFi chip | nRF7002 (on-board) | nRF7002 (EB2 shield) |
| WiFi interface | QSPI | SPI |
| Board target | `nrf7002dk/nrf5340/cpuapp` | `nrf54lm20dk/nrf54lm20a/cpuapp` |
| Shield flag | Not needed | `-DSHIELD=nrf7002eb2` required |
| Provisioner stability | ✅ Stable | ⚠️ Reset bug (confirmed 2026-05) |
| WiFi reconnect | ✅ Stable | Fixed in firmware (2026-05) |
| Generation | Older | Newer |

---

## Throughput: SPI vs QSPI

**Session:** 50a3219c (2026-01-06, title: "Throughput differences between nRF54 devices")

The nRF54LM20DK uses SPI to connect to the nRF7002EB2 shield, while the nRF7002DK
uses QSPI (Quad SPI) for the on-board chip. QSPI has ~4x higher theoretical bandwidth.

**Implication:** For WiFi throughput benchmarks (iperf/zperf), expect lower throughput
numbers on nRF54LM20DK. If optimizing throughput, the sQSPI interface (if available on
the platform) can close the gap. See the WCS-121 test suite results for measured numbers.

---

## When a Bug is Board-Specific

**Rule:** When a WiFi issue appears on nRF54LM20DK but NOT on nRF7002DK with the same
firmware, suspect:

1. **SPI timing** — the shield interface has different initialization sequences
2. **Shield driver** — nRF7002EB2 has its own board support code
3. **CPU architecture difference** — nRF54LM20A vs nRF5340 may handle timing differently
4. **Power sequencing** — the shield requires external power rails that may not be stable
   at the exact point the driver initializes

**Confirmed board-specific issues (2026):**
- Provisioner reset bug (session bf04a4eb) — nRF7002DK unaffected

---

## Debugging Strategy: Always Baseline on nRF7002DK

When encountering a WiFi issue on nRF54LM20DK:

1. Flash identical firmware to nRF7002DK
2. Reproduce or confirm it doesn't occur on nRF7002DK
3. If DK works: board-specific path (SPI/shield/driver)
4. If DK also fails: firmware logic bug (both boards affected)

This two-board strategy saved significant debugging time in sessions d1cdeb42 and bf04a4eb.

---

## Build Differences

```sh
# nRF7002DK — no shield needed
west build -p -b nrf7002dk/nrf5340/cpuapp <app> -d build_nrf7002dk

# nRF54LM20DK — shield required
west build -p -b nrf54lm20dk/nrf54lm20a/cpuapp <app> -d build_nrf54lm20dk \
  -- -DSHIELD=nrf7002eb2
```

---

## When to Use Which Board

| Use case | Recommended |
|----------|-------------|
| Quick dev/test | nRF7002DK (simpler, no shield) |
| Production validation | Both boards |
| Provisioner testing | nRF7002DK baseline first, then nRF54LM20DK |
| Throughput benchmarks | Both — document which was used |
| Memfault crash debug | nRF7002DK (more stable for reproducible testing) |

---

## Related Pages
- [[nrf7002dk]] — nRF7002DK entity page
- [[nrf54lm20dk-plus-nrf7002eb2]] — nRF54LM20DK entity page with known issues
- [[wifi-debugging-patterns]] — Failure patterns and which board they occur on
- [[ncs-build-system]] — Build commands for both boards
