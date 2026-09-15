---
name: esp32p4-multimedia
description: "ESP32-P4 显示/摄像头多媒体驱动调试。MIPI-DSI 显示、MIPI-CSI 摄像头、campreview 30fps 零拷贝、ISP 白平衡、JPEG 编码器。Use when: ESP32-P4 显示、摄像头、DSI、CSI、campreview、camera preview、零拷贝、白平衡、SC2336、EK79007、JPEG 编码。"
---

# ESP32-P4 显示/摄像头多媒体驱动

## 链路概览

```
SC2336 摄像头 → MIPI-CSI → ISP → DW-GDMA → 显示缓冲(fb0) → MIPI-DSI → EK79007 面板
```

- 显示：EK79007 1024×600 RGB565，`/dev/fb0` 双缓冲（`yres_virtual=1200`，`fblen=2457600`）
- 摄像头：SC2336，`/dev/video0` V4L2（I2C chip ID 0xcb3a）
- DW-GDMA：从 ESP-IDF 移植到 NuttX

## 验证命令

```bash
nsh> ls /dev              # 应有 fb0、video0
nsh> campreview 180       # 180 帧预览，输出 60/120/180 帧的 fps
nsh> campreview -h        # 用法
```

正常输出末尾应是 `campreview: 180 frames, 30.0 fps` + `zero-copy`。

## 零拷贝的关键（30fps 的来由）

- 全链路 RGB565，避免逐像素格式转换（6fps → 15fps → 30fps 的演进）
- fb0 双缓冲（`yres_virtual=1200`），摄像头 DMA 直写显示缓冲
- 每帧仅 msync 1.23MB，无 CPU memcpy
- 判断是否零拷贝：campreview 输出 `zero-copy` 而非 `copy`；`fblen` 应为 2457600

## ISP 白平衡

SC2336 出 RAW8(BGGR)，ISP 做 demosaic → RGB565。raw Bayer 天然绿偏（绿像素占一半），用 CCM
对角线静态增益校正：

```c
#define ISP_WB_GAIN_R  (ISP_CCM_ONE * 185 / 100)  /* 1.85x */
#define ISP_WB_GAIN_G  (ISP_CCM_ONE)              /* 1.00x */
#define ISP_WB_GAIN_B  (ISP_CCM_ONE * 175 / 100)  /* 1.75x */
```

偏绿调高 R/B，偏红调低 R。这是固定增益，真正的 AWB（`ISP_AWB_*`）需控制环路，属后续工作。

## 屏幕红屏自检

开机时 `esp_mipi_dsi_start_refresh()` 把 framebuffer 填红。**开机纯红屏 = DSI 链路全通**
（PHY→Host→Bridge→DMA→面板）。不是红屏先查显示，不要查 camera。

## JPEG 编码器

`CONFIG_ESP32P4_JPEG_ENCODER` 注册 `/dev/video1`（V4L2 M2M）。但 campilot 的「camera→JPEG→
MiMo」链路会卡死——根因是 ESP-HAL 的 JPEG/DMA handler 破坏 callee-saved 寄存器（见
esp32p4-interrupt-clic），非 JPEG 驱动本身。
