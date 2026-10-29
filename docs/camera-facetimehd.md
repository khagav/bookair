# FaceTime HD camera (Broadcom 14e4:1570) — full recipe

适用：MacBook Air 6,x/7,x（A1465/A1466）、MacBook Pro 11,x/12,x。

摄像头是 **PCIe** 设备（`02:00.0 Multimedia controller [0480]: Broadcom 720p
FaceTime HD Camera [14e4:1570]`），**不是** USB UVC，`uvcvideo` 不认，所以
`/dev/video0` 不会自动出现。Linux 上可用的是逆向工程驱动 `facetimehd`
（patjak/facetimehd，tag 0.7.2）+ 两块 Apple 私有 blob。

## 本机实测结论（2026-10-29）

- 驱动 `facetimehd-dkms 0.7.2-1`、固件 `facetimehd-firmware 1:1.43_5-2`、
  校准 `facetimehd-data 5.1.5769-2`，kernel `7.2.8-9-omarchy`。
- 结果：`/dev/video0`，1280×720@30（YUYV/YVYU），校准后肤色正常，
  开机自动加载（`/etc/modules-load.d/facetimehd.conf` + PCI modalias）。
- OBS 里用 **Video Capture Device (V4L2)** 源选 “Apple Facetime HD”（PipeWire
  也能枚举到 `v4l2:/dev/video0`，但 V4L2 源最稳）。
- 颜色验证：校准前 HUE 188.8°（偏青绿）、校准后 138.5°（自然）。

## 三个 AUR 包

| 包 | 作用 | 来源 |
|---|---|---|
| `facetimehd-dkms` | 驱动（源码 patjak/facetimehd @ 0.7.2） | GitHub，构建 |
| `facetimehd-firmware` | `firmware.bin` 1.36 MB（必须） | Apple `OSXUpd10.11.5.dmg`（Range 取 2.8 MB 片段） |
| `facetimehd-data` | `*_01XX.dat` ×4，传感器校准（可选但强烈建议） | Apple `bootcamp5.1.5769.zip`（517 MB） |

## 第一步：先试编译，别急着装

DKMS 0.7.2 较老，新内核常有 V4L2 API 改动导致编译失败。装之前先当普通用户
试编译一次：

```bash
git clone --depth 1 --branch 0.7.2 https://github.com/patjak/facetimehd.git /tmp/fhd/src
make -C /lib/modules/$(uname -r)/build M=/tmp/fhd/src modules
```

看到 `facetimehd.ko` + BTF 输出 = 能编过。失败就换 `facetimehd-dkms-git`
或给 0.7.2 打补丁。

## 第二步：固件（firmware.bin）

AUR `facetimehd-firmware` 用 **部分下载**（HTTP Range 只取 2.8 MB 片段，
不用下 723 MB dmg）：

- URL `https://updates.cdn-apple.com/2019/cert/041-88431-20191011-e7ee7d98-2878-4cd9-bc0a-d98b3a1e24b1/OSXUpd10.11.5.dmg`
- Range `204909802-207733123`
- `curl -r RANGE | xzcat -qq -Q | cpio --format odc -i -d` 解出
  `System/Library/Extensions/AppleCameraInterface.kext/Contents/MacOS/AppleCameraInterface`
- `dd bs=1 skip=81920 count=603715` → `firmware.bin.gz`，gunzip → `firmware.bin`（1,425,412 B）
- 校验：驱动 hash `f56e68a880b65767335071531a1c75f3cfd4958adc6d871adf8dbf3b788e8ee1`，
  固件 hash `e3e6034a67dfdaa27672dd547698bbc5b33f47f1fc7f5572a2fb68ea09d32d3d`
- 装到 `/usr/lib/firmware/facetimehd/firmware.bin`

## 第三步：校准数据（*_01XX.dat ×4）

`bootcamp5.1.5769.zip`（**517 MB**，sha256
`4ede2c8ef240708c850237a3e5911094ed6adae1734258e4639bc9069a814b1e`）：

```bash
7z x bc.zip 'BootCamp/Drivers/Apple/AppleCamera64.exe'
7z x AppleCamera64.exe          # 得到 AppleCamera.sys
dd bs=1 skip=1663920 count=33060 if=AppleCamera.sys of=9112_01XX.dat
dd bs=1 skip=1644880 count=19040 if=AppleCamera.sys of=1771_01XX.dat
dd bs=1 skip=1606800 count=19040 if=AppleCamera.sys of=1871_01XX.dat
dd bs=1 skip=1625840 count=19040 if=AppleCamera.sys of=1874_01XX.dat
```

`.dat` 校验值（本机从 GitHub Release 实测，与 GitHub 官方 `digest` 一致）：

```
4dd756fa8460d8dc3d78d0d76944b2f92275d1fe9c83181bbc8292c81c005f1a  9112_01XX.dat
756c2bb7c5e55b395449e43a0be1cb7c40c37dfc6c2b5abfaffb8ae70ff0fc4b  1771_01XX.dat
bf36fbde0668ab7e44368b584f9fa64b5945b01003d04c6e3c6f22c0be0fd5f3  1871_01XX.dat
ffde89e7819ac16a9eb1c8f0bc6dba0e980b508b2022507679d901c190f7cef8  1874_01XX.dat
```

装到 `/usr/lib/firmware/facetimehd/`。**不装也能出画面，但会偏青绿**；日志
对应报错 `Direct firmware load for facetimehd/1871_01XX.dat failed with error -2`。

> 下载方法（Apple 慢/连不上）见 [download-relay.md](download-relay.md) ——
> 已把 4 个 `.dat` 发布成可复用 Release：`khagav/crdroid-nabu-build`
> `releases/tag/relay-facetimehd-data`。

## 第四步：安装与加载

```bash
sudo pacman -S --needed 7zip                 # 仅当要现场解 517 MB zip
# 三个包：见 docs 或 AUR（facetimehd-dkms / facetimehd-firmware / facetimehd-data）
echo facetimehd > /etc/modules-load.d/facetimehd.conf   # 开机自启
sudo usermod -aG video "$USER"                        # 权限组（重启生效；logind 当场给 ACL）
sudo modprobe facetimehd
```

验证：

```bash
lsmod | grep facetimehd
ls -l /dev/video0        # crw-rw----+ root video —— '+' = logind ACL 已授权当前用户
v4l2-ctl -d /dev/video0 --list-formats-ext
ffmpeg -f v4l2 -i /dev/video0 -frames:v 1 /tmp/cam.jpg
journalctl -k -b | grep -i facetime   # 期望: Loaded firmware 1392kb, Full memory verification succeeded
```

## 坑与对策

| 坑 | 现象 | 对策 |
|---|---|---|
| 新内核编译失败 | DKMS 报 V4L2 错误 | 先试编；不行换 `facetimehd-dkms-git` 或打补丁 |
| `/tmp` 被清 | 517 MB 下载、AUR clone 全没 | 大文件放 `~/.cache/`，别放 `/tmp` |
| `video` 组不生效 | `id -nG` 里没有 video | 重启；logind ACL 当场可用 |
| Secure Boot | 模块签名拒绝 | `/sys/kernel/security/lockdown` 应为 `[none]` |
| sudo 密码 | 非交互 shell 拿不到密码 | `setsid nohup foot -e bash 脚本` 弹真实终端输一次密码 |
| pgrep 自匹配 | 误报"仍在下载" | `pgrep -f '[b]ootcamp5.1'` 用方括号避免匹配自己 |
| Apple 源连不上 | curl 卡死/超时 | 用 [download-relay.md](download-relay.md) 的 GitHub 转存 |

## 参考

- 驱动源码 https://github.com/patjak/facetimehd（tag 0.7.2）
- 固件提取 https://github.com/patjak/facetimehd/wiki/Extracting-the-firmware
- 校准提取 https://github.com/patjak/facetimehd/wiki/Extracting-the-sensor-calibration-files
- FAQ https://github.com/patjak/facetimehd/wiki/FAQ
