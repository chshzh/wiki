---
title: macOS Host + Ubuntu VM Wi-Fi Sniffer Architecture
created: 2026-08-14
updated: 2026-08-14
type: concept
tags: [wifi, sniffer, wireshark, macos, ubuntu, vmware-fusion, monitor-mode]
sources: [session:local]
confidence: high
---

# macOS Host + Ubuntu VM Wi-Fi Sniffer Architecture

Architecture, configuration, and performance characteristics for capturing raw 802.11 monitor-mode frames on an Ubuntu Linux VM (VMware Fusion / UTM) and streaming them live into native Wireshark on macOS.

---

## 1. Problem & Architecture

### The Problem
* **macOS Wi-Fi Dongle Limitation:** macOS does not allow third-party USB Wi-Fi dongles to enter raw 802.11 monitor mode with full Radiotap headers (RSSI, noise floor, MCS rate, channel width, A-MPDU/A-MSDU subframes).
* **Linux USB Wi-Fi Driver Support:** Linux has in-kernel drivers (e.g., `mt7921u`, `rtw89`) for Wi-Fi 6/6E/7 USB adapters (like Netgear A9000).
* **Display / UI Preference:** Running Wireshark directly inside a VM or over X11/VNC has high latency and loses macOS Retina display rendering and native keyboard shortcuts.

### Architecture
```text
 ┌────────────────────────────────────────────────────────┐
 │                      macOS Host                        │
 │  ┌──────────────────────────────────────────────────┐  │
 │  │      Native Wireshark GUI (Retina / Smooth)      │  │
 │  └──────────────────────────▲───────────────────────┘  │
 │                             │ stdin pipe (pcapng stream)
 │                  ssh -4 ... │ (7+ Gbps Virtual NAT Link)
 └─────────────────────────────┼──────────────────────────┘
                               │
 ┌─────────────────────────────┼──────────────────────────┐
 │                      Ubuntu Linux VM                   │
 │  ┌──────────────────────────┴───────────────────────┐  │
 │  │        dumpcap / tcpdump (Headless capture)      │  │
 │  └──────────────────────────▲───────────────────────┘  │
 │                             │ mon0 (Monitor Mode)      │
 │  ┌──────────────────────────┴───────────────────────┐  │
 │  │    USB Wi-Fi Dongle (Netgear A9000 / MT7921au)   │  │
 │  └──────────────────────────────────────────────────┘  │
 └────────────────────────────────────────────────────────┘
```

---

## 2. Link Performance & Throughput

Benchmarked on Apple Silicon (M4 Pro) with VMware Fusion NAT networking:

* **VM $\rightarrow$ Mac (Capture Stream Direction):** **7.41 Gbps** (0 retransmissions).
* **Mac $\rightarrow$ VM:** **~460 Mbps** (asymmetric due to software segmentation vs TCP Segmentation Offload).
* **Wi-Fi Air Saturation Requirement:** Saturated 160 MHz Wi-Fi 6 captures generate at most **150–300 Mbps**.
* **Verdict:** The in-memory virtual NAT link exceeds maximum Wi-Fi air bandwidth by >25×, guaranteeing zero packet drops caused by transport bandwidth.

---

## 3. Key Technical Pitfalls & Fixes

### 1. SSH Password Prompt Corrupts Binary Pcap Pipe
* **Symptom:** `sudo: A terminal is required to authenticate` or Wireshark error `End of file on pipe magic during open.`
* **Cause:** SSH cannot interactively prompt for sudo password inside a pipe; passing `-t` forces a PTY which converts `\n` to `\r\n`, corrupting binary pcap magic bytes.
* **Fix:** Grant passwordless sudo for capture tools in `/etc/sudoers.d/capture-nopasswd`:
  ```text
  <user> ALL=(ALL) NOPASSWD: /usr/bin/dumpcap, /usr/sbin/dumpcap, /usr/bin/tcpdump, /usr/sbin/tcpdump, /usr/local/bin/setup_mon0.sh
  ```

### 2. IPv6 Link-Local mDNS Scope Resolution
* **Symptom:** Connecting to `sniffer.local` hangs or prompts for unknown host fingerprint on `fe80::...%bridge101`.
* **Cause:** macOS Bonjour resolves `.local` names to IPv6 link-local addresses first by default.
* **Fix:** Always pass the **`-4`** flag to SSH:
  ```bash
  ssh -4 <user>@sniffer.local "sudo dumpcap -i mon0 -P -w -" | /Applications/Wireshark.app/Contents/MacOS/Wireshark -k -i -
  ```

### 3. NetworkManager Silent Channel Retuning
* **Symptom:** `mon0` is set to channel 165, but Wireshark continues capturing on channel 1 or disconnects.
* **Cause:** NetworkManager periodically scans with unmanaged adapters and retunes the shared PHY.
* **Fix:** Permanently mark the adapter's MAC as unmanaged in `/etc/NetworkManager/NetworkManager.conf`:
  ```ini
  [keyfile]
  unmanaged-devices=mac:<MAC_ADDRESS>
  ```

### 4. USB Pass-Through Detach on Reboot
* **Symptom:** `mon0` disappears after Ubuntu reboot because VMware Fusion leaves USB attached to macOS host.
* **Fix:** In VMware Fusion: `Virtual Machine` $\rightarrow$ `Settings` $\rightarrow$ `USB & Bluetooth` $\rightarrow$ set Wi-Fi adapter's **Plug In Action** to **"Connect to Linux"**.

---

## 4. Automation Components (`ubuntu-wireshark-wifi-sniffer`)

* **`setup_mon0.sh`**: Headless setup script that auto-detects `wlx*` interfaces, sets channel, unmanages NetworkManager, and provides `--install` one-touch VM setup.
* **`setup-mon0.service`**: `oneshot` systemd unit running on boot with `--wait 15` polling loop to initialize `mon0` before capture.
* **`sniffer-info`**: Zenity GUI and CLI utility providing macOS connection strings and desktop guide launcher.
* **`sniffer-desktop-notify.sh`**: Desktop login hook popping up the Mac guide or USB disconnected warning dialog.
