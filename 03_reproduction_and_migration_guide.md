# 联想 YOGA Pad Pro 12.7 AI（TB520FU / SM8650）主线 Linux 与 Arch Linux ARM 异机二次复刻指南

> 设备型号：Lenovo YOGA Pad Pro 12.7 AI（TB520FU / Lapis / 高通骁龙 8 Gen 3 / SM8650）  
> 适用场景：在另一台全新电脑（Linux 主机）上，重新构建或快速部署本项目的完整引导链与 Arch Linux ARM 系统。

---

## 0. 安全红线（必须时刻牢记）

1. **绝对严禁执行任何 `fastboot flash` 命令！**
2. 每次引导平板只允许使用临时引导命令：
   ```bash
   fastboot boot out/tb520fu-edk2-uefi-boot.img
   ```
3. 任何时候写入 ESP 或 `userdata`，必须先在 TWRP 下核对设备节点：
   - 槽位 a ESP：`/dev/block/sde19`（容量 511 MiB）
   - 根文件系统：`/dev/block/sda16`（容量 206.2 GiB）

---

## 方案 A：最快复刻（使用现有产物包，耗时 ~3 分钟）

### 1. 部署 ESP 分区 (sde19)
```bash
adb shell "mkdir -p /mnt/esp && mount -t vfat /dev/block/sde19 /mnt/esp"
adb push staging/tb520fu-esp/EFI/BOOT/BOOTAA64.EFI /mnt/esp/EFI/BOOT/BOOTAA64.EFI
adb push staging/tb520fu-esp/EFI/BOOT/grub.cfg /mnt/esp/EFI/BOOT/grub.cfg
adb push staging/tb520fu-esp/EFI/BOOT/Image /mnt/esp/EFI/BOOT/Image
adb push staging/tb520fu-esp/EFI/BOOT/sm8650-lenovo-tb520fu.dtb /mnt/esp/EFI/BOOT/sm8650-lenovo-tb520fu.dtb
adb shell "sync && umount /mnt/esp"
```

### 2. 部署 Arch Linux ARM Rootfs (sda16)
```bash
adb shell "mke2fs -t ext4 -F -L archroot /dev/block/sda16"
adb shell "mkdir -p /mnt/userdata && mount -t ext4 /dev/block/sda16 /mnt/userdata"
adb push sources/ArchLinuxARM-aarch64-latest.tar.gz /mnt/userdata/rootfs.tar.gz
adb shell "tar -xz --numeric-owner -f /mnt/userdata/rootfs.tar.gz -C /mnt/userdata && rm -f /mnt/userdata/rootfs.tar.gz"

adb shell "
printf '/dev/sda16 / ext4 rw,noatime,errors=remount-ro 0 1\n' > /mnt/userdata/etc/fstab && \
printf 'tb520fu-arch\n' > /mnt/userdata/etc/hostname && \
rm -f /mnt/userdata/etc/resolv.conf && \
printf 'nameserver 1.1.1.1\nnameserver 8.8.8.8\n' > /mnt/userdata/etc/resolv.conf && \
mkdir -p /mnt/userdata/etc/systemd/system/getty.target.wants && \
ln -sf /usr/lib/systemd/system/serial-getty@.service /mnt/userdata/etc/systemd/system/getty.target.wants/serial-getty@ttyGS0.service && \
ln -sf /usr/lib/systemd/system/getty@.service /mnt/userdata/etc/systemd/system/getty.target.wants/getty@tty0.service && \
sync && umount /mnt/userdata
"

adb reboot bootloader
fastboot boot out/tb520fu-edk2-uefi-boot.img
```
