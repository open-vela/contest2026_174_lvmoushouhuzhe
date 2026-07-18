# ESP32-P4X-Function-EV-Board 适配 (openvela vendor 层)

## 概述

本目录为 ESP32-P4X-Function-EV-Board 在 openvela 上的 vendor 板级适配层。
通过本配置，可在 openvela (基于 NuttX) 系统上支持 ESP32-P4 开发板。

## 硬件规格

| 参数 | 说明 |
|------|------|
| 主控 | ESP32-P4，双核 RISC-V，主频 400MHz |
| 内存 | 32MB PSRAM |
| 显示接口 | MIPI DSI |
| 摄像头接口 | MIPI CSI |
| USB | USB 2.0 OTG |
| 无线连接 | Wi-Fi 6 + BLE 5（通过板载 ESP32-C6 模组） |
| 存储 | SD 卡插槽 |
| 音频 | I2S 音频编解码器 |

## 目录结构

```
esp32p4-function-ev-board/
├── configs/openvela/defconfig   # openvela 内核配置
├── scripts/Make.defs            # 编译脚本定义
├── CMakeLists.txt               # CMake 构建入口
├── Kconfig                      # 板级 Kconfig 选项
└── README.md                    # 本文件
```

## 构建方法

```bash
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela --cmake -j8
```

## 依赖说明

本适配依赖 NuttX 上游公共仓库中的 ESP32-P4 芯片层和板级源码，主要包括：

- **芯片层 (chip layer)**：`arch/risc-v/src/esp32p4/`
- **板级源码 (board sources)**：`boards/risc-v/esp32p4/`

这些代码需要通过向 NuttX 公共仓库提交 PR 的方式合入。在 PR 合入之前，
可通过本地补丁或 vendor overlay 的方式进行开发调试。

## 参考资料

- [ESP32-P4 技术参考手册](https://www.espressif.com/sites/default/files/documentation/esp32-p4_technical_reference_manual_en.pdf)
- [ESP32-P4-Function-EV-Board 用户指南](https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32p4/esp32-p4-function-ev-board/index.html)
- [openvela 官方文档](https://open-vela.github.io/)
