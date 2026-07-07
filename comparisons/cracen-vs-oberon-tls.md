---
title: CRACEN vs Oberon — TLS Crypto Backends on nRF54LM20DK and nRF7002DK
created: 2026-05-13
updated: 2026-07-07
type: comparison
tags: [ncs, crypto, tls, cracen, oberon, nrf54lm20, nrf5340, mbedtls, psa, stack]
sources: []
confidence: high
---

# CRACEN vs Oberon — TLS Crypto Backends on nRF54LM20DK and nRF7002DK

In NCS projects that run TLS (HTTPS, MQTT-TLS, OTA download) on Wi-Fi, the crypto
backend is selected automatically at build time based on the SoC. This page documents
the differences observed in `nordic-wifi-memfault` and the NCS `nrf_security` subsystem.

## Backend Selection

Selection is fully automatic — no Kconfig to set manually.

```
# nrf/subsys/nrf_security/src/drivers/cracen/Kconfig
config HAS_HW_NRF_CRACEN
    def_bool (SOC_SERIES_NRF54L && !SOC_NRF54LS05A && !SOC_NRF54LS05B) || SOC_SERIES_NRF71

# nrf/subsys/nrf_security/src/drivers/Kconfig
config PSA_CRYPTO_DRIVER_CRACEN
    default y          # when HAS_HW_NRF_CRACEN

config PSA_CRYPTO_DRIVER_OBERON
    default y if !HAS_HW_NRF_CRACEN
```

| Board | SoC | Backend |
|---|---|---|
| nRF54LM20DK + nRF7002EB2 | nRF54LM20A (`SOC_SERIES_NRF54L`) | **CRACEN** (hardware) |
| nRF7002DK | nRF5340 | **Oberon** (software library) |

## What CRACEN Is

CRACEN is an on-chip hardware crypto IP core in nRF54L-series SoCs with two sub-engines:

| Sub-engine | Accelerates |
|---|---|
| **sxsymcrypt** (CryptoMaster) | AES-GCM/CBC/CTR, SHA-2/3, HMAC, ChaCha20 |
| **silexpk / BA414 PKE** | ECC scalar multiply, ECDSA, ECDH, RSA mod-exp |

CRACEN on nRF54LM20A is the **LITE variant** (`CRACEN_HW_VERSION_LITE`), distinct from
the BASE variant on nRF54L15. It has several hardware workarounds active at build time:
`PSA_NEED_CRACEN_MULTIPART_WORKAROUNDS`, `PSA_NEED_CRACEN_IKG_INTERRUPT_WORKAROUND`,
`PSA_NEED_CRACEN_MEMORY_ACCESS_WORKAROUND`.

**Oberon** is a prebuilt software library (ARM Cortex-M optimized) that implements
the same PSA API entirely in CPU code — no hardware offload, no kernel events.

## Execution Model During TLS Handshake

The same application code and PSA API calls are used on both boards. The call depth differs.

**CRACEN path (nRF54LM20DK):**
```
mbedtls_ssl_handshake()
  └─ psa_sign_hash() / psa_key_agreement()
       └─ cracen_psa_asymmetric_sign() / cracen_psa_key_agreement()
            └─ sx_pk_* [submit to BA414 PKE hardware]
                 └─ cracen_wait_for_pke_interrupt()
                      └─ nrf_security_event_wait()
                           └─ k_event_wait_internal()  ← thread SUSPENDED here, CPU free
```

**Oberon path (nRF7002DK):**
```
mbedtls_ssl_handshake()
  └─ psa_sign_hash() / psa_key_agreement()
       └─ oberon_ecdsa_sign() / oberon_ecdh_key_agreement()
            └─ [pure CPU arithmetic — returns immediately, CPU busy throughout]
```

## Stack Depth Impact

The interrupt-wait call chain adds stack depth on CRACEN. Measured values from
`nordic-wifi-memfault` thread analyzer (post-OTA run):

| Source | Extra depth |
|---|---|
| AES record operation (CryptoMaster) | ~+600 B |
| ECDSA/ECDH handshake (BA414 PKE) | ~+1500 B |

This is why `prj.conf` is sized for CRACEN as the baseline, and
`boards/nrf7002dk_nrf5340_cpuapp.conf` reduces most TLS thread stacks:

| Thread | nRF54LM20DK (CRACEN) | nRF7002DK (Oberon) |
|---|---|---|
| `mqtt_helper_thread` | 8192 | 4096 |
| `mflt_ota_triggers` | 10240 | 8192 |
| `memfault_http` | 8192 | 8192 |
| `downloader` | 8192 | 6144 |
| `app_https_client` | 8192 | 6144 |
| `app_mqtt_client` | 8192 | 6144 |

Note: nRF7002DK OTA/downloader threads are **larger than the CRACEN-free baseline**
because `CONFIG_MBEDTLS_ECDSA_C=y` (needed for ECDHE-ECDSA cipher suites) pushes
deep arithmetic recursion in software — the OTA thread overflowed at 6144 B without
the raise to 8192.

## Benefits of CRACEN Over Software Oberon

Despite the larger stack allocations, CRACEN wins on every operational dimension:

| Dimension | nRF7002DK (Oberon) | nRF54LM20DK (CRACEN) |
|---|---|---|
| TLS handshake time | ~300–500 ms | ~5–15 ms |
| CPU during TLS | 100% occupied | Free (thread suspended) |
| Key storage | Keys in RAM (readable) | KMU: keys in hardware, never in CPU |
| RNG source | CC312/Oberon PRNG | Hardware TRNG in CRACEN |
| Side-channel hardening | Software countermeasures | Hardware countermeasures |
| Power per TLS handshake | High (long CPU time) | Low (short burst, HW efficient) |

The KMU (Key Management Unit) is activated by:
```
CONFIG_MBEDTLS_PSA_CRYPTO_BUILTIN_KEYS=y   # auto-set when HAS_HW_NRF_CRACEN
```
Keys provisioned into the KMU are never readable by software — not by the app,
not by a debugger, not by a compromised firmware image.

## The RAM "Cost" Is Not What It Seems

The larger stacks are **one-time allocations** that sit mostly idle. While the CRACEN
thread is suspended at `k_event_wait_internal()`, those stack bytes are unused and
the CPU runs other threads. On nRF5340, shallower stacks are fully consumed during
the entire duration of software math — the CPU cannot run anything else.

The 108 KB `MBEDTLS_HEAP_SIZE` is identical on both boards. On nRF54LM20DK, CRACEN
offloads working buffers (ECC point coordinates, intermediate hashes) into its own
hardware SRAM — reducing heap pressure during handshakes.

## No Application Code Changes Required

The PSA Crypto API (`psa_sign_hash`, `psa_key_agreement`, `psa_cipher_encrypt`, …)
is identical on both boards. The only differences between boards are:
1. Kconfig (NCS auto-selects the right PSA driver based on SoC)
2. Stack sizes in `boards/<board>.conf`

## Related Pages

- [embedded-system-general-debugging](../concepts/embedded-system-general-debugging.md)
- [memfault-version-requirements](../concepts/memfault-version-requirements.md)
- [psa-crypto-tfm-vs-no-tfm](psa-crypto-tfm-vs-no-tfm.md) — the orthogonal TF-M-isolation-vs-not axis (independent of CRACEN/Oberon driver choice)
