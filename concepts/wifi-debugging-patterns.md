---
title: WiFi Debugging Patterns
created: 2026-05-31
updated: 2026-07-01
type: concept
tags: [wifi, debug, pattern, failure, hardware]
sources: [session:d1cdeb42, session:69ca6368, session:bf04a4eb, session:db5832e8, session:54f9d0bc, https://nrfconnectdocs.nordicsemi.com/ncs/latest/nrf/protocols/wifi/sap_mode/sap.html]
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

## Pattern N+1: P2P_CLIENT Static IP Slot Conflict

**Context:** nordic-wifi-app-template on nRF54LM20DK+nRF7002EB2 (P2P_CLIENT mode)
**Session:** 5317c7d8 (2026-06-13)
**Symptom:** `net_if_ipv4_addr_add(iface, "192.168.7.2", MANUAL)` returns NULL immediately after connect; DHCP fallback shows `FAIL1: net_ipv4_create` in infinite loop.

**Root cause 1 — IPv4 slot occupied by boot-assigned address:**
`CONFIG_NET_IF_MAX_IPV4_COUNT=1` (only one IPv4 address slot per interface).
`net_config_init` assigns `CONFIG_NET_CONFIG_MY_IPV4_ADDR` ("192.168.7.1") as `NET_ADDR_OVERRIDABLE` at boot, filling the only slot. Static IP add for "192.168.7.2" then fails because the `rm("192.168.7.2")` finds nothing (the slot has "192.168.7.1"), leaving the slot occupied.

**Fix:** Before `net_if_ipv4_addr_add`, explicitly `rm(CONFIG_NET_CONFIG_MY_IPV4_ADDR)` to clear the boot-assigned address, then `rm(client_ip)` for reconnect cleanup:
```c
zsock_inet_pton(AF_INET, "192.168.7.1", &cfg_ip);  /* = CONFIG_NET_CONFIG_MY_IPV4_ADDR */
net_if_ipv4_addr_rm(iface, &cfg_ip);
net_if_ipv4_addr_rm(iface, &client_ip);
```

**Root cause 2 — DHCP restarted by wpa_supplicant after static IP stop:**
`driver_zephyr.c` calls `net_dhcpv4_restart(iface)` from the `hostap_handler` thread ~4 ms after CONNECT_RESULT fires (inside `set_supp_port(authorized=true)`). Any `net_dhcpv4_stop` in the CONNECT_RESULT callback is overwritten 4 ms later.

**Fix:** Schedule a deferred `net_dhcpv4_stop` work item with 100 ms delay from the CONNECT_RESULT callback. This fires after the supplicant's restart and keeps DHCP permanently stopped:
```c
/* In net_event_mgmt.c after static IP assignment succeeds: */
k_work_reschedule(&dhcp_stop_work, K_MSEC(100));
```

**DHCP diagnostic note:** Even with a static IP assigned, `net_ipv4_create` fails in DHCP DISCOVER because this function inspects the interface's unicast address table and fails if the routing/source selection logic can't resolve 0.0.0.0 against the existing manual address. This is a known limitation on the nRF7002 platform — bypass DHCP entirely for P2P_CLIENT.

**Key configs to check:**
```kconfig
CONFIG_NET_IF_MAX_IPV4_COUNT=1   # check build .config — single slot only
CONFIG_NET_CONFIG_MY_IPV4_ADDR="192.168.7.1"  # always occupies the slot at boot
```

---

## Pattern N+2: AP-Initiated Disconnect — reason=34 (DISASSOC_LOW_ACK)

**Context:** nordic-wifi-memfault on nRF7002DK (F4CE3600230A), customer lab (Assa Abloy)
**Session:** 54f9d0bc (2026-06-30)
**Symptom:** Repeated AP-initiated disconnects in a busy lab. First `reason=6`, then
`reason=34` twice. Supplicant: `CTRL-EVENT-DISCONNECTED bssid=... reason=34`.

**Reason codes (IEEE 802.11):**
- **reason=6** `CLASS2_FRAME_FROM_NONAUTH_STA` — AP's STA table was cleared (AP reboot /
  radio restart / aging timer) while supplicant still thought it was COMPLETED. Device
  only finds out on next data TX. AP-side administration.
- **reason=34** `DISASSOC_LOW_ACK` — AP retransmitted downlink frames, got **no ACKs**,
  hit retry limit, kicked the STA. RF or device-side cause (the radio stopped answering).

**Diagnostic method — read the nRF70 FW stats blob, not just the log.** Parse with the
project parser (see skill `chsh-sk-ncs-tc-nrf70-fw-stats`):
```bash
HDR=/opt/nordic/ncs/v3.3.0/modules/lib/nrf_wifi/fw_if/umac_if/inc/fw/host_rpu_sys_if.h
python3 nordic-wifi-memfault/script/nrf70_fw_stats_parser.py "$HDR" <serial>_nrf70-fw-stats_*.bin
```

**What the stats showed (and how to read them):**
| Counter | Value | Reads as |
|---------|-------|----------|
| `rssi_avg` | −48 dBm | Strong link → **range/weak-signal ruled out** |
| OFDM CRC fail | 10.7% (vs DSSS 0.32%) | **Congested 2.4 GHz ch 11** — fast rates fail, robust rates survive |
| `tx_timeout` / `tx_pkt_cnt` | 3464 / 13437 | Heavy medium contention |
| `rpu_hw_lockup_count` / `..._recovery_done` | 1 / 1 | **RPU locked up once & self-recovered** — radio-silent window → missed ACKs → reason 34 |
| `tx_packet_deauth/disassoc` | 0 / 0 | Device sent no teardown |
| `rx_packet_deauth` | 2 | Device **received** 2 deauths → **AP-initiated** (matches 2 reported disconnects) |

**Conclusion:** two overlapping causes — congested channel + an RPU lockup. The lockup is
the device-side smoking gun and the more escalation-worthy one.

**Key gotchas:**
- Stats are **cumulative since boot**, not per-event. One lockup + one disconnect is
  *consistent with*, not *proof of*, lockup-as-cause. Re-capture after the **next** event
  and diff `rpu_hw_lockup_count` / `reset_cmd_cnt` to confirm timing.
- The parser's **"UMAC control path stats"** block (`cmd_*`/`event_*`) is **misaligned** —
  values like `cmd_set_wiphy: 3440128` are artifacts. PHY/LMAC/UMAC-TX/UMAC-RX are correct.
- Use the **nrf70** (`nrf_wifi` module) header, not `nrf/drivers/wifi/nrf71/...`.

**Next steps for reason-34 in the field:** move AP to a quiet channel (test congestion
hypothesis); sniffer at the AP to see unACK'd retries; escalate to dev team if the lockup
counter increments at the disconnect.

---

## Pattern N+3: P2P_GO Multi-Client — Hardware Supports It, WPS Re-Arm Gate Is the Real Blocker

**Context:** nordic-wifi-app-template ("zego" bricks) on nRF7002DK/nRF54LM20DK+nRF7002EB2
**Topic:** Whether a P2P Group Owner (GO) can serve several P2P clients (GCs) simultaneously
**Correction history:** An earlier version of this pattern claimed nRF70 AP mode (SoftAP/P2P_GO)
was hard-limited to 1 station in NCS v3.3.0. That was **wrong** — it over-generalized from one
Nordic reference sample's self-imposed Kconfig. Corrected 2026-07-01 after the user reported
testing SoftAP with 3 simultaneous clients in this exact template.

**What's actually true:**

1. The real governing setting is the generic Zephyr subsystem Kconfig
   `CONFIG_WIFI_MGMT_AP_MAX_NUM_STA` (`zephyr/subsys/net/l2/wifi/Kconfig`), `range 1 2007,
   default 4` — **not** nRF70-specific. It flows straight into hostapd's `max_num_sta` field
   (`zephyr/modules/hostap/src/hapd_main.c` and `supp_api.c`). Zephyr's own test config uses 10;
   an NXP shield overlay uses 8.
2. `nrf/samples/wifi/softap/Kconfig`'s `SOFTAP_SAMPLE_MAX_STATIONS` (`range 1 1`) is **only**
   that one demo app's own array-size cap — not a driver/firmware ceiling. Don't cite it as
   evidence of a hard limit; it just means Nordic's sample author was conservative/hadn't
   updated the sample.
3. This project already proves multi-station works: `docs/dev-specs/1-architecture.md` documents
   `CONFIG_WIFI_MGMT_AP_MAX_NUM_STA=3` (reduced from the Zephyr default of 4) for SoftAP, and the
   user confirmed 3 simultaneous SoftAP clients tested and working on real hardware.
4. `MAX_PEERS 5` in `modules/lib/nrf_wifi/fw_if/umac_if/inc/system/fmac_structs.h` is the
   driver's internal peer-table array size — comfortably above the 3-4 station configs actually
   used here, not a "currently 1" ceiling.

**Since P2P_GO and SoftAP run the identical hostapd AP code path** (per this project's own
`net_event_mgmt.c` comments), `CONFIG_WIFI_MGMT_AP_MAX_NUM_STA` applies equally to both — a
P2P_GO group in this codebase should structurally accept the same station count as SoftAP does.

**The real blocker for P2P_GO multi-client is application-level, not hardware:** P2P clients
(unlike SoftAP clients) can only join via a live WPS handshake — there's no static SSID+PSK to
just connect with. `wifi_run_p2p_go_mode()` arms WPS PBC once; `wifi_p2p_go_cancel_wps_timer()`
disarms it on the *first* client's `NET_EVENT_WIFI_AP_STA_CONNECTED`, and it is only re-armed
(`wifi_p2p_go_rearm_wps_pin()`) on that client's `NET_EVENT_WIFI_AP_STA_DISCONNECTED`. A second
GC therefore has no way to complete its WPS join while the first stays connected — this is a
deliberate single-peer design choice (matches the PRD: "P2P_GO | First P2P client associates"),
not a technical ceiling.

**To support several concurrent P2P clients:** keep re-arming WPS PBC after each connect
(instead of cancelling it) while `sta_count < CONFIG_WIFI_MGMT_AP_MAX_NUM_STA`, mirroring the
station-counting bookkeeping SoftAP already does via `MAX_SOFTAP_STATIONS`/`connected_stations[]`
in `net_event_mgmt.c`.

**Lesson: don't generalize a driver/hardware ceiling from a single reference sample's Kconfig
default.** Cross-check against the generic Zephyr subsystem Kconfig and, ideally, this
project's own tested/documented configuration before asserting a hard limit exists.

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

## Pattern N+4: BLE Provisioning DHCP-Never-Binds — wifi_prov_core Disconnect+Connect Race Wedges ctrl_iface

**Context:** nordic-wifi-app-template on nRF54LM20DK+nRF7002EB2 (STA mode, BLE provisioning)
**Session:** 2026-07-03
**Symptom:** After BLE provisioning (`SET_CONFIG`), `NET_EVENT_WIFI_CONNECT_RESULT: success` and
`STA: restarting DHCP on wlan0` fire normally, but `L3-NET_EVENT_IPV4_DHCP_BOUND` never fires.
Interface stays on the boot-time fallback IP (`192.168.7.1` / `CONFIG_NET_CONFIG_MY_IPV4_ADDR`).
The next BLE `GET_STATUS` poll's `NET_REQUEST_WIFI_IFACE_STATUS` call times out ~15s later with
`wpa_supp: 'SIGNAL_POLL' command timed out.` Current mitigation: a 25s DHCP-bind watchdog
(`CONFIG_ZEGO_WIFI_BLE_PROV_DHCP_WATCHDOG_SEC`) does a full `sys_reboot(SYS_REBOOT_WARM)`.

**Root cause — confirmed by source, not guessed:** Nordic's vendored `wifi_prov_core` library
issues `NET_REQUEST_WIFI_DISCONNECT` immediately followed by `NET_REQUEST_WIFI_CONNECT`, on the
same thread, with **no wait** for the disconnect to actually complete:

```362:406:nrf/subsys/net/lib/wifi_prov_core/wifi_prov_handler.c
net_mgmt(NET_REQUEST_WIFI_DISCONNECT, iface, NULL, 0);
...
rc = net_mgmt(NET_REQUEST_WIFI_CONNECT, iface, &cnx_params, sizeof(...));
```

Both calls route through Zephyr's hostap port, which serializes every WiFi request behind **one
global static mutex** and processes them on **one statically-allocated thread** created once at
boot (`hostap_handler`, `SYS_INIT(init, APPLICATION, 0)` in `zephyr/modules/hostap/src/supp_main.c`):

```72:72:zephyr/modules/hostap/src/supp_api.c
K_MUTEX_DEFINE(wpa_supplicant_mutex);
```

If `CONNECT` is dispatched before `DISCONNECT`'s response is drained, the ctrl_iface's
request/response bookkeeping wedges: the 802.11/L2 state machine can still limp along and later
report a genuine `CONNECT_RESULT: success` (it's a pushed async event, unaffected), but the
**synchronous command channel** used by `NET_REQUEST_WIFI_IFACE_STATUS`/`SIGNAL_POLL` and by the
driver's "port authorized" signal (which gates the data plane, including outbound DHCP frames)
stays jammed. Hence: CONNECT_RESULT fires, DHCP never binds, and any later status query times out.

**Why nRF54LM20DK+nRF7002EB2 and not nRF7002DK:** the EB2 shield talks over **SPI** (vs. on-board
**QSPI**), which widens the timing window between the library's DISCONNECT and CONNECT dispatch —
making the race far more likely to land. See [[nrf7002dk-vs-nrf54lm20dk]].

**Is a lighter recovery possible than full reboot?** Investigated and largely **no**, for this
specific failure class:
- Zephyr has no supported way to abort a wedged `hostap_handler` thread and safely rebuild the
  `wpa_global`/wpa_supplicant/driver-context structs in place — aborting mid-blocking-call would
  leave `wpa_supplicant_mutex` (and possibly the nRF70 driver's `vif_lock`) permanently locked,
  which is worse than the current wedge.
- Nordic *does* ship a lighter mechanism, `CONFIG_NRF_WIFI_RPU_RECOVERY` (default y), which does
  `net_if_down()` + `net_if_up()` (RPU-only cold boot, preserves app/BLE state) — but it is only
  wired to a genuine **RPU hardware watchdog interrupt** (`hal_interrupt.c` → `hal_rpu_recovery()`
  → `nrf_wifi_rpu_recovery_cb()`), not to host-side ctrl_iface/mutex timeouts. It will not fire for
  this race.
- This matches a known, currently open upstream issue with the identical circular-wait shape:
  **zephyrproject-rtos/zephyr#97512** ("nRF70: deadlock on interface power down") — `net_if_down()`
  holds `vif_lock` while triggering async handlers that also need `vif_lock`.
- **Untested idea, not yet implemented:** a tier-1 `net_if_down()`+`net_if_up()` attempt (own short
  timeout) before falling back to `sys_reboot()` — only helps if the wedge happens to be an
  RPU-firmware-level hang rather than the pure host-side mutex deadlock; never worse than today
  since it falls through to the existing reboot on its own timeout. To classify which failure class
  is actually occurring before investing in this, capture UART with
  `CONFIG_WPA_SUPP_LOG_LEVEL_DBG=y` + `CONFIG_LOG_MODE_IMMEDIATE=y` and look for RPU
  watchdog/"unresponsive" lines at the moment of the wedge.
- **Real fix** (not yet applied): patch `wifi_prov_handler.c`'s `prov_set_config_handler` to wait
  for `NET_EVENT_WIFI_DISCONNECT_RESULT` (or a bounded polled-disconnected check) before issuing
  `NET_REQUEST_WIFI_CONNECT`. This is vendored `sdk-nrf` code (pinned via `west.yml`, not tracked in
  the `zego` self-repo) — any direct edit is wiped by the next `west update` unless tracked as a
  maintained patch.

**A/B/C confirmation (2026-07-03, side-by-side captures) — isolates the bug to SET_CONFIG, not the board:**

| Case | Path | Result |
|---|---|---|
| nRF7002DK, `Set_config` (BLE) | `DISCONNECT`→`CONNECT`, connect succeeds on 1st attempt | `DHCP_BOUND` in 7s ✓ |
| nRF54LM20DK+EB2, `Set_config` (BLE) | `DISCONNECT`→`CONNECT`, 1st attempt fails `pre-scan retry (status=1)` → forced IF_DOWN/IF_UP → 2nd attempt succeeds | `DHCP_BOUND` **never** fires, `SIGNAL_POLL` times out 15s later ✗ |
| nRF54LM20DK+EB2, next boot, `CONNECT_STORED` (no preceding DISCONNECT) | Single `CONNECT`, no disconnect race | `DHCP_BOUND` in 3s ✓ |

The third row is the decisive data point: **the identical nRF54LM20DK+EB2 hardware/driver gets DHCP
bound cleanly** when the connect isn't preceded by an immediate `DISCONNECT` — ruling out a general
board/driver limitation. The nRF54LM20DK+EB2's slower SPI-shield path (vs. nRF7002DK's on-board
QSPI) is what makes `SET_CONFIG`'s connect consistently race ahead of the disconnect settling / the
pre-connect scan completing (visible as the extra `pre-scan retry (status=1)` + forced `IF_DOWN`/
`IF_UP` cycle nRF7002DK never hits) — and it's that compressed disconnect→retry→reconnect sequence
that leaves the ctrl_iface synchronous command channel permanently wedged, even though the L2 state
machine eventually reports a "success" CONNECT_RESULT.

---

## Pattern N+5 (open, single occurrence): CONNECT_STORED scan-reject retries then 10s stall + local timeout

**Context:** nordic-wifi-memfault on nRF54LM20DK+nRF7002EB2, immediately after the v3.3.0→v3.4.0
migration session (2026-07-07) — first hardware boot cycle after the rebuild.
**Symptom:** On reboot (stored credentials, no BLE, plain `CONNECT_STORED` path — same path
Pattern N+4's third row found binds DHCP in 3s), this run instead showed:
```
Auto-connecting with stored credentials...
Reject scan trigger since one is already pending / Failed to initiate AP scan   (x3, ~1s apart)
SME: Trying to authenticate → Trying to associate → Associated → CTRL-EVENT-SUBNET-STATUS-UPDATE
                                                                  ← 10s stall, nothing happens
Authentication with <bssid> timed out.
CTRL-EVENT-DISCONNECTED reason=3 locally_generated=1   (supplicant self-deauthed, not AP-kicked)
L2-NET_EVENT_WIFI_CONNECT_RESULT: pre-scan retry (status=1)
L2-NET_EVENT_IF_DOWN
```
Net effect: `memfault_core`'s periodic upload never got network connectivity (Memfault "not
connected" as reported by the developer) — this is a Wi-Fi reconnect failure, not a Memfault-side
bug.
**Not yet root-caused** — single occurrence, not loop-tested. Distinguishing question for next
time: is the 3x "scan already pending" + 10s post-Associated stall a pre-existing AP/RF flakiness
(same family as Pattern N+2's reason=34), or a new regression from something in the same session's
changes (candidates to rule out: the newly-adopted `zego/bricks/ux` brick's extra `SYS_INIT`
priority-95 init work and its own Wi-Fi-state zbus listener competing for scheduling at boot,
right when auto-connect fires). **Rule out cleanly:** loop-test reboot ~10x with the same firmware
to establish a failure rate; if it reproduces at similar timing every time, capture with
`CONFIG_WPA_SUPP_LOG_LEVEL_DBG=y` to see what's happening in the 10s gap between Associated and
the timeout.

---

## Related Pages
- [[nrf7002dk-vs-nrf54lm20dk]] — Board behavior differences that affect WiFi
- [[nrf54lm20dk-plus-nrf7002eb2]] — Board-specific notes
- [[memfault-workflow]] — How to correlate WiFi failures with Memfault crash reports
- [[ncs-build-system]] — Build command for WiFi credential overlays
- [[zephyr-assert-usage]] — Enabling `CONFIG_ASSERT` on a debug build to catch invariant violations faster during root-cause isolation
- [[wifi-power-save-listen-interval]] — Power-save wakeup mode (DTIM vs listen interval) quantization behavior on nRF70
- Skill `chsh-sk-ncs-tc-nrf70-fw-stats` — parse the nRF70 FW stats blob and interpret the counters (used in Pattern N+2)
