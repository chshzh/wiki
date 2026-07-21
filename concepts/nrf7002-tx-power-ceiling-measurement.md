---
title: nRF7002 TX Power Ceiling Measurement (EVM-Based Method)
created: 2026-07-17
updated: 2026-07-17
type: concept
tags: [wifi, hardware, ncs, rf-compliance, tx-power]
sources: [raw/articles/nrf7002-tx-power-ceiling-evm-support-answer-2026-07-17.md]
confidence: medium
---

# nRF7002 TX Power Ceiling Measurement (EVM-Based Method)

The nRF7002 datasheet tabulates a nominal max TX power of **13 dBm**, but the *actual*
ceiling applied at runtime is firmware-controlled and board-specific — determining the real
achievable max power for a given board design requires an empirical EVM sweep, not just
reading the datasheet number. This is documented in Nordic app note **NAN_043**.

## How the firmware TX power ceiling works

> "The TX power ceiling applied by the firmware for system operation is the smallest of the
> default TX power ceiling and the board ceiling as specified in the devicetree overlay file.
> To use the highest possible TX power ceiling, set all board ceilings to a value greater than
> or equal to the default ceilings."
> — [NAN_043: TX power ceilings](https://docs.nordicsemi.com/r/bundle/nan_043/page/app/nan_043/tx_power_ceilings.html)

In other words, two independent ceilings exist and firmware always applies `min(default, board)`:
- **Default TX power ceiling** — built into the firmware/regulatory tables.
- **Board ceiling** — set per-board in the devicetree overlay file, meant to cap power for a
  specific PCB/antenna design (e.g. to stay within a regulatory or thermal limit for that design).

To measure the *chip's* maximum achievable power rather than a board-imposed cap, all board
ceilings must first be raised to be ≥ the default ceilings — otherwise the board overlay value,
not the true maximum, is what gets measured.

## Measurement procedure

Nordic's documented procedure ([NAN_043: max TX power procedure and commands](https://docs.nordicsemi.com/r/bundle/nan_043/page/app/nan_043/max_tx_power_procedure_and_commands.html)):

1. Start at a high TX power setting (above the expected ceiling).
2. Measure SEM (Spectral Emission Mask) and EVM (Error Vector Magnitude).
3. If either fails, step down by **1 dB** and re-measure.
4. Repeat until both SEM and EVM pass — the last passing power level is `TXPWR_max` for that
   board/antenna combination.

This is a standard step-down characterization sweep: rather than trusting the datasheet's
nominal number, it empirically finds the actual ceiling for the specific PCB under test.

## EVM pass/fail thresholds

The EVM limits used as the pass/fail criteria are per-modulation, defined by
[NAN_043: TX performance](https://docs.nordicsemi.com/r/bundle/nan_043/page/app/nan_043/tx_performance.html)
and ultimately traceable to the IEEE 802.11 standard. For example:

- **MCS0** (BPSK, 1/2 code rate) — IEEE 802.11 EVM requirement: **−5 dB**.

Lower (more negative) EVM in dB is better/stricter; higher-order modulations (higher MCS) have
tighter — i.e. more negative — EVM requirements, which is why the max achievable TX power
tends to be lower at higher MCS indices even on the same board.

## Related

- [nrf7002dk](../entities/nrf7002dk.md) — board this measurement applies to
- [fcc-ce-target](fcc-ce-target.md) — why board-level TX power/RF characterization doesn't
  transfer between boards sharing the same chip
- [fcc-ce-conducted-radiated-rf-test-methods](fcc-ce-conducted-radiated-rf-test-methods.md) —
  conducted vs. radiated measurement methodology this procedure typically runs under
