---
name: esp32p4-network-stack
description: "ESP32-P4 有线网络栈与 timer/EMAC/mbedTLS 调试。esp_hr_timer_init 跳过 ROM PMP fault、EMAC link timer、mbedTLS 与 ESP-HAL sdkconfig 冲突、mimonet 四阶段自检。Use when: ESP32-P4 网络、EMAC、以太网、timer、mbedTLS、DNS、mimonet、TLS、有线网络。"
---

# ESP32-P4 网络栈与 timer/EMAC/mbedTLS

## esp_hr_timer_init 的 ROM PMP fault（必看）

`esp_hr_timer_init()` 若真调 `esp_timer_init()`，会走 ESP-HAL 的 ROM 中断分配
（`esp_intr_alloc` → `get_desc_for_int` → `esp_intr_enable`），在 `mepc=0x4fc05ebc` 解引用
NULL 触发 PMP Load fault，导致 AppBringUp panic + LP WDT 复位循环（boot 到 PSRAM init 后静默）。

**修复**：跳过 `esp_timer_init()`，`g_hr_timer_initialized = true; return OK;`（假成功）。
代价是 EMAC 的 link-check timer 不可用（`create link timer failed`），网络链路降级。

注意：这个 skip 在 commit 10d79a9b173 曾被误删（清理调试标记时），已重新恢复并详细注释。

## EMAC / 网络现状

- `esp_eth_driver_install(218): create link timer failed` 是预期（timer 被 skip）
- 后果：`eth0` 的 ioctl 配置失败（`SIOCSIFNETMASK/ADDR/DSTADDR` 全部 errno=25），DNS 解析
  `h_errno=1`，MiMo 云端链路走不通
- 要真正调通网络，需修复 ESP-HAL timer 的 ROM fault（移植 esp_intr_alloc 到 NuttX 原生中断）

## mbedTLS 与 ESP-HAL sdkconfig 冲突

ESP-HAL 3rdparty 的静态 `sdkconfig.h` 会 `#undef` 67 个 `CONFIG_MBEDTLS_*`（为 ESP-IDF 的
mbedTLS v4 准备的），编译 ESP-HAL 源文件时破坏 NuttX 的 mbedTLS v3.4 配置，导致 check_config
失败。workaround 在 `apps/crypto/mbedtls/include/mbedtls/mbedtls_config.h` 末尾加 `#undef` 掉
不需要的特性（ECJPAKE、ECDH 等）。根本修复是改 ESP-HAL fork 的 sdkconfig.h。

## mimonet 四阶段自检

`apps/examples/mimonet` 做 TCP → TLS → HTTP 四阶段连通性自检，用于验证网络栈。DNS 失败时
会停在 TLS 阶段（`gethostbyname ... h_errno=1` → `TLS connect failed`），这是网络配置问题，
非代码 bug。

## 配置要点（defconfig）

- `CONFIG_ESPRESSIF_EMAC=y` + `CONFIG_ESPRESSIF_ETH_DMA_BUFFER_SIZE=512`
- `CONFIG_CRYPTO_MBEDTLS=y` + 相关 ECDH/GCM 特性
- 网络启动：`CONFIG_NETINIT_THREAD=y`、`CONFIG_NSH_NETINIT=y`（静态 IP 或 DHCP）
