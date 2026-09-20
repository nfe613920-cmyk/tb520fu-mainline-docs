# 📖 Lenovo YOGA Pad Pro 12.7 AI (TB520FU / SM8650) Mainline Linux & Arch ARM Complete Bring-up Documentation

本仓库收录联想小新 Pad Pro 12.7 AI 舒视版 / 超轻薄版 (代号 **TB520FU / Lapis**, 高通骁龙 8 Gen 3 / SM8650) 移植 Mainline Linux v7.2.6 及原生 Arch Linux ARM 的全流程完整技术文档与二次复刻指南。

---

## 🔗 项目代码仓库索引 (Ecosystem Repositories)

| 仓库名称 | 作用说明 | 链接 |
| :--- | :--- | :--- |
| **`mu_aloha_platforms`** (Branch: `8650-test`) | EDK2 UEFI 固件源码 (生成 `uefi.img`) | [mu_aloha_platforms](https://github.com/nfe613920-cmyk/mu_aloha_platforms/tree/8650-test) |
| **`tb520fu-device-tree`** | 设备树源码 DTS 及 NT36532 Dual-DSI 显存映射 | [tb520fu-device-tree](https://github.com/nfe613920-cmyk/tb520fu-device-tree) |
| **`tb520fu-linux-kernel`** | Mainline Linux 7.2.6 内核 defconfig、补丁与构建脚本 | [tb520fu-linux-kernel](https://github.com/nfe613920-cmyk/tb520fu-linux-kernel) |
| **`tb520fu-boot-kit`** | 独立 GRUB 2.14 EFI 二进制、诊断 initramfs 与 Arch 部署套件 | [tb520fu-boot-kit](https://github.com/nfe613920-cmyk/tb520fu-boot-kit) |
| **`tb520fu-mainline-docs`** | 本仓库：全流程实操排错与复刻指南 Markdown | [tb520fu-mainline-docs](https://github.com/nfe613920-cmyk/tb520fu-mainline-docs) |

---

## 📚 文档目录

1. [`01_uefi_to_grub_bringup_guide.md`](./01_uefi_to_grub_bringup_guide.md): 阶段 1 - UEFI 编译、启动及 GRUB 引导器加载指南
2. [`02_mainline_kernel_and_arch_bringup_guide.md`](./02_mainline_kernel_and_arch_bringup_guide.md): 阶段 2 & 3 - 主线内核编译、双 DSI 物理屏亮屏排错、以及 Arch Linux ARM 裸机部署全记录 (103 步完整日志)
3. [`03_reproduction_and_migration_guide.md`](./03_reproduction_and_migration_guide.md): 二次复刻与异机迁移全流程指南
