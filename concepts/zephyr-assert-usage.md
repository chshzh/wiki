---
title: Zephyr CONFIG_ASSERT — Behavior, Sub-Options, and When to Enable
created: 2026-07-07
updated: 2026-07-07
type: concept
tags: [ncs, zephyr, kconfig, debug, pattern]
sources: []
confidence: high
---

# Zephyr `CONFIG_ASSERT`

## What it does

`CONFIG_ASSERT` (`zephyr/subsys/debug/Kconfig`) gates whether the `__ASSERT()` macro
family compiles to real runtime checks or to nothing.

```c
config ASSERT
	bool "__ASSERT() macro"
	default y if TEST
```

- **`CONFIG_ASSERT=n`** (default for non-test builds): every `__ASSERT()` /
  `__ASSERT_NO_MSG()` call site compiles away completely — zero flash/RAM cost, zero
  runtime cost. This is why production/release builds ship with it off.
- **`CONFIG_ASSERT=y`**: the condition is evaluated at runtime. On failure it prints a
  message via `__ASSERT_PRINT` (file/line/condition, depending on sub-options) and calls
  `assert_post_action()`, which by default triggers a fatal error (system halt/oops).
- Automatically forced on for Twister/ztest builds (`default y if TEST`) since tests rely
  on assertions firing to catch bugs.

## Sub-options (only visible when `ASSERT=y`)

| Kconfig | Effect |
|---|---|
| `CONFIG_ASSERT_LEVEL` (0–2, default 2) | 0 = checks run but no message printed; 1 = checks + compile-time warning in every including file; 2 = checks, no warning. Note: level 0 still keeps the runtime check + `assert_post_action()`, unlike `ASSERT=n` which removes the check entirely. |
| `CONFIG_ASSERT_VERBOSE` | Whether the failure message prints at all. |
| `CONFIG_ASSERT_NO_MSG_INFO` | Drop the custom message text from `__ASSERT_MSG_INFO`-style calls. |
| `CONFIG_ASSERT_NO_COND_INFO` | Drop the stringified condition from the printed message. |
| `CONFIG_ASSERT_NO_FILE_INFO` | Drop file/line from the printed message. |
| `CONFIG_SPIN_VALIDATE` | Spinlock validation framework, depends on `ASSERT`; adds ~3KB. `default y if !FLASH || FLASH_SIZE > 32`. |

The `NO_*_INFO` options exist specifically to trim flash usage on constrained targets while
still keeping the pass/fail check itself.

## Don't confuse with `CONFIG_ASSERT_ON_ERRORS`

`CONFIG_ASSERT_ON_ERRORS` (`zephyr/Kconfig.zephyr`) is unrelated — it's a Bluetooth-stack
option that asserts on certain function return codes instead of just logging them. Same
"assert" word, different subsystem, different Kconfig file.

## Recommended usage pattern for NCS apps in this workspace

- **Release/production builds**: leave `CONFIG_ASSERT=n` (the default). Saves flash/RAM
  and avoids a full system halt in the field from a non-fatal logic error — see
  [ncs-build-system](ncs-build-system.md) for the flash-budget pressure these apps
  already run under (Memfault + TLS eat a lot of the budget).
- **Development/debugging**: set `CONFIG_ASSERT=y` (e.g. via a debug overlay, not
  committed `prj.conf`) to catch invariant violations immediately with a clear
  file:line message, instead of silent corruption that surfaces later as a much
  harder-to-diagnose crash or hang. Pairs naturally with the debug workflow in
  `chsh-sk-ncs-3.2-debug`.
- If flash is tight on a debug build but you still want the check (not just the
  message), use `CONFIG_ASSERT_LEVEL=0` or the `NO_*_INFO` variants rather than
  disabling `ASSERT` outright — you keep the halt-on-violation behavior, just quieter.

## Related Pages
- [ncs-build-system](ncs-build-system.md) — general Kconfig/build patterns for these NCS apps
- [wifi-debugging-patterns](wifi-debugging-patterns.md) — runtime failure patterns where a targeted debug-build assert would help isolate root cause faster
