---
title: NCS v3.3.0 — Full CLAUDE.md Reference
created: 2026-05-11
updated: 2026-05-11
type: concept
tags: [ncs, nrf54, embedded, ai-agent, claude, workflow, tools, ppk2, power, gpio, saleae, memfault]
confidence: high
sources:
  - /opt/nordic/ncs/v3.3.0/CLAUDE.md
  - /Users/chsh/.claude/wiki/concepts/eedp-platform.md
---

# NCS v3.3.0 Workspace

This is a Nordic **nRF Connect SDK v3.3.0** west workspace.
SDK source repos (`nrf/`, `zephyr/`, `modules/`, etc.) are read-only upstream code.
The two custom application projects are the primary development targets.

---

## Repository Layout

| Folder | Description |
|--------|-------------|
| `nrf/` | nRF Connect SDK — libraries, samples, subsystems |
| `zephyr/` | Zephyr RTOS |
| `modules/` | Third-party modules (crypto, HAL, debug, fs …) |
| `nrfxlib/` | Nordic binary/source libraries |
| `bootloader/mcuboot/` | MCUboot bootloader |
| `nordic-wifi-webdash/` | **Custom app** — browser-based Wi-Fi dashboard |
| `nordic-wifi-memfault/` | **Custom app** — Memfault observability reference |

---

## Custom Application Projects

### nordic-wifi-webdash

Browser dashboard (buttons, LEDs, system info) served from device flash.
Four Wi-Fi modes: SoftAP, STA, P2P_GO (default), P2P_CLIENT. Mode stored in NVS.

| Board | West build target | Extra args |
|-------|-------------------|------------|
| nRF7002DK | `nrf7002dk/nrf5340/cpuapp` | — |
| nRF54LM20DK + nRF7002EB2 | `nrf54lm20dk/nrf54lm20a/cpuapp` | `-DSHIELD=nrf7002eb2` |

Docs: [`nordic-wifi-webdash/docs/`](nordic-wifi-webdash/docs/)

| Path | Contents |
|------|---------|
| `docs/pm-prd/` | Product requirements |
| `docs/dev-specs/` | Architecture and module specs |
| `docs/qa-test/` | QA reports and test plans |

### nordic-wifi-memfault

Memfault integration reference: crash reporting, OTA, metrics, Wi-Fi provisioning over BLE.
Requires `overlay-app-memfault-project-info.conf` with `CONFIG_MEMFAULT_NCS_PROJECT_KEY` set.

| Board | West build target | Extra args |
|-------|-------------------|------------|
| nRF7002DK | `nrf7002dk/nrf5340/cpuapp` | `-DEXTRA_CONF_FILE="overlay-app-memfault-project-info.conf"` |
| nRF54LM20DK + nRF7002EB2 | `nrf54lm20dk/nrf54lm20a/cpuapp` | `-DSHIELD=nrf7002eb2 -DEXTRA_CONF_FILE="..."` |

Docs: [`nordic-wifi-memfault/README.md`](nordic-wifi-memfault/README.md)

---

## Skills Reference

Invoke these skills (by name) for NCS-specific tasks:

| Task | Skill |
|------|-------|
| Build / flash / west commands | `chsh-sk-ncs-env` |
| Full project lifecycle (PRD → Specs → Code → QA) | `chsh-sk-ncs-workflow` |
| Author or update a PRD | `chsh-sk-ncs-prd` |
| Write engineering specs from a PRD | `chsh-sk-ncs-spec` |
| Implement code from specs | `chsh-sk-ncs-project` |
| Format C/C++ with clang-format | `chsh-sk-ncs-formatter` |
| Optimize RAM/Flash usage | `chsh-sk-ncs-memory` |
| Generate Test Report / QA Report | `chsh-sk-ncs-test` |
| Debug firmware (UART, crashes, loop tests) | `chsh-sk-ncs-debug` |
| Migrate app to a newer NCS version | `chsh-sk-ncs-migrate` |
| Wi-Fi UDP throughput benchmarking | `chsh-sk-ncs-wifi-throughput` |
| Git commit + push | `chsh-sk-git` |

---

## Code Intelligence (GitNexus)

Each repo has its own GitNexus index. See the per-project `AGENTS.md` for repo-specific
resources and functional-area skill files:
- [`nrf/AGENTS.md`](nrf/AGENTS.md) — sdk-nrf (152 k symbols)
- [`zephyr/AGENTS.md`](zephyr/AGENTS.md) — zephyr (518 k symbols) — **read-only upstream**
- [`nordic-wifi-webdash/AGENTS.md`](nordic-wifi-webdash/AGENTS.md) — 708 symbols
- [`nordic-wifi-memfault/AGENTS.md`](nordic-wifi-memfault/AGENTS.md) — 841 symbols

> If any GitNexus tool warns the index is stale, run `npx gitnexus update` in the repo root.

---

## Embedded Development Tools

Optional hardware/software tools for development, debugging, and testing. All are **opt-in** — activate only what the current task requires.

| Tool | Purpose | Interface | When to Enable |
|------|---------|-----------|----------------|
| GPIO Shell | Press buttons / read LEDs on target boards | UART → Zephyr shell (`/dev/ttyUSB0`) | UI interaction tests, board automation |
| JLink / Debugger | Flash firmware, reset, debug, coredump | MCP (`jlink_mcp`) or `west flash` CLI | Flashing, crash analysis, GDB |
| Saleae Logic Analyzer | Capture & decode SPI/QSPI/UART signals | MCP (`logic-analyzer-ai-mcp`) | Protocol debugging, bus timing |
| Router Control | Control Wi-Fi network environment | SSH via `paramiko` | Reconnection tests, packet loss simulation |
| Memfault | Crash reports, OTA status, metrics | REST API (`api.memfault.com`) | Post-crash analysis, fleet monitoring |
| PPK2 Power Profiler | Real-time µA/mA current measurement | Serial (`ppk2-api-python`) | Power optimization, sleep/wake profiling |
| UART Monitor | Read firmware log output | Serial (`pyserial`) | Log capture, boot tracing, loop tests |

### GPIO Shell — Hardware Control

nRF7002DK (nRF5340) runs Zephyr with `CONFIG_GPIO_SHELL=y`. Send commands over UART at 115200 baud:

```sh
gpio set gpio0 11    # press button (transistor pulls GPIO low)
gpio clear gpio0 11  # release button
gpio get gpio0 28    # read LED → 0 or 1
```

### JLink / Debugger

Each nRF54LM20DK has on-board JLink OB. Enable MCP server `jlink_mcp` in Cursor when flashing or debugging. CLI alternative:

```sh
nrfutil sdk-manager toolchain launch --ncs-version=v3.3.0 -- \
  west flash -d <app>/build --recover --dev-id <JLINK_SN>
nrfutil device reset --serial-number <JLINK_SN>
addr2line -e build/zephyr/zephyr.elf 0x<FAULT_ADDR>  # decode crash
```

### Saleae Logic Analyzer

Enable MCP server `logic-analyzer-ai-mcp`. Key tools: `mcp_saleae_capture`, `mcp_saleae_decode` (SPI/QSPI/UART), `mcp_saleae_trigger`, `mcp_saleae_export`.

### Router Control

SSH to ASUS BE92U (Merlin firmware). Use `paramiko`:

```python
import paramiko, os
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect("192.168.1.1", username="admin", password=os.getenv("ROUTER_PASS"))
ssh.exec_command("service restart_wireless")           # simulate AP reboot
ssh.exec_command("iptables -A FORWARD -m mac --mac-source AA:BB:CC:DD:EE:FF -j DROP")  # block device
```

### Memfault

No MCP — call REST API directly. Auth: `Authorization: Basic <base64(:<project_key>)>`.

```python
import requests, base64, os
KEY = os.getenv("MEMFAULT_PROJECT_KEY")
headers = {"Authorization": "Basic " + base64.b64encode(f":{KEY}".encode()).decode()}
BASE = "https://api.memfault.com/api/v0/organizations/<org>/projects/<project>"
requests.get(f"{BASE}/issues", headers=headers)            # list crashes
requests.get(f"{BASE}/devices/{serial}", headers=headers)  # device info
```

### PPK2 Power Profiler

Install: `pip install ppk2-api`. Port: `/dev/ttyACM0` (macOS: `/dev/tty.usbmodem*`).

```python
from ppk2_api.ppk2_api import PPK2_MP
import time

ppk2 = PPK2_MP("/dev/ttyACM0")
ppk2.get_modifiers()
ppk2.use_source_meter()        # PPK2 powers and measures DUT
ppk2.set_source_voltage(3300)  # mV
ppk2.start_measuring()

time.sleep(5)                  # run test scenario here

data = ppk2.get_data()
samples = ppk2.get_samples(data)
print(f"Average: {sum(samples)/len(samples):.1f} µA")
ppk2.stop_measuring()
```

Use `use_ampere_meter()` instead when the DUT is powered externally.

### Development Loop

```
edit → west build → JLink flash → UART log → [Saleae capture] → [PPK2 measure] → [Memfault review] → edit …
```

Activate only the tools the task requires. For code+flash+log, only UART monitor and JLink are needed.

---

## Conventions

- Module code lives in `src/modules/<name>/` — no direct calls between modules; use Zbus channels.
- Partition maps are in `pm_static_<board>.yml` (static memory layout for MCUboot + app slots).
- Board-specific overlays use underscores: `nrf7002dk_nrf5340_cpuapp.overlay`.
- Specs live in `docs/dev-specs/`; the entry point is `overview.md`.
- Do not modify upstream repos (`nrf/`, `zephyr/`, `modules/`, `nrfxlib/`, `bootloader/`) unless patching.
