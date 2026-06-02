# Wiki Index

> Content catalog. Every wiki page listed under its type with a one-line summary.
> Read this first to find relevant pages for any query.
> Last updated: 2026-06-02 | Total pages: 29

## Entities

- [hermes-setup-on-linux-with-nas-nfs-backup](entities/hermes-setup-on-linux-with-nas-nfs-backup.md) — NAS NFS architecture: Hermes data on UGreen NAS, recovery by mounting NFS, zero reconfig
- [nrf7002dk](entities/nrf7002dk.md) — nRF7002DK (PCA10143): board target, USB port, behavior notes, project support matrix
- [nrf54lm20dk-plus-nrf7002eb2](entities/nrf54lm20dk-plus-nrf7002eb2.md) — nRF54LM20DK+nRF7002EB2 (PCA10184): shield flag, provisioner reset bug, SPI interface notes

## Concepts

### Nordic / NCS
- [eedp-platform](concepts/eedp-platform.md) — NCS v3.3.0 workspace reference (repo layout, build targets, skills) + EEDP: AI-controlled physical hardware (GPIO/JLink/Saleae/PPK2/Router) for embedded testing
- [ncs-app-versioning](concepts/ncs-app-versioning.md) — Four-field `MAJOR.MINOR.PATCH.APP` versioning scheme for NCS workspace apps
- [ncs-build-system](concepts/ncs-build-system.md) — West build commands, zephyr-env.sh gotcha, OVERLAY_CONFIG→EXTRA_CONF_FILE, board targets, flash partitioning
- [ncs-version-migration](concepts/ncs-version-migration.md) — Iterative grep/fix pattern, known API changes per NCS version, realistic time estimates
- [memfault-version-requirements](concepts/memfault-version-requirements.md) — Memfault's allowed characters, ordering algorithm, and `v`-prefix caveat
- [memfault-workflow](concepts/memfault-workflow.md) — Build→flash→symbol upload→release→deploy loop; MQTT OTA topic; crash/WiFi reconnect interaction
- [wifi-debugging-patterns](concepts/wifi-debugging-patterns.md) — Six failure patterns: timeout reboot loop, provisioner reset, headset autoconnect, wrong password, BLE provisioning, P2P steps
- [embedded-system-general-debugging](concepts/embedded-system-general-debugging.md) — Debugging lessons from sQSPI driver implementation: barriers, multi-device testing, loop scripts
- [mcp-nordic-mcp-tools](concepts/mcp-nordic-mcp-tools.md) — Nordic MCP server tools and best practices for NCS build/flash/UART workflows
- [github-actions-ncs-ci](concepts/github-actions-ncs-ci.md) — GitHub Actions CI for NCS firmware: Docker container approach, rolling releases, pre-built firmware test loop, idempotent git am, artifact naming, manifest pitfalls
- [west-update-internals](concepts/west-update-internals.md) — How `west update` works: workspace discovery, manifest resolution, manifest-rev branch, detached HEAD semantics, floating vs pinned revisions

### AI / Dev Tools
- [claude-code-config-files](concepts/claude-code-config-files.md) — Six config file locations (global MCPs in .claude.json, settings.json, CLAUDE.md, project .claude/) with restore checklist
- [cursor-skills-and-agents](concepts/cursor-skills-and-agents.md) — Skills vs agents: invocation model, delegation pattern, design rules for `SKILL.md` and `~/.claude/agents/`
- [deepseek-claude-code](concepts/deepseek-claude-code.md) — DeepSeek Anthropic API 兼容层直连 Claude Code：配置、模型策略、验证
- [svg-pptx-agent-generation](concepts/svg-pptx-agent-generation.md) — AI Agent 视觉内容生成完整工作流：SVG（纯 XML 手写）→ PPTX（PptxGenJS），以电车运行管理 PPT 为案例

### Homelab / Networking
- [hermes-architecture](concepts/hermes-architecture.md) — Hermes Agent 源码架构与功能全景
- [hermes-docker-compose-deployment](concepts/hermes-docker-compose-deployment.md) — Hermes Agent + WebUI Docker Compose 部署：init 容器权限修正、三服务编排、NAS 持久化（~5 分钟阅读）
- [linux-vm-docker-permission](concepts/linux-vm-docker-permission.md) — Linux VM / Docker UID 权限对齐：UID 不匹配根因、五种解决模式、排错清单
- [dns-over-https-doh](concepts/dns-over-https-doh.md) — DNS-over-HTTPS: how it protects browsing in office environments, threat model, DoH vs DoT, setup guide
- [ttyd-web-terminal-setup](concepts/ttyd-web-terminal-setup.md) — ttyd Web 终端：安装、使用、systemd 自启、oh-my-bash 时钟前缀修复、与 noVNC/SSH 对比
- [wireguard-comprehensive-guide](concepts/wireguard-comprehensive-guide.md) — WireGuard 全面指南：架构、组网（hub-spoke/site-to-site/mesh）、嵌入式应用（OpenWrt/Zephyr/nRF9160/boringtun）、性能对比
- [wireguard-openwrt-china-tunnel](concepts/wireguard-openwrt-china-tunnel.md) — OpenWrt WireGuard 翻墙隧道：完整配置 + 5 排坑 + 多设备扩展

## Comparisons

- [cracen-vs-oberon-tls](comparisons/cracen-vs-oberon-tls.md) — CRACEN (hardware, nRF54LM20DK) vs Oberon software mbedTLS (nRF7002DK): execution model, stack depth, KMU, speed trade-offs for TLS
- [memfault-mcp-vs-cli](comparisons/memfault-mcp-vs-cli.md) — Memfault MCP server (read-only fleet query, crash decode) vs Memfault CLI (symbol upload, OTA release, cohort deploy): capability matrix + combined release verification workflow
- [agentmemory-vs-llm-wiki](comparisons/agentmemory-vs-llm-wiki.md) — Why session logs ≠ experience; when to use each memory system; agentmemory as raw material, wiki as distilled product
- [nrf7002dk-vs-nrf54lm20dk](comparisons/nrf7002dk-vs-nrf54lm20dk.md) — SPI vs QSPI throughput, provisioner stability, two-board debug strategy, when a bug is board-specific

## Queries
