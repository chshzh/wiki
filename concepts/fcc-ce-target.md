---
title: FCC/CE Target — What Actually Needs Certification
created: 2026-07-17
updated: 2026-07-17
type: concept
tags: [regulatory, fcc, ce, rf-compliance, wifi, hardware, ncs]
sources: [raw/articles/fcc-ce-wifi-requirements-claude-chat-2026-07-17.md]
confidence: medium
---

# FCC/CE Target — What Actually Needs Certification

FCC (US) and CE (EU) RF-compliance requirements are triggered by the same underlying test:
is the product an **"intentional radiator"** — does it deliberately transmit RF energy and
form a *complete* transmitter with an antenna that can actually radiate? The trigger is not
"does it contain a Wi-Fi chip."

## FCC side (US)

- An intentional radiator needs equipment authorization before it can be marketed — for Wi-Fi
  this is the **Certification** procedure, which grants an **FCC ID**.
- **Modular transmitter approval**: a self-contained RF module (chip + matching network +
  antenna + shielding, built to be reused across host designs) can get its own FCC ID. A host
  manufacturer using a certified module skips full technical compliance testing/filing for the
  radio portion — but still needs its own authorization for unintentional-radiator emissions
  (Part 15 Subpart B), and must put a "Contains FCC ID" label on the finished product.
- **Developmental-device carve-out**: equipment still in the conceptual/developmental/design/
  pre-production stage can be marketed to business/commercial/industrial/scientific buyers (not
  consumers), provided the buyer is told in writing it isn't yet FCC-authorized and will be
  brought into compliance before general distribution. This is the regulatory hook that lets
  dev kits and eval boards ship to engineers without full Certification.

## CE side (EU)

The EU Radio Equipment Directive (RED) works differently in mechanics, same in spirit: the
manufacturer placing a radio product on the market tests it against the relevant harmonized
standards —
- RF: EN 300 328 / EN 301 893 (Wi-Fi bands) — see [[fcc-ce-conducted-radiated-rf-test-methods]]
- EMC: EN 301 489
- Safety: EN 62368-1

— and self-issues a **Declaration of Conformity (DoC)**. There's no formal "modular grant"
system like the FCC's, but module vendors' test reports get reused by the host manufacturer in
practice, because the host manufacturer is who legally has to sign the DoC.

## Chip vs. module vs. end product

| Layer | What it is | Certification |
|-------|-----------|----------------|
| Bare chip (e.g. nRF7002 IC) | Not a complete transmitter — no antenna, no fixed matching network | Nothing to certify on its own |
| Module | Chip + matching network + antenna + shielding, packaged as a reusable, independently-testable RF building block | Gets the FCC modular grant / CE test report downstream customers rely on |
| End product | Whatever the customer ships — bare chip + own RF layout, or a pre-certified module | Manufacturer's own authorization/DoC for the whole assembly |

Nordic sells the chip. It's typically module-making partners — or customers doing their own
PCB/antenna design — who build the module layer and carry its certification.

## Why "same chip" doesn't mean "same compliance"

FCC/CE compliance is a property of the **whole RF assembly** (PCB layout, matching network,
antenna, enclosure) — not the chip. Two boards using the same chip but different
antennas/layouts are two different RF designs, and neither's test data transfers to the other.

Concretely, on [[nrf7002dk]] vs [[nrf54lm20dk-plus-nrf7002eb2]] (EB II): FCC testing on the DK
was likely pre-compliance/reference validation for that specific reference design — not a
Certification grant covering "the nRF7002" as a part number. EB II's own RF compliance hasn't
been separately characterized; it inherits the chip's RF performance in principle, but not any
board-level certification. A customer shipping EB II's design (or something close) in an end
product would need to test/certify that specific PCB and antenna implementation themselves.
This is also why dev kits and eval boards are marketed under the developmental-device
provision rather than run through full Certification — they aren't meant to end up as retail
radio products, so partners building a purpose-made reusable module around the chip are the
ones positioned to get the modular grant / CE report.

## Caveat

Not legal advice. The exact boundary of "developmental marketing" under EU RED enforcement
varies more by member state than the FCC's rule does. Before this goes into official
customer-facing documentation, sanity-check with Nordic's regulatory/compliance team —
especially on the CE side.

## Related Pages
- [fcc-ce-conducted-radiated-rf-test-methods](fcc-ce-conducted-radiated-rf-test-methods.md) — how the RF portion of certification is actually measured
- [../entities/nrf7002dk.md](../entities/nrf7002dk.md) — board this FCC/CE discussion originated from
- [../entities/nrf54lm20dk-plus-nrf7002eb2.md](../entities/nrf54lm20dk-plus-nrf7002eb2.md) — EB II, the board whose compliance status prompted this page
- [wifi-debugging-patterns](wifi-debugging-patterns.md) — sibling Wi-Fi/hardware concept page
