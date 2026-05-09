# Hybrid Mount

<img src="https://raw.githubusercontent.com/Hybrid-Mount/meta-hybrid_mount/main/icon.svg" align="right" width="120" />

![Language](https://img.shields.io/badge/Language-Rust-orange?style=flat-square&logo=rust)
![Platform](https://img.shields.io/badge/Platform-Android-green?style=flat-square&logo=android)
![License](https://img.shields.io/badge/License-Apache--2.0-blue?style=flat-square)
![Version](https://img.shields.io/badge/Version-4.0-8A2BE2?style=flat-square)

Hybrid Mount 是面向 **KernelSU** 与 **APatch** 的挂载编排元模块。
通过统一的策略引擎，将模块文件合并到 Android 分区，并支持三种挂载后端：

- **OverlayFS** — 分层挂载，兼容性优先。
- **Magic Mount** — bind mount，适合直接路径替换或回退场景。
- **Kasumi** — LKM 内核驱动，提供运行时 hide/spoof/stealth 能力。

内置 **SolidJS WebUI**，支持图形化管理、实时状态监控和配置编辑。

**[🇺🇸 English](README.md)**

---

## 目录

- [特性](#特性)
- [快速开始](#快速开始)
- [挂载方式](#挂载方式)
- [WebUI](#webui)
- [配置说明](#配置说明)
- [Kasumi](#kasumi)
- [策略参考](#策略参考)
- [CLI 命令](#cli-命令)
- [架构说明](#架构说明)
- [构建方式](#构建方式)
- [运维建议](#运维建议)
- [开源协议](#开源协议)

---

## 特性

- **三种后端，统一策略引擎** — 支持按路径粒度分配 OverlayFS、Magic Mount 或 Kasumi。
- **确定性规划** — 冲突在计划阶段检出，而非启动时随机出现。
- **内置 WebUI** — 通过浏览器或 WebView 管理模块、编辑配置、监控运行时状态、控制 Kasumi 特性。
- **Kasumi 运行时集成** — LKM 自动加载、mirror 路由、mount 隐藏、maps/statfs 伪装、UID 隐藏、uname 伪装、kstat 规则。
- **恢复友好** — 残留运行时文件自动清理；配置错误时可通过 `gen-config` 重置。
- **自动化友好** — 基于 Unix socket 的 JSON 守护进程协议，便于脚本和外部控制器调用。

---

## 快速开始

### 安装

1. 在设备上安装 [KernelSU](https://kernelsu.org/) 或 [APatch](https://apatch.dev/)。
2. 从 [GitHub Releases](https://github.com/Hybrid-Mount/meta-hybrid_mount/releases) 下载最新版本的 Hybrid Mount 刷入包。
3. 通过 Root 管理器的模块安装器刷入 ZIP。
4. 重启设备。Hybrid Mount 将自动检测运行环境并应用默认 overlay 策略。

### 安装后

```bash
# 查看运行时状态
hybrid-mount daemon status

# 列出已检测到的模块
hybrid-mount modules

# 在浏览器中打开 WebUI
# （守护进程启动时会将 URL 打印到 logcat）
```

### 更改模块的挂载方式

```toml
# /data/adb/hybrid-mount/config.toml
[rules.my_module]
default_mode = "magic"

[rules.my_module.paths]
"system/bin/problematic_binary" = "ignore"
```

---

## 挂载方式

| 模式 | 后端 | 适用场景 |
|------|------|----------|
| `overlay` | OverlayFS | 无冲突地新增或替换文件的模块。默认模式。 |
| `magic` | Bind mount | 需要逐文件直接替换的模块；OverlayFS 不可用时的回退方案。 |
| `kasumi` | Kasumi LKM | 需要显式 mirror 路由或运行时 hide/spoof 能力的模块。 |
| `ignore` | — | 排除特定路径，不进行任何挂载处理。 |

### OverlayFS 存储模式

OverlayFS 后端支持两种 upper/work 层存储策略：

- `ext4`（默认）— 创建 ext4 磁盘镜像。重启后持久保留，支持 xattr。
- `tmpfs` — 使用 tmpfs 挂载。易失性、更轻量，但重启后丢失。

```toml
overlay_mode = "ext4"
```

### 回退行为

当 `enable_overlay_fallback = true` 时，计划走 OverlayFS 但无法挂载的模块（内核不支持 overlay）会自动以 Magic Mount 重试。这可减少不稳定内核上的启动失败概率。

---

## WebUI

Hybrid Mount 内置 **基于 SolidJS 的 WebUI**，由守护进程通过本地 TCP socket 提供服务。守护进程启动时会将访问 URL 打印到 logcat。

### 功能

- **状态面板** — 实时挂载统计、活跃分区、存储模式、守护进程健康状态。
- **模块管理** — 列出所有已检测模块及其生效的挂载方式；交互式修改模块策略。
- **配置编辑器** — 完整的 config.toml 编辑，带校验，支持逐模块路径规则配置。
- **Kasumi 控制面板** — LKM 状态、规则列表、特性开关、uname 配置、maps/kstat 规则管理。

### 访问方式

WebUI 运行在 `http://127.0.0.1:<随机端口>`，使用加密访问令牌。守护进程管理整个生命周期，无需额外的 Web 服务器。在设备上使用浏览器或 WebView 访问；远程访问可通过 ADB 端口转发。

---

## 配置说明

默认路径：`/data/adb/hybrid-mount/config.toml`。

### 顶层字段

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `moduledir` | string | `/data/adb/modules` | 模块目录。 |
| `mountsource` | string | 自动检测 | 运行来源标识（`KSU`、`APatch`）。 |
| `partitions` | list | `[]` | 额外受管分区。 |
| `overlay_mode` | `ext4` \| `tmpfs` | `ext4` | Overlay upper/work 存储模式。 |
| `disable_umount` | bool | `false` | 跳过 umount（仅调试使用）。 |
| `enable_overlay_fallback` | bool | `false` | OverlayFS 不可用时，将 overlay 模块回退到 Magic Mount。 |
| `default_mode` | `overlay` \| `magic` \| `kasumi` | `overlay` | 全局默认挂载策略。 |
| `rules` | map | `{}` | 按模块和路径的细粒度挂载策略。 |

### 示例

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

Kasumi 是 **LKM 内核驱动**后端。除挂载路由外，还提供一系列运行时 hide 和 spoof 能力。

### 启用条件

`kasumi.enabled = true` 仅使后端可用。Hybrid Mount 在满足以下条件之一时才会实际启用 Kasumi 运行时：

- 生成的挂载计划中包含 Kasumi 管理的模块或路径。
- 配置了任一辅助特性（hidexattr、mount hide、maps spoof、statfs spoof、UID 隐藏、uname 伪装、cmdline 替换、kstat 规则或用户 hide 规则）。

### 关键配置项

| 字段 | 作用 |
| --- | --- |
| `kasumi.enabled` | Kasumi 集成总开关。 |
| `kasumi.lkm_autoload` | 启动时自动加载 Kasumi LKM。 |
| `kasumi.lkm_dir` | LKM 搜索目录。 |
| `kasumi.lkm_kmi_override` | 可选的 KMI 版本覆盖，用于 LKM 匹配。 |
| `kasumi.mirror_path` | Kasumi 规则使用的 mirror 根目录（默认 `/dev/kasumi_mirror`）。 |
| `kasumi.enable_kernel_debug` | 开启内核侧调试日志。 |
| `kasumi.enable_stealth` | 显式启用 stealth 模式。 |
| `kasumi.enable_hidexattr` | 兼容模式总开关，联动启用 stealth、mount hide、maps spoof、statfs spoof。 |
| `kasumi.enable_mount_hide` | 全局或按路径模式隐藏挂载点。 |
| `kasumi.mount_hide.path_pattern` | 挂载隐藏的路径匹配模式。 |
| `kasumi.enable_maps_spoof` | 启用 `/proc/<pid>/maps` 伪装。 |
| `kasumi.maps_rules` | 按 inode/device 的 maps 重写规则。 |
| `kasumi.enable_statfs_spoof` | 启用 `statfs` 伪装。 |
| `kasumi.statfs_spoof.path` / `.spoof_f_type` | 按路径的 statfs 伪装配置。 |
| `kasumi.hide_uids` | 对 Kasumi 查询隐藏的 UID 集合。 |
| `kasumi.uname.*` | 结构化 uname 伪装（sysname、release、version、machine）。 |
| `kasumi.cmdline_value` | 替换 `/proc/cmdline` 内容。 |
| `kasumi.kstat_rules` | 按目标的 stat 元数据伪装规则。 |

### 常用命令

```bash
# 状态与诊断
hybrid-mount kasumi status
hybrid-mount kasumi version
hybrid-mount kasumi features
hybrid-mount kasumi list          # 列出活跃规则
hybrid-mount lkm status

# 启用/禁用运行时特性
hybrid-mount kasumi enable
hybrid-mount kasumi disable

# 挂载隐藏
hybrid-mount kasumi mount-hide enable --path-pattern /dev/kasumi_mirror

# statfs 伪装
hybrid-mount kasumi statfs-spoof enable --path /system --f-type 0x794c7630

# Maps 伪装规则
hybrid-mount kasumi maps add \
  --target-ino 1 --target-dev 2 \
  --spoofed-ino 3 --spoofed-dev 4 \
  --path /dev/kasumi_mirror/system/bin/sh

# Kstat 伪装规则
hybrid-mount kasumi kstat upsert \
  --target-ino 11 --target-path /system/bin/app_process64 \
  --spoofed-ino 22 --spoofed-dev 33

# 规则管理
hybrid-mount kasumi rule add --target /system/bin/tool --source /data/adb/modules/my_module/system/bin/tool
hybrid-mount kasumi rule merge --target /system/lib64 --source /data/adb/modules/my_module/system/lib64
hybrid-mount kasumi rule hide --path /system/bin/su
hybrid-mount kasumi rule delete --path /system/bin/old_tool
```

---

## 策略参考

### 优先级

当多个策略可能同时命中时，按以下顺序评估：

1. **路径级覆盖** — `rules.<module>.paths["<path>"]`
2. **模块级默认** — `rules.<module>.default_mode`
3. **全局默认** — `default_mode`

### 行为矩阵

| 规则结果 | 后端可用？ | `enable_overlay_fallback` | 最终行为 |
| --- | --- | --- | --- |
| `overlay` | 是 | 任意 | 使用 OverlayFS 挂载。 |
| `overlay` | 否 | `false` | 跳过并标记失败。 |
| `overlay` | 否 | `true` | 回退为 Magic Mount 重试。 |
| `magic` | 不适用 | 任意 | 使用 Magic Mount 挂载。 |
| `kasumi` | 是 | 任意 | 走 Kasumi 路由。 |
| `kasumi` | 否 | 任意 | 跳过 Kasumi 映射。 |
| `ignore` | 不适用 | 任意 | 不挂载。 |

### 实用场景

- **模块大部分路径走 overlay，仅单个文件走 magic**：模块默认设为 `overlay`，对冲突路径配置 `magic`。
- **临时排除某个冲突文件**：将该路径设为 `ignore`。
- **内核 OverlayFS 不稳定**：配置 `enable_overlay_fallback = true`。

---

## CLI 命令

```bash
hybrid-mount [OPTIONS] [COMMAND]
```

### 全局参数

| 参数 | 说明 |
|------|------|
| `-c, --config <PATH>` | 指定配置文件路径。 |
| `-m, --moduledir <PATH>` | 覆盖模块目录。 |
| `-s, --mountsource <SOURCE>` | 覆盖来源标识。 |
| `-p, --partitions <CSV>` | 覆盖分区列表。 |

### 子命令

| 命令 | 说明 |
|------|------|
| `gen-config` | 生成默认配置文件。 |
| `show-config` | 以 JSON 格式输出生效配置。 |
| `save-config --payload <HEX_JSON>` | 从 WebUI 负载保存配置。 |
| `save-module-rules --module <ID> --payload <HEX_JSON>` | 更新单模块规则。 |
| `modules` | 列出已检测模块。 |
| `daemon status` | 查询守护进程运行时状态。 |
| `daemon stop` | 停止守护进程。 |
| `kasumi ...` | Kasumi 管理（参见 [Kasumi](#kasumi)）。 |
| `lkm load / unload / status` | LKM 生命周期管理。 |
| `hide list / add / remove / apply` | 用户 hide 规则管理。 |

---

## 架构说明

```
┌─────────────────────────────────────────────┐
│                  config.toml                  │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│               模块清单扫描                    │
│         扫描模块目录，分类条目                  │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│               挂载规划器                      │
│    评估规则 (路径 > 模块 > 全局)               │
│    生成 overlay / magic / kasumi 计划         │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│               执行器                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │ OverlayFS│ │  Magic   │ │   Kasumi     │ │
│  │ 执行器   │ │  Mount   │ │   执行器     │ │
│  └──────────┘ └──────────┘ └──────────────┘ │
└──────────────────┬──────────────────────────┘
                   ▼
┌─────────────────────────────────────────────┐
│            运行时状态 + 守护进程               │
│    持久化状态 → Unix socket → WebUI/CLI       │
└─────────────────────────────────────────────┘
```

### 源码结构

```
src/
├── conf/          配置模型、TOML 加载器、CLI 定义
├── domain/        核心类型：MountMode、ModuleRules、路径匹配
├── core/
│   ├── inventory/ 模块发现与列表
│   ├── ops/       挂载计划生成、各后端执行器
│   ├── daemon/    Unix socket 服务器、HTTP/SSE、协议
│   ├── api/       WebUI 端点负载构建
│   └── startup/   启动流程、恢复、重试逻辑
├── mount/
│   ├── overlayfs/ OverlayFS 后端（ext4 镜像 / tmpfs）
│   ├── magic_mount/ Bind mount 后端
│   └── kasumi/    Kasumi 规则编译、运行时、状态
├── sys/           底层：挂载 syscall、LKM 加载/卸载、Kasumi UAPI
└── utils/         日志、路径工具、校验

webui/
├── src/
│   ├── routes/    页面组件（状态、配置、模块、Kasumi、关于）
│   ├── components/ 共享 UI 组件（导航栏、提示、骨架屏）
│   └── lib/       API 桥接、状态管理、编解码器、国际化
└── locales/       9 种语言国际化

xtask/             构建与发布自动化
module/            模块打包脚本与静态资源
```

---

## 构建方式

### 环境要求

- Rust nightly（参见 `rust-toolchain.toml`）
- Android NDK r27+ 和 `cargo-ndk`
- Node.js 20+ 和 pnpm（用于 WebUI）

### 命令

```bash
# 完整构建（二进制 + WebUI）→ output/
cargo run -p xtask -- build --release

# 仅构建二进制（跳过 WebUI）
cargo run -p xtask -- build --release --skip-webui

# 本地 arm64 调试构建
./scripts/build-local.sh

# 打入预编译的 Kasumi LKM .ko 资产
./scripts/build-local.sh --release --kasumi-lkm-dir /path/to/kasumi-lkm

# WebUI 开发服务器（热重载）
cd webui && pnpm install && pnpm dev

# 代码检查
cargo run -p xtask -- lint
cd webui && pnpm lint

# 运行测试
cargo +nightly test
cd webui && pnpm test
```

### Release 编译配置

Release 使用 `opt-level = 3`、`lto = "fat"`、`codegen-units = 1`、`strip = true`、`panic = "abort"` 以获得最小二进制体积。

---

## 运维建议

- **挂载来源自动检测**：新安装会默认自动检测运行环境。仅在自动检测失败时才需显式设置 `mountsource`。
- **配置错误恢复**：执行 `hybrid-mount gen-config` 重置为默认配置，然后逐步恢复规则。
- **Kasumi LKM**：LKM 必须与当前内核匹配。如果自动检测的 KMI 不正确，请使用 `lkm_kmi_override` 覆盖。
- **`kasumi kstat clear-config`**：仅清除持久化配置。已下发到内核的 kstat 规则在 LKM 重载或运行时重建前仍然有效。
- **减小体积**：建议优先从依赖特性裁剪和 release profile 调优入手，再考虑重构。

---

## 开源协议

基于 [Apache-2.0](LICENSE) 许可。
