---
title: FCC/CE Conducted and Radiated RF Test Methods
created: 2026-07-17
updated: 2026-07-17
type: concept
tags: [regulatory, fcc, ce, rf-compliance, rf-test, wifi, hardware, ncs]
sources: [raw/articles/fcc-ce-wifi-requirements-claude-chat-2026-07-17.md]
confidence: medium
contested: true
---

# FCC/CE Conducted and Radiated RF Test Methods

Both FCC and CE RF-compliance testing split into two measurement types. Which one applies —
or whether both are required — depends mainly on whether the RF output is accessible via a
connector. See [[fcc-ce-target]] for what triggers certification in the first place.

## Conducted testing

Preferred method where available: connect directly to the device's RF output via a
cable/attenuator, bypassing the antenna entirely. This isolates the transmitter's own
performance from the antenna's influence.

Under FCC (**ANSI C63.10**) and **ETSI EN 300 328**, conducted measurements cover:
- Fundamental transmit power at the antenna port
- Occupied bandwidth (typically the 99% method)
- Unwanted/spurious and harmonic emissions, checked against out-of-band limits

## Radiated (non-conducted / over-the-air) testing

Used when there's no accessible RF connector — output is estimated from antenna
characteristics and radiated performance instead. EN 300 328 is explicit that this method
applies only to integral-antenna equipment without a temporary antenna connector. The
resulting **EIRP** (effective isotropic radiated power) is measured and recorded at a proper
test site.

On the FCC side, the same fundamental power gets measured both ways — conducted at the antenna
port, and as radiated EIRP using calibrated antennas at an accredited facility — plus a
separate radiated pass purely for spurious/harmonic emissions.

## Why you often need both, even with a connector

Having an antenna connector lets you skip radiated testing for the *fundamental output power*
(conducted power + known antenna gain converts to EIRP mathematically). But **radiated
spurious/harmonic testing is frequently still required**, because those emissions can leak from
PCB traces and the enclosure itself, not just out the antenna port — a cable connected only at
the RF output won't catch them. Real test reports list radiated spurious spot checks performed
to confirm the module remains compliant once installed in the host, in addition to conducted
measurements.

## MIMO / multi-antenna provisions (general ETSI rule — does not apply to nRF7002)

ETSI EN 300 328 has specific provisions for smart-antenna systems using symmetrical power
distribution across multiple transmit chains: the standard requires the device be configured
with only one active chain where possible, and if that's not possible, the tested chain's
result gets mathematically scaled up to represent the full system. Conducted per-chain
measurement plus a documented correction can substitute for a full radiated multi-chain test.

> **Correction to source material:** the conversation this page was ingested from claimed this
> provision was "relevant to nRF7002, which supports MIMO." That's wrong — **nRF7002 is a
> single-antenna (SISO) Wi-Fi companion IC, not MIMO-capable.** The EN 300 328 multi-chain
> provision above is a real rule but doesn't apply to nRF7002-based designs. Flagged
> 2026-07-17; the immutable raw source in `raw/articles/` still contains the original
> (incorrect) claim, per wiki convention — corrections live here, not in raw/.

## Tying back to nRF7002 / EB II / DK

- **Conducted** characterization of the chip's transmitter (power, PSD, in-band spurious) can be
  done once, off a test point/U.FL connector, largely independent of which board it sits on —
  genuinely reusable data across [[nrf7002dk]] and [[nrf54lm20dk-plus-nrf7002eb2]] (EB II).
- **Radiated** performance (actual EIRP, radiated spurious/harmonics) depends on the antenna,
  PCB ground plane, trace routing, and enclosure of that *specific* board — the DK's radiated
  numbers don't transfer to EB II even with an identical chip, because EB II is a different
  physical RF assembly. Shared silicon ≠ shared radiated compliance data. See [[fcc-ce-target]]
  for the certification-scope reasoning behind this.

## Related Pages
- [fcc-ce-target](fcc-ce-target.md) — what triggers FCC/CE certification and who certifies what
- [../entities/nrf7002dk.md](../entities/nrf7002dk.md) — SISO board referenced throughout this page
- [../entities/nrf54lm20dk-plus-nrf7002eb2.md](../entities/nrf54lm20dk-plus-nrf7002eb2.md) — EB II, whose radiated data doesn't transfer from the DK
- [wifi-debugging-patterns](wifi-debugging-patterns.md) — sibling Wi-Fi/hardware concept page
