<h3 align="center">
  <a href=#><img src="https://raw.githubusercontent.com/armbian/.github/master/profile/logosmall.png" alt="Armbian logo"></a>
  <br><br>
</h3>

# GL-MT2500 v1 Armbian

本分支为 GL.iNet GL-MT2500 / Brume 2 v1 提供 Armbian 支持。当前启动策略是：

1. U-Boot 优先从 USB 第 5 分区的 `/boot/extlinux/extlinux.conf` 启动 Armbian。
2. U 盘不存在、损坏或 extlinux 加载失败时，自动启动 eMMC 中原有的 OpenWrt。
3. Armbian 只写入 U 盘；eMMC 的 OpenWrt、BL2、环境和射频数据保持不变。
4. 正常安装可通过原厂 OpenWrt 的 SSH 只更新 FIP，无需拆机或连接串口。

当前 v1 参考产物：

```text
image: output/images/Armbian-unofficial_26.11.0-trunk_Gl-mt2500_trixie_edge_6.16.12.img
size: 2189426688 bytes
SHA256: 76f21287ef086e750ef1079ab830afef86a3ee10785690bc589ecbf860023f4f
root UUID: 1aa540fd-c972-49c7-879a-df4997566423
kernel: 6.16.12-edge-filogic
U-Boot: 2025.04
FIP size: 673313 bytes
FIP SHA256: 15bb94395ce8354d20a57bdd3338dcd052ce6da2bbb953591734a8e568033a04
```

重新构建会改变 UUID、initramfs 和哈希，实际刷写时始终以当次生成的 `.sha` 文件和 `sha256sum` 输出为准。

## 刷机前准备

需要以下设备：

- GL-MT2500 v1。v2 的 2.5G PHY 不同，不要混用镜像。
- 独立、稳定的 5V/3A USB-C 电源。
- 一个允许完全覆盖的 8 GB 或更大 U 盘。
- Linux 编译机、一根连接编译机与 MT2500 LAN 口的网线，并能 SSH 登录原厂 OpenWrt。
- 可选：用于排障和恢复的 3.3V USB TTL 串口，参数为 `115200 8N1`。
- 至少 25 GB 可用磁盘空间。

不要用电脑 USB 口给 MT2500 供电。实机已确认供电不足会导致 BL2 或 Linux 阶段冷复位。需要用 TTL 恢复时，只连接 `GND`、设备 `TX` 和设备 `RX`，TX/RX 交叉，不要连接 TTL 的 VCC。

刷 U-Boot 时只允许覆盖 eMMC 的 `fip` 分区。不要写 BL2，不要擦除 `u-boot-env` 或 `rf`，不要执行 `env erase`、`env default -a; saveenv` 或 `saveenv`。

## 1. 准备 Linux 主机

Debian/Ubuntu 安装依赖：

```bash
sudo apt update
sudo apt install -y \
  ca-certificates curl git xz-utils zstd \
  util-linux parted e2fsprogs dosfstools \
  openssh-client
```

Debian 可切换到清华源。先确认发行版代号，再写入 deb822 源：

```bash
. /etc/os-release
DEBIAN_CODENAME="$VERSION_CODENAME"
sudo tee /etc/apt/sources.list.d/debian.sources >/dev/null <<EOF
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/debian
Suites: ${DEBIAN_CODENAME} ${DEBIAN_CODENAME}-updates ${DEBIAN_CODENAME}-backports
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/debian-security
Suites: ${DEBIAN_CODENAME}-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
EOF
sudo apt update
```

## 2. 获取源码并编译 v1

```bash
git clone https://github.com/JiaY-shi/armbian.git
cd armbian
```

已有仓库时：

```bash
git switch gl-mt2500
git pull --ff-only origin gl-mt2500
```

构建 Debian Trixie CLI 镜像：

```bash
./compile.sh build \
  BOARD=gl-mt2500 \
  BRANCH=edge \
  RELEASE=trixie \
  BUILD_DESKTOP=no \
  BUILD_MINIMAL=no \
  KERNEL_CONFIGURE=no \
  KERNEL_GIT=shallow \
  KERNELSOURCE=https://mirrors.tuna.tsinghua.edu.cn/git/linux-stable.git \
  KERNELBRANCH=branch:linux-6.16.y \
  DOWNLOAD_MIRROR=china \
  PESTER_TERMINAL=no
```

首次构建必须联网。缓存完整后才可以增加 `OFFLINE_WORK=yes`。构建成功后检查：

```bash
find output/images -maxdepth 1 -type f -name '*Gl-mt2500*6.16.12*.img*' -ls
find output/debs -maxdepth 1 -type f -name 'linux-u-boot-gl-mt2500-edge*.deb' -ls
cd output/images
sha256sum -c Armbian-unofficial_26.11.0-trunk_Gl-mt2500_trixie_edge_6.16.12.img.sha
cd ../..
```

检查镜像 GPT：

```bash
sfdisk --dump output/images/Armbian-unofficial_26.11.0-trunk_Gl-mt2500_trixie_edge_6.16.12.img
```

必须有 `bl2`、`ubootenv`、`factory`、`fip` 和根分区共 5 个分区。第 4 分区 `fip` 必须是 Linux filesystem 类型，不能是 EFI System；根分区必须从扇区 `32768` 开始。

## 3. 首次写入 U 盘

插入 U 盘后同时核对总线、型号、序列号和容量：

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,TRAN,TYPE
lsblk -o NAME,SIZE,FSTYPE,LABEL,PARTLABEL,MOUNTPOINTS
```

下面的 `/dev/sdX` 必须替换为已确认的整块 U 盘，不能使用分区名，也不能凭盘符猜测：

```bash
IMG=output/images/Armbian-unofficial_26.11.0-trunk_Gl-mt2500_trixie_edge_6.16.12.img
sudo umount /dev/sdX1 /dev/sdX2 /dev/sdX3 /dev/sdX4 /dev/sdX5 2>/dev/null || true
sudo dd if="$IMG" of=/dev/sdX bs=4M status=progress conv=fsync
sync
sudo fdisk -l /dev/sdX
```

做完整回读校验：

```bash
IMG_BYTES="$(stat -c %s "$IMG")"
sudo head -c "$IMG_BYTES" /dev/sdX | sha256sum
sha256sum "$IMG"
sudo e2fsck -fn /dev/sdX5
```

两个 SHA256 必须一致，`e2fsck -fn` 必须没有文件系统错误。写入大容量 U 盘后，备份 GPT 暂时位于镜像尾部属于正常现象，首次启动的 Armbian 扩容服务会处理剩余空间。

## 4. 备份 eMMC 唯一数据

升级 FIP 前先启动设备原有 OpenWrt。下面假设 OpenWrt 地址为 `192.168.8.1`：

```bash
mkdir -p backups/gl-mt2500
ssh root@192.168.8.1 'dd if=/dev/mmcblk0boot0 bs=512 count=2048 2>/dev/null' \
  > backups/gl-mt2500/mmcblk0boot0-bl2.bin
ssh root@192.168.8.1 'dd if=/dev/mmcblk0p2 bs=512 2>/dev/null' \
  > backups/gl-mt2500/u-boot-env.bin
ssh root@192.168.8.1 'dd if=/dev/mmcblk0p3 bs=512 2>/dev/null' \
  > backups/gl-mt2500/rf.bin
ssh root@192.168.8.1 'dd if=/dev/mmcblk0p4 bs=512 2>/dev/null' \
  > backups/gl-mt2500/fip.bin
sha256sum backups/gl-mt2500/*
```

确认文件非空，并复制到另一块物理磁盘。`rf` 和 `u-boot-env` 含有本机 MAC、序列号等唯一数据，不能用其他设备的备份替代。

## 5. 提取 U-Boot FIP

```bash
UBOOT_DEB="$(find output/debs -maxdepth 1 -type f \
  -name 'linux-u-boot-gl-mt2500-edge*.deb' \
  -printf '%T@ %p\n' | sort -nr | head -n 1 | cut -d' ' -f2-)"
mkdir -p output/deploy/uboot
dpkg-deb -x "$UBOOT_DEB" output/deploy/uboot
FIP=output/deploy/uboot/usr/lib/linux-u-boot-edge-gl-mt2500/u-boot_sdmmc.fip
stat -c '%n %s bytes' "$FIP"
sha256sum "$FIP"
```

FIP 必须明显小于 eMMC `fip` 分区的 2 MiB 容量。当前参考值为 673313 字节，SHA256 为 `15bb94395ce8354d20a57bdd3338dcd052ce6da2bbb953591734a8e568033a04`。

## 6. 通过 OpenWrt SSH 只刷 eMMC FIP

正常安装不需要拆机或连接 TTL。保持设备运行原厂 OpenWrt，将刚刚提取的 FIP 上传到内存文件系统：

```bash
scp "$FIP" root@192.168.8.1:/tmp/u-boot-mt2500.fip
ssh root@192.168.8.1
```

在 OpenWrt 中核对目标分区和上传文件：

```sh
cat /sys/class/block/mmcblk0p4/size
wc -c /tmp/u-boot-mt2500.fip
sha256sum /tmp/u-boot-mt2500.fip
```

GL-MT2500 v1 的 `/sys/class/block/mmcblk0p4/size` 必须显示 `4096` 个 512 字节扇区，即 2 MiB。上传文件的大小和 SHA256 必须与编译机上的 `FIP` 完全一致。分区不存在、分区大小不同、FIP 大于分区或哈希不一致时都必须停止。

确认使用独立、稳定的 5V/3A 电源。下面的命令只覆盖 eMMC 的 `fip` 分区，写完后立即落盘并回读相同字节数：

```sh
FIP=/tmp/u-boot-mt2500.fip
FIP_BYTES="$(wc -c < "$FIP")"
dd if="$FIP" of=/dev/mmcblk0p4 bs=512
sync
dd if=/dev/mmcblk0p4 bs=1 count="$FIP_BYTES" 2>/dev/null | sha256sum
sha256sum "$FIP"
```

回读值和文件值两次 SHA256 必须一致。不要写 BL2，不要擦除 `u-boot-env` 或 `rf`，不要执行 `env erase`、`env default -a; saveenv` 或 `saveenv`。

校验成功后关闭设备，插入已经写好 Armbian 镜像的 U 盘，再重新上电。U-Boot 会优先启动 U 盘，失败时自动回退到 eMMC OpenWrt。

## 7. 可选的 TTL 串口与 TFTP 恢复

正常安装不需要本节。只有在排障、手动选择启动项或 OpenWrt 已无法启动时，才需要打开设备连接 3.3V TTL。

原生 Linux 通常直接生成 `/dev/ttyUSB0`：

```bash
sudo modprobe usbserial
sudo modprobe cp210x
sudo picocom -b 115200 /dev/ttyUSB0
```

在 WSL2 中使用 Windows COM 口时，先关闭 PuTTY 和其他串口程序，再用管理员 PowerShell 执行 `usbipd list`、`usbipd bind --busid <BUSID>` 和 `usbipd attach --wsl --busid <BUSID>`。CP210x 常见 VID:PID 为 `10c4:ea60`。

恢复时可以在 Linux 主机准备 TFTP：

```bash
sudo apt install -y dnsmasq-base picocom usbutils
TFTP_ROOT="$(pwd)/output/deploy/tftp"
mkdir -p "$TFTP_ROOT"
cp backups/gl-mt2500/fip.bin "$TFTP_ROOT/fip.bin"
LAN_IF=enp3s0
sudo ip addr add 192.168.1.10/24 dev "$LAN_IF"
sudo ip link set "$LAN_IF" up
sudo dnsmasq --no-daemon --port=0 --enable-tftp \
  --tftp-root="$TFTP_ROOT" --listen-address=192.168.1.10 --bind-interfaces
```

在 U-Boot 倒计时期间按空格，必须先确认 v1 的 `part_addr=3400` 和 `part_size=1000`：

```text
mmc dev 0
part start mmc 0 fip part_addr
part size mmc 0 fip part_size
printenv part_addr part_size
setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.10
tftpboot 0x50000000 fip.bin
hash sha256 0x50000000 ${filesize}
setexpr cnt ${filesize} + 0x1ff
setexpr cnt ${cnt} / 0x200
test 0x${cnt} -le 0x${part_size}
mmc write 0x50000000 ${part_addr} ${cnt}
mmc read 0x51000000 ${part_addr} ${cnt}
hash sha256 0x51000000 ${filesize}
```

只有分区边界正确，并且主机文件、RAM 与 eMMC 回读的 SHA256 全部一致时才能重启。恢复时只能使用该设备自己的 `fip.bin`，绝不能使用其他设备的 `rf` 或 `u-boot-env`。

## 8. 验证 USB 优先启动

保持 U 盘和 LAN 连接，使用独立电源重新上电。如需观察完整启动日志，再连接可选的 TTL 串口。正常日志应依次出现：

```text
Model: GL.iNet GL-MT2500
U-Boot 2025.04
USB XHCI
1 Storage Device(s) found
Retrieving file: /boot/extlinux/extlinux.conf
Loading: /boot/Image
Loading: /boot/uInitrd
Loading: /boot/dtb/mediatek/mt7981b-glinet-gl-mt2500-v1.dtb
```

自动启动命令为：

```text
usb start; sysboot usb 0:5 any ${scriptaddr} /boot/extlinux/extlinux.conf; run boot_openwrt
```

专用 U-Boot 只在 RAM 中把原 OpenWrt 的已知旧 `bootcmd=run boot_system` 迁移为上述命令，不写回环境，不覆盖设备唯一数据。若停在 `GL-MT2500>`，执行 `run bootcmd` 可重新运行完整启动链。

首次进入 Armbian 会扩展 U 盘第 5 分区，并要求设置 root 密码和普通用户。未连接串口时，可从上级路由器的 DHCP 租约中找到 MT2500 地址，再通过 SSH 完成首次登录。登录后验证：

```bash
cat /proc/device-tree/model
uname -a
findmnt /
lsblk -o NAME,SIZE,MODEL,TRAN,FSTYPE,MOUNTPOINTS
ip -br link
ip -br addr
ip route
systemctl --failed
grep -E '^(CONFIG_PREEMPT_NONE|CONFIG_HZ=|CONFIG_USB=|CONFIG_USB_XHCI_HCD=|CONFIG_USB_XHCI_MTK=|CONFIG_USB_STORAGE=|CONFIG_MMC=)' \
  /boot/config-$(uname -r)
```

预期型号为 `GL.iNet GL-MT2500`，内核为 `6.16.12-edge-filogic`，根文件系统来自 USB `/dev/sda5`，有线网口获得 DHCP 地址。内核使用 `CONFIG_PREEMPT_NONE=y` 和 `CONFIG_HZ=100`，不是实时内核。

## 9. 验证 OpenWrt 回退

串口进入 U-Boot 后执行：

```text
printenv boot_openwrt
run boot_openwrt
```

设备应从 eMMC 的 `kernel` GPT 分区加载 OpenWrt FIT，并挂载 eMMC `rootfs`。正常自动启动时，仅当 USB/extlinux 失败才执行相同回退命令，因此拔掉或损坏 U 盘后仍应进入 OpenWrt。

## 10. 后续不拔 U 盘更新镜像

不能在 Armbian 正从 `/dev/sda5` 运行时覆盖同一个 `/dev/sda`。以后保留 U 盘插在 MT2500 上，在 U-Boot 按空格并执行 `run boot_openwrt`，临时启动 eMMC OpenWrt。

在 OpenWrt 中先确认 `/` 和 `/overlay` 不在 U 盘上，并卸载可能自动挂载的分区：

```sh
cat /proc/cmdline
mount
cat /proc/partitions
block info
dmesg | grep -Ei 'usb|scsi|sd[a-z]'
umount /dev/sda5 2>/dev/null || true
```

确认 U 盘是 `/dev/sda` 后，在 Linux 构建机通过 LAN 流式写入，不需要把 2 GiB 镜像存入 OpenWrt RAM：

```bash
IMG=output/images/Armbian-unofficial_26.11.0-trunk_Gl-mt2500_trixie_edge_6.16.12.img
OPENWRT=root@<OpenWrt-IP>
IMG_BYTES="$(stat -c %s "$IMG")"
IMG_SECTORS="$(((IMG_BYTES + 511) / 512))"
sha256sum "$IMG"
dd if="$IMG" bs=4M status=progress | \
  ssh "$OPENWRT" 'dd of=/dev/sda bs=4M && sync'
```

完整回读校验：

```bash
ssh "$OPENWRT" "dd if=/dev/sda bs=512 count=$IMG_SECTORS 2>/dev/null" | sha256sum
sha256sum "$IMG"
```

两个 SHA256 一致后执行 `ssh "$OPENWRT" reboot`。U-Boot 会自动优先启动新写入的 Armbian。SSH 中断不会损坏 eMMC OpenWrt，可以重新写 U 盘。

## 11. 故障恢复

- 接 U 盘后反复冷复位：先更换独立 5V/3A 电源和 USB-C 线，不要添加实时内核、GPIO 常开或 regulator 容错补丁。
- U-Boot 看不到 U 盘：执行 `usb stop`、`usb start`、`usb tree` 和 `part list usb 0`，检查 U 盘与供电。
- 上电停在 `GL-MT2500>`：旧 U-Boot 会被任意串口字符打断；最终 U-Boot 只接受空格。当前可执行 `run bootcmd`。
- initramfs 找不到 root UUID：比较 `blkid /dev/sda5` 与 `/boot/extlinux/extlinux.conf` 中的 `root=UUID=`。
- 出现失败的 `efi.mount`：使用最终镜像重写 U 盘；最终第 4 分区已经不是 EFI System。
- USB 无法启动：等待自动回退，或在串口执行 `run boot_openwrt`。
- 恢复旧 FIP：把本机备份的 `fip.bin` 放入 TFTP 目录，按相同边界和回读流程只写 `fip`。绝不能使用其他设备的 `rf` 或 `u-boot-env`。

## Purpose of This Repository

The **Armbian Linux Build Framework** creates customizable OS images based on **Debian** or **Ubuntu** for **single-board computers (SBCs)** and embedded devices.

It builds a complete Linux system including kernel, bootloader, and root filesystem, giving you control over versions, configuration, firmware, device trees, and system optimizations.

The framework supports **native**, **cross**, and **containerized** builds for multiple architectures (`x86_64`, `aarch64`, `armhf`, `riscv64`) and is suitable for development, testing, production, or automation.

> **Looking for prebuilt images?** Use [Armbian Imager](https://github.com/armbian/imager/releases) — the easiest way to download and flash Armbian to your SD card or USB drive. Available for Linux, macOS, and Windows.

## Quick Start

```bash
git clone https://github.com/armbian/build
cd build
./compile.sh
```

<a href="#how-to-build-an-image-or-a-kernel"><img src=".github/README.gif" alt="Build demonstration" width="100%"></a>

## Build Host Requirements

### Hardware
- **RAM:** ≥8GB (less with `KERNEL_BTF=no`)
- **Disk:** ~50GB free space
- **Architecture:** x86_64, aarch64, or riscv64

### Operating System
- **Native builds:** Armbian/Debian 13 (Trixie)
- **Containerized:** Any Docker-capable Linux
- **Windows:** WSL2 with Armbian/Debian 13 (Trixie)

### Software
- Superuser privileges (`sudo` or root)
- Up-to-date system (outdated Docker or other tools can cause failures)

## Resources

- **[Documentation](https://docs.armbian.com/Developer-Guide_Overview/)** — Comprehensive guides for building, configuring, and customizing
- **[Website](https://www.armbian.com)** — News, features, and board information
- **[Blog](https://blog.armbian.com)** — Development updates and technical articles
- **[Forums](https://forum.armbian.com)** — Community support and discussions

## Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on reporting issues, submitting changes, and contributing code.

## Support

### Community Forums
Get help from users and contributors on troubleshooting, configuration, and development.
👉 [forum.armbian.com](https://forum.armbian.com)

### Real-time Chat
Join discussions with developers and community members on IRC or Discord.
👉 [Community Chat](https://docs.armbian.com/Community_IRC/)

### Paid Consultation
For commercial projects, guaranteed response times, or advanced needs, paid support is available from Armbian maintainers.
👉 [Contact us](https://www.armbian.com/contact)

## Contributors

Thank you to everyone who has contributed to Armbian!

<a href="https://github.com/armbian/build/graphs/contributors">
  <img alt="Contributors" src="https://contrib.rocks/image?repo=armbian/build" />
</a>

## Armbian Partners

Our [partnership program](https://forum.armbian.com/subscriptions) supports Armbian's development and community. Learn more about [our Partners](https://armbian.com/partners).
