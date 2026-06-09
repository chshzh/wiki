---
title: WiFi Debugging Patterns
created: 2026-05-31
updated: 2026-05-31
type: concept
tags: [wifi, debug, pattern, failure, hardware]
sources: [session:d1cdeb42, session:69ca6368, session:bf04a4eb, session:db5832e8]
confidence: high
---

# WiFi Debugging Patterns

## Pattern 1: Timeout Reboot Loop (Wrong Fix)

**Context:** nordic-wifi-memfault on nRF54LM20DK+nRF7002EB2  
**Session:** d1cdeb42 (2026-05-21, 19 turns)  
**Symptom:** Device reboots on WiFi reconnect after first boot  

**Root cause:** When WiFi connection timed out on boot, the code rebooted immediately
instead of retrying with stored credentials.

```
Boot → attempt WiFi connect → timeout → REBOOT (wrong)
Boot → attempt WiFi connect → timeout → retry with stored creds (correct)
```

**Fix:** Change WiFi timeout handler to retry with stored credentials rather than
triggering a reboot. Verify the fix by power-cycling the board and confirming it
reconnects without rebooting on the second boot.

**Lesson:** "Stack overflow" was the initial framing (the session title), but the actual
root cause was control-flow logic, not memory. The reboot-on-timeout looked like a crash.
**Always verify whether a "stack overflow" is actually a panic/reboot from a policy
decision before diving into memory analysis.**

---

## Pattern 2: Provisioner Reset on nRF54LM20DK (Board-Specific Bug)

**Context:** nordic-wifi-memfault on nRF54LM20DK+nRF7002EB2  
**Session:** bf04a4eb (2026-05-15, 1 turn)  
**Symptom:** Device resets when attempting to pair from nRF Wi-Fi Provisioner app.
nRF7002DK does **not** have this issue.

**Board difference:** nRF54LM20DK+nRF7002EB2 uses SPI over a shield connection to the
nRF7002 chip, while the nRF7002DK has it on-board. The shield path adds
initialization timing differences that can expose provisioning race conditions.

**Diagnostic approach:**
1. First confirm the nRF7002DK works with the same firmware — establishes a clean
   baseline
2. Capture UART logs from nRF54LM20DK at the moment of reset
3. Look for provisioner BLE event sequence in logs before the crash

**See also:** [[nrf7002dk-vs-nrf54lm20dk]] for the hardware comparison.

---

## Pattern 3: Headset Autoconnect — Found IP, Won't Connect

**Context:** nordic-wifi-audio on nRF5340 Audio DK  
**Session:** 69ca6368 (2026-05-29, 4 turns)  
**Symptom:** Headset found the target gateway IP but did not automatically connect.

**Diagnostic steps used:**
1. Build and flash **without erase** (`west flash` without `--erase`) to preserve WiFi
   credentials stored in settings partition
2. Capture UART logs from both boards simultaneously (headset + gateway) using VCOM0
3. Compare log timestamps — look for where headset stops progressing

**Key insight:** "Found IP" means discovery works but the connection handshake is
failing silently. Look for:
- Role mismatch (both headset, both gateway)
- Wrong port or unexpected state in the gateway
- Timing: gateway not ready when headset attempts connect

**Board note:** Both nRF5340 Audio DK boards use VCOM0 for UART logging. When
monitoring two boards, use two terminal sessions with explicit port names.

---

## Pattern 4: Wrong WiFi Password (Silent Failure)

**Context:** nordic-wifi-audio on nRF5340 Audio DK  
**Session:** db5832e8 (2026-04-17, 5 turns)  
**Symptom:** WiFi connection fails with no useful error visible in application logs.

**Root cause:** Wrong password in stored credentials.

**Lesson:** WiFi credential errors often surface as generic "connection failed" or
timeout without indicating the cause. If everything looks correct (board powered,
AP in range, firmware functional on another board), test with a known-good AP
or temporarily switch to `CONFIG_WIFI_CREDENTIALS_STATIC` with hardcoded SSID/pass.

---

## Pattern 5: BLE Provisioning with Existing WiFi Credentials

**Context:** nordic-wifi-memfault / nordic-wifi-webdash  
**Session:** de80a0e8 (2025-12-17, 12 turns)  
**Symptom/Topic:** BLE provisioning flow interaction when WiFi credentials already exist.

**Core tension:** The nRF Wi-Fi Provisioner app sends new credentials over BLE. If the
device already has stored credentials, there are two behaviors depending on implementation:
- Overwrite stored credentials immediately → may disconnect active session
- Queue new credentials for next connect → safe but may confuse user

**Lesson:** When debugging provisioning failures, check whether the device has pre-existing
credentials in the settings partition. Use `west flash --erase` to start from a clean
settings partition, then re-test provisioning. If provisioning works on clean flash but
not with existing credentials, the conflict-resolution logic is the issue.

---

## Pattern 6: WiFi P2P Connection Steps

**Context:** nordic-wifi-webdash on nRF7002DK  
**Sessions:** 1f3afd05 (2026-01-15), 89531aad (2026-01-16)  
**Topic:** Connecting two nRF7002DK devices in WiFi P2P (Wi-Fi Direct) mode

**Build command for P2P:**
```sh
west build -p -b nrf7002dk/nrf5340/cpuapp nordic-wifi-webdash \
  -d build_nrf7002dk -- -DSNIPPET=wifi-p2p
```

**P2P connection sequence:**
1. One device becomes Group Owner (GO), other becomes Client
2. GO starts a soft-AP — client discovers it
3. Key: both devices must use the `wifi-p2p` snippet overlay
4. P2P group formation can take 5-15 seconds — wait before declaring it failed

**Common failure:** One device missing the `-DSNIPPET=wifi-p2p` flag — it falls back
to STA mode and can't complete group formation with the GO.

---

## General WiFi Debugging Workflow

```
1. Establish a WORKING BASELINE first
   → Does nRF7002DK work with the same firmware?
   → Does the board work with a simpler WiFi sample?

2. Capture UART logs during the failure
   → Use: screen /dev/tty.usbmodem... 115200
   → Or the chsh-ag-terminal agent for structured capture

3. Identify WHERE in the connect sequence it fails:
   Scan → Association → DHCP → TLS handshake → App connect
   Each phase has distinct log markers.

4. If board-specific: compare nRF7002DK vs nRF54LM20DK behavior
   → Isolates hardware vs firmware issues

5. Check credentials: flash and connect with static credentials to rule out
   stored-credential corruption
```

---

## UART Monitoring Reference

| Board | USB serial port pattern | VCOM index |
|-------|------------------------|------------|
| nRF54LM20DK | `/dev/tty.usbmodem0010518067141` (seen in logs) | VCOM0 |
| nRF7002DK | `/dev/tty.usbmodem<id>` (macOS) | VCOM0 |
| nRF5340 Audio DK | `/dev/tty.usbmodem<id>` | VCOM0 |

**Quick connect:**
```sh
screen /dev/tty.usbmodem<id> 115200
```
Or use Termite / nRF Terminal for logging to file.

---

## Related Pages
- [[nrf7002dk-vs-nrf54lm20dk]] — Board behavior differences that affect WiFi
- [[nrf54lm20dk-plus-nrf7002eb2]] — Board-specific notes
- [[memfault-workflow]] — How to correlate WiFi failures with Memfault crash reports
- [[ncs-build-system]] — Build command for WiFi credential overlays
