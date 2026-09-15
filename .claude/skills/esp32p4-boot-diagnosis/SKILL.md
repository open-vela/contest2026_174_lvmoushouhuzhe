---
name: esp32p4-boot-diagnosis
description: "ESP32-P4 boot 卡死与 panic 定位。RAMLOG 环形缓冲区挖 panic dump、gdb 读 mepc/mtval/mcause 定位 fault、boot 到哪个阶段。Use when: ESP32-P4 boot 卡死、panic、fault、崩溃、静默卡死、PSRAM init 后卡死、定位 crash。"
---

# ESP32-P4 boot 卡死与 panic 定位

## 判断 boot 卡在哪个阶段

boot 顺序（对应串口输出）：ROM bootloader → PSRAM init（`PSRAM init OK, size=33554432`）→
PSRAM memtest（`SPI SRAM memory test OK`）→ esp_bringup → NSH（`NuttShell (NSH)`）。

串口在某个阶段后静默 = 卡在下一个初始化。例如「PSRAM init OK 后静默」= 卡在 memtest 或
之后的 bringup（历史案例：esp_timer 回归导致 LP WDT 复位循环，见 m2 memory）。

## RAMLOG 挖 panic dump（串口被吞时，最有用）

`CONFIG_RAMLOG=y` + `CONFIG_RAMLOG_SYSLOG=y` 时，`_alert`/`dump_assert_info` 的完整 dump
（task 名、backtrace、全部寄存器）写进 16KB 环形缓冲 `g_sysbuffer`。即使串口/USJ 卡死也能
用 gdb dump 出来（具体命令见 esp32p4-serial-debug）。

dump 里最有价值：`task: xxx`（哪个任务崩）、`EXCEPTION: ... EPC/MTVAL/MCAUSE`、全部 32 个
通用寄存器、`backtrace`。

## gdb 定位 fault 指令

```bash
$GDB -batch -ex "file $ELF" -ex "target extended-remote :3333" -ex "monitor halt" \
     -ex "info registers mepc mtval mcause" -ex "bt" -ex "x/8i \$pc" -ex "detach" -ex "quit"
```

- **mepc** = fault 指令地址（用 addr2line 定位函数）
- **mtval** = fault 访问的地址（如 0x64、0x691 这类小地址 = NULL 指针 + 结构体偏移）
- **mcause** = 异常类型（0x38000005 = Load access fault，0x2 = Illegal instruction）

## 常见 fault 模式

- `mtval = 0x691`（= 1676 + 5）：s1 被破坏成 5 后，`1676(s1)` 访问 g_running_tasks 时 fault
  —— 见 esp32p4-interrupt-clic（ESP-HAL handler 破坏 s1）
- `mtval = 0x30200073`（mret 机器码）+ mcause=2：mret 返回 U-mode 触发 Illegal instruction
  —— return_from_exception 漏了 MPP=M-mode 强制
- `mcause = 0x38000005` + mepc 在 ROM（0x4fc0xxxx）：ROM 函数解引用 NULL，常见于 esp_timer
  等 ESP-HAL ROM 调用链

## 定位流程

1. 串口抓 boot 日志，确认卡在哪个阶段
2. OpenOCD + gdb 读 mepc/mtval/mcause
3. RAMLOG dump panic dump（拿 task 名 + backtrace + 寄存器）
4. addr2line 定位 mepc 到函数
5. 反汇编 mepc 附近找 fault 指令
6. 必要时 git log 找该行为的修复/回归历史
