# 联想 YOGA Pad Pro 12.7 AI (TB520FU / SM8650) EDK2 UEFI 编译与 GRUB 引导全流程指南

> **目标**：本文档提供从零开始编译适用于联想 YOGA Pad Pro 12.7 AI（骁龙 8 Gen 3 / 代号 `Lapis` / 型号 `TB520FU`）的 EDK2 UEFI 引导镜像的完整标准流程，并包含 **Linux 内核 12 色带显存诊断对照表**，使后续开发者或 Agent 能够基于当前的色带进展直接推进主线 Linux 内核与终端的完全引导。

---

## 1. 硬件规格与引导架构

- **目标机型**：联想 YOGA Pad Pro 12.7 AI (`TB520FU` / `Lapis`)
- **SoC 芯片**：高通第三代骁龙 8（Qualcomm Snapdragon 8 Gen 3 / `SM8650` / `Pineapple`）
- **CPU 核心**：8 核（1×Cortex-X4 + 5×Cortex-A720 + 2×Cortex-A520）
- **内存与存储**：12GB LPDDR5X + 256GB UFS 4.0
- **屏幕与显存**：12.7 英寸 2944x1840（32bpp Framebuffer 基址 `0xD5100000`）
- **代码仓库**：`https://github.com/nfe613920-cmyk/mu_aloha_platforms.git`（分支：`8650-test`）
- **唯一基准 UEFI 镜像**：`out/tb520fu-edk2-uefi-boot.img`
- **防砖安全红线**：**严禁向设备物理闪存执行 `fastboot flash boot`**，全程必须使用 `fastboot boot` 内存引导。

---

## 2. 编译环境与工具链准备

```bash
# Ubuntu / Debian:
sudo apt update && sudo apt install -y \
    build-essential uuid-dev iasl git gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
    clang llvm lld python3 python3-venv python3-pip ccache curl

# Arch Linux / CachyOS:
sudo pacman -S --needed \
    base-devel acpica git aarch64-linux-gnu-gcc clang llvm lld \
    python python-pip ccache
```

---

## 3. 源码拉取与构建

```bash
git clone https://github.com/nfe613920-cmyk/mu_aloha_platforms.git -b 8650-test
cd mu_aloha_platforms
git submodule update --init --recursive
./build_tb520fu_uefi.sh
```

---

## 4. Linux 内核 12 色带显存白盒诊断对照表

| 序号 | 显存物理地址 | 颜色 | 对应内核代码位置 | 代表已完成的启动阶段 | 状态 |
| :-: | :--- | :---: | :--- | :--- | :---: |
| **#1** | `0xD5900000` | 🟪 品红 | `arch/arm64/kernel/head.S` | `init_kernel_el` 执行完毕，接管 CPU | ✅ 通过 |
| **#2** | `0xD5A00000` | 🟦 青色 | `arch/arm64/kernel/head.S` | `__cpu_setup` 返回，准备开 MMU | ✅ 通过 |
| **#3** | `0xD5B00000` | 🟨 黄色 | `arch/arm64/kernel/setup.c` | 进入 `setup_arch()`，MMU 就绪 | ✅ 通过 |
| **#4** | `0xD5C00000` | 🟩 绿色 | `arch/arm64/kernel/setup.c` | `setup_machine_fdt()` 校验 DTB | ✅ 通过 |
| **#5** | `0xD5D00000` | ⬜ 白色 | `arch/arm64/kernel/setup.c` | `setup_arch()` 顺利结束 | ✅ 通过 |
| **#6** | `0xD5E00000` | 🟧 橙色 | `init/main.c: start_kernel` | 内核通用子系统初始化完成 | ✅ 通过 |
| **#7** | `0xD5F00000` | 🟥 红色 | `init/main.c: smp_init` 前 | 准备唤醒 7 个从 CPU | ✅ 通过 |
| **#8** | `0xD6000000` | 🟦 蓝色 | `init/main.c: smp_init` 后 | 8 个 CPU 核心全部上线 | ✅ 通过 |
| **#9** | `0xD6100000` | 🟪 紫色 | `init/main.c: do_initcalls` | 准备执行驱动模块队列 | ✅ 通过 |
| **#10** | `0xD6200000` | 🌸 粉色 | `arch_initcall` | 展开设备树节点 | ✅ 通过 |
| **#11** | `0xD6300000` | 🍏 Lime绿 | `drivers/video/fbdev/simplefb.c` | `simplefb_probe()` 驱动入口 | ✅ 通过 |
| **#12** | `0xD6400000` | 🟦 青色 | `drivers/video/fbdev/simplefb.c` | `register_framebuffer()` 成功（**8 只 Tux 企鹅**） | ✅ 通过 |
