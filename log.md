# Wiki Log

> Chronological record of all wiki actions. Append-only.
> Actions: ingest, update, query, lint, create, archive, delete

## [2026-07-07] update | ncs-version-migration — nordic-wifi-memfault v3.3.0→v3.4.0 full build fix (session d34b1836)
- Added v3.4.0 mbedTLS findings: `MBEDTLS_X509_LIBRARY`/`MBEDTLS_TLS_LIBRARY` promptless (delete), `MBEDTLS_ECP_DP_SECP384R1_ENABLED` renamed to `PSA_WANT_ECC_SECP_R1_384`, `MBEDTLS_ECDSA_C` now needs explicit `MBEDTLS_ECP_C=y`.
- Added empirical clarification: Kconfig strict mode does NOT treat deprecated/experimental/not-secure warnings as fatal by themselves (confirmed identical warnings present in both a failing and the final passing build) — only "defined without a type", "undefined symbol", "not directly user-configurable", and dependency value-mismatch warnings actually abort the build.
- Added new section: Memfault Firmware SDK bump (independent of NCS/mbedTLS) — `MEMFAULT_COREDUMP_STORAGE_RRAM`→`_NRF_RRAM` rename, `CONFIG_MEMFAULT_FOTA` removed (→ `CONFIG_MEMFAULT_ZEPHYR_FOTA`), `memfault_fota_start()`→`memfault_zephyr_fota_start()` + header path change, `MEMFAULT_FOTA_CLI_CMD` fully dead.
- Added new section: zego brick refactor gotcha — adopting `zego/bricks/ux` (banner moved here from `zego/bricks/wifi`) into a project with its own similarly-named local `ux` module caused a `ZBUS_LISTENER_DEFINE` symbol name collision at link time (`app_wifi_state_listener`); fixed by renaming the local listener, not by removing either module.

## [2026-07-03] update | wifi-debugging-patterns — root-caused BLE provisioning DHCP-never-binds bug
- Added Pattern N+4: confirmed via source (`wifi_prov_handler.c`, Zephyr hostap `supp_main.c`/`supp_api.c`) that the DHCP-bind watchdog reboot (nRF54LM20DK+nRF7002EB2) is a documented mitigation for a real upstream race — `wifi_prov_core`'s `SET_CONFIG` fires DISCONNECT immediately followed by CONNECT with no wait, wedging wpa_supplicant's single global ctrl_iface mutex/thread.
- Researched whether a lighter (non-full-reboot) recovery exists: `CONFIG_NRF_WIFI_RPU_RECOVERY` does `net_if_down/up` but only triggers on RPU hardware watchdog interrupts, not host-side ctrl_iface timeouts — doesn't apply here. Linked matching open upstream bug zephyrproject-rtos/zephyr#97512.
- No code fix applied yet (user stopped at diagnosis stage) — real fix would require patching vendored `sdk-nrf` (not tracked in `zego` repo).

## [2026-07-03] update | wifi-debugging-patterns — A/B/C log comparison confirms SET_CONFIG-only, not board-general
- User supplied side-by-side nRF7002DK (works) vs nRF54LM20DK+EB2 (fails) BLE-provisioning captures, plus the same nRF54LM20DK's next-boot CONNECT_STORED capture (works, no preceding DISCONNECT). Added as decisive evidence to Pattern N+4: rules out a general nRF54LM20DK/driver limitation — confirms the wedge is specific to wifi_prov_core's SET_CONFIG disconnect+connect race.

## [2026-05-29] lint | Wiki review — 1 P0 fixed, 0 P1, 14 P2 (info)
- **P0 fixed:** concepts/eedp-platform.md — corrupted frontmatter opener (`chsh-sk-ncs-test ---`) stripped; YAML now parses correctly.
- **P2 (info):** 14 pages have `sources: []` — valid per SCHEMA.md for session-derived content.
- **P2 (info):** 8 pages over 200 lines — no splits applied; all well-structured.
- **No orphans, no broken links, no index gaps, no stale content.**
- **Report:** skills/chsh-sk-llm-wiki-review/report-2026-05-29.md


- **Renamed:** concepts/mcp-nrflow-tools.md → concepts/mcp-nordic-mcp-tools.md
- **Updated links (6 files):** index.md, comparisons/memfault-mcp-vs-cli.md, concepts/eedp-platform.md, concepts/github-actions-ncs-ci.md, concepts/embedded-system-general-debugging.md, concepts/cursor-skills-and-agents.md
- **Cleaned:** Removed remaining `mcp.nrflow` table column headers inside the page (→ "Nordic MCP"), updated server history note to omit old server name
- **log.md historical entries preserved** (append-only, factual record of 2026-05-29 session)

## [2026-05-29] update + create | nordic-mcp MCP consolidation + Memfault MCP wiki
- **mcp.json:** Removed `nrflow` server (confirmed byte-identical duplicate of `nordic-mcp` — same 6 tools, 3 resources, 20 knowledge sources). Removed `nordic-semiconductor-docs` (kapa.ai, single tool `search_nordic_semi_knowledge` — corpus covered by `mcp_nordic-mcp_nordicsemi_search_sources`).
- **Updated:** concepts/mcp-nrflow-tools.md — renamed title/frontmatter to `mcp.nordic-mcp`, replaced all `mcp_nrflow_*` tool references with `mcp_nordic-mcp_*`, added server history note (nrflow + nordic-semiconductor-docs removal rationale), added cross-reference to new comparison page.
- **Created:** comparisons/memfault-mcp-vs-cli.md — full capability matrix (MCP = read-only: device_search SQL, trace_get with logs, metrics_list, listReboots; CLI = write: symbol upload, OTA release, deploy, abort), combined OTA release + verification workflow, crash debug workflow, tool reference for all 8 MCP tools, limitations section.
- **Updated:** index.md — fixed mcp-nrflow-tools summary, added memfault-mcp-vs-cli under Comparisons, pages 20→21, date 2026-05-29.
- **Skills updated (11 files):** All `mcp_nrflow_` → `mcp_nordic-mcp_` in chsh-sk-ncs-env, chsh-sk-ncs-3.1-coding, chsh-sk-ncs-3.2-debug, chsh-sk-ncs-3.3-memopt, chsh-sk-ncs-4.1-verification, chsh-sk-ncs-4.2-validation, chsh-sk-ncs-2-spec, chsh-sk-ncs-migrate, chsh-sk-git-release, eedp-platform.md ref. Added Memfault MCP tools table + post-deploy verification guidance to chsh-sk-memfault-cli.



## [2026-05-26] lint | Wiki review — 2 concrete issues, 11 orphans
- **Report:** skills/chsh-ag-llm-wiki-review/report-2026-05-26.md
- **P0:** my-ncs-claude.md — 6 broken markdown links to external repo files → converted to plain text paths
- **P1:** hermes-file-map-customization.md — listed in index but file not found → removed from index.md
- **P1:** my-ncs-claude.md — existed on disk but missing from index → added to index.md under AI/Dev Tools
- **Updated:** index.md — added my-ncs-claude, removed hermes-file-map-customization (page count unchanged at 20)
- **Orphans:** 11/20 (55%) — expected for young wiki, no auto-fixes
- **No stale content, no source drift, no contested pages**

## [2026-05-26] update | EEDP v0.7 — added Serial Terminal module
- **Updated:** concepts/eedp-platform.md — added "2. Serial Terminal" (ttyd-based Web UART, 4 ports), renumbered 2→8: HW Ctrl→Serial Terminal→JLink→PPK2→Saleae→Router→Wireshark→Web. Total 8 modules.
- **Updated:** raw/assets/eedp-architecture-v0.3.svg — redrawn for 8-module layout, added Serial Terminal box, repositioned PPK2 and Wireshark
- **Created:** /mnt/CharlieII/eedp-intro.html — reader-friendly intro article (7 chapters, interactive SVG, FAQ)

## [2026-05-25] update | EEDP — added PPK2 as 6th module
- **Updated:** concepts/eedp-platform.md — added §3. Power Profiler / PPK2 (current measurement, 200nA–1A), renumbered modules 3→6, added to target hardware table, MCP ecosystem table, CLAUDE.md template, eedp-controller.py template, load table, open questions. Bumped to v0.5.
- **Created:** raw/assets/eedp-ppk2-workflow.html — standalone PPK2 workflow page with connection diagram, CLI commands, use cases, EEDP integration points
- **Updated:** index.md — date bump (no page count change)

## [2026-05-25] update | EEDP v0.6 — added Wireshark/tshark
- **Updated:** concepts/eedp-platform.md — added "6. Network Protocol Analysis / Wireshark" module (tshark + tcpdump via Router SSH), updated tool table + MCP index + version history. 7 modules total.
- **Updated:** raw/assets/eedp-architecture-v0.3.svg — added Wireshark module block
- **Key integration:** tshark CLI on PC, tcpdump remote capture via Router SSH, paired with Router Control for WiFi connect/disconnect frame analysis
- **Updated:** concepts/eedp-platform.md — added "实施策略" section: 3-layer architecture (CLAUDE.md + scripts/ + on-demand MCP), file structure, startup self-test flow, load table. Bumped to v0.4.
- **Asset:** wiki/raw/assets/eedp-architecture-v0.3.svg — standalone SVG for GitHub/Obsidian rendering

## [2026-05-10] create | EEDP platform concept
- **Created:** concepts/eedp-platform.md — Embodied Embedded Development Platform: AI-driven hardware control via GPIO shell (nRF7002DK), JLink debug, Saleae LA, router control, Memfault API. 5-module architecture, 3× nRF54LM20DK+nRF7002EB2.
- **Assets:** /mnt/CharlieII/eedp-architecture-v0.3.html (detailed SVG diagram)
- **Updated:** index.md (pages 16→17)
- **Decision:** nRF7002DK Zephyr GPIO Shell + direct GPIO wiring to target boards (replaces earlier solenoid/servo approach). EEDP Controller on PC, not embedded. AI-agent agnostic (not Hermes-dependent). Named "Embodied" for physical interaction parity.

- **Created:** concepts/github-actions-ncs-ci.md — Docker container approach, rolling "latest" release, pre-built firmware test loop, common failure patterns, caching strategy, west.yml manifest
- **Updated:** index.md (pages 15→16)
- **Updated:** skills/chsh-dev-ncs-debug/SKILL.md — added Mode G (GHA monitoring + pre-built firmware test loop), updated Related Skills table


- **Created:** concepts/deepseek-claude-code.md — DeepSeek Anthropic API 兼容层配置，V4-Pro/V4-Flash 模型策略，验证方法
- **Updated:** index.md (pages 10→11)

## [2026-05-05] merge | 双 wiki 合并：/mnt/CharlieII/wiki → /mnt/CharlieII/claude/wiki
- **Source (retired):** `/mnt/CharlieII/wiki`（11 页 + raw/）
- **Target (canonical):** `/mnt/CharlieII/claude/wiki`（保留 2 个独有页面 ncs-app-versioning, memfault-version-requirements）
- **方法:** `rsync -av` 覆盖骨架，独有文件手动保留
- **更新:** index.md 合并为 13 页，`.env` 更新指向
- **清理:** 源目录 `/mnt/CharlieII/wiki/` 暂未删除

## [2026-04-30] create | WiFi Knowledge Wiki initialized
- Domain: Wi-Fi / Wireless Networking
- Structure created with SCHEMA.md, index.md, log.md
- Wiki location: /workspace/wiki/

## [2026-04-30] create | Core WiFi wiki pages
- Created: concepts/wifi-standards-evolution.md — Full standards history Wi-Fi 1-8
- Created: concepts/wifi-6-6e-7.md — Wi-Fi 6/6E/7 technical deep dive (OFDMA, MLO, 4096-QAM, 320 MHz)
- Created: concepts/wifi-security.md — WPA2, WPA3, SAE, OWE, PMF, KRACK, WPS
- Created: entities/wifi-hardware-ecosystem.md — Chipsets, routers, mesh, antennas, deployment
- Created: comparisons/wpa2-vs-wpa3.md — Side-by-side comparison
- Updated: index.md — Added all 5 pages
- Sources: Wi-Fi Alliance, IEEE 802.11 standards, Qualcomm, Broadcom, MediaTek product docs

## [2026-05-01] merge | Wireless + Homelab wikis merged
- **Action:** Merged `/opt/data/home/wiki/` → `/opt/data/workspace/wiki/`
- **Domain expanded:** Wi-Fi + Homelab, networking, self-hosted infrastructure
- **New page:** concepts/wireguard-comprehensive-guide.md (moved from home wiki)
- **Duplicate handled:** concepts/wireguard-openwrt-china-tunnel.md — kept workspace version (includes multi-device expansion section)
- **SCHEMA.md updated:** Combined tag taxonomy (Standard, Protocol, Hardware, Security, Deployment, Networking, Homelab, Technology, Meta)
- **index.md updated:** Total pages: 7
- **Originating session:** WireGuard research (protocol design, networking topologies, embedded Zephyr/nRF/IoT)
- Conflicts: none

## [2026-05-01] delete | WiFi test content removed
- **Action:** User confirmed WiFi research was test data — deleted all 5 WiFi pages
- **Deleted:** concepts/wifi-standards-evolution.md, concepts/wifi-6-6e-7.md, concepts/wifi-security.md, entities/wifi-hardware-ecosystem.md, comparisons/wpa2-vs-wpa3.md
- **Cleaned:** index.md (pages 7→2, removed Entities/Comparisons/Queries sections), SCHEMA.md (domain narrowed to Homelab/networking/WireGuard/OpenWrt)
- **Removed empty dirs:** comparisons/, entities/
- **Remaining pages:** wireguard-comprehensive-guide, wireguard-openwrt-china-tunnel

## [2026-05-01] create | Hermes architecture wiki page
- **Created:** concepts/hermes-architecture.md — Full codebase architecture analysis (v0.11.0)
- **Content:** Agent loop, tool system, gateway, skills, memory, context compression, cron, transports, deployment modes, key statistics, design principles
- **Reading time:** ~10 minutes
- **Sources:** Source code analysis of /opt/hermes/ (~625K lines, 1303 source files)
- **Cron job:** Scheduled to monitor GitHub releases and auto-update this page

## [2026-05-02] ingest + create | AI SVG & PPTX generation
- **Prompt:** "请生成一篇wiki，以这个为例介绍svg和pptx的生成管理和过程"
- **Case study:** 有轨电车运行管理系统 PPT（11 页, 3 张 SVG, 310 KB）
- **RAW saved:** 
  - raw/articles/ppt-gen-tram-ops-script.md — Complete 630-line PptxGenJS script
  - raw/articles/ai-svg-pptx-generation-process.md — Full generation process narrative
- **Created:** concepts/svg-pptx-agent-generation.md — SVG（纯XML手写）+ PPTX（PptxGenJS JS API）生成工作流指南
  - Covers: SVG element mapping, color palette management, dark/light slide strategy, PptxGenJS pitfalls, full workflow diagram, SVG vs PPTX decision matrix
- **Updated:** index.md (pages 3→4), concepts/hermes-architecture.md (added cross-reference)
- **Cross-references:** svg-pptx-agent-generation ⇄ hermes-architecture

## [2026-05-03] create + cleanup | NAS Ubuntu VM wiki page + Hermes dir cleanup
- **Rewrote as NAS NFS architecture page:** entities/hermes-setup-on-linux-with-nas-nfs-backup.md — NAS is single source of truth, new machines recover by mounting NFS
  - Covers: NFS mount details (<nas-ip>:/volume1/<volume>, NFSv3, 3.6TB), what lives on NAS (hermes/, webui/, wiki/, backup/), zero-reconfig recovery, 5 pitfalls
- Also fixed: Telegram gateway (systemd 203/EXEC, stale paths → corrected), WebUI workspace → /mnt/CharlieII
- **Updated:** index.md (pages 4→5, added Entities section), concepts/hermes-architecture.md (added cross-ref to nas-ubuntu24-vm)
- **Cleanup /mnt/CharlieII/hermes/:** Removed Docker-era exploration leftovers
  - Deleted: webui/ (old Docker WebUI state), src/ (empty), sandboxes/singularity/, bin/tirith (12MB), images/ (empty), tram_ops_research.md, response_store.db*, models_dev_cache.json
- **Hermes WebUI setup:** Installed nesquena/hermes-webui at /mnt/CharlieII/hermes-webui/, state at webui-state/, bound 0.0.0.0:8787
  - Fixed tab completion: eval "$(hermes completion bash)" in ~/.bashrc
  - CLI session bridge: requires show_cli_sessions toggle in WebUI Settings
- **Auto-start:** systemd services for gateway + WebUI with linger → survive reboots
  - WebUI: manual systemd unit created (~/.config/systemd/user/hermes-webui.service)
- **Wiki relocated:** /mnt/CharlieII/hermes/workspace/wiki/ → /mnt/CharlieII/wiki/
  - WIKI_PATH updated in .env to /mnt/CharlieII/wiki

## [2026-05-03] create | DNS-over-HTTPS (DoH) wiki page
- **Created:** concepts/dns-over-https-doh.md
- **Content:** How DoH encrypts DNS, office threat model (what IT can/can't see), DoH vs DoT comparison, browser + systemd setup via cloudflared, limitations (SNI/ECH, managed devices, VPN as stronger alternative)
- **Updated:** index.md (pages 5→6)

## [2026-05-03] ingest | Docker permissions + Hermes docker-compose
- **Raw sources saved:**
  - raw/articles/linux-docker-permissions-source.md ← /mnt/CharlieII/Permissions.txt
  - raw/articles/hermes-docker-compose-source.md ← /mnt/CharlieII/hermes_backup/docker-compose.yaml
- **Created:** concepts/linux-vm-docker-permission.md — Linux 权限模型、UID 对齐（含实战：Synology NAS uid=1002 + Ubuntu VM uid=1000 对齐全过程）、Docker 容器权限（init container / user 指令 / userns-remap）、chmod/chown 速查
- **Created:** concepts/hermes-docker-compose-deployment.md — 三服务编排（init→agent+webui）、named volume 共享、NAS 路径映射、部署步骤与排错
- **Updated:** index.md (pages 6→8)
- **Note:** Claude chat source (ca49b286...) was private/unreachable; wiki pages synthesized from the two referenced files alone.

## [2026-05-03] create | ttyd Web 终端设置
- **Created:** concepts/ttyd-web-terminal-setup.md — ttyd 安装与使用、`-W` 可写模式、systemd 开机自启、常见问题（oh-my-bash 时钟前缀 `THEME_SHOW_CLOCK=false`、非登录 shell PATH 修复、hermes completion 条件加载）、与 noVNC/SSH 对比
- **Updated:** index.md (pages 8→9)
- **Context:** 绿联 UGOS Pro noVNC 不支持剪贴板通道（SPICE/QEMU Agent 安装后仍不可用），ttyd 作为浏览器端原生终端替代方案
- **Learned:** ttyd 非 login shell，只读 `.bashrc` 不读 `.profile`；`~/.local/bin` 需在 `.bashrc` 里显式加到 PATH；oh-my-bash font 主题时钟需在加载前用 `export THEME_SHOW_CLOCK=false` 关闭

## [2026-05-05] create + update | Hermes file map wiki page + state dir fix
- **Created:** concepts/hermes-file-map-customization.md — Physical file layout: where repos live, which files to customize (config.yaml, .env, SOUL.md, webui .env, systemd units) vs auto-managed
- **Updated:** entities/hermes-setup-on-linux-with-nas-nfs-backup.md — Fixed stale `webui-state/` → `webui/` references (3 locations: table, .env example, text)
- **Updated:** index.md (pages 9→10)

## [2026-05-07] create | concepts/memfault-version-requirements.md
- Sourced from: Memfault docs (https://docs.memfault.com/docs/platform/software-version-hardware-version)
- Covers: allowed characters, natsort ordering, v-prefix caveat, OTA ordering across NCS bumps

## [2026-05-07] create | concepts/embedded-system-general-debugging.md
- Sourced from: sQSPI/MSPI driver debugging session (NCS v3.3.0, nRF54LM20DK + nRF7002 EB-II)
- Covers: baseline testing, variable isolation, multi-device comparison, barrier patterns, loop testing, nordicsemi_uart_monitor.py best practice

## [2026-05-07] create | concepts/mcp-nrflow-tools.md
- Sourced from: mcp_nrflow_nordicsemi_workflow_ncs resource content
- Covers: tool inventory, resource descriptions, comparison vs manual scripting, multi-device best practices

## [2026-05-07] update | index.md + concepts/*.md + SCHEMA.md
- Fixed [[wikilinks]] to GitHub-compatible markdown links in all concept pages and index (15 total pages)
- Fixed VCOM0/VCOM1 mapping: UART30 = VCOM0 on nRF54LM20DK (not VCOM1)
- Updated embedded-system-general-debugging Lesson 10: prefer nordicsemi_uart_monitor.py over manual Python serial
- Updated SCHEMA.md: document markdown link convention instead of wikilinks
- Added wiki/raw/ to .gitignore (51MB hardware PDFs/Altium — not for git)

## [2026-05-07] skills-restructure | chsh-dev-git-release, chsh-ag-skill-create, chsh-ag-skill-review, chsh-dev-ncs-project
- Created `chsh-dev-git-release`: release tagging, GitHub CI watch, firmware download/flash/verify loop
- Renamed `cursor-create-skill` → `chsh-ag-skill-create` (updated frontmatter name)
- Created `chsh-ag-skill-review`: daily skill health audit — structure, size, dedup, dead links, cross-ref gaps, brainstorm section for future expansion
- Cleaned up `chsh-dev-ncs-project`: removed stale `debug/SKILL.md` (duplicated chsh-dev-ncs-debug), removed stale INDEX.md, README.md, ENHANCEMENT_SUMMARY.md (referenced non-existent files); updated SKILL.md sub-skills reference table

## [2026-05-07] skills-audit | ran chsh-ag-skill-review on ~/.claude/skills
- 28 skills reviewed, 17 real issues (P0=4, P1=2, P2=11)
- P0 fixes applied: restored YAML frontmatter to 4 sub-skills under chsh-dev-ncs-project (architecture/, protocols/, protocols/webserver/, wifi/) — frontmatter was either missing or wrapped in ```` ```skill ```` code fence so Cursor couldn't parse it; fixed dead links to relative paths
- P1 fixes applied: added `chsh-dev-git-release` to Related Skills tables of chsh-dev-git-commit, chsh-dev-ncs-debug, chsh-dev-ncs-workflow, chsh-qa-ncs-test
- Report: skills/chsh-ag-skill-review/report-2026-05-07.md

## [2026-05-07] rename | llm-wiki → chsh-ag-llm-wiki
- Renamed personal skill `llm-wiki` to `chsh-ag-llm-wiki` (consistent `chsh-ag-*` naming for agent-management skills)
- Rewrote frontmatter description to include WHEN trigger (was just one-liner)
- Created companion audit skill `chsh-ag-llm-wiki-review`: schema validation, broken/orphan link audit, index completeness, staleness, source drift via sha256, page-size split candidates, contradiction surface

## [2026-05-07] create | chsh-dev-ncs-migrate skill
- New skill: NCS version migration (single-hop and multi-hop)
- Workflow: baseline (build+flash+test on source) → plan hops → per-hop (toolchain switch, west.yml bump, apply migration guide, build clean, flash, smoke test, commit) → final functional verification
- Authoritative sources: Nordic release notes + per-version migration guides (URLs in skill)
- Includes Decision Gates table — when the migration guide doesn't cover something, STOP and ask via AskQuestion (out-of-tree patches, third-party SHA pins, functional regressions)
- Cross-referenced from chsh-dev-ncs-workflow Skill Reference and chsh-dev-ncs-debug Related table

## [2026-05-07] lint | wiki audit, 5 issues found, all auto-fixed
- P0: added missing `sources:` field to deepseek-claude-code.md, wireguard-openwrt-china-tunnel.md
- P1 + schema clarification: SCHEMA.md now explicitly allows URLs and `[]` in `sources:` (was previously raw/ paths only)
- P1: cleaned `sources: [wireguard.com]` → `[https://www.wireguard.com/]` on wireguard-comprehensive-guide.md
- P2: fixed invalid date `updated: 2026-05-07v2` on embedded-system-general-debugging.md
- 6 oversized pages (>200L) and 6 orphans surfaced as info-only — no auto-fixes
- Report: skills/chsh-ag-llm-wiki-review/report-2026-05-07.md

## [2026-05-08] skills-restructure | renamed all chsh-* skills to chsh-sk-* prefix
- Renamed 17 skill folders (chsh-ag-*, chsh-dev-*, chsh-pm-*, chsh-qa-*, chsh-txt-*) to chsh-sk-* prefix
- gitnexus-* skills kept as-is (intentional separate namespace)
- Moved subagent chsh-ag-git from ~/.cursor/agents/ to ~/.claude/agents/ (Cursor supports both paths)
- Updated all frontmatter name: fields and internal skill cross-references (comprehensive sed pass)
- Fixed 4 sub-skill frontmatter names under chsh-sk-ncs-project (P0: were still chsh-dev-*)
- Report: skills/chsh-sk-skill-review/report-2026-05-08.md

## [2026-05-12] create | west-update-internals
- Created concepts/west-update-internals.md — How `west update` works
- Topics: workspace discovery (.west/config), active manifest resolution, per-project git sequence (fetch → update-ref → checkout), manifest-rev branch mechanics, floating vs pinned revisions, detached HEAD "from" vs "at" semantics, self-project exclusion, common pitfalls
- Source: live debugging session (west update from inside project, dirty working tree, git status "from 6999f26" vs rev-parse HEAD = 1bcfaeb)
- Cross-references: github-actions-ncs-ci, ncs-app-versioning, embedded-system-general-debugging
- index.md: added entry under Nordic/NCS, bumped total to 18

## [2026-05-13] create | cracen-vs-oberon-tls
- Created comparisons/cracen-vs-oberon-tls.md
- Source: live exploration session — NCS nrf_security Kconfig + nordic-wifi-memfault board conf files
- Topics: PSA driver auto-selection (HAS_HW_NRF_CRACEN), CRACEN LITE variant on nRF54LM20A,
  interrupt-wait execution model vs synchronous Oberon, stack depth deltas (+600 B AES, +1500 B ECC),
  per-thread stack table, KMU hardware key storage, TRNG, TLS handshake timing, CPU offload benefit
- Cross-references: embedded-system-general-debugging, memfault-version-requirements
- index.md: added entry under Comparisons, bumped total to 19

## [2026-05-08] lint | wiki audit, 0 issues found
- No old skill refs in wiki pages
- All 16 pages pass frontmatter validation
- Index complete (16 pages on disk = 16 in index)
- Log: 160 lines, well under rotation threshold
- 12 orphan pages (info only — expected for young wiki)
- 6 oversized pages (info only — no auto-fix)
- Tag taxonomy in SCHEMA.md is informal (defers to log.md) — flagged for future formalization

## [2026-05-13] create | cursor-skills-and-agents
- Created concepts/cursor-skills-and-agents.md
- Source: live design session (skills/agents discussion, chsh-sk-memfault refactor, chsh-ag-memfault creation)
- Topics: skill vs agent distinction, invocation model (auto vs explicit), skill→agent delegation pattern (when justified vs redundant), agent design rules (hard rules, Step 0, AskQuestion gates, scope boundaries, API pitfall docs), skill design rules (500-line limit, Self-Update Policy, Related Skills table, name/directory match)
- Key finding: skill wrapper over agent is only justified when the skill does real pre-processing work (build decisions, env setup); if it only relays, the agent's description field suffices
- Key finding: GET /releases/{VER} on Memfault does NOT return activations — must fetch via GET /deployments filtered by status=done
- Cross-references: mcp-nrflow-tools, hermes-architecture, eedp-platform
- index.md: added entry under AI/Dev Tools, bumped total to 20
- Tags added: cursor (new)

## [2026-05-31] ingest | agentmemory deep mining → 8 new NCS experience pages

Mined agentmemory database covering 50+ sessions from 2025-10 through 2026-05.
Sessions analysed include: d1cdeb42, 299515f9, 0ebeff51, 69ca6368, bf04a4eb,
b51223d2, df6d96ae, f157d027, 558f2717, 50a3219c, b6ca3b52, 89531aad, 1f3afd05,
de80a0e8, 620407bd, 8e6ec0c1, 2ac79fc1, c153ab22, c8bc6f97, db5832e8.

Pages created (merged from /Users/chsh/.claude/wiki):
- entities/nrf7002dk.md
- entities/nrf54lm20dk-plus-nrf7002eb2.md
- concepts/ncs-build-system.md
- concepts/ncs-version-migration.md
- concepts/memfault-workflow.md
- concepts/wifi-debugging-patterns.md (also updated with BLE provisioning + P2P patterns)
- comparisons/agentmemory-vs-llm-wiki.md
- comparisons/nrf7002dk-vs-nrf54lm20dk.md

Source wiki /Users/chsh/.claude/wiki deleted after merge.
index.md: added 8 entries, bumped total to 28.

## [2026-06-02] create | claude-code-config-files
- Created concepts/claude-code-config-files.md
- Source: live session — user asked about Claude Code config locations after observing ~/.claude.json
- Topics: ~/.claude.json (global + project-scoped MCPs, allowedTools, auth), ~/.claude/settings.json (model/theme/permissions), ~/.claude/CLAUDE.md, <project>/.claude/settings.json (committed), <project>/.claude/settings.local.json (gitignored), <project>/CLAUDE.md; restore checklist for new machines
- Key finding: ~/.claude.json is the *only* place global and project-scoped MCP definitions are stored — must be backed up separately, app manages it, not human-edited
- Cross-references: mcp-nordic-mcp-tools, cursor-skills-and-agents
- index.md: added entry under AI/Dev Tools, bumped total to 29

## [2026-06-02] config | set WIKI_PATH to ~/.claude/wiki
- Added `export WIKI_PATH="$HOME/.claude/wiki"` to ~/.zshrc
- ~/wiki (1 page, NCS-only stub created today) left in place but not the canonical wiki
- Canonical wiki confirmed as ~/.claude/wiki (29 pages)

## [2026-06-02] admin | deleted ~/wiki
- Removed ~/wiki (4 files: SCHEMA.md, index.md, log.md, concepts/kconfig-nrf-security-mbedtls-warnings.md)
- Content not migrated (user chose not to)
- ~/.claude/wiki is now the only wiki on this machine

## [2026-06-05] ingest | nordic-wifi-webdash memory comparison session
- Source: live session — measured webserver vs no-webserver on nRF7002DK (STA-only, NCS v3.3.0)
- Key findings:
  - webserver costs +44.5 KB Flash / +18.6 KB RAM on nRF5340 cpuapp
  - nRF5340 app core: 1 MB FLASH (full, PM disabled), 448 KB RAM (512 KB − 64 KB hci_ipc)
  - Flash breakdown: HTTP parser+server (~21 KB), webserver.c + static assets (~9.4 KB), mDNS+DNS-SD (~4 KB), remainder
  - RAM breakdown: HTTP client contexts × 10 (~15 KB), thread stack (2 KB), static buffers (1.5 KB)
  - CMakeLists fix: webserver.c was unconditionally compiled; now gated on CONFIG_WEBSERVER_MODULE
  - HTTP linker sections + gzip asset generation also gated on CONFIG_HTTP_SERVER
  - webdash-as-GUI analysis: not advisable on nRF7002DK (flash pressure), cautious yes on nRF54LM20DK, dev-only
- Files created:
  - concepts/nordic-wifi-webdash-memory.md (new page)
- Files updated:
  - entities/nrf7002dk.md (added Flash/RAM budget section with measured numbers)
  - index.md (added new page entry, bumped total to 30)
  - nordic-wifi-webdash/README.md (added Feature Overlay Builds section with memory table)

## [2026-06-08] lint | Wiki review — 0 P0, 0 P1, 7 P2 (oversized pages, deferred) | `d483f86`
- Checked 30 pages: frontmatter ✓, orphans ✓, broken links ✓, index ✓
- 7 split candidates: eedp-platform(431), wireguard-comprehensive-guide(369), github-actions-ncs-ci(367), hermes-architecture(330), wireguard-openwrt-china-tunnel(298), svg-pptx-agent-generation(285), dns-over-https-doh(202)

## [2026-06-13] debug | P2P_CLIENT static IP root cause found and fixed | `5317c7d8`
- `CONFIG_NET_IF_MAX_IPV4_COUNT=1` + `CONFIG_NET_CONFIG_MY_IPV4_ADDR="192.168.7.1"` occupies the only IPv4 slot at boot; must explicitly rm("192.168.7.1") before add("192.168.7.2")
- `driver_zephyr.c:set_supp_port` calls `net_dhcpv4_restart` 4ms after CONNECT_RESULT; counteract with deferred 100ms `net_dhcpv4_stop` work item
- wiki/concepts/wifi-debugging-patterns.md updated with Pattern N+1
- [2026-06-30] update | wifi-debugging-patterns.md: added Pattern N+2 (reason=34 DISASSOC_LOW_ACK; nRF70 FW stats diagnosis — RSSI/OFDM-CRC/tx_timeout/rpu_hw_lockup; reason 6 vs 34). Cross-linked new skill chsh-sk-ncs-tc-nrf70-fw-stats. Session 54f9d0bc, device F4CE3600230A.

## [2026-07-01] update | wifi-debugging-patterns.md: Pattern N+3 written, then corrected same day
- Initial (wrong) claim: P2P_GO/SoftAP hard-limited to 1 station on nRF70 in NCS v3.3.0, based on `nrf/samples/wifi/softap/Kconfig` (`SOFTAP_SAMPLE_MAX_STATIONS` `range 1 1`) + driver `MAX_PEERS=5`
- **User correction:** had already set and tested SoftAP with 3 simultaneous clients in this exact template
- Root cause of my error: conflated one Nordic reference sample's self-imposed array-size Kconfig with an actual driver/hardware ceiling. The real generic knob is `CONFIG_WIFI_MGMT_AP_MAX_NUM_STA` (`zephyr/subsys/net/l2/wifi/Kconfig`, range 1-2007, default 4) feeding hostapd's `max_num_sta` directly — project's own `docs/dev-specs/1-architecture.md` documents `=3`, tested and working
- Corrected finding: P2P_GO shares the identical hostapd AP code path as SoftAP, so the same station-count ceiling applies structurally; the actual reason P2P_GO only takes 1 client today is that `wifi_p2p_go_cancel_wps_timer()` disarms WPS PBC after the first connect and only re-arms on disconnect — an application-level design choice, not a hardware limit
- index.md: bumped `updated` to 2026-07-01, summary line corrected to reflect WPS-gate finding not HW limit

## [2026-07-02] update | ncs-build-system.md: Kconfig select-vs-depends-on circular dependency pattern + MBEDTLS_LEGACY_CRYPTO_C correction
- Fixed a real circular dependency (`ZEGO_NETWORK depends-on ZEGO_WIFI depends-on NETWORKING`, with `NETWORKING`'s `default y if ZEGO_NETWORK` unable to ever fire) in `zego/bricks/wifi` and `zego/bricks/network` Kconfig by replacing `depends on` + conditional `default` with `select` at the mandatory-requirement points — new pattern section documents this generically.
- Corrected an existing wiki section that said `MBEDTLS_LEGACY_CRYPTO_C` should be removed as deprecated — it's actually required on CRACEN/PSA-only boards (nRF54LM20DK) for `zego/memonitor`'s mbedTLS heap-debug linker symbols; ported the existing memfault fix into nordic-wifi-app-template/prj.conf instead of deleting it.
- Verified via pristine-build `.config` byte-diff against pre-change baselines on nRF54LM20DK + nRF7002DK for both nordic-wifi-app-template and nordic-wifi-memfault — zero diff, confirming ~16 lines of hardcoded `prj.conf` overrides (across both apps) and several board `.conf` lines could be safely commented out (not deleted) as now-redundant with brick defaults. Session a3f9bf71.

## [2026-07-03] update | ncs-version-migration.md: v3.3.0→v3.4.0 section (Mbed TLS 4.1.0 / TF-PSA-Crypto)
- Migrated `zego` from NCS v3.3.0 to v3.4.0: `CONFIG_NRF_SECURITY` became a computed symbol (hard Kconfig error if assigned), `CONFIG_MBEDTLS_LEGACY_CRYPTO_C` removed entirely, `CONFIG_MBEDTLS_MEMORY_DEBUG` no longer forwarded to generated PSA crypto headers (nrf_security's `psa_crypto_want_config.cmake` whitelist gap) — broke `zego/memonitor`'s mbedTLS heap stat, defaulted to `n` until Nordic exposes a forwarding path.
- New triage pattern for post-migration Kconfig deprecation warnings: grep the symbol across the app tree first — silent = inherited Zephyr/vendor default (app-controllable, e.g. `WIFI_NM_WPA_SUPPLICANT_LEGACY_CRYPTO`/`_WEP`); found only via `select` in vendor Kconfig (e.g. Nordic's `hostap_crypto/Kconfig` unconditionally selecting `MBEDTLS_ECP_C`/`BIGNUM_C`/`DECLARE_PRIVATE_IDENTIFIERS`) = not fixable without disabling the feature that needs it.
- Disabled `CONFIG_WIFI_NM_WPA_SUPPLICANT_WEP=n` in `nordic-wifi-app-template/prj.conf` (obsolete, unused by this app), kept `LEGACY_CRYPTO=y` for STA-mode TKIP interop with older/mixed-mode routers — user's explicit tradeoff choice. Verified via pristine rebuild on nRF54LM20DK: WEP warnings gone, build succeeds (FLASH 58.79%, RAM 85.11%). Session 4ea9621c.

## [2026-07-07] update | ncs-build-system.md: FIXED_PARTITION_* deprecation + WPA legacy-crypto warning triage
- Cleared all app-owned build warnings in `nordic-wifi-memfault`: renamed 3 files' + `pm_config.h`'s `FIXED_PARTITION_ID/OFFSET/SIZE` → `PARTITION_*` (deprecated in `zephyr/include/zephyr/storage/flash_map.h`, mechanical no-op rename); left the vendored `memfault-firmware-sdk` copy of the same pattern untouched (not app-owned, reverts on `west update`).
- Applied the same `WIFI_NM_WPA_SUPPLICANT_WEP=n` / `LEGACY_CRYPTO=n` decision already made in `nordic-wifi-app-template` (session 4ea9621c) to `nordic-wifi-memfault` too — this app is also personal WPA2/WPA3-PSK only, no legacy-AP requirement in its docs. Eliminated 8 deprecated/experimental/insecure warnings per board and shrank FLASH by ~4.5 KB (RC4/MD5 code dropped).
- Remaining warnings (`NRF_PLATFORM_LUMOS`, `NETCORE_HCI_IPC` choice, `MBEDTLS_ECP_C`/`BIGNUM_C`/`DECLARE_PRIVATE_IDENTIFIERS`, `PSA_WANT_ALG_WPA3_SAE_*`, `WIFI_NM_WPA_SUPPLICANT` experimental) confirmed as SoC/sysbuild defaults or hard functional dependencies (WPA3 + ECDSA-cert TLS support) — new wiki section documents the triage so this isn't re-investigated from scratch next time. Verified via pristine rebuild on both boards, zero errors, memory unchanged besides the FLASH drop above. Session d34b1836.
- Left the 5 vendored `memfault-firmware-sdk` deprecated-macro warnings (`FIXED_PARTITION_ADDRESS`/`_SIZE`, v1.40.0 pinned) untouched — confirmed via GitHub that upstream hasn't renamed them even in the latest release; `__DEPRECATED_MACRO` is a compile-time-only annotation (expansion unchanged), so zero functional risk. User's explicit choice to leave as-is over patching or filing upstream.
- Found and fixed a real CI gap while investigating: `nordic-wifi-memfault/.github/workflows/{validation,release}.yml` were missing the "Apply vendored nrf_security patches" step that `zego/.github/workflows/*.yml` already has — this app also needs `zego/patches/nrf_security/0001-forward-mbedtls-memory-debug.patch` (same `CONFIG_ZEGO_MEMONITOR_ZVIEW` mbedTLS-heap dependency documented in `prj.conf`) but CI was silently building without it. Ported the step verbatim from `zego`'s workflows into both files, right after `west update`/before the build step.

## [2026-07-07] create | psa-crypto-tfm-vs-no-tfm
- Created comparisons/psa-crypto-tfm-vs-no-tfm.md — Mbed TLS 4.x split (legacy TLS/X.509 now calls PSA Crypto API for all primitives) and the two NCS PSA implementation standards: TF-M Crypto Service (isolated, IPC) vs Oberon PSA Crypto (in-process, no isolation)
- Source: user question about not using TF-M with PSA crypto due to memory limitations; verified via Nordic docs (`nordicsemi_search_sources`) + live `nordic-wifi-memfault` `.config` (nRF54LM20A/cpuapp: `CONFIG_PSA_CRYPTO_DRIVER_CRACEN=y`, `CONFIG_BUILD_WITH_TFM` not set at all — working proof this config runs WPA2/WPA3 + concurrent mbedTLS TLS sessions with no TF-M)
- Key finding: TF-M is only "Experimental" on nRF54LM20A in NCS v3.4.0 (Oberon PSA Crypto is "Supported") — skipping TF-M is the recommended path on this chip family for RAM-constrained apps, not a workaround with gaps
- Key finding: CRACEN's KMU hardware key slots (`PSA_KEY_LOCATION_CRACEN_KMU`) work without TF-M too — hardware-backed keys don't strictly require TF-M isolation on nRF54L/LM; noted a community devzone report of `PSA_ERROR_NOT_SUPPORTED` on KMU persistent-key generation as a caveat to verify empirically if pursued
- Cross-referenced from comparisons/cracen-vs-oberon-tls.md (orthogonal driver-choice axis vs this isolation-choice axis)
- index.md: added entry under Comparisons, bumped total to 31

## [2026-07-07] fix | ncs-build-system.md: ZView LookupError on __weak extern k_heap symbols
- `west zview live` crashed with `LookupError: Symbol 'net_buf_mem_pool_rx_bufs' address not found.` on nordic-wifi-memfault (nRF54LM20DK, `CONFIG_NET_BUF_FIXED_DATA_SIZE=y`). Root cause: `zego/bricks/memonitor/src/memonitor.c` declares `extern __weak struct k_heap net_buf_mem_pool_rx_bufs;` so it compiles under any Kconfig combo, but this build never defines it (real symbol is `net_buf_fixed_rx_bufs` instead) — DWARF still records the bare `extern` as a declaration-only `DW_TAG_variable` DIE, and ZView's `elf_inspector.py::_parse()` collected it anyway (missing the same `DW_AT_declaration` guard already present on the sibling `DW_TAG_structure_type` branch).
- Fixed in `modules/tools/zview/src/backend/elf_inspector.py` (own repo, `chshzh/zview`, not vendored/third-party): added the `DW_AT_declaration` skip to the `DW_TAG_variable` branch, bumped `_CACHE_SCHEMA_VERSION` 2→3 to auto-invalidate stale caches. Verified directly via `ElfInspector` against the real ELF: the two undefined weak symbols now drop out, all 5 real heaps still resolve. New wiki section covers the general pattern (DWARF scanners over `__weak extern` globals must filter declaration-only DIEs). Session d34b1836.

## [2026-07-07] create | zephyr-assert-usage
- Created concepts/zephyr-assert-usage.md — documents `CONFIG_ASSERT` (zephyr/subsys/debug/Kconfig): compiles `__ASSERT()` to nothing when off (zero cost), or to a runtime check + `assert_post_action()` fatal error when on; covers sub-options (`ASSERT_LEVEL`, `ASSERT_VERBOSE`, `ASSERT_NO_MSG_INFO`/`NO_COND_INFO`/`NO_FILE_INFO`, `SPIN_VALIDATE`) and the distinction from the unrelated Bluetooth-stack `CONFIG_ASSERT_ON_ERRORS`.
- Recommendation for this workspace's NCS apps: leave off for release (flash/RAM budget is already tight — see ncs-build-system.md), enable via a debug overlay during development/debugging to catch invariant violations with file:line info.
- Cross-referenced from ncs-build-system.md and wifi-debugging-patterns.md; index.md updated, total pages 31→32.

## [2026-07-15] create | wifi-power-save-listen-interval
- Created concepts/wifi-power-save-listen-interval.md — documents the actual nRF70 STA power-save wake cadence, derived empirically via `wifi-fund/l6` Lesson 6 exercise + Power Profiler on nRF7002DK against an ASUS RT-BE92U AP.
- Key finding: `listen_interval` (units: beacon intervals, set via `WIFI_PS_PARAM_LISTEN_INTERVAL`) is quantized by firmware to the **nearest whole multiple of the DTIM period** before being used as the actual wake schedule — e.g. `listen_interval=10` with `dtim_period=3` yields a real wake cadence of `3×3=9` beacons, not 10. Formula: `wake_period = round(listen_interval/dtim_period) * dtim_period * beacon_interval_ms`.
- Secondary finding: `wifi status`'s reported `Beacon Interval` is in **TU (1 TU = 1.024ms)**, not ms — must be converted before use in any wake-timing math (200 TU → 204.8ms real, confirmed via DTIM-mode measurement: 614ms / 3 ≈ 204.7ms).
- Also documents `ps_exit_strategy` (`custom` vs `tim`) interaction: `tim` forces every-DTIM wake regardless of `wakeup_mode`/`listen_interval`, which is what exposed the quantization rule was independent of exit strategy (switching exit strategy alone didn't change the measured listen-interval cadence).
- Cross-referenced from wifi-debugging-patterns.md and nrf7002dk.md (both updated with backlinks); index.md updated, total pages 32→33.

## [2026-07-17] ingest | FCC and CE requirements for WiFi products (shared Claude chat)
- Ingested raw/articles/fcc-ce-wifi-requirements-claude-chat-2026-07-17.md (source_url: https://claude.ai/share/5a9fece2-e2df-447b-b2ee-a1e7d61d7db6). Initial WebFetch failed (JS-rendered share page returned empty shell); resolved via Playwright browser after user authenticated the session — share link required org login, not publicly readable.
- Created concepts/fcc-ce-target.md — intentional-radiator trigger, FCC modular transmitter approval + developmental-device marketing carve-out (why dev kits ship without full Certification), CE RED self-issued DoC (no modular-grant equivalent), chip-vs-module-vs-end-product framing, and why nRF7002DK's FCC pre-compliance testing doesn't transfer to nRF54LM20DK+nRF7002EB2 despite sharing the chip.
- Created concepts/fcc-ce-conducted-radiated-rf-test-methods.md — conducted (ANSI C63.10 / EN 300 328, antenna-port measurement) vs radiated/OTA (EIRP at a test site) methodology, why radiated spurious/harmonic checks are often still required even with a conducted connector, and the EN 300 328 multi-chain smart-antenna provision.
- **Correction flagged during ingest**: the source conversation claimed the EN 300 328 multi-chain provision was "relevant to nRF7002, which supports MIMO" — user (Charlie) confirmed nRF7002 is SISO, not MIMO-capable. Left the raw source untouched (immutable per convention) and added an explicit correction block in fcc-ce-conducted-radiated-rf-test-methods.md; set `contested: true` in that page's frontmatter.
- Updated entities/nrf7002dk.md and entities/nrf54lm20dk-plus-nrf7002eb2.md with new "RF Compliance Status" sections + backlinks to both new pages (bumped `updated` on both).
- index.md: added both new pages under Concepts > Nordic/NCS (alphabetical), total pages 33→35, last-updated bumped to 2026-07-17.
