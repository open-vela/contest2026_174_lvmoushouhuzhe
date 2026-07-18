# ESP32-P4 Camera + Display 验证计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 ESP32-P4 上启用 MIPI-DSI 显示、MIPI-CSI Camera，使用 nxcamera 工具验证 Camera 采集并在 framebuffer 上显示，同时开启常用 NSH 命令（ps、free 等）。

**Architecture:** 通过 menuconfig 启用 MIPI-DSI/CSI 驱动、VIDEO 框架、nxcamera 应用，更新 defconfig 后重新构建烧录验证。

**Tech Stack:** NuttX V4L2 video framework, MIPI-DSI (EK79007 panel), MIPI-CSI (OV5647 sensor), framebuffer, nxcamera app

---

## 前置条件

- ✅ NSH 已启动（L0 完成）
- ✅ 构建系统工作正常（CMake --cmake -j8）
- ESP32-P4 MIPI-DSI 驱动代码已在 `nuttx/arch/risc-v/src/esp32p4/esp_mipi_dsi.c`
- ESP32-P4 Camera 驱动代码已在 `nuttx/arch/risc-v/src/esp32p4/esp_camera.c`
- nxcamera 应用在 `apps/system/nxcamera/`

---

## Task 1: 启用常用 NSH 命令

**目标：** 开启 ps、free、top、cat、ls 等常用命令，让系统更易调试。

- [ ] **Step 1: 通过 menuconfig 启用**

```bash
cd /data/work/work/workspace/openvela_workspace
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela menuconfig
```

在 menuconfig 中启用：
- `Application Configuration → NSH Library → Disable Individual commands` → 确保 ps/free/cat/ls/mount 等未被 disable
- `RTOS Features → Tasks and Scheduling → Task name size` → 设为 32（让 ps 显示任务名）
- `File Systems → PROCFS` → 确保已启用（free 命令依赖）
- `RTOS Features → Performance Monitoring → Stack coloring` → 可选，用于 stack 使用量分析

- [ ] **Step 2: savedefconfig**

```bash
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela savedefconfig
```

将更新的 defconfig 拷贝回专属仓：
```bash
cp cmake_out/esp32p4-function-ev-board_openvela/defconfig \
   contest2026_174_lvmoushouhuzhe/board/esp32p4-function-ev-board/configs/openvela/defconfig
```

- [ ] **Step 3: 构建验证**

```bash
rm -rf cmake_out/esp32p4-function-ev-board_openvela
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela --cmake -j8
```

- [ ] **Step 4: 烧录验证 ps/free**

```bash
esptool.py -c esp32p4 -p /dev/ttyACM0 -b 921600 write_flash 0x2000 \
  cmake_out/esp32p4-function-ev-board_openvela/nuttx.bin
```

NSH 中验证：
```
nsh> ps
nsh> free
nsh> uname -a
```

- [ ] **Step 5: Commit**

```bash
cd contest2026_174_lvmoushouhuzhe
git add board/esp32p4-function-ev-board/configs/openvela/defconfig
git commit -m "feat(defconfig): enable ps, free, and common NSH commands"
```

---

## Task 2: 启用 MIPI-DSI 显示 (Framebuffer)

**目标：** 启用 MIPI-DSI 驱动，注册 /dev/fb0 设备，验证屏幕显示。

- [ ] **Step 1: menuconfig 启用 DSI + FB**

启用以下配置：
```
CONFIG_ESP32P4_MIPI_DSI=y         # ESP32-P4 MIPI-DSI 驱动
CONFIG_VIDEO_FB=y                  # Framebuffer 框架
CONFIG_BOARD_LATE_INITIALIZE=y     # 在 board_late_initialize 中初始化驱动
```

注意：根据 steering 文档，`BOARD_LATE_INITIALIZE` 可能导致 USB hang。
如果 hang，需要用 fbinit 命令替代（在 board_app_initialize 中手动触发）。

- [ ] **Step 2: 确认 bringup 代码中 DSI 初始化**

检查 `nuttx/boards/risc-v/esp32p4/esp32p4-function-ev-board/src/esp32p4_bringup.c` 中：

```c
#ifdef CONFIG_ESP32P4_MIPI_DSI
  ret = esp_mipi_dsi_initialize();
  if (ret < 0)
    {
      syslog(LOG_ERR, "ERROR: esp_mipi_dsi_initialize failed: %d\n", ret);
    }
#endif
```

如果不存在，需要添加。

- [ ] **Step 3: 构建、烧录、验证**

烧录后验证：
```
nsh> ls /dev/fb0
nsh> fbinfo          # 如果有这个命令
```

屏幕应显示内容（可能是黑色，因为没有写入数据）。

- [ ] **Step 4: 验证 fb 写入**

使用 dd 写入测试数据到 fb：
```
nsh> dd if=/dev/zero of=/dev/fb0 bs=1024 count=100
```

或使用 LVGL demo（如果启用）。

---

## Task 3: 启用 MIPI-CSI Camera (V4L2)

**目标：** 启用 Camera 驱动，注册 /dev/video0，验证图像采集。

- [ ] **Step 1: menuconfig 启用 Camera**

```
CONFIG_ESP32P4_CAMERA=y           # ESP32-P4 MIPI-CSI Camera
CONFIG_VIDEO=y                     # Video 框架
CONFIG_VIDEO_STREAM=y              # V4L2 capture stream
CONFIG_I2C=y                       # I2C（Camera sensor 通信）
CONFIG_I2C_DRIVER=y
```

- [ ] **Step 2: 确认 bringup 中 Camera 初始化**

```c
#ifdef CONFIG_ESP32P4_CAMERA
  ret = esp_camera_initialize();
  if (ret < 0)
    {
      syslog(LOG_ERR, "ERROR: esp_camera_initialize failed: %d\n", ret);
    }
#endif
```

- [ ] **Step 3: 构建、烧录、验证**

```
nsh> ls /dev/video0
```

---

## Task 4: 启用 nxcamera 并验证 Camera→Display 链路

**目标：** 使用 nxcamera 工具从 Camera 采集并输出到 framebuffer。

- [ ] **Step 1: menuconfig 启用 nxcamera**

```
CONFIG_SYSTEM_NXCAMERA=y
CONFIG_NXCAMERA_MAINTHREAD_STACKSIZE=8192
```

- [ ] **Step 2: 构建、烧录**

- [ ] **Step 3: 验证 Camera→FB 流程**

```
nsh> nxcamera
nxcam> output /dev/fb0
nxcam> input /dev/video0
nxcam> stream
```

预期：摄像头图像实时显示在 7 寸屏幕上。

- [ ] **Step 4: Commit 最终 defconfig**

```bash
cd contest2026_174_lvmoushouhuzhe
git add board/esp32p4-function-ev-board/configs/openvela/defconfig
git commit -m "feat(defconfig): enable MIPI-DSI display, Camera, and nxcamera

- CONFIG_ESP32P4_MIPI_DSI: 7-inch 1024x600 LCD via MIPI-DSI
- CONFIG_ESP32P4_CAMERA: OV5647 via MIPI-CSI, V4L2 /dev/video0
- CONFIG_VIDEO_FB: Framebuffer /dev/fb0
- CONFIG_SYSTEM_NXCAMERA: Camera→Display streaming tool
- Common NSH commands (ps, free, top) enabled"
```

---

## Task 5: 处理 BOARD_LATE_INITIALIZE 可能的 hang

**目标：** 如果 Task 2/3 中启用 BOARD_LATE_INITIALIZE 导致 USB 无响应，用替代方案。

- [ ] **Step 1: 如果 hang，改用 fbinit 命令方式**

创建 `apps/examples/fbinit/` 应用，在 NSH 中手动触发初始化：
```
nsh> fbinit    # 手动初始化 DSI + Camera
```

参考 steering 文档中已验证的 fbinit 方案。

- [ ] **Step 2: 或者在 board_app_initialize 中初始化**

修改 `esp32p4_appinit.c` 中的 `board_app_initialize()`，只做不阻塞的初始化。

---

## 风险与注意事项

| 风险 | 影响 | 应对 |
|------|------|------|
| BOARD_LATE_INITIALIZE 导致 USB hang | 系统无串口输出 | 使用 fbinit 命令替代或 board_app_initialize |
| PSRAM 未启用 | FB 需要大 buffer（1024×600×2=1.2MB） | 需要启用 SPIRAM 提供 PSRAM heap |
| Camera I2C 通信失败 | /dev/video0 不注册 | 检查 I2C GPIO 配置（GPIO7/8） |
| DSI PHY PLL 不 lock | 屏幕无显示 | 需 OpenOCD 调试 PHY 寄存器 |

## 依赖关系

```
Task 1 (NSH 命令) → 独立，可先做
Task 2 (DSI) → 可能需要 PSRAM（Task 额外）
Task 3 (Camera) → 依赖 I2C
Task 4 (nxcamera) → 依赖 Task 2 + Task 3
Task 5 (hang fix) → 只在 Task 2/3 遇到问题时执行
```

## PSRAM 启用（可能需要的前置任务）

如果 framebuffer 需要 PSRAM（1024×600×2 = 1.2MB > 内部 SRAM）：

```
CONFIG_ESPRESSIF_SPIRAM=y
CONFIG_ESPRESSIF_SPIRAM_MODE_HEX=y
CONFIG_MM_REGIONS=2
```

这需要 esp-hal-3rdparty 中的 PSRAM 初始化正确工作（之前已有相关修复）。
