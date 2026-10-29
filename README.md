# bookair — MacBookAir 7,2 (A1466) Linux hardware & driver repair

One place for every Mac hardware/driver fix on this machine, so next time (fresh
install, new kernel, re-flash) it's all reusable.

| 项目 | 设备 | 驱动/方案 | 状态 | 文档 |
|---|---|---|---|---|
| 摄像头 FaceTime HD | Broadcom 14e4:1570 (PCIe) | `facetimehd-dkms` + Apple firmware + 校准 | ✅ 修复 2026-10-29 | [docs/camera-facetimehd.md](docs/camera-facetimehd.md) |
| 无线网卡 | Broadcom BCM4360 14e4:43a0 | `broadcom-wl` (dkms) | ✅ 工作 | [docs/wifi-broadcom.md](docs/wifi-broadcom.md) |
| Apple 私有 blob 下载 | — | GitHub CI 转存 + 加速器 | ✅ 实测 | [docs/download-relay.md](docs/download-relay.md) |

- 机型：`MacBookAir7,2`（A1466，2015）
- 系统：Omarchy 4.0.6（Arch）/ kernel `7.2.8-9-omarchy`
- 全部命令在本机实测过；踩坑记录在各文档内。

## 下次取用（快速恢复）

```bash
# 基础依赖（新装系统后）
sudo pacman -S --needed base-devel git dkms linux-omarchy-headers

# 摄像头：见 docs/camera-facetimehd.md（3 个 AUR 包，Apple blob 可用中转 Release）
# 开机自启 + 权限 + 立即生效
echo facetimehd > /etc/modules-load.d/facetimehd.conf
sudo usermod -aG video "$USER"
sudo modprobe facetimehd

# 验证
ls -l /dev/video0 && ffmpeg -f v4l2 -i /dev/video0 -frames:v 1 /tmp/cam.jpg
```

## 目录

- `reference/macbook-facetimehd-linux/` — 摄像头 skill 原件（SKILL.md/README.md）
- `docs/` — 分设备完整步骤与踩坑

## 中文速答

- 这是 MacBookAir7,2（2015）在 Linux 上所有硬件/驱动修复的统一仓库。
- 摄像头不是 USB 设备，是 PCIe，需要逆向工程驱动 + 两块 Apple 私有 blob。
- Apple 下载慢或连不上时用 `docs/download-relay.md` 的 GitHub 转存 + 加速器法。
