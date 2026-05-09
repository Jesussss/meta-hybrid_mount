# Hybrid Mount

<img src="https://raw.githubusercontent.com/Hybrid-Mount/meta-hybrid_mount/main/icon.svg" align="right" width="120" />

![Language](https://img.shields.io/badge/Language-Rust-orange?style=flat-square&logo=rust)
![Platform](https://img.shields.io/badge/Platform-Android-green?style=flat-square&logo=android)
![License](https://img.shields.io/badge/License-Apache--2.0-blue?style=flat-square)
![Version](https://img.shields.io/badge/Version-4.0-8A2BE2?style=flat-square)

Hybrid Mount is a mount orchestration metamodule for **KernelSU** and **APatch**.
It merges module files into Android partitions through a unified policy engine backed by three mount backends:

- **OverlayFS** — layered mounts for broad compatibility.
- **Magic Mount** — bind-mount for direct path replacement or fallback.
- **Kasumi** — LKM-backed routing with runtime hide, spoof, and stealth features.

A built-in **SolidJS WebUI** provides graphical management, live state monitoring, and configuration editing.

**[🇨🇳 中文文档](README_ZH.md)**

---

## Table of Contents

- [Features](#features)
- [Quick Start](#quick-start)
- [Mount Modes](#mount-modes)
- [WebUI](#webui)
- [Configuration](#configuration)
- [Kasumi](#kasumi)
- [Policy Reference](#policy-reference)
- [CLI](#cli)
- [Architecture](#architecture)
- [Build](#build)
- [Operational Notes](#operational-notes)
- [License](#license)

---

## Features

- **Three backends, one policy engine** — assign paths to OverlayFS, Magic Mount, or Kasumi with per-path granularity.
- **Deterministic planning** — conflicts are detected at plan time, not discovered randomly at boot.
- **Built-in WebUI** — manage modules, edit configuration, monitor runtime state, and control Kasumi features from a browser or WebView.
- **Kasumi runtime integration** — LKM autoload, mirror routing, mount hiding, maps/statfs spoofing, UID hiding, uname spoofing, and kstat rules.
- **Recovery-friendly** — stale runtime files are cleaned automatically; misconfigurations can be reset with `gen-config`.
- **Automation-friendly** — JSON-over-Unix-socket daemon protocol for scripting or external controllers.

---

## Quick Start

### Installation

1. Install [KernelSU](https://kernelsu.org/) or [APatch](https://apatch.dev/) on your device.
2. Download the latest Hybrid Mount release ZIP from [GitHub Releases](https://github.com/Hybrid-Mount/meta-hybrid_mount/releases).
3. Flash the ZIP through your root manager's module installer.
4. Reboot. Hybrid Mount will auto-detect your environment and apply the default overlay policy.

### Post-install

```bash
# Check runtime status
hybrid-mount daemon status

# List detected modules
hybrid-mount modules

# Open the WebUI in your browser
# (the daemon prints the URL to logcat on startup)
```

### Changing mount mode for a module

```toml
# /data/adb/hybrid-mount/config.toml
[rules.my_module]
default_mode = "magic"

[rules.my_module.paths]
"system/bin/problematic_binary" = "ignore"
```

---

## Mount Modes

| Mode | Backend | Best for |
|------|---------|----------|
| `overlay` | OverlayFS | Modules that add or replace files without conflicts. Default mode. |
| `magic` | Bind mount | Modules that need direct per-file replacement; fallback when OverlayFS is unavailable. |
| `kasumi` | Kasumi LKM | Modules requiring explicit mirror routing or runtime hide/spoof features. |
| `ignore` | — | Excluding specific paths from any mount processing. |

### OverlayFS storage modes

The OverlayFS backend supports two storage strategies for the upper/work layers:

- `ext4` (default) — creates an ext4 disk image. Persists across reboots, supports xattr.
- `tmpfs` — uses a tmpfs mount. Volatile, lighter weight, but lost on reboot.

```toml
overlay_mode = "ext4"
```

### Fallback behavior

When `enable_overlay_fallback = true`, modules planned for OverlayFS that cannot mount (kernel lacks overlay support) automatically retry as Magic Mount. This reduces boot-time failures on kernels with unstable overlay support.

---

## WebUI

Hybrid Mount includes a **SolidJS-based WebUI** served by the daemon over a local TCP socket. The daemon prints the access URL to logcat on startup.

### Capabilities

- **Status dashboard** — live mount statistics, active partitions, storage mode, daemon health.
- **Module management** — list all detected modules with their effective mount modes; apply mode changes interactively.
- **Configuration editor** — full config.toml editing with validation, including per-module path rules.
- **Kasumi control panel** — LKM status, rule listing, feature toggles, uname configuration, maps/kstat rules.

### Access

The WebUI runs on `http://127.0.0.1:<random-port>` with a cryptographic access token. The daemon manages the lifecycle — no separate web server needed. On-device, use a browser or WebView; remotely, forward the port via ADB.

---

## Configuration

Default path: `/data/adb/hybrid-mount/config.toml`.

### Top-level fields

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `moduledir` | string | `/data/adb/modules` | Module source directory. |
| `mountsource` | string | auto-detect | Runtime source tag (`KSU`, `APatch`). |
| `partitions` | list | `[]` | Extra managed partitions. |
| `overlay_mode` | `ext4` \| `tmpfs` | `ext4` | Overlay upper/work storage mode. |
| `disable_umount` | bool | `false` | Skip umount operations (debug only). |
| `enable_overlay_fallback` | bool | `false` | Retry overlay-planned modules as Magic Mount when OverlayFS is unavailable. |
| `default_mode` | `overlay` \| `magic` \| `kasumi` | `overlay` | Global default mount policy. |
| `rules` | map | `{}` | Per-module and per-path mount policies. |

### Example

```toml
moduledir = "/data/adb/modules"
mountsource = "KSU"
partitions = ["system", "vendor"]
overlay_mode = "ext4"
enable_overlay_fallback = true
default_mode = "overlay"

[rules.viper4android]
default_mode = "magic"

[rules.viper4android.paths]
"system/etc/audio_policy.conf" = "overlay"

[rules.sensitive_module]
default_mode = "kasumi"

[rules.sensitive_module.paths]
"system/bin/helper" = "kasumi"
"system/etc/placeholder" = "ignore"
```

---

## Kasumi

Kasumi is the **LKM-backed** backend. Beyond mount routing, it provides a suite of runtime hide and spoof capabilities.

### Activation

Setting `kasumi.enabled = true` makes the backend available. The Kasumi runtime is actually enabled when at least one of these conditions is met:

- The mount plan contains a Kasumi-managed module or path.
- An auxiliary feature is configured (hidexattr, mount hide, maps spoof, statfs spoof, UID hiding, uname spoof, cmdline replacement, kstat rules, or user hide rules).

### Key config fields

| Field | Purpose |
| --- | --- |
| `kasumi.enabled` | Master switch for Kasumi integration. |
| `kasumi.lkm_autoload` | Auto-load the Kasumi LKM during startup. |
| `kasumi.lkm_dir` | LKM search directory. |
| `kasumi.lkm_kmi_override` | Optional KMI version override for LKM selection. |
| `kasumi.mirror_path` | Mirror root used by Kasumi rules (default `/dev/kasumi_mirror`). |
| `kasumi.enable_kernel_debug` | Toggle kernel-side debug logging. |
| `kasumi.enable_stealth` | Explicit stealth mode. |
| `kasumi.enable_hidexattr` | Compatibility umbrella — enables stealth, mount hide, maps spoof, and statfs spoof together. |
| `kasumi.enable_mount_hide` | Hide mounts globally or by path pattern. |
| `kasumi.mount_hide.path_pattern` | Path pattern for mount hiding. |
| `kasumi.enable_maps_spoof` | Enable `/proc/<pid>/maps` spoofing. |
| `kasumi.maps_rules` | Per-inode/device maps rewrite rules. |
| `kasumi.enable_statfs_spoof` | Enable `statfs` spoofing. |
| `kasumi.statfs_spoof.path` / `.spoof_f_type` | Path-scoped statfs spoof configuration. |
| `kasumi.hide_uids` | UIDs to hide from Kasumi-aware queries. |
| `kasumi.uname.*` | Structured uname spoof (sysname, release, version, machine). |
| `kasumi.cmdline_value` | Replacement `/proc/cmdline` content. |
| `kasumi.kstat_rules` | Per-target stat metadata spoof rules. |

### Commands

```bash
# Status and diagnostics
hybrid-mount kasumi status
hybrid-mount kasumi version
hybrid-mount kasumi features
hybrid-mount kasumi list          # list active rules
hybrid-mount lkm status

# Enable / disable runtime features
hybrid-mount kasumi enable
hybrid-mount kasumi disable

# Mount hiding
hybrid-mount kasumi mount-hide enable --path-pattern /dev/kasumi_mirror

# statfs spoofing
hybrid-mount kasumi statfs-spoof enable --path /system --f-type 0x794c7630

# Maps spoof rules
hybrid-mount kasumi maps add \
  --target-ino 1 --target-dev 2 \
  --spoofed-ino 3 --spoofed-dev 4 \
  --path /dev/kasumi_mirror/system/bin/sh

# Kstat spoof rules
hybrid-mount kasumi kstat upsert \
  --target-ino 11 --target-path /system/bin/app_process64 \
  --spoofed-ino 22 --spoofed-dev 33

# Rule management
hybrid-mount kasumi rule add --target /system/bin/tool --source /data/adb/modules/my_module/system/bin/tool
hybrid-mount kasumi rule merge --target /system/lib64 --source /data/adb/modules/my_module/system/lib64
hybrid-mount kasumi rule hide --path /system/bin/su
hybrid-mount kasumi rule delete --path /system/bin/old_tool
```

---

## Policy Reference

### Precedence

When multiple policies could apply to a path, evaluation order is:

1. **Path-level override** — `rules.<module>.paths["<path>"]`
2. **Module-level default** — `rules.<module>.default_mode`
3. **Global default** — `default_mode`

### Behavior matrix

| Rule result | Backend available? | `enable_overlay_fallback` | Effective behavior |
| --- | --- | --- | --- |
| `overlay` | Yes | any | Mount with OverlayFS. |
| `overlay` | No | `false` | Skip and report as failed. |
| `overlay` | No | `true` | Retry as Magic Mount. |
| `magic` | n/a | any | Mount with Magic Mount. |
| `kasumi` | Yes | any | Route through Kasumi. |
| `kasumi` | No | any | Skip Kasumi mapping. |
| `ignore` | n/a | any | Do not mount. |

### Practical recipes

- **One problematic binary on bind mount, rest on overlay**: set module default to `overlay`, override the binary path to `magic`.
- **Temporarily exclude a conflicting file**: set the path to `ignore`.
- **Kernel with flaky OverlayFS**: set `enable_overlay_fallback = true`.

---

## CLI

```bash
hybrid-mount [OPTIONS] [COMMAND]
```

### Global options

| Flag | Description |
|------|-------------|
| `-c, --config <PATH>` | Custom config file path. |
| `-m, --moduledir <PATH>` | Override module directory. |
| `-s, --mountsource <SOURCE>` | Override source tag. |
| `-p, --partitions <CSV>` | Override partition list. |

### Subcommands

| Command | Description |
|---------|-------------|
| `gen-config` | Generate a default config file. |
| `show-config` | Print effective config as JSON. |
| `save-config --payload <HEX_JSON>` | Save config from a WebUI payload. |
| `save-module-rules --module <ID> --payload <HEX_JSON>` | Update rules for one module. |
| `modules` | List detected modules. |
| `daemon status` | Query daemon runtime state. |
| `daemon stop` | Stop the daemon. |
| `kasumi ...` | Kasumi management (see [Kasumi](#kasumi)). |
| `lkm load / unload / status` | LKM lifecycle management. |
| `hide list / add / remove / apply` | User hide rule management. |

---

## Architecture

```
┌─────────────────────────────────────────────┐
│                  config.toml                  │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│              Inventory Discovery              │
│         Scan module tree, classify entries    │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│              Mount Planner                    │
│    Evaluate rules (path > module > global)    │
│    Generate overlay / magic / kasumi plan     │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│              Executors                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ OverlayFS│ │  Magic   │ │   Kasumi     │ │
│  │ executor │ │  Mount   │ │   executor   │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│            Runtime State + Daemon             │
│   Persist state → Unix socket → WebUI/CLI     │
└─────────────────────────────────────────────┘
```

### Source layout

```
src/
├── conf/          Config schema, TOML loader, CLI definition
├── domain/        Core types: MountMode, ModuleRules, path matching
├── core/
│   ├── inventory/ Module discovery and listing
│   ├── ops/       Mount plan generation and per-backend execution
│   ├── daemon/    Unix socket server, HTTP/SSE, protocol
│   ├── api/       Payload builders for WebUI endpoints
│   └── startup/   Boot sequence, recovery, retry logic
├── mount/
│   ├── overlayfs/ OverlayFS backend (ext4 image / tmpfs)
│   ├── magic_mount/ Bind-mount backend
│   └── kasumi/    Kasumi rule compilation, runtime, status
├── sys/           Low-level: mount syscalls, LKM load/unload, Kasumi UAPI
└── utils/         Logging, path utilities, validation

webui/
├── src/
│   ├── routes/    Page components (Status, Config, Modules, Kasumi, Info)
│   ├── components/ Shared UI components (NavBar, Toast, Skeleton)
│   └── lib/       API bridge, stores, codecs, i18n
└── locales/       9-language internationalization

xtask/             Build and release automation
module/            Module packaging scripts and static assets
```

---

## Build

### Prerequisites

- Rust nightly (from `rust-toolchain.toml`)
- Android NDK r27+ and `cargo-ndk`
- Node.js 20+ and pnpm (for WebUI)

### Commands

```bash
# Full release package (binary + WebUI) → output/
cargo run -p xtask -- build --release

# Binary only (skip WebUI)
cargo run -p xtask -- build --release --skip-webui

# Local arm64 debug build
./scripts/build-local.sh

# Local build with prebuilt Kasumi LKM .ko assets
./scripts/build-local.sh --release --kasumi-lkm-dir /path/to/kasumi-lkm

# WebUI dev server (hot reload)
cd webui && pnpm install && pnpm dev

# Lint everything
cargo run -p xtask -- lint
cd webui && pnpm lint

# Run tests
cargo +nightly test
cd webui && pnpm test
```

### Release profile

The release profile uses `opt-level = 3`, `lto = "fat"`, `codegen-units = 1`, `strip = true`, and `panic = "abort"` for minimal binary size.

---

## Operational Notes

- **Mount source auto-detection**: fresh installs detect the runtime environment automatically. Only set `mountsource` explicitly if auto-detection fails.
- **Recovery from bad config**: run `hybrid-mount gen-config` to reset to defaults, then reapply rules incrementally.
- **Kasumi LKM**: the LKM must match the running kernel. Use `lkm_kmi_override` if the auto-detected KMI is incorrect.
- **`kasumi kstat clear-config`**: only removes persisted config. Existing kernel-side rules persist until LKM reload or runtime rebuild.
- **Binary size**: prefer dependency feature trimming and profile tuning before invasive refactoring.

---

## License

Licensed under [Apache-2.0](LICENSE).
