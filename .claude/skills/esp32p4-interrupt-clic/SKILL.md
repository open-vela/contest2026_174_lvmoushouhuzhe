---
name: esp32p4-interrupt-clic
description: "ESP32-P4 CLIC 中断模式与 ESP-HAL ABI 兼容问题。CLIC 用 intthresh 屏蔽、ESP-HAL handler 不遵循 ABI 破坏 s0-s5、mret 返回 U-mode、尾调用优化。Use when: ESP32-P4 中断、CLIC、s1 破坏、callee-saved、mret、Illegal instruction、中断 handler、tail call、ESP-HAL ABI。"
---

# ESP32-P4 CLIC 中断与 ESP-HAL ABI 问题

## CLIC 中断屏蔽机制

ESP32-P4 用 CLIC（Core Local Interrupt Controller），与标准 RISC-V 不同：

- **屏蔽用 `mintthresh`（intthresh CSR）**，不是 MIE。`up_irq_save()` 用
  `SWAP_CSR(INTTHRESH, RISCV_MAX_INTTHRESH)` 屏蔽，`up_irq_restore()` 恢复。
- `up_irq_enable()` 只设 MIE=1，**不提升 intthresh**，所以 handler 期间仍允许嵌套中断。
- 中断触发条件：`level > mintthresh_level`（严格大于）。

## 最深的坑：ESP-HAL handler 不遵循 RISC-V ABI

ESP-HAL 的外设 ISR（systimer、DMA、JPEG 等，通过 `esp_intr_alloc` 注册）编译自 ESP-IDF，
**不遵循标准 RISC-V ABI**：把 irq 号存进 `s1`，并破坏 `s0-s5`（标准里是 callee-saved，必须
保存恢复）。

而 `riscv_doirq` 用 `s1` 作 `g_running_tasks` 基准（`lui s1,0x4ff4d` + `1676(s1)`），`s2/s4/s5`
作 g_readytorun/g_interrupt_context/restore_context。handler 返回后这些寄存器被破坏，下一次
访问 g_running_tasks 时 `1676(5)=0x691` fault。

**证据**（串口 debug 输出）：`[IRQDBG] irq=39 s1=0x27`（s1 = irq 号）。

**NuttX 侧 workaround**（commit 2f5c1d2742b）：`riscv_doirq_top` 里 IRQ_DISPATCH 前后用内联
汇编保存/恢复 s0-s5 到栈。**根本修复是重编译 ESP-HAL 用标准 ABI（s0-s11 callee-saved）**。

## mret 返回 U-mode（Illegal instruction）

`return_from_exception` 之前的 MIE/MPIE 修复漏了 MPP。CLIC 下 MPP 被破坏成 U-mode，mret 试图
返回 U-mode 触发 Illegal instruction（mtval=0x30200073）。补 `li t0,STATUS_PPP; or s0,s0,t0`
强制 MPP=M-mode（commit 23186c8612a）。注意 `ori s0,s0,STATUS_PPP` 非法（0x1800 超 12 位
立即数，必须 `li`+`or`）。

## 尾调用优化

`esp_isr_demultiplexing` 里 `(*handler)(handler_arg)` 被编译器尾调用优化（`jalr s0`，rd=x0），
handler 返回直接跳回 riscv_doirq，跳过 epilogue，callee-saved 寄存器不被恢复。用
`__attribute__((optimize("no-optimize-sibling-calls")))` 可靠禁用（volatile barrier 不可靠，
加 debug 代码后尾调用会重现）。
