---
name: esp32p4-serial-debug
description: "ESP32-P4 (OpenVela/NuttX) 板级串口调试与故障定位。termios+select 直读 USB-Serial-JTAG 避开 pyserial 阻塞、esptool 复位代替 monitor reset halt 避免断线、OpenOCD+gdb 读寄存器、RAMLOG 挖 panic dump。Use when: ESP32-P4 串口、USB-JTAG 断线、串口无响应、boot 卡死、campreview、panic 定位、读取 RAMLOG、esp32p4 serial debug、烧录 nuttx。"
---

# ESP32-P4 板级串口调试

ESP32-P4-Function-EV-Board 在 OpenVela/NuttX 下调试的踩坑经验，全部经过板上实测。

## 核心结论（先看这个）

1. **串口是 USB-Serial-JTAG（303a:1001）**，设备号 `/dev/ttyACM*`，复位后编号会变（ACM0→1→2）。
2. **pyserial 的 `serial.Serial(port, baud)` 默认 DTR 握手会阻塞**，必须 `dsrdtr=False`，或直接用 termios+select（见下）。
3. **`monitor reset halt` 几乎必然断开 USB-JTAG**（libusb NO_DEVICE），复位用 esptool 代替。
4. **gdb 连接会反复报 `vMustReplyEmpty` / `GDB missing ack`**，是 USB-JTAG 状态不稳，重启 OpenOCD 或重新插拔 USB 可恢复。
5. **panic 的完整信息（task 名、backtrace、全部寄存器）在 RAMLOG 环形缓冲区里**，即使串口被吞也能用 gdb dump 出来。

## 串口捕获（避开 pyserial 阻塞）

用 termios + select + fcntl 非阻塞，不要用 pyserial 的 `read()`：

```python
import os, termios, select, time, fcntl

fd = os.open('/dev/ttyACM1', os.O_RDWR | os.O_NOCTTY)
flags = fcntl.fcntl(fd, fcntl.F_GETFL)
fcntl.fcntl(fd, fcntl.F_SETFL, flags | os.O_NONBLOCK)
t = termios.tcgetattr(fd)
t[0]=0; t[1]=0; t[2]=termios.CS8|termios.CREAD|termios.CLOCAL; t[3]=0
t[4]=termios.B115200; t[5]=termios.B115200
t[6][termios.VMIN]=0; t[6][termios.VTIME]=0   # 关键：非阻塞读
termios.tcsetattr(fd, termios.TCSANOW, t)

def drain(secs):
    end = time.time() + secs
    while time.time() < end:
        r,_,_ = select.select([fd], [], [], 0.2)
        if r:
            try: c = os.read(fd, 8192)
            except BlockingIOError: continue
            sys.stdout.write(c.decode('utf-8','replace')); sys.stdout.flush()
```

## USB-JTAG 断线处理

- **触发**：`monitor reset halt`、反复重启 OpenOCD、gdb 反复连接，都会让 USB-JTAG 断线
  （`ls /dev/ttyACM*` 消失或编号变）。
- **恢复**：`pkill -9 -f openocd-esp32` → 等 USB 重新枚举（轮询 `ls /dev/ttyACM*`）→ 重启 OpenOCD。
- **复位用 esptool**（比 monitor reset halt 稳）：
  ```bash
  ESPTOOL=/data/work/work/workspace/openvela_workspace/esp32-p4/esp/tools/python_env/idf6.0_py3.12_env/bin/esptool.py
  $ESPTOOL -c esp32p4 -p /dev/ttyACM1 --before default-reset --after hard-reset chip_id
  ```

## OpenOCD + gdb（读寄存器 / 反汇编）

```bash
OCD=/data/work/work/workspace/openvela_workspace/esp32-p4/esp/v6.0.2/esp/tools/openocd-esp32/v0.12.0-esp32-20260424/openocd-esp32
$OCD/bin/openocd -s $OCD/share/openocd/scripts -f board/esp32p4-builtin.cfg &   # gdb server :3333

GDB=/data/work/work/workspace/openvela_workspace/esp32-p4/esp/tools/tools/riscv32-esp-elf-gdb/17.1_20260402/riscv32-esp-elf-gdb/bin/riscv32-esp-elf-gdb
ELF=cmake_out/esp32p4-function-ev-board_openvela/nuttx
$GDB -batch -ex "set pagination off" -ex "file $ELF" \
     -ex "target extended-remote :3333" -ex "monitor halt" \
     -ex "info registers mepc mtval mcause" -ex "bt" -ex "detach" -ex "quit"
```

**读寄存器定位 fault 的关键**：`mepc`（fault 指令地址）、`mtval`（fault 访问地址）、
`mcause`（异常类型）。这些 CSR 在 panic 死循环里仍然保留，可直接读。

**addr2line/nm 路径**（不在 PATH）：
`/data/work/work/workspace/openvela_workspace/esp32-p4/esp/v6.0.2/esp/tools/riscv32-esp-elf/esp-15.2.0_20251204/riscv32-esp-elf/bin/`

## RAMLOG 挖 panic dump（串口被吞时）

`CONFIG_RAMLOG=y` + `CONFIG_RAMLOG_SYSLOG=y` 时，`_alert`/`dump_assert_info` 的完整 panic
dump 写进 16KB 环形缓冲区 `g_sysbuffer`。即使串口/USJ 卡死，也能用 gdb dump 出来：

```bash
# g_sysbuffer 地址（用 nm 查，可能变）
$GDB -batch -ex "file $ELF" -ex "target extended-remote :3333" -ex "monitor halt" \
     -ex "dump binary memory /tmp/ramlog.bin 0x4ff51eb8 0x4ff55eb8" -ex "detach" -ex "quit"
strings -n 4 /tmp/ramlog.bin   # 看 EXCEPTION / backtrace / task: / 寄存器 dump
```

panic dump 里最有价值的是：`task: xxx`（哪个任务崩）、`EPC/MTVAL/MCAUSE`、全部 32 个
通用寄存器、`backtrace`。

## esptool 烧录

```bash
# 必须手动 elf2image + --ram-only-header（build.sh 自动生成的不带 ram-only header）
$ESPTOOL --chip esp32p4 elf2image -fs 16MB -fm dio -ff 80m --ram-only-header -o nuttx.bin nuttx
$ESPTOOL -c esp32p4 -p /dev/ttyACM1 -b 921600 --before default-reset --after hard-reset \
    write-flash 0x2000 nuttx.bin
```

## 已知固件坑（详情见 memory m1/m2）

- **ESP-HAL 中断 handler 不遵循 RISC-V ABI**：把 irq 号存 s1 并破坏 s0-s5（callee-saved），
  导致 `riscv_doirq` 访问 g_running_tasks 时 fault。NuttX 侧已打补丁（保存/恢复 s0-s5），
  根本修复需重编译 ESP-HAL 用标准 ABI。
- **timer/EMAC**：跳过 `esp_timer_init`（避免 ROM PMP fault）导致 EMAC link timer 不可用、
  网络不通（DNS 失败是预期）。
