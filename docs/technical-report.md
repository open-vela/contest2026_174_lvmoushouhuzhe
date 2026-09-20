# 2026 首届 openvela AI 硬件开发者大赛 · 技术报告（终稿）

> 本文件按大赛《作品提交模板·第二节技术报告》整理，已基于最新代码同步（apps `f157001f6` / nuttx `65158c38a` / 专属仓 `967285f`），无剩余待填项。
> 正文插图均为 Mermaid 流程图，review 确认后可整体导入飞书渲染成图。

---

## 一、信息表

| 项目 | 内容 |
|------|------|
| 作品名称 | 绿盟守护者 |
| 队伍名称 | 绿盟守护者（编号 174） |
| 团队分工 | 姚靖威（独立完成：硬件适配、驱动开发、应用开发、调试、文档全部工作） |
| 选题方向 | AI 硬件产品创新 + 新硬件平台适配（组合） |

---

## 二、摘要

绿盟守护者是一款面向绿植养护的 AI 硬件产品：解决绿植缺乏全天候看护、主人因忙碌或出差错过浇水施肥时机导致绿植死亡的痛点。设备端 camera 定时拍照，经 MiMo 大模型识别绿植健康状态（缺水/缺营养/病虫害），异常时推送飞书/微信提醒。

技术上在乐鑫 ESP32-P4 上完成 OpenVela（NuttX 内核）全新硬件平台适配，打通 MIPI-DSI 显示（EK79007 1024×600 RGB565）与 MIPI-CSI 摄像头（SC2336）链路，移植 DW-GDMA，实现摄像头 DMA 直写显示缓冲的零拷贝预览达 30fps；另完成 32MB PSRAM 上电适配、ISP 白平衡、有线网络栈与 mbedTLS、MiMo 客户端。落地 openvela「图形 + 多媒体 + AI」能力。

### 产品思路（图示）

> 设备端**定时抓拍**绿植照片 → 云端 MiMo **目标识别**健康状态 → 异常时**推送飞书/微信提醒**。
> 注：当前代码 `campilot once` 为命令触发单次抓拍；「定时抓拍」为产品规划形态，可由 NuttX timer / work queue 周期调度实现。

**图 P1｜产品系统架构（端 / 云 / 用户三层）**

```mermaid
graph TB
    subgraph DEV["设备端 · ESP32-P4"]
        T["定时器<br/>周期触发"]
        CAM["SC2336 摄像头<br/>拍照"]
        JPEG["JPEG 编码"]
        APP["campilot + mimo_client"]
    end
    subgraph CLOUD["云端"]
        MIMO["MiMo 大模型<br/>vision 图像识别"]
    end
    subgraph USER["用户端"]
        FEISHU["飞书 / 微信提醒"]
        REPORT["看护报告"]
    end
    T --> CAM --> JPEG --> APP
    APP -->|"HTTP/TLS 上传图像"| MIMO
    MIMO -->|"识别结果"| APP
    APP -->|"异常推送"| FEISHU
    MIMO --> REPORT
```

**图 P2｜核心产品业务流程（定时抓拍 → 目标识别 → 异常推送）**

```mermaid
flowchart TD
    START["开始"] --> TIMER["定时器触发<br/>按预设间隔"]
    TIMER --> CAP["摄像头拍照<br/>SC2336 采集一帧"]
    CAP --> ENC["JPEG 编码压缩"]
    ENC --> UP["上传 MiMo vision<br/>HTTP/TLS + base64"]
    UP --> REC["MiMo 目标识别<br/>绿植健康状态"]
    REC --> JUDGE{"识别结果"}
    JUDGE -->|"健康"| LOG["记录看护日志"]
    JUDGE -->|"缺水"| W1["推送 缺水提醒"]
    JUDGE -->|"缺营养"| W2["推送 施肥提醒"]
    JUDGE -->|"病虫害"| W3["推送 除虫提醒"]
    LOG --> WAIT["等待下一定时周期"]
    W1 --> WAIT
    W2 --> WAIT
    W3 --> WAIT
    WAIT --> TIMER
```

**图 P3｜端云交互时序**

```mermaid
sequenceDiagram
    participant T as "定时器"
    participant D as "设备 campilot"
    participant M as "MiMo 云端"
    participant F as "飞书/微信"
    loop "每个定时周期"
        T->>D: 触发抓拍
        D->>D: camera 拍照 + JPEG 编码
        D->>M: POST v1/chat/completions (base64 图像)
        M-->>D: 识别结果 缺水/缺营养/病虫害/健康
        alt 异常
            D->>F: 推送提醒
        else 健康
            D->>D: 记录日志
        end
    end
```

**图 P4｜产品状态机**

```mermaid
stateDiagram-v2
    [*] --> IDLE: 开机
    IDLE --> CAPTURE: 定时触发
    CAPTURE --> ENCODE: 拍照完成
    ENCODE --> UPLOAD: JPEG 就绪
    UPLOAD --> RECOGNIZE: 云端识别
    RECOGNIZE --> NOTIFY: 异常
    RECOGNIZE --> IDLE: 健康 记录日志
    NOTIFY --> IDLE: 推送完成
```

**图 P5｜绿植健康目标识别分类**

```mermaid
graph LR
    IMG["绿植照片"] --> MIMO["MiMo vision 识别"]
    MIMO --> C1["健康"]
    MIMO --> C2["缺水"]
    MIMO --> C3["缺营养"]
    MIMO --> C4["病虫害"]
    C1 --> OK["记录 + 继续定时"]
    C2 --> A2["飞书提醒浇水"]
    C3 --> A3["飞书提醒施肥"]
    C4 --> A4["飞书提醒除虫"]
```

---

## 三、正文

### 3.1 绪论

**项目背景与问题定义**：openvela 是面向 AIoT 的开源操作系统，亟需扩展硬件生态；ESP32-P4 是乐鑫首款双核 RISC-V（400MHz）+ 原生 MIPI-DSI/CSI 的 SoC，此前无 openvela 支持，缺乏显示与摄像头驱动，开发者无法在该平台使用图形与多媒体能力。

**技术难点**：

1. MIPI-DSI 显示链路（DSI PHY → Host → Bridge → DMA → 面板）端到端打通
2. DW-GDMA 控制器从 ESP-IDF 到 NuttX 的移植
3. 32MB PSRAM 上电时序（MPLL LDO 寄存器直写）
4. MIPI-CSI 摄像头驱动 + ISP demosaic/白平衡
5. 摄像头 DMA 直写显示缓冲的零拷贝路径

**创新点**：

1. 全链路 RGB565 + 双缓冲，摄像头 DMA 直写显示缓冲，30fps 零拷贝预览
2. ISP CCM 静态白平衡（R 1.85× / G 1.0× / B 1.75×），无 AWB 硬件时校正 raw Bayer 绿偏

### 3.2 系统方案设计

**总体架构**（数据流）：`SC2336 摄像头 → MIPI-CSI → ISP → DW-GDMA → 显示缓冲(fb0) → MIPI-DSI → EK79007 面板`

**方案论证与选型**：选 ESP32-P4 因其原生 MIPI-DSI/CSI + 大容量 PSRAM + 双核 RISC-V，可承载摄像头预览等多媒体负载；端侧完成采集与显示，云端（MiMo）承担大模型推理，断网/弱网时本地仍可完成预览、抓帧、编码等降级功能。

**关键模块设计**：显示驱动（双缓冲 fb0）、摄像头驱动（V4L2 ring 模式）、DW-GDMA 适配层、ISP 白平衡。

**图 1｜系统总体架构（分层）**

```mermaid
graph TB
    subgraph APP["应用层 · apps/examples"]
        D1["campreview<br/>30fps 零拷贝预览"]
        D2["campilot<br/>AI 交互"]
        D3["camcap / jpegenc<br/>抓帧 / JPEG 编码"]
    end

    subgraph SYS["系统层 · OpenVela/NuttX"]
        B1["/dev/fb0<br/>framebuffer 双缓冲"]
        B2["/dev/video0<br/>V4L2 ring 模式"]
        B3["RISC-V 中断 / CLIC"]
    end

    subgraph DRV["板级驱动 · arch/risc-v/src/esp32p4"]
        C1["esp_mipi_dsi.c<br/>MIPI-DSI 显示"]
        C2["esp_mipi_csi.c<br/>MIPI-CSI 摄像头"]
        C3["esp_dw_gdma_nuttx.c<br/>DW-GDMA 移植"]
        C4["esp_irq.c<br/>中断分发修复"]
        C5["ISP 静态白平衡"]
    end

    subgraph HW["硬件 · ESP32-P4-Function-EV-Board"]
        A1["ESP32-P4 SoC<br/>双核 RISC-V 400MHz"]
        A2["PSRAM 32MB @200MHz"]
        A3["EK79007 面板<br/>MIPI-DSI 1024×600 RGB565"]
        A4["SC2336 摄像头<br/>MIPI-CSI"]
    end

    D1 --> B2
    D1 --> B1
    B2 --> C2
    B1 --> C1
    C1 --> A3
    C2 --> A4
    C3 -.DMA 直写.-> B1
    C4 --> B3
    C5 --> C2
    A1 --- A2
    D2 -.云端调用.-> CLOUD["MiMo 云端大模型"]
```

**图 2｜摄像头→显示零拷贝数据流（核心）**

```mermaid
graph LR
    S["SC2336<br/>CMOS 传感器"] -->|"RAW Bayer"| I["ISP<br/>demosaic + 白平衡"]
    I -->|"RGB565"| D["DW-GDMA<br/>DMA 直写"]
    D -->|"零 CPU 拷贝"| F0["/dev/fb0 buffer0"]
    D -.交替.-> F1["/dev/fb0 buffer1"]
    F0 --> P["MIPI-DSI<br/>EK79007 面板"]
    F1 --> P
    V["/dev/video0<br/>V4L2 ring 连续灌流"] -.驱动帧.-> D
```

### 3.3 核心算法与技术原理

**AI 算法实现**：端侧无神经网络模型，AI 能力体现在云端 MiMo 大模型接入（`mimo_client` 库 + `campilot` 应用，HTTP/TLS 鉴权，支持文本/图像/语音多模态）。MiMo 采用 HTTP Bearer token 鉴权，POST /v1/chat/completions（文本/图像多模态），端侧由 mimo_client 库 + campilot 应用封装（mbedTLS 建 TLS 连接）。

**关键机制设计**：零拷贝调度——CSI 驱动以 `V4L2_BUF_MODE_RING` 连续灌流，DW-GDMA 将帧直写显示缓冲，避免 CPU memcpy。

**openvela 系统能力运用**：落地「图形」（`/dev/fb0` framebuffer）与「多媒体」（`/dev/video0` V4L2）两项；用到 openvela 的 framebuffer、V4L2、DMA 子系统，优化建议见 3.7。

**图 3｜双缓冲切换时序**

```mermaid
sequenceDiagram
    participant CSI as "MIPI-CSI / DW-GDMA"
    participant B0 as "fb0 buffer0"
    participant B1 as "fb0 buffer1"
    participant DSI as "MIPI-DSI 面板"
    loop "每帧约 33ms @30fps"
        CSI->>B0: DMA 直写第 N 帧
        DSI->>B1: 显示第 N-1 帧
        Note over B0,B1: 双缓冲切换，读写并行不冲突
        CSI->>B1: DMA 直写第 N+1 帧
        DSI->>B0: 显示第 N 帧
    end
```

**图 4｜零拷贝 vs memcpy vs 逐像素转换（三条路径）**

```mermaid
flowchart TD
    START["摄像头帧 RGB565"] --> Q{"采用哪条路径？"}
    Q -->|"路径 A：逐像素转换"| A1["CPU RGB565 to RGB888"] --> A2["写屏"] --> A3["约 6 fps"]
    Q -->|"路径 B：memcpy"| B1["CPU memcpy 全帧"] --> B2["写屏"] --> B3["约 15 fps"]
    Q -->|"路径 C：零拷贝（当前）"| C1["DW-GDMA 直写显示缓冲"] --> C2["无 CPU 参与"] --> C3["约 30 fps"]
    style C3 fill:#c8e6c9
```

**图 5｜AI 应用链路（MiMo）**

```mermaid
graph LR
    P["campilot<br/>拍照-分析-通知"] --> MC["mimo_client 库<br/>文本 图像 语音多模态"]
    MC --> TLS["mbedTLS TLS 连接"]
    TLS --> API["MiMo 云端<br/>HTTP Bearer<br/>POST v1/chat/completions"]
    P --> FN["feishu_notify<br/>飞书 微信提醒"]
```

### 3.4 系统实现

**软件/固件架构**：NuttX 板级驱动（`esp_mipi_dsi.c`、`esp_mipi_csi.c`、DW-GDMA）+ apps 示例（`campreview` / `camcap` / `jpegenc` / `mimonet` / `campilot`）。

**图 6｜软件模块架构**

```mermaid
graph TB
    subgraph NUTTX["nuttx 仓 feat/esp32p4-support"]
        N1["esp_mipi_dsi.c"]
        N2["esp_mipi_csi.c"]
        N3["esp_dw_gdma_idf.c / _nuttx.c"]
        N4["esp_irq.c 中断修复"]
        N5["gt911_board.c 触摸 shim"]
    end
    subgraph APPS["apps 仓 dev-ai-contest-2026"]
        P1["campreview"]
        P2["camcap"]
        P3["jpegenc V4L2 M2M"]
        P4["campilot LVGL UI"]
        P5["mimo_client / feishu_notify"]
    end
    subgraph BOARD["专属仓 board/esp32p4-function-ev-board"]
        B1["defconfig"]
        B2["board_secrets.h.example"]
    end
    N1 --> P1
    N2 --> P1
    N2 --> P2
    N2 --> P3
    N5 --> P4
    P4 --> P5
    B1 -.构建配置.-> NUTTX
```

**数据流与关键流程**：开机红屏自检 → `campreview` 拉起 CSI 流 → DMA 直写 fb0 → 双缓冲切换 → 30fps。

**图 7｜Boot 启动流程**

```mermaid
stateDiagram-v2
    [*] --> PSRAM_ON: 上电复位
    PSRAM_ON --> RED_SCREEN: DSI 链路初始化
    RED_SCREEN --> NSH_READY: 内核启动完成
    NSH_READY --> PREVIEW: campreview 180
    PREVIEW --> NSH_READY: 帧数跑完退出
    note right of PSRAM_ON
        PSRAM 32MB 上电
        MPLL LDO 寄存器直写
    end note
    note right of RED_SCREEN
        面板输出纯红自检
        DSI PHY-Host-Bridge-DMA-面板全通
    end note
```

**图 8｜campreview 预览执行流程**

```mermaid
flowchart TD
    A["nsh campreview 180"] --> B["打开 /dev/video0"]
    B --> C["V4L2 设置格式<br/>1024×600 RGB565"]
    C --> D["申请 ring 缓冲<br/>V4L2_BUF_MODE_RING"]
    D --> E["启动 CSI 连续灌流"]
    E --> F["打开 /dev/fb0<br/>mmap 显示缓冲"]
    F --> G["DW-GDMA 帧直写 fb0"]
    G --> H{"帧数小于 180 ?"}
    H -->|是| G
    H -->|否| I["打印 fps 统计<br/>30.0fps 退出"]
```

**硬件设计与适配**：全新硬件平台适配（**是**）。芯片 ESP32-P4，开发板 ESP32-P4-Function-EV-Board；新开发驱动：MIPI-DSI 显示、MIPI-CSI 摄像头、DW-GDMA、GT911 触摸、JPEG 编码器（V4L2 M2M）。适配难点：DSI 时序寄存器配置、CSI ring 缓冲要求、PSRAM 上电时序、RISC-V 中断/CLIC 与 ESP-HAL ABI 适配。关键 BOM：ESP32-P4 SoC（双核 RISC-V 400MHz）+ 32MB PSRAM（200MHz，MPLL LDO 直写）+ 16MB SPI Flash + EK79007 MIPI-DSI 面板（1024×600 RGB565）+ SC2336 MIPI-CSI 摄像头。

**图 9｜MIPI-DSI 显示链路**

```mermaid
graph LR
    FB["/dev/fb0<br/>显示缓冲 1024×600 RGB565"] --> HOST["DSI Host<br/>esp_mipi_dsi.c"]
    HOST --> PHY["DSI PHY<br/>D-PHY 2-lane"]
    PHY --> BRIDGE["Bridge IC 板载"]
    BRIDGE --> DMA["DMA 传输"]
    DMA --> PANEL["EK79007 面板<br/>RGB565"]
    style PANEL fill:#ffe6cc
```

**图 10｜MIPI-CSI 采集链路**

```mermaid
graph LR
    S["SC2336<br/>I2C chip ID 0xcb3a"] -->|"RAW Bayer"| CSI["MIPI-CSI RX<br/>esp_mipi_csi.c"]
    CSI --> ISP["ISP<br/>demosaic + CCM 白平衡<br/>R1.85 G1.0 B1.75"]
    ISP -->|"RGB565"| GDMA["DW-GDMA"]
    GDMA --> VBUF["/dev/video0<br/>V4L2 ring buffer"]
    style ISP fill:#e6f2ff
```

**图 11｜RISC-V 中断/CLIC 分发流程**

```mermaid
flowchart TD
    IRQ["外设中断<br/>DSI CSI DMA JPEG"] --> TRAP["trap 进入 exception_common"]
    TRAP --> DISP["riscv_dispatch_irq"]
    DISP --> DEMUX{"esp_isr_demultiplexing<br/>ESP-HAL 中断分发"}
    DEMUX --> SAVE["保存 callee-saved s0-s5"]
    SAVE --> CALL["调用 ESP-HAL ISR"]
    CALL --> RESTORE["恢复 s0-s5"]
    RESTORE --> RET["return_from_exception<br/>MPP=M-mode / CLIC"]
    RET --> DONE["回到被中断任务"]
    style SAVE fill:#fff3cd
    style RESTORE fill:#fff3cd
```

**图 12｜ESP-HAL ABI 破坏→修复时序**

```mermaid
sequenceDiagram
    participant T as "被中断任务 C编译"
    participant H as "NuttX ISR 入口"
    participant E as "ESP-HAL ISR"
    T->>H: 中断发生
    Note over H: 标准 RISC-V ABI：s0-s5 为 callee-saved，被调方须保存恢复
    H->>E: 分发到 ESP-HAL handler
    Note over E: 问题：ESP-HAL 把 irq 号写入 s1，并破坏 s0-s5（未按 ABI 保存）
    E-->>H: 返回
    Note over H: 修复：入口保存 s0-s5，返回前恢复
    H-->>T: 返回被中断任务
    Note over T: 寄存器状态完好，不再出现 Store/AMO fault 或 Illegal instruction
```

**应用/交互端**：NSH 命令行交互 + `campreview` 实时预览。

**自定义 Skill**：已沉淀 6 个自建 Skill + 1 个路由 Agent（esp32p4-serial-debug / build-flash / boot-diagnosis / interrupt-clic / multimedia / network-stack，及 esp32p4-debugger）；开发中另使用 openvela-build、nuttx-driver-development、contest-log-collector 等 Skill。

### 3.5 系统测试与结果分析

**测试环境**：ESP32-P4-Function-EV-Board + OpenVela，ESPTool 烧录，USB-Serial/JTAG 控制台。

**功能测试**：

| 功能 | 结果 |
|------|------|
| 开机显示自检（纯红屏） | 通过 |
| `/dev/fb0` 双缓冲 | 通过（yres_virtual=1200，fblen=2457600） |
| `/dev/video0` 摄像头 | 通过（SC2336 I2C chip ID 0xcb3a） |
| campreview 预览 | 通过，30.0fps 零拷贝 |

**性能测试**（1024×600 RGB565，每帧 PSRAM 流量约 1.23MB）：

| 阶段 | fps |
|------|-----|
| RGB565→RGB888 逐像素转换 | 6.0 |
| 全链路 RGB565 memcpy | 15.0 |
| 双缓冲零拷贝（当前） | **30.0** |

**图 13｜预览性能对比（fps）**

```mermaid
xychart-beta
    title "campreview 预览性能对比"
    x-axis ["逐像素转换", "memcpy", "零拷贝双缓冲"]
    y-axis "fps" 0 --> 35
    bar [6, 15, 30]
```

> 注：`xychart-beta` 需 Mermaid 10.x+；旧渲染器可跳过本图，数据同上表。

**可靠性与稳定性**：boot 到 NSH 稳定（已修复 esp_timer 回归导致的 LP WDT 复位循环）；campreview 连续多轮 180 帧预览稳定 30.0fps；PSRAM 32MB memtest（SPI SRAM memory test）通过。

### 3.6 AI-Native 开发说明

| 指标 | 数据 |
|------|------|
| AI Coding 代码占比 | 100%（口径：全部代码由 AI 辅助生成，人类负责需求定义、代码审查、集成与真机验证） |
| 使用的 AI 工具 | Claude Code、Kiro |
| MCP 工具使用情况 | 飞书 MCP、Gerrit MCP、IDE MCP 等 |
| Skills 使用与新增情况 | 使用 openvela-build / nuttx-driver-development / contest-log-collector 等；新增 6 个自建 Skill + 1 个 Agent（esp32p4-serial-debug 等，详见 3.4） |
| Token 使用总量 | 64 个会话、累计 33808 事件（口径：logs/ 里 manifest 的 event_count 累计，含 claude-code + kiro） |

**补充**：AI 工具在驱动寄存器调试、DSI/CSI 时序排查、PSRAM 上电等环节显著提升效率；遇到的问题：AI 对硬件寄存器细节易误判，需结合寄存器 dump 与示波器人工校验。

### 3.7 总结与展望

**成果总结**：完成 ESP32-P4 上 OpenVela 全新硬件平台适配——MIPI-DSI 显示与 MIPI-CSI 摄像头链路端到端打通、DW-GDMA 移植、32MB PSRAM 上电、RISC-V 中断/CLIC 适配，实现 30fps 零拷贝预览，boot 稳定到 NSH。

**应用前景与商业价值**：可作为 openvela 在 RISC-V + 多媒体 SoC 上的参考 BSP，用于 AI 摄像头、智能门锁/门铃、工业视觉等端侧设备。目标受众：家庭绿植爱好者、办公室绿化、小型园艺商家。痛点：绿植常因主人疏忽（出差/忙碌）错过浇水、施肥、病虫害防治时机而死亡。商业模式：设备端 camera 定时拍照 → MiMo 大模型识别绿植健康状态 → 异常（缺水/缺营养/病虫害）时推送飞书/微信提醒主人，可衍生订阅看护报告与硬件销售。

**不足与未来工作**：campilot 云端链路未跑通（DNS 解析失败）；AWB 尚未闭环（当前为固定增益）；JPEG 编码与触摸未完成板上验证。

---

## 四、评审维度对照

| 评审维度（分值） | 对应章节 |
|------|------|
| 技术难度（30） | 3.2 / 3.3 / 3.4 / 3.5 |
| 产品创新性（20） | 摘要 / 3.1 |
| 项目完整度（20） | 3.5 / 源码 / 展示照片 |
| AI 开发（10） | 3.3 / 3.6 |
| 商业潜力（10） | 3.7 |
| 展示效果（10） | 演示视频 / 海报 / PPT |

---

## 附：图清单与导入飞书说明

**图清单**（共 18 张：产品思路 P1–P5 + 技术架构 1–13）：

| 编号 | 图名 | 位置 |
|------|------|------|
| P1 | 产品系统架构（端/云/用户） | 摘要·产品思路 |
| P2 | 核心产品业务流程 | 摘要·产品思路 |
| P3 | 端云交互时序 | 摘要·产品思路 |
| P4 | 产品状态机 | 摘要·产品思路 |
| P5 | 绿植健康目标识别分类 | 摘要·产品思路 |
| 1 | 系统总体架构（分层） | 3.2 |
| 2 | 零拷贝数据流（核心） | 3.2 |
| 3 | 双缓冲切换时序 | 3.3 |
| 4 | 三条路径性能对比 | 3.3 |
| 5 | AI 应用链路（MiMo） | 3.3 |
| 6 | 软件模块架构 | 3.4 |
| 7 | Boot 启动流程 | 3.4 |
| 8 | campreview 执行流程 | 3.4 |
| 9 | MIPI-DSI 显示链路 | 3.4 |
| 10 | MIPI-CSI 采集链路 | 3.4 |
| 11 | RISC-V 中断/CLIC 分发 | 3.4 |
| 12 | ESP-HAL ABI 修复时序 | 3.4 |
| 13 | 预览性能对比（fps） | 3.5 |

**导入飞书步骤**：确认本稿无误后，在飞书技术报告文档中逐段粘贴，Mermaid 代码块语言选 `Mermaid` 即自动渲染成图。
