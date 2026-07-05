# Tenda BE12 Pro ImmortalWrt

这个仓库用于维护 Tenda BE12 Pro 的 ImmortalWrt 构建。迁移旧仓库改动时，以已经实机刷入并能稳定进系统的旧仓库提交作为基线：

- 旧仓库 boot-good 提交：`4433932a6c7d705f026f9878edb230b2aa426925`
- 内核版本标识：`Linux 6.6.94 ... r0+33509-4433932a6c`
- 设备识别：`Machine model: Tenda BE12 Pro`，`board_name: tenda,be12-pro`
- 当前仓库上游基底：`padavanonly/immortalwrt-mt798x-6.6` 的 `mt798x-mt799x-6.6-mtwifi`

## Boot-Good 证据

`4433932a6c7d705f026f9878edb230b2aa426925` 刷入后，系统日志和手工命令确认以下关键链路正常：

- 分区布局正常：`kernel` 为 `0x780000-0xd80000`，`ubi` 为 `0xd80000-0x6780000`，并且 `ubi0`、`rootfs_data` 能挂载。
- PCIe 正常枚举 MT7992：日志出现 `pci 0000:01:00.0: [14c3:7992]`，后续 `mt7992 0000:01:00.0` 固件加载完成。
- 以太网设备正常：`eth0`、`eth1`、`eth2` 都创建，`lan3/lan4/lan5` 作为 `eth0` 下的 DSA 口存在。
- HNAT 正常：`wan = eth2`、`lan = eth0`、`lan2 = eth1`、`ppd = eth0`、`gmac num = 3`、`ppe num = 2`，并且 `PPE0/PPE1 hwnat start`。
- 无线接口正常：`ra0`、`rai0` 创建并处于 `UP`，日志出现 `MAC Init Done`。

已知非致命日志不要误判成失败：

- 早期 PCIe `Failed to get clk index: 0 ret: -517`、`failed to get max link width`，后续仍能枚举 MT7992。
- 早期 AN8855 `Failed to register DSA switch: -517`，后续仍能注册 `lan3/lan4/lan5`。
- `eth0: mtu greater than device maximum` 和 `error -22 setting MTU to 1504 to include DSA overhead`，当前 boot-good 版本仍可正常进系统。
- WiFi `RT_CfgSetMacAddress() invalid length (0)`、DPD runtime calibration 日志，当前 boot-good 版本仍能启动 `ra0/rai0`。

## 迁移状态

旧仓库 `d54961259e4ada4d01e1186dbfd454c738104971..4433932a6c7d705f026f9878edb230b2aa426925` 里和 BE12 Pro 相关、可迁移的改动已经同步到本仓库：

- 构建配置：`full.config`、`minimal.config`、`mt7987_mt7992.config`、`mt7988_mt7990.config`。
- feeds/默认值：iStore feed、zram 默认 `128M`、turboacc HNAT 默认启用。
- PHY/DSA：EN8811H 使用 `air_en8811h.ko`，AN8855 使用 DSA 路径，移除旧 AN8855 GSW/vendor EN8811H 文件。
- MTK WiFi 用户态：`mwctl` wrapper、testmode fallback、AP isolation、WPA2/WPA3 mixed ApCli、默认 profile 参数。
- 内核/target：MTK HNAT PPE 路由、BA 处理修正、MT7987 DTSI、`include/image.mk`、BE12 Pro network/image rule。

保留的差异是故意的：

- BE12 Pro DTS 不回退到纯旧仓库版本；当前以旧仓库 boot-good DTS 为底盘，保留 OpenWrt 修正：`label-mac-device = &gmac0`、gmac/WiFi MAC offset、PCIe 子节点 address/size cells、`wifi@0,0`。
- 主 workflow 保持只构建 `mini` 和 `full`；passwall 通过独立 workflow 构建。
- Hiveton H5000M 不是这个仓库的维护目标，旧项目里 Hiveton DTS/image package 差异不迁移。
- `02_network` 的 BE12 Pro 内容等价，只保留本仓库的 tab 缩进。

## 无限重启 / 启动失败高风险项

下面这些改动可能直接导致无限重启、无法挂载 rootfs、或刷入后无法进入系统。修改前必须能说明为什么，修改后必须用串口或完整启动日志验证。

- 分区表和 NAND/NMBM 相关 DTS：`Bootloader`、`u-boot-env`、`Factory`、`kernel`、`ubi`、`CFG`、`MISC2` 的地址和大小不能随手改。`kernel`/`ubi` 错位会导致 kernel 找不到 rootfs 或覆盖数据。
- BE12 Pro DTS 路径和 image rule：当前使用 `DEVICE_DTS := mt7987a-tenda-be12-pro`、`DEVICE_DTS_DIR := ../dts`。路径错会让镜像带错 DTB 或构建出不可启动镜像。
- kernel/FIT/sysupgrade 封装：`KERNEL_LOADADDR := 0x40000000`、`KERNEL_INITRAMFS`、`IMAGE/sysupgrade.bin`、`tenda-mkdualimageheader` 都属于启动链。改错会导致 U-Boot 校验、解包或加载失败。
- HNAT DTS 映射：boot-good 值是 `mtketh-wan = "eth2"`、`mtketh-lan = "eth0"`、`mtketh-lan2 = "eth1"`、`mtketh-max-gmac = <3>`、`mtketh-ppe-num = <2>`、`ext-devices-prefix = "dummy"`。这些值和实际 `eth0/eth1/eth2` 顺序不一致时，可能触发网络初始化异常或 HNAT 相关重启。
- turboacc HNAT 默认值：`fastpath_mh_eth_hnat` 当前应默认 `1`。旧问题窗口里禁用 HNAT 默认值和重启问题高度相关，未重新实机验证前不要改成默认 `0`。
- EN8811H/AN8855 PHY/DSA 驱动：BE12 Pro 的 WAN 侧日志显示 `eth2` 使用 `Airoha EN8811H`，LAN 侧依赖 `AN8855` DSA。`air_en8811.ko`/`air_en8811h.ko`、`CONFIG_AIR_EN8811H_PHY`、AUTOLOAD 顺序或删除 vendor driver 文件都可能导致 WAN/LAN 初始化失败。
- MTK HNAT 内核 PPE 路由：`NR_GMAC2_PORT`/`NR_GMAC3_PORT` 一类改动属于高风险内核行为，必须放到基础镜像能开机之后再单独迁移。
- `&ssusb`、`&tphyu3port0`、PCIe `wifi@0,0`、MAC/nvmem offset：这类 DTS 节点一般先表现为 USB/WiFi/MAC 异常，但如果影响驱动 probe 顺序，也可能带来启动链不稳定。
- 删除 zram 或把默认值设得过小：BE12 Pro 内存约 512M，full 固件服务多时可能 OOM。`minimal` 和 `full` 都保留 `zram-swap`/`kmod-zram`，默认值先按 `128M` 验证。

## 刷机后验证

每次刷入后至少保留以下输出，用来和 boot-good 基线对比：

```sh
ip link
ubus call system board
logread | grep -iE 'mt7992|pcie|hnat|eth|nvmem|mac'
cat /sys/kernel/debug/hnat/hook_toggle 2>/dev/null
```

最低通过标准：

- `ubus call system board` 返回 `board_name: tenda,be12-pro`。
- `ip link` 能看到 `eth0`、`eth1`、`eth2`、`lan3`、`lan4`、`lan5`、`ra0`、`rai0`。
- `logread` 能看到 MT7992 枚举和固件启动、HNAT `PPE0/PPE1 hwnat start`、`eth2` 使用 `Airoha EN8811H`。

---

<img src="https://avatars.githubusercontent.com/u/53193414?s=200&v=4" alt="logo" width="200" height="200" align="right">

# Project ImmortalWrt

ImmortalWrt is a fork of [OpenWrt](https://openwrt.org), with more packages ported, more devices supported, default optimized profiles and localization modifications for mainland China users.<br/>
Compared to upstream, we allow to use (non-upstreamable) modifications/hacks to provide better feature/performance/support.

Default login address: http://192.168.6.1 or http://immortalwrt.lan, username: __root__, password: _none_.

## Download
Built firmware images are available for many architectures and come with a package selection to be used as WiFi home router. To quickly find a factory image usable to migrate from a vendor stock firmware to ImmortalWrt, try the *Firmware Selector*.

- [ImmortalWrt Firmware Selector](https://firmware-selector.immortalwrt.org/)

If your device is supported, please follow the **Info** link to see install instructions or consult the support resources listed below.

## Development
To build your own firmware you need a GNU/Linux, BSD or macOS system (case sensitive filesystem required). Cygwin is unsupported because of the lack of a case sensitive file system.<br/>

  ### Requirements
  To build with this project, Debian 11 is preferred. And you need use the CPU based on AMD64 architecture, with at least 4GB RAM and 25 GB available disk space. Make sure the __Internet__ is accessible.

  The following tools are needed to compile ImmortalWrt, the package names vary between distributions.

  - Here is an example for Debian/Ubuntu users:<br/>
    - Method 1:
      <details>
        <summary>Setup dependencies via APT</summary>

        ```bash
        sudo apt update -y
        sudo apt full-upgrade -y
        sudo apt install -y ack antlr3 asciidoc autoconf automake autopoint binutils bison build-essential \
          bzip2 ccache clang cmake cpio curl device-tree-compiler ecj fastjar flex gawk gettext gcc-multilib \
          g++-multilib git gnutls-dev gperf haveged help2man intltool lib32gcc-s1 libc6-dev-i386 libelf-dev \
          libglib2.0-dev libgmp3-dev libltdl-dev libmpc-dev libmpfr-dev libncurses-dev libpython3-dev \
          libreadline-dev libssl-dev libtool libyaml-dev libz-dev lld llvm lrzsz mkisofs msmtp nano \
          ninja-build p7zip p7zip-full patch pkgconf python3 python3-pip python3-ply python3-docutils \
          python3-pyelftools qemu-utils re2c rsync scons squashfs-tools subversion swig texinfo uglifyjs \
          upx-ucl unzip vim wget xmlto xxd zlib1g-dev zstd
        ```
      </details>
    - Method 2:
      ```bash
      sudo bash -c 'bash <(curl -s https://build-scripts.immortalwrt.org/init_build_environment.sh)'
      ```

  Note:
  - Do everything as an unprivileged user, not root, without sudo.
  - Using CPUs based on other architectures should be fine to compile ImmortalWrt, but more hacks are needed - No warranty at all.
  - You must __not__ have spaces or non-ascii characters in PATH or in the work folders on the drive.
  - If you're using Windows Subsystem for Linux (or WSL), removing Windows folders from PATH is required, please see [Build system setup WSL](https://openwrt.org/docs/guide-developer/build-system/wsl) documentation.
  - Using macOS as the host build OS is __not__ recommended. No warranty at all. You can get tips from [Build system setup macOS](https://openwrt.org/docs/guide-developer/build-system/buildroot.exigence.macosx) documentation.
  - For more details, please see [Build system setup](https://openwrt.org/docs/guide-developer/build-system/install-buildsystem) documentation.

  ### Quickstart
  1. Run `git clone -b mt798x-mt799x-6.6-mtwifi --single-branch --filter=blob:none https://github.com/padavanonly/immortalwrt-mt798x-24.10 immortalwrt-mt798x-24.10` to clone the source code.
  2. Run `cd immortalwrt-mt798x-24.10` to enter source directory.
  3. Run `./scripts/feeds update -a` to obtain all the latest package definitions defined in feeds.conf / feeds.conf.default
  4. Run `./scripts/feeds install -a` to install symlinks for all obtained packages into package/feeds/
  5. Copy the configuration file for your device from the `defconfig` directory to the project root directory and rename it `.config`
     
     ```
     # MT7988_mt7990
     cp -f defconfig/mt7988_mt7990.config .config

     # MT7987_mt7992
     cp -f defconfig/mt7987_mt7992.config .config

     # MT7988_mt7992
     cp -f defconfig/mt7988_mt7992.config .config

     
  6. Run `make` to build your firmware. This will download all sources, build the cross-compile toolchain and then cross-compile the GNU/Linux kernel & all chosen applications for your target system.

  ### Related Repositories
  The main repository uses multiple sub-repositories to manage packages of different categories. All packages are installed via the OpenWrt package manager called opkg. If you're looking to develop the web interface or port packages to ImmortalWrt, please find the fitting repository below.
  - [LuCI Web Interface](https://github.com/immortalwrt/luci): Modern and modular interface to control the device via a web browser.
  - [ImmortalWrt Packages](https://github.com/immortalwrt/packages): Community repository of ported packages.
  - [OpenWrt Routing](https://github.com/openwrt/routing): Packages specifically focused on (mesh) routing.
  - [OpenWrt Video](https://github.com/openwrt/video): Packages specifically focused on display servers and clients (Xorg and Wayland).

## Support Information
For a list of supported devices see the [OpenWrt Hardware Database](https://openwrt.org/supported_devices)
  ### Documentation
  - [Quick Start Guide](https://openwrt.org/docs/guide-quick-start/start)
  - [User Guide](https://openwrt.org/docs/guide-user/start)
  - [Developer Documentation](https://openwrt.org/docs/guide-developer/start)
  - [Technical Reference](https://openwrt.org/docs/techref/start)

  ### Support Community
  - Support Chat: group [@ctcgfw_openwrt_discuss](https://t.me/ctcgfw_openwrt_discuss) on [Telegram](https://telegram.org/).
  - Support Chat: group [#immortalwrt](https://matrix.to/#/#immortalwrt:matrix.org) on [Matrix](https://matrix.org/).

## License
ImmortalWrt is licensed under [GPL-2.0-only](https://spdx.org/licenses/GPL-2.0-only.html).

## Acknowledgements
<table>
  <tr>
    <td><a href="https://dlercloud.com/"><img src="https://user-images.githubusercontent.com/22235437/111103249-f9ec6e00-8588-11eb-9bfc-67cc55574555.png" width="183" height="52" border="0" alt="Dler Cloud"></a></td>
    <td><a href="https://www.jetbrains.com/"><img src="https://resources.jetbrains.com/storage/products/company/brand/logos/jb_square.png" width="120" height="120" border="0" alt="JetBrains Black Box Logo logo"></a></td>
    <td><a href="https://sourceforge.net/"><img src="https://sourceforge.net/sflogo.php?type=17&group_id=3663829" alt="SourceForge" width=200></a></td>
  </tr>
</table>
