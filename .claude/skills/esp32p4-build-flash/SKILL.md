---
name: esp32p4-build-flash
description: "ESP32-P4 (OpenVela/NuttX) 固件编译与烧录。build.sh 编译、elf2image --ram-only-header、esptool 烧录、改 config 后必须全量重建。Use when: ESP32-P4 编译、build、烧录、flash、nuttx.bin、elf2image、全量重建、编译报错。"
---

# ESP32-P4 固件编译与烧录

## 编译

```bash
cd /data/work/work/workspace/openvela_workspace
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela --cmake -j8
```

产物在 `cmake_out/esp32p4-function-ev-board_openvela/nuttx`。

## 烧录（关键：手动 elf2image + --ram-only-header）

`build.sh` 末尾自动生成的 `nuttx.bin` **不带 ram-only header**，烧录后 boot 会反复 LP WDT 复位。
必须手动重新生成：

```bash
cd cmake_out/esp32p4-function-ev-board_openvela
ESPTOOL=/data/work/work/workspace/openvela_workspace/esp32-p4/esp/tools/python_env/idf6.0_py3.12_env/bin/esptool.py
$ESPTOOL --chip esp32p4 elf2image -fs 16MB -fm dio -ff 80m --ram-only-header -o nuttx_fixed.bin nuttx
$ESPTOOL -c esp32p4 -p /dev/ttyACM1 -b 921600 --before default-reset --after hard-reset \
    write-flash 0x2000 nuttx_fixed.bin
```

- 产物正常约 750KB；214KB 说明漏了 `--ram-only-header`。
- `/dev/ttyACM*` 编号烧录后会变，每次重新 `ls` 确认。

## 改 config 后必须全量重建

改过 `defconfig`（如 CONFIG_ARCH_INTERRUPTSTACK、CONFIG_ESP32P4_JPEG_ENCODER）后，
**增量编译不可靠**（.config 不重新生成，旧对象文件缓存）。全量重建：

```bash
rm -rf cmake_out/esp32p4-function-ev-board_openvela
./build.sh vendor/espressif/boards/esp32p4/esp32p4-function-ev-board/configs/openvela --cmake -j8
```

验证 config 生效：`grep CONFIG_XXX cmake_out/.../.config`（defconfig 改了但 .config 没变 = 没重新 lunch）。

## 工具链路径备忘

- esptool：`esp32-p4/esp/tools/python_env/idf6.0_py3.12_env/bin/esptool.py`
- nm/addr2line：`esp32-p4/esp/v6.0.2/esp/tools/riscv32-esp-elf/esp-15.2.0_20251204/riscv32-esp-elf/bin/`（不在 PATH）
- 构建日志：`build.sh ... 2>&1 | tail`，错误会打印 `Error: ############# build fail ##############`
