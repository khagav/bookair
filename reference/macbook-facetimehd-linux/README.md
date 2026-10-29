# macbook-facetimehd-linux

让 Linux 下的苹果 MacBook 内置 FaceTime HD 摄像头（Broadcom 720p FaceTime HD Camera,
PCIe **14e4:1570**）正常工作。

适用机型：MacBook Air 6,x / 7,x（A1465/A1466，含 MacBookAir7,2）、MacBook Pro 11,x / 12,x。
这些摄像头是 PCIe 设备，**不是** USB UVC，`uvcvideo` 不认，需要逆向工程的
[`facetimehd`](https://github.com/patjak/facetimehd) 驱动，以及两块只有 Apple 才提供的 blob：

| blob | 来源 | 作用 |
|---|---|---|
| `firmware.bin` | Apple `OSXUpd10.11.5.dmg`（723 MB，只按 Range 取 2.8 MB 片段） | 必须，没有它摄像头完全不出画面 |
| `*_01XX.dat` ×4 | Apple `bootcamp5.1.5769.zip`（517 MB） | 可选，传感器校准；缺了画面偏青绿 |

完整步骤、踩坑记录与命令见 [`SKILL.md`](SKILL.md)。

## 状态

- **已验证**：2026-10-28 在 MacBookAir7,2（A1466）/ Omarchy（Arch）/ kernel `7.2.8-9-omarchy`
  上实装。结果：`/dev/video0` 1280×720@30（YUYV/YVYU）、校准后肤色正常、随开机自动加载。
- 驱动 `facetimehd-dkms 0.7.2-1`、固件 `facetimehd-firmware 1:1.43_5-2`、校准 `facetimehd-data 5.1.5769-2`。

## 快速开始（Arch / Omarchy）

```bash
# 0) 先试编译，别急着装（老驱动 vs 新内核常有 V4L2 改动）
git clone --depth 1 --branch 0.7.2 https://github.com/patjak/facetimehd.git /tmp/fhd/src
make -C /lib/modules/$(uname -r)/build M=/tmp/fhd/src modules

# 1) 三个 AUR 包（firmware 从 Apple dmg 取，data 从 Apple BootCamp 取）
yay -S facetimehd-dkms facetimehd-firmware facetimehd-data

# 2) 开机自启 + 权限 + 加载
echo facetimehd > /etc/modules-load.d/facetimehd.conf
usermod -aG video "$USER"
modprobe facetimehd

# 3) 验证
ls /dev/video0 && ffmpeg -f v4l2 -i /dev/video0 -frames:v 1 /tmp/cam.jpg
```

> ⚠️ `/tmp` 会被 systemd-tmpfiles 在重启时清空，中途下载的 517 MB 文件要放 `~/.cache/`。
> `video` 组重启后才生效（但 logind 的 ACL 会让当前会话当场可用）。
