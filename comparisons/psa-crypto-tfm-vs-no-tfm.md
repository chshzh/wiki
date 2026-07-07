---
title: PSA Crypto With vs Without TF-M — Mbed TLS 4.x Architecture in NCS v3.4.0
created: 2026-07-07
updated: 2026-07-07
type: comparison
tags: [ncs, crypto, tls, mbedtls, psa, tfm, cracen, nrf54lm20, security]
sources: [https://nrfconnectdocs.nordicsemi.com/ncs/latest/nrf/security/crypto/crypto_architecture.html, https://nrfconnectdocs.nordicsemi.com/ncs/latest/nrf/security/psa_certified_api_overview.html, https://nrfconnectdocs.nordicsemi.com/ncs/latest/nrf/releases_and_maturity/software_maturity.html]
confidence: high
---

# PSA Crypto With vs Without TF-M — Mbed TLS 4.x Architecture in NCS v3.4.0

Mbed TLS 4.x split the library into two layers: the legacy TLS/X.509 API
(`mbedtls_ssl_*`, `mbedtls_x509_*`) now calls into the **PSA Crypto API** for
every underlying crypto primitive, instead of implementing its own
bignum/cipher/hash code directly. This is purely an internal refactor — the
PSA Crypto API is implementation-agnostic about *where* the crypto actually
executes. TF-M is one such implementation, not a requirement.

> "The API will work for applications with and without Trusted Firmware-M (TF-M)."
> — Nordic PSA Certified Security Framework overview

## Two PSA Crypto Implementation Standards in NCS

| | TF-M Crypto Service | Oberon PSA Crypto (no TF-M) |
|---|---|---|
| Execution | PSA core + drivers run isolated in the Secure Processing Environment; app calls in via IPC across the Secure/Non-secure boundary | PSA core runs in-process with the app, no isolation boundary |
| Isolation | Keys/crypto state protected from a compromised non-secure app | None — keys/state live in the same privilege domain as app code |
| RAM/flash cost | Higher — separate SPE partition, IPC framework, duplicated crypto lib | Lower — no partition, no IPC overhead |
| nRF54LM20A support (NCS v3.4.0) | **Experimental** | **Supported** (both CRACEN hw driver and nrf_oberon sw driver) |
| Nordic's own guidance | Needed when isolation from non-secure app bugs is a hard requirement | "Suitable for applications that prioritize simplicity and do not require the additional security isolation provided by TF-M... ideal for resource-constrained applications" |

See [cracen-vs-oberon-tls](cracen-vs-oberon-tls.md) for the orthogonal axis —
which PSA *driver* (hardware CRACEN vs software Oberon) executes the crypto.
That choice is independent of TF-M: both drivers work with or without it.

## Not Using TF-M Is a Supported, Recommended Choice on RAM-Constrained Boards

This is not a workaround with hidden gaps — it is the configuration Nordic
explicitly designed for memory-constrained apps, and TF-M is only
"Experimental" (not yet "Supported") on nRF54LM20A anyway as of NCS v3.4.0.

Confirmed via `nordic-wifi-memfault`'s actual build config (nRF54LM20A/cpuapp,
Secure-only target, no `/ns` variant):

```
CONFIG_MBEDTLS_PSA_CRYPTO_C=y
CONFIG_MBEDTLS_PSA_CRYPTO_DRIVERS=y
CONFIG_PSA_CRYPTO_DRIVER_CRACEN=y
# CONFIG_BUILD_WITH_TFM is not set at all
```

This project has run WPA2/WPA3 Wi-Fi auth plus concurrent mbedTLS TLS
sessions (HTTPS, MQTT, Memfault uploads) throughout an entire debugging
session with no TF-M — a working existence proof on this exact chip family.

## What You Actually Give Up Without TF-M

**Isolation, not functionality.** Without TF-M, PSA Crypto state and key
material live in the same privilege domain as application code — a
memory-corruption bug in app code could theoretically reach crypto internals.
With TF-M, that is walled off in the Secure world. This is a
defense-in-depth trade-off, not a capability loss.

**Persistent key storage backend differs, not disappears.** If the app uses
`psa_set_key_lifetime(PSA_KEY_LIFETIME_PERSISTENT, ...)`, without TF-M the
storage backend becomes Zephyr's Secure Storage subsystem or NCS's Trusted
Storage library instead of TF-M's Internal Trusted Storage (ITS). Both still
provide confidentiality/integrity — just without TF-M's Platform Root of
Trust mediating access.

**Hardware-wrapped keys still work without TF-M.** CRACEN's KMU (Key
Management Unit) hardware key slots are directly accessible via
`PSA_KEY_LOCATION_CRACEN_KMU` without TF-M — real hardware-backed keys do not
require TF-M on nRF54L/LM parts. One nuance: without TF-M, the KMU driver's
temporary key-material push buffer during an operation lives in a fixed
non-secure RAM region (`0x20000000`–`0x20000060`, `kmu_push_area`) rather than
inside the Secure enclave — a minor isolation detail per Nordic's KMU
programming model docs, not a functional block. A community devzone thread
(Andreas u-blox, nRF54L10) reports `psa_generate_key()` returning
`PSA_ERROR_NOT_SUPPORTED` (-134) for KMU-backed persistent keys — if pursuing
this path, verify the exact `psa_set_key_id()`/usage-scheme Kconfig
combination against Nordic's KMU guide rather than assuming it "just works".

## Cryptographic Support Matrix — nRF54LM20A (NCS v3.4.0)

| Implementation | Status |
|---|---|
| Oberon PSA Crypto — CRACEN (hardware) | Supported |
| Oberon PSA Crypto — nrf_oberon (software) | Supported |
| TF-M Crypto Service | Experimental |
| nrf_cc3xx | Not available (nRF54L series has no CryptoCell) |

## Bottom Line

Given RAM constraints, skipping TF-M and using Oberon PSA Crypto (+ CRACEN
hardware driver where available) is the recommended path on nRF54L/LM
devices — not a compromise with hidden functional gaps. Only reach for TF-M
if a hard isolation requirement exists (e.g. certified secure-boot/PSA
Certified Level product requirement) and accept it is still experimental on
this chip family as of NCS v3.4.0.

## Related Pages

- [cracen-vs-oberon-tls](cracen-vs-oberon-tls.md) — the orthogonal hardware-vs-software driver axis
- [ncs-version-migration](../concepts/ncs-version-migration.md) — Mbed TLS 4.1.0 / TF-PSA-Crypto migration notes (v3.3.0→v3.4.0)
- [ncs-build-system](../concepts/ncs-build-system.md) — `MBEDTLS_LEGACY_CRYPTO_C` and PSA-only board Kconfig patterns
