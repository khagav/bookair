# Wi-Fi — Broadcom BCM4360 (14e4:43a0) 无线网卡

本机网卡：`03:00.0 Network controller: Broadcom BCM4360 802.11ac Dual Band
Wireless Network Adapter [14e4:43a0] (rev 03)`。

Linux 上 BCM4360 用 `broadcom-wl`（AUR/额外仓库的 DKMS 驱动）最稳；内核自带的
`b43`/`brcmsmac` 对这个芯片支持差。

## 本机实测（2026-10-29）

- 包：`broadcom-wl`（dkms，版本 `6.30.223.271`），状态：
  ```bash
  dkms status
  # broadcom-wl/6.30.223.271, 7.2.8-9-omarchy, x86_64: installed
  ```
- 模块 `wl` 已加载（`lsmod | grep '^wl'`），wifi 工作。

## 检查/恢复

```bash
sudo modprobe wl            # 加载
dkms status                 # 确认内核匹配
sudo pacman -S --needed broadcom-wl    # 或 AUR broadcom-wl-dkms
```

## 常见坑

- 换内核后 `wl` 没自动重编 → `sudo dkms autoinstall`。
- 与 `b43`/`brcmsmac` 冲突：确保 `modprobe.d` 里黑名单了它们（若装过）。
- 蓝牙（如果同卡）由 `btusb` 处理，与 wl 无冲突。
