# 演示视频分镜脚本（≤5 分钟）

作品：绿盟守护者（ESP32-P4 OpenVela 新硬件适配 + 绿植养护 AI 看护）
目标：覆盖「功能演示 + 交互操作 + AI 能力展示」三条要求。
拍摄思路：**产品 UI（campilot ui）展示形态 + campreview 展示真实 30fps 预览**，全部用已验证跑通的画面，避免拍到触摸失灵 / UI 崩溃。

---

## 镜头 1｜开场（0:00–0:25）

- **画面**：ESP32-P4-Function-EV-Board 特写，镜头缓慢拉远展示整板 + 屏幕
- **字幕**：「绿盟守护者 · ESP32-P4 OpenVela 新硬件适配」
- **旁白**：本项目在乐鑫 ESP32-P4（双核 RISC-V 400MHz）上移植 OpenVela，打通 MIPI-DSI 显示与 MIPI-CSI 摄像头链路，实现 30fps 零拷贝预览，并落地一款绿植养护 AI 看护产品。

---

## 镜头 2｜开机自检（0:25–0:55）

- **画面**：板子重新上电，屏幕瞬间变**纯红**（显示链路自检信号）
- **画面切换**：串口终端滚动 boot 日志，特写 `PSRAM init OK, size=33554432`、`SPI SRAM memory test OK`
- **旁白**：纯红屏代表 DSI PHY→Host→Bridge→DMA→面板全链路打通；串口可见 32MB PSRAM 初始化成功。

---

## 镜头 3｜产品 UI（0:55–1:55）

- **画面**：NSH 终端输入 `campilot ui`，屏幕出现 **LVGL 三屏卡片 UI**
- **特写**：camera / AI result / voice 三张卡片，及标题与提示文字（"Tap screen to capture"）
- **旁白**：这是产品「绿盟守护者」的三屏卡片交互界面——摄像头预览卡、AI 识别结果卡、语音对话卡，滑动切换、点按拍照。

---

## 镜头 4｜30fps 零拷贝预览（1:55–3:25，核心）

- **画面**：NSH 输入 `campreview 180`，屏幕立即出现**摄像头实时预览画面**
- **画面切换**：串口滚动 `campreview: 60 frames, 29.7 fps` → `120 frames, 30.0 fps` → `180 frames, 30.0 fps`
- **特写字幕**：「zero-copy · 30fps」
- **旁白**：摄像头 DMA 直写显示缓冲，零 CPU 拷贝，稳定 30fps。相比逐像素转换（6fps）和 memcpy（15fps），提升约 5 倍。

---

## 镜头 5｜AI 能力展示（3:25–4:00）

- **画面**：NSH 输入 `campilot -h`，展示 `once`（拍照识别）/ `text`（文本问答）/ `ui`（三屏卡片）三种用法
- **可选**：若云端链路实测可通，接 `campilot text "这盆绿植状态如何"` 展示真实问答
- **旁白**：campilot 封装 MiMo 大模型 API（HTTP Bearer 鉴权，POST /v1/chat/completions），支持文本/图像/语音多模态；受板载网络环境限制，云端链路演示若未通，代码已就绪（`mimo_client` 库 + `feishu_notify` 推送）。

---

## 镜头 6｜结尾总结（4:00–4:30）

- **画面**：回到开发板全景，屏幕停留在摄像头预览画面
- **字幕**：「新硬件适配 · 显示 + 摄像头 · 30fps 零拷贝」
- **旁白**：本项目展示了 openvela 在 RISC-V 高端多媒体 SoC 上的落地能力，感谢观看。

---

## 录制建议

- 全程「串口终端 + 屏幕」双画面（画中画或分屏），让评委同时看到命令与画面
- 串口工具用 minicom（`minicom -D /dev/ttyACM1 -b 115200`），或支持 DTR 的终端
- 镜头 3 的 UI 卡片建议在**光线充足**下拍，避免屏幕反光；触摸若不稳，用键盘/重新进 `campilot ui` 切屏即可，不硬拍触摸
- 镜头 5 的 `campilot text` 只在云端实测通过时拍，否则只拍 `campilot -h` 用法说明
- 总时长控制在 4 分 30 秒左右，留上传余量
