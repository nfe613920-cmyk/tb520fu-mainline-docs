# 联想 YOGA Pad Pro 12.7 AI（TB520FU / SM8650）从零编译 UEFI 到启动 Arch Linux ARM 全流程完整教程

> **设备型号**：Lenovo YOGA Pad Pro 12.7 AI 舒视版 / 超轻薄版（代号 `Lapis`，型号 `TB520FU`）  
> **芯片平台**：高通第三代骁龙 8（Qualcomm Snapdragon 8 Gen 3 / `SM8650` / `Pineapple`）  
> **操作系统**：Mainline Linux v7.2.6 + Arch Linux ARM aarch64 (裸机 ext4 根文件系统)  
> **适用场景**：一站式从零构建、编译、部署与引导指南，包含所有踩坑点与排错方案。

---

## 目录
1. [硬件规格与全局引导架构](#1-硬件规格与全局引导架构)
2. [安全红线与准备工作](#2-安全红线与准备工作)
3. [第一阶段：编译 EDK2 UEFI 固件](#3-第一阶段编译-edk2-uefi-固件)
4. [第二阶段：构建主线 Linux 内核与设备树](#4-第二阶段构建主线-linux-内核与设备树)
5. [第三阶段：构建独立 GRUB 2.14 ARM64 EFI](#5-第三阶段构建独立-grub-214-arm64-efi)
6. [第四阶段：制作支持 switch_root 的 Initramfs](#6-第四阶段制作支持-switch_root-的-initramfs)
7. [第五阶段：ESP 引导分区与 Arch Linux 裸机部署](#7-第五阶段esp-引导分区与-arch-linux-裸机部署)
8. [第六阶段：启动测试与登录验证](#8-第六阶段启动测试与登录验证)
9. [高频故障与核心避坑指南](#9-高频故障与核心避坑指南)

---

## 1. 硬件规格与全局引导架构

### 1.1 硬件参数规格
- **CPU**：8 核 Kryo 架构（1×3.3GHz Cortex-X4 + 5×Cortex-A720 + 2×Cortex-A520）
- **内存与存储**：12 GB LPDDR5X + 256 GB UFS 4.0
- **屏幕与显存**：12.7 英寸 Dual-DSI Video Mode 液晶屏（NT36532 控制器，分辨率 $2944 \times 1840$ @ 144Hz，32bpp Framebuffer 物理基址 `0x00000000D5100000`，容量 43MB）
- **按键**：物理音量加减键（支持在 GRUB 菜单中充当上下方向键）

### 1.2 引导链拓扑图
```text
[开机上电]
   │
   ▼
[高通 ABL 引导层] ──(执行 fastboot boot 临时引导)──► 内存地址 0xF3800000
   │
   ▼
[BootShim + EDK2 UEFI (mu_aloha_platforms)] 
   │  ├─ 驱动按时序注册: RamPartition -> Smem -> ShmBridge -> DalTlmm -> UFSDxe -> ButtonsDxe
   │  └─ 初始化 Framebuffer (2944x1840 简单帧缓冲区 GOP)
   │
   ▼
[GRUB 2.14 Standalone EFI (BOOTAA64.EFI)] ──(位于 ESP 分区 sde19 /EFI/BOOT/)
   │  └─ 通过 ($esp) 自动寻址加载内核 Image 与 sm8650-lenovo-tb520fu.dtb
   │
   ▼
[Mainline Linux v7.2.6 (Image + DTB)]
   │  ├─ SimpleFB 驱动接管屏幕 (显示 8 只 Tux 企鹅，368x115 终端字符矩阵)
   │  ├─ 关键内核参数维持 Dual-DSI 时钟供电 (clk_ignore_unused pd_ignore_unused)
   │  └─ 执行内嵌 initramfs (/init)
   │
   ▼
[Initramfs 诊断与挂载层]
   │  ├─ 创建静态设备节点 /dev/console (5:1) 与 /dev/null (1:3)
   │  ├─ 自动探测并挂载 userdata 分区 (/dev/sda16, ext4) 到 /newroot
   │  └─ 执行 exec switch_root /newroot /sbin/init
   │
   ▼
[Arch Linux ARM 原生系统 (Bare-Metal systemd)]
   │  ├─ 屏幕终端: getty@tty0 (可在物理屏幕上显示登录提示符)
   │  └─ 调试串口: serial-getty@ttyGS0 (通过 USB 线免驱免串口线直接登录)
```

---

## 2. 安全红线与准备工作

### 2.1 ⚠️ 绝对安全红线
1. **绝对严禁执行任何 `fastboot flash` 刷写命令！**  
   任何固件测试只使用内存临时引导：`fastboot boot out/tb520fu-edk2-uefi-boot.img`。
2. **严禁在未确认分区编号时写入磁盘！**  
   在 TWRP 下必须严格核对：
   - 槽位 a ESP 分区：`/dev/block/sde19`（容量 511MB）
   - 用户数据根分区：`/dev/block/sda16`（容量 206.2GB）

### 2.2 宿主编译机依赖安装（以 Arch Linux / Ubuntu 为例）
```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y \
    build-essential uuid-dev iasl git gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
    clang llvm lld python3 python3-venv python3-pip ccache curl bc bison flex \
    libssl-dev dwarves cpio rsync perl android-tools-adb android-tools-fastboot

# Arch Linux / CachyOS / Manjaro
sudo pacman -S --needed \
    base-devel acpica git aarch64-linux-gnu-gcc aarch64-linux-gnu-binutils \
    clang llvm lld python python-pip ccache bc bison flex openssl pahole \
    cpio rsync perl android-tools
```

---

## 3. 第一阶段：编译 EDK2 UEFI 固件

### 3.1 拉取 UEFI 平台源码
```bash
mkdir -p ~/tb520fu-mainlinux && cd ~/tb520fu-mainlinux
git clone https://github.com/nfe613920-cmyk/mu_aloha_platforms.git -b 8650-test
cd mu_aloha_platforms
git submodule update --init --recursive
```

### 3.2 配置 Python 虚拟环境与 pytool
```bash
python3 -m venv ~/tb520fu-mainlinux/edk2-venv
source ~/tb520fu-mainlinux/edk2-venv/bin/activate
pip install --upgrade pip
pip install edk2-pytool-library edk2-pytool-extensions
```

### 3.3 编译 UEFI 镜像
运行项目内的一键构建脚本（包含子模块补丁自动检测与注入）：
```bash
cd ~/tb520fu-mainlinux/mu_aloha_platforms
./build_tb520fu_uefi.sh
```
编译完成后，产物位于 `~/tb520fu-mainlinux/out/tb520fu-edk2-uefi-boot.img`（约 2.1 MB）。

---

## 4. 第二阶段：构建主线 Linux 内核与设备树

### 4.1 获取 Linux v7.2.6 源码
```bash
cd ~/tb520fu-mainlinux
git clone --depth=1 --branch v7.2.6 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git linux-v7.2.6
```

### 4.2 注入 TB520FU 设备树 (DTS)
在 `linux-v7.2.6/arch/arm64/boot/dts/qcom/sm8650-lenovo-tb520fu.dts` 中配置 SimpleFB 显存节点：
```dts
/dts-v1/;
#include "sm8650-qrd.dts"

/ {
	model = "Lenovo YOGA Pad Pro 12.7 AI (TB520FU)";
	compatible = "lenovo,tb520fu", "qcom,sm8650";

	chosen {
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;

		framebuffer: framebuffer@d5100000 {
			compatible = "simple-framebuffer";
			reg = <0x0 0xd5100000 0x0 0x02b00000>;
			width = <2944>;
			height = <1840>;
			stride = <(2944 * 4)>;
			format = "x8r8g8b8";
			status = "okay";
		};
	};
};
```
并在 `arch/arm64/boot/dts/qcom/Makefile` 中追加构建目标：
```makefile
dtb-$(CONFIG_ARCH_QCOM) += sm8650-lenovo-tb520fu.dtb
```

### 4.3 编译内核 Image 与 DTB
```bash
mkdir -p ~/tb520fu-mainlinux/build/linux-v7.2.6-tb520fu
cd ~/tb520fu-mainlinux/linux-v7.2.6

# 采用纯净高通配置构建
make O=../build/linux-v7.2.6-tb520fu ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- CC="ccache aarch64-linux-gnu-gcc" defconfig

# 确保关键驱动内建: SimpleFB, UFS, USB Gadget, ext4
scripts/config --file ../build/linux-v7.2.6-tb520fu/.config \
    -e FB_SIMPLE -e PHY_QCOM_QMP -e SCSI_UFS_QCOM -e EXT4_FS \
    -e PHY_SNPS_EUSB2 -e PHY_QCOM_EUSB2_REPEATER -e USB_G_SERIAL

make O=../build/linux-v7.2.6-tb520fu ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- CC="ccache aarch64-linux-gnu-gcc" -j$(nproc) Image qcom/sm8650-lenovo-tb520fu.dtb
```

> [!IMPORTANT]
> 内核 Image 尺寸必须控制在 **44.7 MB** 以内，防止触发 ARM64 恒等映射（IDMAP）页表溢出死锁。

---

## 5. 第三阶段：构建独立 GRUB 2.14 ARM64 EFI

### 5.1 编译 GRUB 2.14 核心
```bash
cd ~/tb520fu-mainlinux
curl -L -O https://mirrors.kernel.org/gnu/grub/grub-2.14.tar.xz
tar -xJvf grub-2.14.tar.xz
mkdir -p build/grub-2.14-arm64-efi && cd build/grub-2.14-arm64-efi

../../grub-2.14/configure \
    --target=aarch64-linux-gnu \
    --with-platform=efi \
    BUILD_CC="ccache gcc" TARGET_CC="ccache aarch64-linux-gnu-gcc" \
    TARGET_OBJCOPY=aarch64-linux-gnu-objcopy

make -j$(nproc)
```

### 5.2 编写 GRUB 配置文件 (`grub.cfg`)
```grub
set default=0
set timeout=5
set timeout_style=menu
set gfxpayload=keep

search --no-floppy --file --set=esp /EFI/BOOT/Image

menuentry "Arch Linux ARM (TB520FU /dev/sda16)" {
    echo "Loading Mainline Kernel & Device Tree..."
    linux ($esp)/EFI/BOOT/Image root=/dev/sda16 rw earlycon console=tty0 clk_ignore_unused pd_ignore_unused regulator_ignore_unused arm-smmu.disable_bypass=0
    devicetree ($esp)/EFI/BOOT/sm8650-lenovo-tb520fu.dtb
}

menuentry "Diagnostic USB Serial Shell (rd.shell=1)" {
    linux ($esp)/EFI/BOOT/Image rd.shell=1 earlycon console=tty0 clk_ignore_unused pd_ignore_unused regulator_ignore_unused arm-smmu.disable_bypass=0
    devicetree ($esp)/EFI/BOOT/sm8650-lenovo-tb520fu.dtb
}
```

### 5.3 打包 Standalone EFI
```bash
./grub-mkstandalone \
    --core-compress=none \
    --format=arm64-efi \
    --output=../../staging/tb520fu-esp/EFI/BOOT/BOOTAA64.EFI \
    --directory=grub-core \
    --modules="part_gpt fat normal linux fdt search search_fs_file configfile all_video gfxterm reboot halt echo test" \
    /boot/grub/grub.cfg=../../staging/tb520fu-esp/EFI/BOOT/grub.cfg
```

---

## 6. 第四阶段：制作支持 switch_root 的 Initramfs

### 6.1 编译静态 Busybox 与创建设备节点表
```bash
mkdir -p ~/tb520fu-mainlinux/initramfs/root/{bin,sbin,usr/bin,usr/sbin,dev,proc,sys,run,tmp,newroot}

cat << 'EOF' > ~/tb520fu-mainlinux/initramfs/dev_nodes.list
nod /dev/console 0600 0 0 c 5 1
nod /dev/null 0666 0 0 c 1 3
EOF
```

### 6.2 编写 PID 1 初始化脚本 (`init`)
在 `~/tb520fu-mainlinux/initramfs/root/init` 中写入：
```sh
#!/bin/busybox sh
export PATH=/bin:/sbin:/usr/bin:/usr/sbin
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mkdir -p /dev/pts /run
mount -t devpts devpts /dev/pts
mount -t tmpfs tmpfs /run
exec </dev/console >/dev/console 2>&1

echo "=== TB520FU Booting Arch Linux ==="
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s

root_dev=/dev/sda16
wait_cnt=0
while [ ! -b "$root_dev" ] && [ "$wait_cnt" -lt 30 ]; do
    sleep 1
    wait_cnt=$((wait_cnt + 1))
    mdev -s
done

mount -t ext4 -o rw "$root_dev" /newroot
mount --move /proc /newroot/proc
mount --move /sys /newroot/sys
mount --move /dev /newroot/dev
mount --move /run /newroot/run
exec switch_root /newroot /sbin/init
```
赋权：`chmod 0755 ~/tb520fu-mainlinux/initramfs/root/init`。

---

## 7. 第五阶段：ESP 引导分区与 Arch Linux 裸机部署

设备进入 **TWRP Recovery** 模式，通过 USB 连接主机：

### 7.1 部署 ESP 引导分区 (`/dev/block/sde19`)
```bash
adb shell "mkdir -p /mnt/esp && mount -t vfat /dev/block/sde19 /mnt/esp"
adb push staging/tb520fu-esp/EFI/BOOT/BOOTAA64.EFI /mnt/esp/EFI/BOOT/
adb push staging/tb520fu-esp/EFI/BOOT/grub.cfg /mnt/esp/EFI/BOOT/
adb push build/linux-v7.2.6-tb520fu/arch/arm64/boot/Image /mnt/esp/EFI/BOOT/
adb push build/linux-v7.2.6-tb520fu/arch/arm64/boot/dts/qcom/sm8650-lenovo-tb520fu.dtb /mnt/esp/EFI/BOOT/
adb shell "sync && umount /mnt/esp"
```

### 7.2 部署 Arch Linux ARM Rootfs (`/dev/block/sda16`)
```bash
# 格式化 userdata 为 clean ext4
adb shell "mkfs.ext4 -F -L arch_rootfs /dev/block/sda16"
adb shell "mkdir -p /mnt/userdata && mount -t ext4 /dev/block/sda16 /mnt/userdata"

# 下载并解压官方 Arch Linux ARM 根文件系统
curl -L -O http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
adb push ArchLinuxARM-aarch64-latest.tar.gz /mnt/userdata/
adb shell "tar -xpf /mnt/userdata/ArchLinuxARM-aarch64-latest.tar.gz -C /mnt/userdata && rm /mnt/userdata/ArchLinuxARM-aarch64-latest.tar.gz"

# 固化系统配置 (fstab, hostname, getty)
adb shell "
echo 'tb520fu-arch' > /mnt/userdata/etc/hostname
echo 'nameserver 1.1.1.1' > /mnt/userdata/etc/resolv.conf
echo '/dev/sda16 / ext4 rw,relatime,errors=remount-ro 0 1' > /mnt/userdata/etc/fstab
mkdir -p /mnt/userdata/etc/systemd/system/getty.target.wants
ln -sf /usr/lib/systemd/system/serial-getty@.service /mnt/userdata/etc/systemd/system/getty.target.wants/serial-getty@ttyGS0.service
ln -sf /usr/lib/systemd/system/getty@.service /mnt/userdata/etc/systemd/system/getty.target.wants/getty@tty0.service
sync && umount /mnt/userdata
"
```

---

## 8. 第六阶段：启动测试与登录验证

### 8.1 临时引导设备
```bash
adb reboot bootloader
fastboot boot out/tb520fu-edk2-uefi-boot.img
```

### 8.2 观察屏幕与交互
1. 设备重启，显示 UEFI 初始化。
2. 屏幕进入 **GRUB 2.14 菜单**，可通过音量上下键自由切换项目。
3. 倒计时结束后自动启动第一项，屏幕顶部显示 **8 只 Tux 企鹅** 与 12 色带白盒诊断条。
4. 屏幕打印 Linux 内核启动日志，随后进入 Arch Linux 登录提示：
   ```text
   tb520fu-arch login:
   ```
5. **登录方式**：
   - 物理屏幕（USB/蓝牙键盘）：输入 `root`（密码 `root`）或 `alarm`（密码 `alarm`）。
   - 主机免驱 USB 串口：在电脑端运行 `minicom -D /dev/ttyACM0` 或 `screen /dev/ttyACM0 115200`，即可直接获得互动 root shell！

---

## 9. 高频故障与核心避坑指南

| 故障现象 | 根因剖析 | 解决方案 |
| :--- | :--- | :--- |
| **屏幕在内核启动瞬间全黑/冻结** | NT36532 Dual-DSI 为 Video 模式（无内置显存），内核关闭了未引用时钟 | 内核 cmdline 必须加入 `clk_ignore_unused pd_ignore_unused regulator_ignore_unused arm-smmu.disable_bypass=0` |
| **屏幕仅亮前两条色带（品红+青色）即死机** | ARM64 恒等页表越界，内核 Image 超过 44.7MB | 移除内核中冗余非高通驱动，精简 Image 尺寸 |
| **出现 unable to open an initial console** | Initramfs 中缺失基础字符设备节点 | 在 `dev_nodes.list` 中注入 `/dev/console` (5:1) 与 `/dev/null` (1:3) |
| **GRUB 报 archelp.c: file not found** | standalone memdisk 环境未带分区前缀 | 在 `grub.cfg` 中使用 `($esp)/EFI/BOOT/...` 明确指定根设备 |
