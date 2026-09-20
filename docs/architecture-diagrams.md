# ESP32-P4 OpenVela 作品 · 架构与流程图（芯片适配 + 产品思路）

> 配套《技术报告》正文，聚焦**新硬件平台适配**主线，补充**产品思路**业务流程。所有图用 Mermaid 绘制，
> 可直接整体复制进飞书文档（```mermaid 代码块）或本地渲染。
> 代码基线：nuttx `65158c38a` / apps `f157001f6` / 专属仓 `967285f`。

## 图目录

| # | 图名 | 类型 | 对应章节 |
|---|------|------|----------|
| 1 | 系统总体架构（分层） | graph | 3.2 |
| 2 | 摄像头→显示零拷贝数据流（核心） | graph | 3.2 / 3.3 |
| 3 | MIPI-DSI 显示链路 | graph | 3.1 / 3.4 |
| 4 | MIPI-CSI 采集链路 | graph | 3.1 / 3.4 |
| 5 | 双缓冲切换时序 | sequence | 3.3 |
| 6 | 零拷贝 vs memcpy vs 逐像素（三条路径） | flowchart | 3.3 / 3.5 |
| 7 | 预览性能对比（fps） | xychart | 3.5 |
| 8 | Boot 启动流程 | state | 3.4 / 3.5 |
| 9 | campreview 预览执行流程 | flowchart | 3.4 |
| 10 | RISC-V 中断/CLIC 分发 | flowchart | 3.4 |
| 11 | ESP-HAL ABI 破坏→修复时序 | sequence | 3.4 |
| 12 | 软件模块架构 | graph | 3.4 |
| 13 | AI 应用链路（MiMo） | graph | 3.3 |
| 14 | 产品系统架构（端/云/用户三层） | graph | 产品思路 |
| 15 | 核心产品业务流程（定时抓拍→识别→推送） | flowchart | 产品思路 |
| 16 | 端云交互时序 | sequence | 产品思路 |
| 17 | 产品状态机 | state | 产品思路 |
| 18 | 绿植健康目标识别分类 | graph | 产品思路 |

---

## 图 1｜系统总体架构（分层）

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

---

## 图 2｜摄像头→显示零拷贝数据流（核心）

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

---

## 图 3｜MIPI-DSI 显示链路

```mermaid
graph LR
    FB["/dev/fb0<br/>显示缓冲 1024×600 RGB565"] --> HOST["DSI Host<br/>esp_mipi_dsi.c"]
    HOST --> PHY["DSI PHY<br/>D-PHY 2-lane"]
    PHY --> BRIDGE["Bridge IC 板载"]
    BRIDGE --> DMA["DMA 传输"]
    DMA --> PANEL["EK79007 面板<br/>RGB565"]
    style PANEL fill:#ffe6cc
```

---

## 图 4｜MIPI-CSI 采集链路

```mermaid
graph LR
    S["SC2336<br/>I2C chip ID 0xcb3a"] -->|"RAW Bayer"| CSI["MIPI-CSI RX<br/>esp_mipi_csi.c"]
    CSI --> ISP["ISP<br/>demosaic + CCM 白平衡<br/>R1.85 G1.0 B1.75"]
    ISP -->|"RGB565"| GDMA["DW-GDMA"]
    GDMA --> VBUF["/dev/video0<br/>V4L2 ring buffer"]
    style ISP fill:#e6f2ff
```

---

## 图 5｜双缓冲切换时序

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

---

## 图 6｜零拷贝 vs memcpy vs 逐像素转换（三条路径）

```mermaid
flowchart TD
    START["摄像头帧 RGB565"] --> Q{"采用哪条路径？"}
    Q -->|"路径 A：逐像素转换"| A1["CPU RGB565 to RGB888"] --> A2["写屏"] --> A3["约 6 fps"]
    Q -->|"路径 B：memcpy"| B1["CPU memcpy 全帧"] --> B2["写屏"] --> B3["约 15 fps"]
    Q -->|"路径 C：零拷贝（当前）"| C1["DW-GDMA 直写显示缓冲"] --> C2["无 CPU 参与"] --> C3["约 30 fps"]
    style C3 fill:#c8e6c9
```

---

## 图 7｜预览性能对比（fps）

```mermaid
xychart-beta
    title "campreview 预览性能对比"
    x-axis ["逐像素转换", "memcpy", "零拷贝双缓冲"]
    y-axis "fps" 0 --> 35
    bar [6, 15, 30]
```

> 注：`xychart-beta` 需 Mermaid 10.x+；旧渲染器可跳过本图，数据同图 6。

---

## 图 8｜Boot 启动流程

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

---

## 图 9｜campreview 预览执行流程

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

---

## 图 10｜RISC-V 中断/CLIC 分发流程

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

---

## 图 11｜ESP-HAL ABI 破坏→修复时序

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

---

## 图 12｜软件模块架构

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

---

## 图 13｜AI 应用链路（MiMo，应用层辅助）

```mermaid
graph LR
    P["campilot<br/>拍照-分析-通知"] --> MC["mimo_client 库<br/>文本 图像 语音多模态"]
    MC --> TLS["mbedTLS TLS 连接"]
    TLS --> API["MiMo 云端<br/>HTTP Bearer<br/>POST v1/chat/completions"]
    P --> FN["feishu_notify<br/>飞书 微信提醒"]
```

---

## 第二部分 · 产品思路流程图（业务层）

> 作品「绿盟守护者」的产品业务逻辑：绿植养护 AI 看护。
> 设备端**定时抓拍**绿植照片 → 云端 MiMo **目标识别**健康状态 → 异常时**推送飞书/微信提醒**。
>
> 注：当前代码 `campilot once` 为命令触发单次抓拍；「定时抓拍」为产品规划形态，
> 可由 NuttX timer / work queue 周期调度实现。

---

### 图 14｜产品系统架构（端 / 云 / 用户三层）

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

---

### 图 15｜核心产品业务流程（定时抓拍 → 目标识别 → 异常推送）

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

---

### 图 16｜端云交互时序

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

---

### 图 17｜产品状态机

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

---

### 图 18｜绿植健康目标识别分类

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

## 附：如何导入飞书

1. 在飞书技术报告文档中，光标定位到目标位置；
2. 插入 `代码块`，语言选 `Mermaid`（或直接粘贴 ```mermaid 围栏代码）；
3. 飞书会自动渲染为图（也可转成可编辑画板）。
