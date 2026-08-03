# ESP32-P4 功能验证指南

针对 ESP32-P4-Function-EV-Board + OpenVela 的本地测试步骤。

## 环境变量

后面的命令都用到这两个路径：

```bash
export WS=/data/work/work/workspace/openvela_workspace
export ESPTOOL=$WS/esp32-p4/esp/tools/python_env/idf6.0_py3.12_env/bin/esptool.py
```

## 构建与烧录

```bash
cd $WS
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela --cmake -j8

cd cmake_out/esp32p4-function-ev-board_openvela
$ESPTOOL --chip esp32p4 elf2image -fs 16MB -fm dio -ff 80m --ram-only-header -o nuttx.bin nuttx
$ESPTOOL -c esp32p4 -p /dev/ttyACM0 -b 921600 --before default-reset --after hard-reset \
    write-flash 0x2000 nuttx.bin
```

两个必须注意的点：

- `build.sh` 末尾的 `elf2image` 步骤**会报错**，这是已知问题（脚本内部调用），手动执行上面那条即可。
- `--ram-only-header` **不能省**。少了它 ROM bootloader 无法正确加载，表现为反复 LP_WDT 复位（USB 每 3 秒断开重连）。产物正常约 300KB；如果是 214KB 说明漏了这个参数，500MB+ 说明误用了 `objcopy`。

改过 `sdkconfig.h` 或 HAL 头文件后**必须全量重建**，增量编译不可靠：

```bash
rm -rf $WS/cmake_out/esp32p4-function-ev-board_openvela
```

## 串口连接

USB CDC 在主机未拉 DTR 时不输出，所以 `cat /dev/ttyACM0` **无效**，必须用支持 DTR 的终端。

脚本方式（适合自动化）：

```bash
cd $WS
./tools/esp32p4-serial.sh alive                    # 存活检测，exit 0=活
./tools/esp32p4-serial.sh nsh "ls /dev"            # 执行命令并取回输出
./tools/esp32p4-serial.sh nsh "xd 0x500A1008 4"    # 读寄存器
./tools/esp32p4-serial.sh read 5                   # 读 5 秒输出
```

交互方式：

```bash
minicom -D /dev/ttyACM0 -b 115200
```

## 开机自检：屏幕应为纯红

启动时 `esp_mipi_dsi_start_refresh()` 会把 framebuffer 填成红色。

**开机看到纯红屏 = 显示链路（DSI PHY → Host → Bridge → DMA → 面板）全通。**
如果不是红屏，问题在显示而非摄像头，先不要查 camera。

摄像头启用时红蓝交替的自检线程不会启动（避免和预览抢 framebuffer）；
若要单独验证显示，关掉 `CONFIG_ESP32P4_CAMERA` 重新构建即可看到红蓝交替。

## 设备节点检查

```bash
nsh> ls /dev
```

应能看到：

| 节点 | 说明 |
|------|------|
| `fb0` | 显示，RGB565 1024×600，双缓冲 |
| `video0` | 摄像头，V4L2，RGB565 1024×600 |

`video0` 存在即说明 SC2336 的 I2C 探测（chip ID 0xcb3a）已通过。

## 摄像头预览

```bash
nsh> campreview          # 连续预览，Ctrl-C 退出
nsh> campreview 180      # 跑 180 帧（约 6 秒）后自动退出
nsh> campreview -h       # 用法
```

正常输出：

```
campreview: fb 1024x600 fmt=11 bpp=16 stride=2048 fblen=2457600 yres_virtual=1200
campreview: preview 1024x600 RGB565 -> RGB565, zero-copy, limited run
campreview: 60 frames, 29.7 fps
campreview: 120 frames, 30.0 fps
```

两个关键指标：

- **`zero-copy`** — 摄像头 DMA 直接写显示缓冲。若显示 `copy`，说明 fb0 没暴露双缓冲
  （`fblen` 应为 2457600、`yres_virtual` 应为 1200），会退化到约 15fps。
- **`30.0 fps`** — sensor 满帧率。明显偏低说明存在瓶颈。

## 性能参考

优化过程的实测数据，用于判断当前是否正常：

| 阶段 | fps | 每帧 PSRAM 流量 |
|------|-----|----------------|
| RGB565→RGB888 逐像素转换 | 6.0 | 读 1.23 + 写 1.84 + msync 1.84 MB |
| 全链路 RGB565，memcpy | 15.0 | 读 1.23 + 写 1.23 + msync 1.23 MB |
| 双缓冲零拷贝（当前） | **30.0** | 仅 msync 1.23 MB |

## 串口无响应的处理

`alive` 报 `DEAD` 时，按顺序试：

1. **进程占用**（OpenOCD 或残留的 python 脚本）：
   ```bash
   fuser -k /dev/ttyACM0; pkill -f openocd; sleep 3
   ```
2. **USB 需重新枚举**（用过 OpenOCD 的 JTAG 通道后常见）—— 重新烧录即可恢复：
   ```bash
   cd $WS/cmake_out/esp32p4-function-ev-board_openvela
   $ESPTOOL -c esp32p4 -p /dev/ttyACM0 -b 921600 --before default-reset \
       --after hard-reset write-flash 0x2000 nuttx.bin
   ```
3. **确认设备号**：烧录后 ttyACM 编号可能变化，`ls /dev/ttyACM*` 确认。

注意 OpenOCD 与烧录不能同时占用 USB，烧录前先关掉 OpenOCD。

## 寄存器速查（排查用）

用 `xd <addr> <len>` 直接读，比起 OpenOCD 更方便且是运行态真值。

显示：

| 地址 | 寄存器 | 期望值 |
|------|--------|--------|
| 0x500A0068 | CMD_MODE_CFG | `0x010F7F02`（DCS 走 LP 模式，面板唤醒的关键） |
| 0x500A0010 | DPI_COLOR_CODING | `0`（RGB565 16-bit config1） |
| 0x500A0034 | MODE_CFG | `0`（video mode） |
| 0x500A0038 | VID_MODE_CFG | `0xFF02` |
| 0x500A080C | 桥 raw_num_total | `0x25800`（153600） |
| 0x500A0804 | 桥 EN | `1` |
| 0x500A0858 | 桥 INT_RAW | bit0=0 表示无 underrun |

摄像头 / ISP：

| 地址 | 寄存器 | 期望值 |
|------|--------|--------|
| 0x500A1008 | ISP_CNTL | `0x80002543`（含 CCM_EN bit8） |
| 0x500A1014 | ISP_CCM_COEF0 | `0x1D9` = 473 = 1.85×256（R 增益） |
| 0x500A101C | ISP_CCM_COEF3 | `0x100` = 256 = 1.0（G 增益） |
| 0x50081018 | DW-GDMA CHEN | bit0=camera，bit1=display |

**`PHY_STATUS`(0x500A00B0) 不能用来判断显示是否正常** —— burst 模式下 lane 在帧间
本来就回到 LP-11，正常显示时也会读到 `0x15B9`。它只能用来确认 PLL lock（bit0）。

## 画面颜色调整

摄像头经 ISP 做 RAW8(BGGR) → demosaic → RGB565。raw Bayer 未做白平衡时天然绿偏
（绿像素占一半，且硅的量子效率在绿光最高），所以 ISP 里用 CCM 对角线加了静态白平衡增益。

偏色时改 `nuttx/arch/risc-v/src/esp32p4/esp_mipi_csi.c`：

```c
#define ISP_WB_GAIN_R  (ISP_CCM_ONE * 185 / 100)  /* 1.85x */
#define ISP_WB_GAIN_G  (ISP_CCM_ONE)              /* 1.00x */
#define ISP_WB_GAIN_B  (ISP_CCM_ONE * 175 / 100)  /* 1.75x */
```

- 偏绿 → 调高 R 和 B
- 偏红 → 调低 R
- 偏蓝/紫 → 调低 B

`ISP_CCM_ONE` = 256 表示 1.0（S4.8 定点，ESP32-P4 rev≥3）。改完需全量重建。

这是固定增益而非测量值。ISP 硬件支持真正的 AWB（`ISP_AWB_*` 加统计寄存器），
但需要实现控制环路，属于后续工作。
