---
title: Wi-Fi Power Save — Listen Interval vs DTIM Wakeup (nRF70)
created: 2026-07-15
updated: 2026-07-15
type: concept
tags: [wifi, power-save, nrf7002, debug, pattern]
sources: []
confidence: high
---

# Wi-Fi Power Save — Listen Interval vs DTIM Wakeup (nRF70)

Empirically derived (via nRF7002DK + Power Profiler, `wifi-fund/l6` Lesson 6
exercise) rule for what actually determines the nRF70 STA's power-save wake
cadence, since the configured values do **not** map to wake period as naively
expected.

## Background

Two independent knobs exist on the nRF70 driver (`enum wifi_ps_wakeup_mode` in
[`zephyr/include/zephyr/net/wifi.h`](../../v3.1.0/zephyr/include/zephyr/net/wifi.h)):

- `WIFI_PS_PARAM_WAKEUP_MODE`: `WIFI_PS_WAKEUP_MODE_DTIM` vs
  `WIFI_PS_WAKEUP_MODE_LISTEN_INTERVAL` — nominally selects whether the STA
  wakes on DTIM beacons or on a configured `listen_interval` (units: beacon
  intervals, set via `WIFI_PS_PARAM_LISTEN_INTERVAL` / shell
  `wifi ps_listen_interval <n>`).
- `WIFI_PS_PARAM_EXIT_STRATEGY`: `WIFI_PS_EXIT_CUSTOM_ALGO` (`custom`, default)
  vs `WIFI_PS_EXIT_EVERY_TIM` (`tim`) — governs when the firmware actually
  decides to exit sleep.

## Key finding: listen_interval is quantized to whole DTIM periods

Regardless of `exit_strategy`, the real measured wake period is:

```
wake_period = round(listen_interval / dtim_period) * dtim_period * beacon_interval
```

where `beacon_interval` must be read from `wifi status` (field `Beacon Interval`)
and converted from **TU to ms**: `beacon_interval_ms = reported_TU * 1.024`
(802.11 Beacon Interval field is in Time Units, 1 TU = 1.024ms — NOT 1ms as
commonly assumed).

Verified on an ASUS RT-BE92U AP (5GHz, `Beacon Interval: 200` TU → 204.8ms
real, `DTIM: 3`):

| `listen_interval` | nearest DTIM multiple | predicted wake period | measured (Power Profiler) |
|---|---|---|---|
| n/a (`WIFI_PS_WAKEUP_MODE_DTIM`) | 1 × 3 | 614.4ms | ~614–620ms |
| 3–5 | 1 × 3 | 614.4ms | ~615ms |
| 6–8 | 2 × 3 | 1228.8ms | (predicted, untested) |
| 9–11 (default was 10) | 3 × 3 | 1843.2ms | ~1846–1856ms |

**Why:** broadcast/multicast frames are only buffered by the AP until the next
DTIM beacon, so the STA gains nothing by waking on a non-DTIM-aligned beacon —
the firmware snaps any requested listen interval to the nearest whole number
of DTIM periods it can actually align sleep to.

## exit_strategy interaction

- `custom` (default): firmware uses the DTIM-quantization rule above; measured
  values for `LISTEN_INTERVAL` and `DTIM` wakeup modes can differ if the
  quantized listen interval resolves to a different DTIM multiple than 1.
- `tim` (`WIFI_PS_EXIT_EVERY_TIM`): forces wake at **every** DTIM beacon
  (`1 × dtim_period × beacon_interval`) regardless of `wakeup_mode` or
  `listen_interval` — this makes `DTIM` and `LISTEN_INTERVAL` modes converge
  to the same measured cadence. Switching exit strategy did **not** change the
  listen-interval measurement in testing (it was already at the same DTIM
  multiple), which is what exposed the quantization rule rather than an
  exit-strategy effect.

## Diagnostic recipe

1. `wifi status` → get real `Beacon Interval` (TU, ×1.024 for ms) and `DTIM`.
2. `wifi ps` → get current `PS listen_interval`, `PS wake up mode`,
   `PS exit strategy`.
3. Compute `round(listen_interval / dtim_period) * dtim_period * beacon_interval_ms`.
4. Measure actual wake cadence with Power Profiler (no application traffic
   during the capture — any socket activity keeps the radio active and skews
   the reading).
5. If measured ≠ predicted, check exit_strategy first (`custom` vs `tim`) and
   confirm no idle-traffic contamination before assuming a firmware anomaly.

## Cross-references

See [wifi-debugging-patterns](wifi-debugging-patterns.md) for other nRF70
STA-mode failure/behavior patterns, and
[nrf7002dk](../entities/nrf7002dk.md) for board-specific notes.
