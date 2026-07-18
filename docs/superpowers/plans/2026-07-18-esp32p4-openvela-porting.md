# ESP32-P4 OpenVela 系统适配 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 ESP32-P4X-Function-EV-Board 的 openvela 适配代码按比赛规范整理到专属仓，并完成从 L0 到 L3 的全功能外设驱动适配。

**Architecture:** 代码分两路提交 — 芯片层/NuttX 板级源码通过 cherry-pick 提 PR 到 open-vela/nuttx 公共仓；vendor 板级适配（defconfig、Make.defs、CMakeLists.txt）放在专属仓 board/ 目录，通过 manifest linkfile 映射到构建树。

**Tech Stack:** NuttX RTOS, RISC-V, ESP32-P4, CMake, esp-hal-3rdparty, openvela build system

---

## 文件结构

### 专属仓（创建/修改）

| 操作 | 文件路径 | 职责 |
|------|----------|------|
| 修改 | `contest2026_174_lvmoushouhuzhe.xml` | manifest linkfile 映射 |
| 创建 | `board/esp32p4-function-ev-board/configs/openvela/defconfig` | openvela 构建配置 |
| 创建 | `board/esp32p4-function-ev-board/scripts/Make.defs` | 构建工具链配置 |
| 创建 | `board/esp32p4-function-ev-board/CMakeLists.txt` | CMake 入口 |
| 创建 | `board/esp32p4-function-ev-board/Kconfig` | 板级可选配置 |
| 创建 | `board/esp32p4-function-ev-board/README.md` | 板级说明 |
| 修改 | `README.md` | 作品说明（替换模板） |
| 创建 | `docs/porting_guide.md` | 适配复现指南 |
| 删除 | `board/contest_board/` | 移除模板占位目录 |

### 公共仓 PR（cherry-pick + 增量）

| 操作 | 文件路径 | 来源 |
|------|----------|------|
| 添加 | `arch/risc-v/src/esp32p4/` | apache/nuttx cherry-pick |
| 添加 | `boards/risc-v/esp32p4/common/` | apache/nuttx cherry-pick |
| 添加 | `boards/risc-v/esp32p4/esp32p4-function-ev-board/` | apache/nuttx + 本地增量 |

---

## Task 1: 清理模板目录并创建板级目录结构

**Files:**
- 删除: `board/contest_board/`（模板占位）
- 创建: `board/esp32p4-function-ev-board/configs/openvela/defconfig`
- 创建: `board/esp32p4-function-ev-board/scripts/Make.defs`
- 创建: `board/esp32p4-function-ev-board/CMakeLists.txt`
- 创建: `board/esp32p4-function-ev-board/Kconfig`
- 创建: `board/esp32p4-function-ev-board/README.md`

- [ ] **Step 1: 删除模板占位目录**

```bash
cd /data/work/work/workspace/openvela_workspace/contest2026_174_lvmoushouhuzhe
rm -rf board/contest_board
```

- [ ] **Step 2: 创建新目录结构**

```bash
mkdir -p board/esp32p4-function-ev-board/configs/openvela
mkdir -p board/esp32p4-function-ev-board/scripts
```

- [ ] **Step 3: 创建 defconfig**

从本地已验证的 openvela 适配代码复制 defconfig：

```bash
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela/defconfig \
   board/esp32p4-function-ev-board/configs/openvela/defconfig
```

验证内容应包含：
- `CONFIG_ARCH="risc-v"`
- `CONFIG_ARCH_CHIP="esp32p4"`
- `CONFIG_ARCH_BOARD="esp32p4-function-ev-board"`
- `CONFIG_ESPRESSIF_USBSERIAL=y`

- [ ] **Step 4: 创建 Make.defs**

从本地已验证代码复制：

```bash
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/scripts/Make.defs \
   board/esp32p4-function-ev-board/scripts/Make.defs
```

- [ ] **Step 5: 创建 CMakeLists.txt**

从本地已验证代码复制：

```bash
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/CMakeLists.txt \
   board/esp32p4-function-ev-board/CMakeLists.txt
```

- [ ] **Step 6: 创建 Kconfig**

创建文件 `board/esp32p4-function-ev-board/Kconfig`：

```kconfig
#
# ESP32-P4 Function EV Board configuration (openvela contest 174)
#

if ARCH_BOARD_ESP32P4_FUNCTION_EV_BOARD

endif
```

- [ ] **Step 7: 创建板级 README.md**

创建文件 `board/esp32p4-function-ev-board/README.md`，内容说明：
- 这是 ESP32-P4X-Function-EV-Board 的 openvela vendor 板级适配
- 构建命令
- 依赖的公共仓 PR

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(board): add esp32p4-function-ev-board vendor adaptation

- Remove template contest_board placeholder
- Add openvela defconfig for ESP32-P4X-Function-EV-Board
- Add Make.defs and CMakeLists.txt build configuration
- Add Kconfig and board README"
```

---

## Task 2: 修改 Manifest 映射

**Files:**
- 修改: `contest2026_174_lvmoushouhuzhe.xml`

- [ ] **Step 1: 修改 manifest XML**

将 `contest2026_174_lvmoushouhuzhe.xml` 内容替换为：

```xml
<?xml version='1.0' encoding='UTF-8'?>
<manifest>
  <!--
    Team 174 repo manifest.
    Includes the full openvela base (openvela.xml) and maps team board/app
    directories into the openvela build tree via <linkfile>.
  -->
  <include name="openvela.xml"/>

  <project path="contest2026_174_lvmoushouhuzhe"
           name="contest2026_174_lvmoushouhuzhe">
    <!-- 板级适配：映射到 vendor/espressif 标准路径 -->
    <linkfile src="board/esp32p4-function-ev-board"
              dest="vendor/espressif/boards/esp32p4/esp32p4-function-ev-board"/>
    <!-- quickapp：保留供后续 UI 应用开发 -->
    <linkfile src="quickapp/hello_quickapp"
              dest="packages/apps/contest2026_174_hello_quickapp"/>
  </project>
</manifest>
```

- [ ] **Step 2: 验证 XML 格式正确**

```bash
xmllint --noout contest2026_174_lvmoushouhuzhe.xml
```

Expected: 无输出（格式正确）。如果 xmllint 不可用，肉眼确认标签闭合即可。

- [ ] **Step 3: Commit**

```bash
git add contest2026_174_lvmoushouhuzhe.xml
git commit -m "feat(manifest): update linkfile to vendor/espressif path

Map board/esp32p4-function-ev-board to the standard
vendor/espressif/boards/esp32p4/ location, matching the
upstream target repo structure for post-contest PR."
```

---

## Task 3: 提交 NuttX 公共仓 PR（芯片层）

**Files:**
- 添加: `arch/risc-v/src/esp32p4/`（含 esp-hal-3rdparty）
- 修改: `arch/risc-v/src/Makefile`（注册新芯片目录）
- 修改: `arch/risc-v/src/CMakeLists.txt`
- 修改: `arch/risc-v/Kconfig`（引入 esp32p4 Kconfig）

- [ ] **Step 1: Fork open-vela/nuttx 到个人 GitHub**

在 GitHub 上 fork `https://github.com/open-vela/nuttx`。

- [ ] **Step 2: Clone 并创建 feature 分支**

```bash
git clone https://github.com/<your-github>/nuttx.git /tmp/openvela-nuttx-fork
cd /tmp/openvela-nuttx-fork
git checkout dev-ai-contest-2026
git checkout -b feat/esp32p4-arch-support
```

- [ ] **Step 3: 添加 apache/nuttx 为 remote 并 fetch**

```bash
git remote add upstream-apache https://github.com/apache/nuttx.git
git fetch upstream-apache master
```

- [ ] **Step 4: Cherry-pick ESP32-P4 芯片层 commits**

从 apache/nuttx 中找到 ESP32-P4 相关 commits 并 cherry-pick：

```bash
# 列出 esp32p4 相关 commits
git log upstream-apache/master --oneline -- arch/risc-v/src/esp32p4/ | head -20

# 按时间顺序 cherry-pick（从最早到最新）
git cherry-pick <commit-hash-1>
git cherry-pick <commit-hash-2>
# ... 逐个 cherry-pick，解决冲突
```

如果 commits 过多且冲突严重，可考虑：
```bash
# 直接从 upstream 拷贝整个目录
git checkout upstream-apache/master -- arch/risc-v/src/esp32p4/
git add arch/risc-v/src/esp32p4/
git commit -m "feat(arch): add ESP32-P4 chip support from upstream NuttX"
```

- [ ] **Step 5: 应用本地增量修改**

将本地 openvela 适配的额外文件（MIPI CSI/DSI 等）复制过来：

```bash
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/nuttx/arch/risc-v/src/esp32p4/esp_mipi_*.c arch/risc-v/src/esp32p4/
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/nuttx/arch/risc-v/src/esp32p4/esp_mipi_*.h arch/risc-v/src/esp32p4/
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/nuttx/arch/risc-v/src/esp32p4/esp_camera*.c arch/risc-v/src/esp32p4/
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/nuttx/arch/risc-v/src/esp32p4/esp_camera*.h arch/risc-v/src/esp32p4/
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/nuttx/arch/risc-v/src/esp32p4/esp_cam_sensor*.c arch/risc-v/src/esp32p4/
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/nuttx/arch/risc-v/src/esp32p4/esp_cam_sensor*.h arch/risc-v/src/esp32p4/

git add -A
git commit -m "feat(esp32p4): add MIPI CSI/DSI and camera driver support"
```

- [ ] **Step 6: 应用 esp-hal-3rdparty 补丁**

确保 esp-hal-3rdparty submodule 指向正确版本并包含补丁：

```bash
cd arch/risc-v/src/esp32p4/esp-hal-3rdparty
git checkout b90b1837cb5ad24747deb4c895246037cc206ce5
git apply /data/work/work/workspace/openvela_workspace/esp32-p4/patches/esp-hal-3rdparty.patch
cd -
git add -A
git commit -m "fix(esp32p4): apply esp-hal-3rdparty patches for openvela compat

- GCC15 ATOMIC_VAR_INIT compatibility
- nxsched_usleep -> nxsig_usleep
- PMP region protection for LP peripherals
- PSRAM related fixes"
```

- [ ] **Step 7: Push 并创建 PR**

```bash
git push -u origin feat/esp32p4-arch-support
```

在 GitHub 上向 `open-vela/nuttx` 的 `dev-ai-contest-2026` 分支创建 PR：
- 标题: `feat(arch): add ESP32-P4 RISC-V chip support`
- 描述: 说明来源（apache/nuttx cherry-pick + openvela 适配增量）

---

## Task 4: 提交 NuttX 公共仓 PR（板级代码）

**Files:**
- 添加: `boards/risc-v/esp32p4/common/`
- 添加: `boards/risc-v/esp32p4/esp32p4-function-ev-board/`

- [ ] **Step 1: 在同一 fork 创建新分支（或合并到同一 PR）**

```bash
cd /tmp/openvela-nuttx-fork
git checkout feat/esp32p4-arch-support
# 可以在同一分支继续，或新建分支
```

- [ ] **Step 2: 从 upstream 获取板级代码**

```bash
git checkout upstream-apache/master -- boards/risc-v/esp32p4/common/
git checkout upstream-apache/master -- boards/risc-v/esp32p4/esp32p4-function-ev-board/
git add boards/risc-v/esp32p4/
git commit -m "feat(boards): add ESP32-P4 function EV board from upstream"
```

- [ ] **Step 3: 应用本地板级增量修改**

将本地 openvela 适配的增量文件覆盖：

```bash
# appinit.c（openvela 新增）
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/nuttx/boards/risc-v/esp32p4/esp32p4-function-ev-board/src/esp32p4_appinit.c \
   boards/risc-v/esp32p4/esp32p4-function-ev-board/src/

# fb0 stub（openvela 新增）
cp /data/work/work/workspace/openvela_workspace/esp32-p4/openvela/nuttx/boards/risc-v/esp32p4/esp32p4-function-ev-board/src/esp32p4_fb0_stub.c \
   boards/risc-v/esp32p4/esp32p4-function-ev-board/src/

git add -A
git commit -m "feat(esp32p4-board): add openvela-specific board sources

- Add board_app_initialize for openvela init flow
- Add framebuffer stub for MIPI-DSI display"
```

- [ ] **Step 4: Push 并创建 PR（如果单独分支）**

如果与 Task 3 在同一分支，直接 push 即可（同一个 PR 包含芯片层 + 板级代码）。

```bash
git push origin feat/esp32p4-arch-support
```

---

## Task 5: 验证专属仓构建

**Files:**
- 无新文件，验证整体构建流程

- [ ] **Step 1: 在 openvela 工作区验证 repo sync 后 linkfile 生效**

```bash
cd /data/work/work/workspace/openvela_workspace
ls -la vendor/espressif/boards/esp32p4/esp32p4-function-ev-board
```

Expected: 该路径是一个 symlink 指向 `contest2026_174_lvmoushouhuzhe/board/esp32p4-function-ev-board`。

注意：如果 repo sync 尚未执行，需要手动创建 symlink 验证：

```bash
mkdir -p vendor/espressif/boards/esp32p4
ln -sf ../../../../contest2026_174_lvmoushouhuzhe/board/esp32p4-function-ev-board \
       vendor/espressif/boards/esp32p4/esp32p4-function-ev-board
```

- [ ] **Step 2: 确认芯片层代码就位**

确认 `nuttx/arch/risc-v/src/esp32p4/` 目录存在且包含完整代码。
（如果公共仓 PR 尚未合入，临时使用本地已有的代码）

- [ ] **Step 3: 执行构建**

```bash
cd /data/work/work/workspace/openvela_workspace
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela --cmake -j8
```

Expected: 构建成功，产出 `vela_ap.bin` 或 `nuttx.bin`。

- [ ] **Step 4: 烧录验证**

```bash
# 使用 esptool 或 eim-cli 烧录
esptool.py --chip esp32p4 write_flash 0x0 <output-binary>
```

连接串口/USB，验证出现 `nsh>` 提示符。

- [ ] **Step 5: 基础功能验证**

在 NSH 中执行：
```
nsh> help
nsh> uname -a
nsh> free
nsh> ps
```

Expected: 所有命令正常输出，系统稳定运行。

---

## Task 6: L1 基础外设适配

**Files:**
- 修改: `board/esp32p4-function-ev-board/configs/openvela/defconfig`

- [ ] **Step 1: 在 defconfig 中启用 L1 外设**

在 `board/esp32p4-function-ev-board/configs/openvela/defconfig` 追加以下配置：

```kconfig
# GPIO
CONFIG_DEV_GPIO=y

# Timer
CONFIG_TIMER=y

# Watchdog
CONFIG_WATCHDOG=y
CONFIG_ESPRESSIF_MWDT0=y

# I2C
CONFIG_I2C=y
CONFIG_I2C_DRIVER=y
CONFIG_SYSTEM_I2CTOOL=y

# SPI
CONFIG_SPI=y
CONFIG_ESPRESSIF_SPI2=y

# RTC
CONFIG_RTC=y
CONFIG_RTC_DRIVER=y
```

注意：实际操作应通过 `menuconfig` 启用后 `savedefconfig`，确保依赖项正确。

```bash
cd /data/work/work/workspace/openvela_workspace
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela menuconfig
# 在 menuconfig 中启用上述选项
# 保存退出后：
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela savedefconfig
```

- [ ] **Step 2: 编译验证**

```bash
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela --cmake -j8
```

Expected: 编译通过。

- [ ] **Step 3: 烧录并验证外设**

烧录后在 NSH 中验证：
```
nsh> gpio -h           # GPIO 帮助
nsh> i2c bus           # I2C 总线列表
nsh> i2c dev -b 0     # I2C 设备扫描
nsh> timer             # 定时器测试
```

- [ ] **Step 4: 将更新后的 defconfig 提交到专属仓**

```bash
cd /data/work/work/workspace/openvela_workspace/contest2026_174_lvmoushouhuzhe
# 确保 defconfig 已通过 savedefconfig 更新
git add board/esp32p4-function-ev-board/configs/openvela/defconfig
git commit -m "feat(defconfig): enable L1 peripherals

Enable GPIO, Timer, Watchdog, I2C, SPI, RTC drivers.
All verified working on hardware."
```

---

## Task 7: L2 高级外设适配

**Files:**
- 修改: `board/esp32p4-function-ev-board/configs/openvela/defconfig`

- [ ] **Step 1: 启用 PSRAM**

通过 menuconfig 启用：
```kconfig
CONFIG_ESP32P4_SPIRAM=y
CONFIG_ESP32P4_SPIRAM_MODE_OCT=y    # 视硬件配置
```

验证：`nsh> free` 应显示扩展后的内存容量（接近 32MB）。

- [ ] **Step 2: 启用 SPI Flash + 文件系统**

```kconfig
CONFIG_ESPRESSIF_SPIFLASH=y
CONFIG_FS_FAT=y
CONFIG_FAT_LFN=y
CONFIG_FS_LITTLEFS=y
```

验证：`nsh> mount` 显示已挂载的文件系统。

- [ ] **Step 3: 启用 Ethernet**

```kconfig
CONFIG_ESPRESSIF_EMAC=y
CONFIG_NET=y
CONFIG_NET_TCP=y
CONFIG_NET_UDP=y
CONFIG_NET_ICMP=y
CONFIG_NET_ICMP_SOCKET=y
CONFIG_SYSTEM_PING=y
CONFIG_NETUTILS_DHCPC=y
```

验证：
```
nsh> ifconfig       # 查看网络接口
nsh> dhcpc eth0     # 获取 IP
nsh> ping 8.8.8.8  # 网络连通性
```

- [ ] **Step 4: 启用 MIPI-DSI LCD 显示**

```kconfig
CONFIG_VIDEO_FB=y
CONFIG_LCD=y
CONFIG_LCD_DEV=y
CONFIG_LCD_FRAMEBUFFER=y
# MIPI-DSI 相关配置（视驱动实现）
CONFIG_GRAPHICS_LVGL=y
CONFIG_EXAMPLES_LVGLDEMO=y
```

验证：烧录后屏幕显示 LVGL demo 界面。

- [ ] **Step 5: 启用 MIPI-CSI Camera**

```kconfig
CONFIG_VIDEO=y
CONFIG_VIDEO_STREAM=y
# Camera 相关配置
CONFIG_EXAMPLES_CAMERA=y
```

验证：`nsh> camera` 命令可以采集图像。

- [ ] **Step 6: 启用 LEDC/PWM 和 ADC**

```kconfig
CONFIG_ESPRESSIF_LEDC=y
CONFIG_ESPRESSIF_ADC=y
CONFIG_PWM=y
```

验证：PWM 输出波形正确，ADC 可读取模拟值。

- [ ] **Step 7: savedefconfig 并提交**

```bash
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela savedefconfig
cd contest2026_174_lvmoushouhuzhe
git add board/esp32p4-function-ev-board/configs/openvela/defconfig
git commit -m "feat(defconfig): enable L2 advanced peripherals

- PSRAM (32MB OCT mode)
- SPI Flash + FAT/LittleFS
- Ethernet with TCP/UDP/ICMP
- MIPI-DSI LCD with LVGL
- MIPI-CSI Camera
- LEDC/PWM and ADC"
```

---

## Task 8: L3 全功能适配

**Files:**
- 修改: `board/esp32p4-function-ev-board/configs/openvela/defconfig`

- [ ] **Step 1: 启用 Wi-Fi（via ESP32-C6 桥接）**

Wi-Fi 通过板载 ESP32-C6-MINI-1 提供，通过 SPI/SDIO 与 P4 通信：

```kconfig
# Wi-Fi hosted（通过 ESP32-C6）
CONFIG_DRIVERS_IEEE80211=y
CONFIG_DRIVERS_WIRELESS=y
CONFIG_ESP32P4_WIFI=y   # 视实际 Kconfig 名称
CONFIG_WIRELESS_WAPI=y
CONFIG_WIRELESS_WAPI_CMDTOOL=y
CONFIG_NETUTILS_DHCPC=y
```

验证：
```
nsh> wapi scan wlan0
nsh> wapi psk wlan0 "your-ssid" "your-password"
nsh> dhcpc wlan0
nsh> ping 8.8.8.8
```

- [ ] **Step 2: 启用 BLE（via ESP32-C6）**

```kconfig
CONFIG_BLUETOOTH=y
CONFIG_DRIVERS_BLUETOOTH=y
CONFIG_NET_BLUETOOTH=y
CONFIG_WIRELESS_BLUETOOTH=y
CONFIG_BTSAK=y
```

验证：
```
nsh> btsak scan start
nsh> btsak scan get
```

- [ ] **Step 3: 启用 USB 2.0**

ESP32-P4 有原生 USB 2.0 OTG，除了当前的 USB Serial Console，还可以启用 USB Host/Device：

```kconfig
# USB Host（如果需要）
CONFIG_USBHOST=y
CONFIG_USBHOST_MSC=y
# USB Device（如果需要额外功能）
CONFIG_USBDEV=y
```

验证：插入 USB 设备后检测到枚举。

- [ ] **Step 4: 启用 TWAI (CAN Bus)**

```kconfig
CONFIG_ESPRESSIF_TWAI0=y
CONFIG_CAN=y
```

验证：CAN 总线收发正常。

- [ ] **Step 5: 启用触摸屏输入**

```kconfig
CONFIG_INPUT=y
CONFIG_INPUT_TOUCHSCREEN=y
```

验证：触摸屏触摸事件能正确上报。

- [ ] **Step 6: savedefconfig 并提交**

```bash
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela savedefconfig
cd contest2026_174_lvmoushouhuzhe
git add board/esp32p4-function-ev-board/configs/openvela/defconfig
git commit -m "feat(defconfig): enable L3 full features

- Wi-Fi via ESP32-C6 bridge
- BLE via ESP32-C6
- USB 2.0 Host/Device
- TWAI (CAN Bus)
- Touchscreen input"
```

---

## Task 9: 编写适配指南文档

**Files:**
- 创建: `docs/porting_guide.md`

- [ ] **Step 1: 创建适配复现指南**

创建 `docs/porting_guide.md`，内容包括：

1. 环境准备（工具链、依赖）
2. 获取代码（repo init/sync 命令）
3. 公共仓 PR 依赖说明（如何手动 cherry-pick）
4. 构建步骤（完整命令）
5. 烧录方法（esptool / eim-cli）
6. 验证步骤（NSH 命令验证各外设）
7. 已知问题与解决方案
8. esp-hal-3rdparty 补丁说明

- [ ] **Step 2: Commit**

```bash
cd contest2026_174_lvmoushouhuzhe
git add docs/porting_guide.md
git commit -m "docs: add complete porting guide for ESP32-P4

Includes environment setup, build steps, flash instructions,
peripheral verification commands, and known issues."
```

---

## Task 10: 更新作品 README

**Files:**
- 修改: `README.md`

- [ ] **Step 1: 替换模板 README 为作品说明**

将 `README.md` 替换为按比赛要求的格式：

```markdown
# ESP32-P4X-Function-EV-Board OpenVela 适配

## 一、作品简介

将 openvela 操作系统完整适配到 ESP32-P4X-Function-EV-Board，实现从基础 BSP
到全功能外设驱动的全链路支持，覆盖 MIPI-CSI/DSI、Wi-Fi、BLE、USB 2.0 等
高级外设，展示 openvela 在高性能 RISC-V AIoT 平台上的完整能力。

## 二、选题方向

新硬件适配 — ESP32-P4 是 openvela 待适配的 RISC-V 架构平台，属于重点加分方向。

## 三、目录结构

- `board/esp32p4-function-ev-board/` — vendor 板级适配（defconfig、构建配置）
- `quickapp/hello_quickapp/` — UI 应用（后续开发）
- `docs/` — 设计文档、适配指南
- `logs/` — AI Coding 日志

## 四、运行方式

详见 [docs/porting_guide.md](docs/porting_guide.md)

## 五、AI Coding 使用说明

本作品全程借助 AI 辅助开发，涵盖方案设计、代码适配、调试定位、文档编写等环节。
完整对话日志见 `logs/` 目录。
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: replace template README with project description"
```

---

## Task 11: 最终检查与提交

**Files:** 无新文件

- [ ] **Step 1: 检查仓库清洁度**

```bash
cd contest2026_174_lvmoushouhuzhe
git status           # 确认无未提交文件
git log --oneline    # 确认 commit 历史清晰
```

- [ ] **Step 2: 确认 .gitignore**

```bash
cp .gitignore.example .gitignore
# 确保 logs/ 不被忽略
# 确保编译产物被忽略
git add .gitignore
git commit -m "chore: activate gitignore from example"
```

- [ ] **Step 3: Push 到专属仓并创建 PR**

```bash
git push origin <your-branch>
```

在 GitHub 上向 `contest2026_174_lvmoushouhuzhe` 主分支创建 PR 并自行合入。

- [ ] **Step 4: 验证公共仓 PR 状态**

确认 `open-vela/nuttx` 的 PR 已创建，记录 PR URL 到 README 或 porting_guide.md 中。

---

## 执行优先级

| 优先级 | Task | 说明 |
|--------|------|------|
| P0 | Task 1-2 | 仓库结构整理（可立即执行） |
| P0 | Task 3-4 | 公共仓 PR（解除构建依赖） |
| P1 | Task 5 | 构建验证（确保端到端流程通） |
| P1 | Task 6 | L1 外设（基础得分） |
| P2 | Task 7 | L2 外设（重要加分） |
| P2 | Task 8 | L3 外设（高阶加分） |
| P1 | Task 9-10 | 文档（评分必要项） |
| P0 | Task 11 | 最终提交 |

## 预计时间

| Task | 预计耗时 |
|------|----------|
| Task 1-2 | 30 分钟 |
| Task 3-4 | 2-4 小时（cherry-pick + 冲突解决） |
| Task 5 | 1 小时 |
| Task 6 | 2-3 小时 |
| Task 7 | 4-6 小时 |
| Task 8 | 6-10 小时（Wi-Fi/BLE 复杂度高） |
| Task 9-10 | 1-2 小时 |
| Task 11 | 30 分钟 |
