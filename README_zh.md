# NanoPC-T4 Arch Linux 安装指南翻译版（SOM-RK3399V2同样有效）

### 指南
  本方案提供了一种十分方便的方法安装Arch Linux

### 安装 Armbian 到 SD 卡.
      下载镜像(选择Focal recommended).
https://www.armbian.com/nanopc-t4/

### 烧写镜像到SD 卡.

在Linux下通过dd命令

    dd if=Armbian_xx.xx.x_Nanopct4_focal_current_x.x.xx_desktop.img of=/dev/diskN bs=1M

这是一个说明如何烧写的视频
https://docs.armbian.com/User-Guide_Getting-Started/

### 从sd卡启动Armbian

一般直接怼上去就会引导

### 把 Arch linux安装到eMMC (假设eMMC 在 /dev/mmcblk2).
    清理在 eMMC 的系统
    # dd if=/dev/zero of=/dev/sdX bs=1M count=32

### emmc重新分区

```
# fdisk /dev/mmcblk2
Command (m for help): g
Created a new GPT disklabel (GUID: 2E750097-829F-614C-AD9E-271DA3413E3E).
Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-30535646, default 2048): 32768
Last sector, +/-sectors or +/-size{K,M,G,T,P} (32768-30535646, default 30535646):
Created a new partition 1 of type 'Linux filesystem' and of size 14.6 GiB.
Command (m for help): w
```

### 格式化分区为ext4类型

    # mkfs.ext4 /dev/mmcblk2p1

### 挂载分区

```
# mkdir root 
# mount /dev/mmcblk2p1 root
```

### 安装系统到emmc上 

```
# wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
# bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C root
```

### 编辑boot.txt

```
# After modifying, run ./mkscr

part uuid ${devtype} ${devnum}:${bootpart} uuid
setenv bootargs console=ttyS2,1500000 root=PARTUUID=${uuid} rw rootwait earlycon=uart8250,mmio32,0xff1a0000
setenv fdtfile rockchip/rk3399-nanopc-t4.dtb

if load ${devtype} ${devnum}:${bootpart} ${kernel_addr_r} /boot/Image; then
  if load ${devtype} ${devnum}:${bootpart} ${fdt_addr_r} /boot/dtbs/${fdtfile}; then
    fdt addr ${fdt_addr_r}
    fdt resize
    if load ${devtype} ${devnum}:${bootpart} ${ramdisk_addr_r} /boot/initramfs-linux.img; then
      booti ${kernel_addr_r} ${ramdisk_addr_r}:${filesize} ${fdt_addr_r};
    else
      booti ${kernel_addr_r} - ${fdt_addr_r};
    fi;
  fi;
fi
```

# 注意：把boot.txt放在Arch root filesystem/boot/boot.txt可以让你在接下来的操作更加便利OvO

### Optional: host root filesystem on NVMe storage.

如果以后要将根文件系统移动到NVMe存储，可以修改'boot.txt'`

并为根分区添加适当的UUID。迁移到NVMe的完整过程如下

超出本指南的范围。

获取NVMe的UUID

```
# blkid | grep nvme
/dev/nvme0n1p1: UUID="a56421bd-4e0d-48cc-b7a9-ee4fbc48aef8" TYPE="ext4" PARTUUID="8a5c1dfe-01"
```

把NVMe的UUI写入 `boot.txt`.

    rootdev=UUID=a56421bd-4e0d-48cc-b7a9-ee4fbc48aef8

### 把boot.txt转换成boot.scr 

```
archlinux上需安装uboot-tools

# mkimage -A arm -O linux -T script -C none -n "U-Boot boot script" -d boot.txt boot.scr
```

### 挂载本机设备到arch根目录系统，在archlinuxarm上安装可以使用arch-chroot可以跳过本步。

```
# mount --bind /proc root/proc
# mount --bind /sys root/sys
# mount --bind /dev root/dev
# mount --bind /dev/pts root/dev/pts
```

### chroot到安装好的系统中
```
# chroot root 或 arch-chroot root
# echo "nameserver 1.1.1.1" > /etc/resolv.conf
# pacman-key --init
# pacman-key --populate archlinuxarm
# pacman -Syu
# pacman -S dialog wpa_supplicant
# exit
```

### 取消挂载文件系统

    # umount root

### 安装nanopc-t4的uboot

```
# dpkg -L linux-u-boot-nanopct4-current
...
/usr/lib/linux-u-boot-current-nanopct4_20.11.6_arm64/idbloader.bin
/usr/lib/linux-u-boot-current-nanopct4_20.11.6_arm64/trust.bin
/usr/lib/linux-u-boot-current-nanopct4_20.11.6_arm64/uboot.img
...
```

提示：uboot镜像可以到[这里](https://github.com/ghhccghk/arch-nanopct4/tree/main/images)获取

### 刷写uboot到emmc
```
# cd /usr/lib/linux-u-boot-current-nanopct4_20.11.6_arm64
# dd if=idbloader.bin of=/dev/mmcblk2 seek=64 conv=notrunc
# dd if=uboot.img of=/dev/mmcblk2 seek=16384 conv=notrunc
# dd if=trust.bin of=/dev/mmcblk2 seek=24576 conv=notrunc
```

关闭电源，拔下sd卡，重新启动即可
