# AudioSub

基于 `WebRTC-Native` 的实时语音转写与标注系统。

A 端讲话，音频经 `WebRTC` P2P 传到 B 端；B 端用本地 `whisper.cpp` 做 `ASR` 生成字幕，并把 A 端通过 `DataChannel` 发来的标注与字幕按统一时间轴对齐融合，再将增强字幕回传给 A 端。系统提供命令行（CLI）与 `Qt` 图形界面（飞书风格聊天）两套前端，二者共用同一核心引擎。

## 功能特性

- `P0` 主链路：`WebRTC` 音频传输 + B 端本地 `ASR`（`whisper.cpp`）+ 实时字幕
- `P1` 标注通道：A 端经 `DataChannel` 发送结构化标注（`seq` / `event_time_ms` / `payload.text`），支持按 `seq` 去重
- `P1` 字幕与标注融合：统一时间轴对齐，输出「字幕正文 / 时间范围 / 对应标注」；未对齐标注单独兜底展示
- 增强项：B → A 字幕回传
- 指标观测：传输 RTT、出字延迟、标注匹配误差；CLI 退出汇总、GUI 顶栏周期性刷新
- 双前端：CLI 与 `Qt` GUI 共用核心引擎，经 C ABI DLL 跨越 `/MT`（WebRTC）与 `/MD`（Qt）两套运行时

## 目录结构

```text
AudioSub/
├── client/
│   ├── audiosub_engine.{h,cc}      核心引擎：WebRTC/信令/ASR/融合接线，对外抛事件回调
│   ├── peer_connection_client.*    WebRTC PeerConnection 封装
│   ├── signaling_client.*          TCP + 行 JSON 信令客户端
│   ├── wasapi_mic_capture.*        WASAPI 麦克风采集（备用音频链路）
│   ├── main.cc                     CLI 外壳，链接核心引擎
│   └── capi/                       C ABI DLL（audiosub_capi），供 GUI 动态调用
├── gui/                            Qt Widgets 前端（audiosub_gui，飞书风格聊天）
├── include/
│   ├── asr/ audio/ core/           公共头文件
│   ├── fusion/                     字幕-标注融合器（header-only）
│   └── proto/                      DataChannel 消息协议
├── src/
│   ├── asr/                        whisper.cpp 引擎实现
│   └── audio/                      PCM 环形缓冲、ASR 音频格式转换
├── signaling/                      Python 信令服务器
├── scripts/                        构建 / SDK 获取 / 自动化测试脚本
├── docs/                           使用说明与技术文档
├── third_party/
│   ├── nlohmann/json.hpp
│   ├── whisper.cpp/                git 子模块
│   └── webrtc-sdk/                 预编译 WebRTC SDK（本地生成，默认不入库）
├── CMakeLists.txt
└── README.md
```

## 当前状态

- [x] 阶段 1：Python 信令服务器 + WebRTC `PeerConnection` / `DataChannel` 文字 P2P
- [x] 阶段 2：A 端麦克风采集，B 端获取远端 `PCM`
- [x] 阶段 3：接入本地 `ASR`（`whisper.cpp`），输出实时字幕
- [x] 阶段 4：标注通道、字幕与标注融合、指标观测与退出汇总
- [x] 阶段 5：`Qt` 图形界面（飞书风格聊天）

## 架构概览

```text
A 讲话 ──WebRTC 音频──► B 收到 PCM ──► whisper.cpp ASR ──► 字幕
A 标注 ──DataChannel──► B 时间轴对齐融合 ──► 增强字幕 ──B→A 回传──► A

                 ┌── audiosub_client.exe   (CLI, /MT)
audiosub_engine ─┤
   核心引擎       └── audiosub_capi.dll (C ABI) ── audiosub_gui.exe (Qt, /MD)
```

## 快速开始

完整步骤、常见问题与 GUI 说明见 **[docs/usage-guide.md](docs/usage-guide.md)**。

```powershell
git clone --recurse-submodules https://github.com/NJUPTzza/AudioSub.git
cd AudioSub
.\scripts\bootstrap.ps1          # 拉 WebRTC SDK 并编译
# 下载 ggml-small.bin 到 third_party/whisper.cpp/models/

python signaling\server.py     # 终端 1：信令
.\build\client\Release\audiosub_client.exe --id B   # 终端 2
.\build\client\Release\audiosub_client.exe --id A   # 终端 3
```

- A 端：`/talk on` 说话，`/note 文本` 发标注，`/quit` 退出  
- 默认音频链路 `--audio-path webrtc`；同机 ASR 异常时可试 `wasapi`  
- GUI：`audiosub_gui.exe --id A|B`（编译前需关闭已运行的 GUI 进程）

## 指标观测

| 指标 | 参考预算 | 含义 |
|------|----------|------|
| 传输 RTT | ≤ 200ms（同机更小） | WebRTC ICE `candidate-pair.current_round_trip_time` |
| 出字延迟 | ≤ 4000ms | B 端：段首帧进 ASR 缓冲 → 字幕产出（`ready_ms = end_ms - start_ms`） |
| 标注匹配误差 | ≤ 500ms | 标注时刻与字幕时间窗的偏差（`MarkMatchError`） |

详见 [docs/latency-metrics.md](docs/latency-metrics.md)。CLI 退出时打印均值 / 峰值汇总，GUI 顶栏约每 500ms 刷新。

## 仓库包含什么

| 已在 Git 中 | 不在 Git 中 |
|-------------|-------------|
| C++ 客户端 / 引擎 / C ABI DLL 源码 | `third_party/webrtc-sdk/`（含 `webrtc.lib`、头文件树）|
| Qt GUI 源码 | `build/` 编译产物 |
| Python 信令服务器 | ASR 模型 `*.bin` |
| CMake 与 PowerShell 脚本 | |
| `nlohmann/json.hpp`、文档 | |

## 自动化测试

WebRTC P2P 链路验证（信令握手 + DataChannel 双向文字）：

```powershell
python .\scripts\test_stage1b.py
```

## 文档导航

| 文档 | 说明 |
|------|------|
| [docs/usage-guide.md](docs/usage-guide.md) | **使用指南**（环境、编译、运行、FAQ） |
| [docs/latency-metrics.md](docs/latency-metrics.md) | 三项时间指标定义 |
| [docs/pipeline-flowchart.md](docs/pipeline-flowchart.md) | 链路流程图速览 |
| [docs/pipeline-deep-dive.md](docs/pipeline-deep-dive.md) | 链路详解 |
| [docs/current-implementation-flow.md](docs/current-implementation-flow.md) | L0～L8 速查 |

更多问题见 [docs/usage-guide.md#7-常见问题](docs/usage-guide.md#7-常见问题)。
