---
name: macbook-facetimehd-linux
description: Make the FaceTime HD camera work on Apple MacBooks running Linux (Arch/Omarchy or similar). Covers Broadcom 720p FaceTime HD Camera [14e4:1570] used in MacBook Air 6,1/6,2/7,1/7,2 (A1465/A1466/A1369) and MacBook Pro 11,x/12,x — the reverse-engineered facetimehd driver, firmware extracted from Apple's macOS OSXUpd10.11.5.dmg, sensor calibration data extracted from Apple BootCamp 5.1.5769, DKMS build on new kernels, auto-load, video group, and the /tmp-wipe pitfall. Use when the user asks about 摄像头驱动 / camera not working on a Mac running Linux / facetimehd / FaceTime HD / Broadcom 14e4:1570 / 苹果本 装 Linux 摄像头 / webcam no /dev/video0 on MacBook Air 1466, or "从 apple 下载摄像头驱动".
agent_created: true
---

# FaceTime HD camera (Broadcom 14e4:1570) on Linux MacBooks

The 2013–2017 MacBook Air (A1465/A1466, board MacBookAir6,1/6,2/7,1/7,2) and
MacBook Pro (11,x/12,x) FaceTime HD camera is **not** a USB UVC device. It is a
PCIe device:

```
02:00.0 Multimedia controller [0480]: Broadcom Inc. and subsidiaries 720p FaceTime HD Camera [14e4:1570]
```

`uvcvideo` never binds to it. On macOS it is driven by Apple's private
`AppleCameraInterface.kext`. On Linux the working solution is the reverse
engineered **`facetimehd`** driver (patjak/facetimehd) plus two blobs that Apple
only ships inside macOS installer packages. This skill is the complete,
executed recipe.

**Verified on hardware 2026-10-28**: MacBookAir7,2 (A1466), Omarchy
(Arch, kernel 7.2.8-9-omarchy), driver 0.7.2, firmware 1.43_5, calibration
5.1.5769. Result: `/dev/video0` 1280×720@30 YUYV/YVYU, natural colours after
calibration, auto-loads on boot. The only things you need from Apple are the
two blobs; everything else is buildable from source.

## 中文速答

- **是什么**：Broadcom 1570 PCIe 摄像头，不是 USB UVC，`uvcvideo` 不认，`/dev/video0` 不会自动出现。
- **驱动**：AUR 包 `facetimehd-dkms`（逆向工程驱动，源码 patjak/facetimehd，tag 0.7.2）。
- **必须的 Apple 固件**：AUR 包 `facetimehd-firmware`，从 Apple CDN 的 `OSXUpd10.11.5.dmg` 里按字节区间抠出 `AppleCameraInterface` → `firmware.bin`（1.36 MB）。
- **可选的 Apple 校准**：AUR 包 `facetimehd-data`，从 Apple `bootcamp5.1.5769.zip`（517 MB）里用 `7z` 解 `AppleCamera64.exe`，再用 `dd` 抠出 4 个 `.dat`。**没有它画面会偏青绿**，有它肤色正常。
- **三条最坑的坑**：(1) 新内核必须自己先编译一遍确认能过；(2) `/tmp` 会被 systemd-tmpfiles 在重启时清空，中途下载的大文件要放 `~/.cache/`；(3) `video` 组要重启后才生效（但 logind 会先给当前登录用户 ACL，所以当场就能用）。

## 硬件与前置条件

```bash
lspci -nn | grep -iE "14e4:1570|Multimedia"   # 确认设备
cat /sys/class/dmi/id/product_name            # 如 MacBookAir7,2
uname -r                                      # 当前内核，需要配套 -headers 包
pacman -Q linux-omarchy-headers               # Arch 上内核 headers 必须已装
```

需要：`base-devel`、`git`、`dkms`、对应内核的 `*-headers`、一个 AUR helper
（本例用 `yay`）或手工 `makepkg`。`7zip` 只在做校准数据时需要（`makedepends`）。

## 第一步：先试编译，再谈安装

DKMS 版本很老（0.7.2），新内核常有 V4L2 API 改动导致编译失败。**安装前先当普通
用户试编译一次**，失败就先用 `facetimehd-dkms-git` 或打补丁，别直接装：

```bash
git clone --depth 1 --branch 0.7.2 https://github.com/patjak/facetimehd.git /tmp/fhd/src
make -C /lib/modules/$(uname -r)/build M=/tmp/fhd/src modules
# 看到 facetimehd.ko + BTF 输出 = 能编过
```

## 第二步：固件（从 Apple 下载）

AUR `facetimehd-firmware` 的 `prepare()` 用**部分下载**（HTTP Range 只取需要的
2.8 MB 片段，不用下整个 723 MB dmg）从 Apple CDN 抓 `AppleCameraInterface`：

- URL：`https://updates.cdn-apple.com/2019/cert/041-88431-20191011-e7ee7d98-2878-4cd9-bc0a-d98b3a1e24b1/OSXUpd10.11.5.dmg`
- Range：`204909802-207733123`
- 用 `xzcat` + `cpio --format odc` 从 dmg 里解出 `System/Library/Extensions/AppleCameraInterface.kext/Contents/MacOS/AppleCameraInterface`
- 然后 `dd bs=1 skip=81920 count=603715` 抠出 `firmware.bin.gz`，gunzip 得到 1,425,412 字节的 `firmware.bin`
- 装到 `/usr/lib/firmware/facetimehd/firmware.bin`

打包后 `sha256sum` 校验（PKGBUILD 里写死了 `DRV_HASH`/`FW_HASH`）。

## 第三步：校准数据（可选，从 Apple 下载）

AUR `facetimehd-data` 下载 `https://download.info.apple.com/Mac_OS_X/031-30890-20150812-ea191174-4130-11e5-a125-930911ba098f/bootcamp5.1.5769.zip`
（**517 MB**，sha256 `4ede2c8ef240708c850237a3e5911094ed6adae1734258e4639bc9069a814b1e`）：

```bash
7z x "BootCamp/Drivers/Apple/AppleCamera64.exe"   # 先 bsdtar/7z 解 zip，再 7z 解 exe
dd bs=1 skip=1663920 count=33060 if='AppleCamera.sys' of='9112_01XX.dat'
dd bs=1 skip=1644880 count=19040 if='AppleCamera.sys' of='1771_01XX.dat'
dd bs=1 skip=1606800 count=19040 if='AppleCamera.sys' of='1871_01XX.dat'
dd bs=1 skip=1625840 count=19040 if='AppleCamera.sys' of='1874_01XX.dat'
```

装到 `/usr/lib/firmware/facetimehd/`。**不装也能出画面，但会偏青绿**；装上后肤色正常。
日志里对应报错是 `Direct firmware load for facetimehd/1871_01XX.dat failed with error -2`。

## 第四步：安装与加载

```bash
yay -S facetimehd-dkms facetimehd-firmware facetimehd-data   # 或逐个 makepkg -si
echo facetimehd > /etc/modules-load.d/facetimehd.conf        # 开机自启
usermod -aG video ajay                                       # 权限组（重启生效）
modprobe facetimehd
```

验证：

```bash
lsmod | grep facetimehd
ls -l /dev/video0                    # crw-rw----+ root video —— '+' 表示 logind ACL 已授权当前用户
v4l2-ctl -d /dev/video0 --list-formats-ext   # YUYV / YVYU, 1280x720@30
ffmpeg -f v4l2 -i /dev/video0 -frames:v 1 /tmp/cam.jpg   # 抓一帧看颜色
journalctl -k -b | grep -i facetime # 期望: Loaded firmware 1392kb, Full memory verification succeeded
```

## 坑与对策（本机实测踩过的）

| 坑 | 现象 | 对策 |
|---|---|---|
| 新内核编译失败 | DKMS 报 V4L2 错误 | 先按第一步试编；不行换 `facetimehd-dkms-git` 或给 0.7.2 打补丁 |
| `/tmp` 被清 | 517 MB 下载、AUR clone 全没了，sha256 全白下 | 大文件放 `~/.cache/`（如 `~/.cache/fhd/`），别放 `/tmp`；重启后要重来 |
| `video` 组不生效 | `id -nG` 里没有 video | 重启；但 logind ACL 让当前会话当场可用 |
| Secure Boot | 模块签名拒绝 | `lockdown` 为 none 时无碍；`/sys/kernel/security/lockdown` 检查 |
| sudo 密码 | 非交互 shell 拿不到密码 | `setsid nohup foot -e bash 脚本` 弹一个真实终端让用户输一次密码；sudo 时间戳同 tty 共享，后续 sudo 不再问 |
| pgrep 自匹配 | `pgrep -f "bootcamp5.1"` 匹配到自己命令行，误报"仍在下载" | 加 `grep -v pgrep`，或按完整文件名判断，或看 sha256 是否已等于期望值 |
| 部分下载校验 | dmg 是 723 MB 但只需 2.8 MB | 用 `curl -r range \| xzcat \| cpio` 流式解，别整包下 |

## 参考资料

- 驱动源码：https://github.com/patjak/facetimehd （tag 0.7.2）
- 固件提取说明：https://github.com/patjak/facetimehd/wiki/Extracting-the-firmware
- 校准提取说明：https://github.com/patjak/facetimehd/wiki/Extracting-the-sensor-calibration-files
- 官方 FAQ：https://github.com/patjak/facetimehd/wiki/FAQ
- 本机实测记录：本 skill 由一次完整实装（2026-10-28，MacBookAir7,2 / Omarchy / kernel 7.2.8）蒸馏而来，所有命令均在本机验证过。
