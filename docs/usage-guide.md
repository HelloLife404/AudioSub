# AudioSub 使用指南

本文说明如何在本机**拉代码、编译、运行** AudioSub（CLI 或 GUI）。技术细节见文末「进一步阅读」。

---

## 1. 环境要求

| 类型 | 要求 |
|------|------|
| 系统 | Windows 10/11 x64 |
| 编译 | Visual Studio 2022 或 Build Tools 2022（勾选 **Desktop development with C++**） |
| 运行信令 | Python 3.8+ |
| 版本管理 | Git |
| GUI（可选） | Qt 6（MSVC 2022 64-bit 组件） |

说明：

- **不需要** `depot_tools`、本地 `webrtc/src`、自行编译 WebRTC
- 使用仓库脚本下载的**预编译 WebRTC SDK**
- `scripts/build.ps1` 会自动查找 VS 自带的 `cmake.exe`

---

## 2. 获取代码与依赖

```powershell
git clone --recurse-submodules https://github.com/NJUPTzza/AudioSub.git
cd AudioSub
```

若已 clone 但未拉子模块：

```powershell
git submodule update --init --recursive
```

### 下载 WebRTC SDK 并构建

**默认地址已配置时**（见 `scripts/fetch-webrtc-sdk.ps1`）：

```powershell
.\scripts\bootstrap.ps1
```

**指定 SDK 地址：**

```powershell
.\scripts\bootstrap.ps1 -Url "https://github.com/NJUPTzza/AudioSub/releases/download/webrtc-sdk-v1.0.0/webrtc-sdk-win-x64-release-771e6489c9.zip"
```

**使用本地已有 SDK 目录：**

```powershell
.\scripts\bootstrap.ps1 -LocalSdkPath "D:\path\to\webrtc-sdk-win-x64-release"
```

也可设置环境变量后执行 `bootstrap.ps1`：

```powershell
$env:AUDIOSUB_WEBRTC_SDK_URL="https://.../webrtc-sdk-....zip"
.\scripts\bootstrap.ps1
```

`bootstrap.ps1` 会检查 Python / VS、按需拉取 SDK，并调用 `build.ps1` 编译。

### 仅重新编译（环境已就绪）

```powershell
.\scripts\build.ps1 -Config Release
```

不需要 GUI 时：

```powershell
cmake -S . -B build -DAUDIOSUB_BUILD_GUI=OFF
cmake --build build --config Release
```

---

## 3. ASR 模型（B 端必需）

B 端默认识别模型：

```text
third_party/whisper.cpp/models/ggml-small.bin
```

请按 [whisper.cpp](https://github.com/ggerganov/whisper.cpp) 说明下载 `small` 模型到该路径。缺失时 B 端会报 `whisper init failed`，无法出字幕。

---

## 4. 运行

### 4.1 启动信令服务器（终端 1）

```powershell
cd AudioSub
python signaling\server.py
```

默认监听 `127.0.0.1:8888`。

### 4.2 命令行（终端 2 / 3）

**B 端（识别方，建议先开）：**

```powershell
.\build\client\Release\audiosub_client.exe --id B
```

**A 端（讲话方）：**

```powershell
.\build\client\Release\audiosub_client.exe --id A
```

可选参数：`--host`、`--port`、`--audio-path wasapi|webrtc`（默认 **webrtc**）。

两边出现 `dc:open` / `pc:connected` 后：

| 端 | 操作 |
|----|------|
| A | `/talk on` 说话 → `/talk off`；`/note 文本` 发标注 |
| B | 等待 `[sub]` 字幕行 |
| 任意 | `/quit` 退出并打印指标汇总 |

### 4.3 图形界面

需已成功编译 GUI（且本机已安装 Qt）。**编译前请关闭正在运行的 `audiosub_gui.exe`**，否则可能报 `LNK1104`。

```powershell
.\build\gui\Release\audiosub_gui.exe --id A
.\build\gui\Release\audiosub_gui.exe --id B
```

- A：点「开始说话」、输入框发标注  
- B：查看字幕气泡  
- 顶栏约每 500ms 刷新三项指标

### 4.4 音频链路说明

| `--audio-path` | 说明 |
|----------------|------|
| `webrtc`（默认） | WebRTC 麦克风 + 3A + Opus，经 RTP 传 B |
| `wasapi` | WASAPI 直采 PCM，经 DataChannel 二进制发送；同机调试若 ASR 幻觉多可尝试 |

---

## 5. 指标说明

顶栏 / 退出汇总为三项（详见 [latency-metrics.md](latency-metrics.md)）：

| 指标 | 含义 |
|------|------|
| 传输 RTT | WebRTC ICE 往返时延 |
| 出字延迟 | B 端：段进 ASR → 字幕产出 |
| 标注误差 | 标注时刻与字幕时间窗偏差 |

同机 RTT 通常很小（约 1～几十 ms）；出字延迟含攒段 + whisper，常为 1～4s。

---

## 6. 环境自检（可选）

```powershell
# 信令 + WebRTC P2P + DataChannel 文字
python .\scripts\test_stage1b.py

# 仅信令
python .\scripts\test_stage1a.py
```

---

## 7. 常见问题

**编译报 `webrtc.lib not found`**  
先运行 `.\scripts\bootstrap.ps1` 或 `.\scripts\fetch-webrtc-sdk.ps1`。

**`LNK1104: 无法打开 audiosub_gui.exe`**  
关闭所有 GUI 窗口，或 `Stop-Process -Name audiosub_gui -Force`，再编译。

**`cmake` 不在 PATH**  
直接运行 `.\scripts\build.ps1`，脚本会定位 VS 自带 CMake。

**GUI 找不到 Qt**  
安装 Qt 6（MSVC 2022 64-bit），设置 `QT_PREFIX_PATH`，或 `-DAUDIOSUB_BUILD_GUI=OFF` 只编 CLI。

**连上 peer 但没有字幕**  
检查 `ggml-small.bin`；确认 A 已 `/talk on` 或 GUI「开始说话」；看 B 端是否有 `[asr]` 报错。

**顶栏 RTT 长期为 0 或暂无样本**  
需 `pc:connected` 后等待几秒；请使用最新编译版本（RTT 在 GetStats 回调里入账）。

**`-Clean` 失败 / build 目录被占用**  
关闭 VS、资源管理器预览、`audiosub_client.exe` / `audiosub_gui.exe` 后重试，或不加 `-Clean`。

---

## 8. 进一步阅读

| 文档 | 内容 |
|------|------|
| [latency-metrics.md](latency-metrics.md) | 三项时间指标定义与代码落点 |
| [pipeline-flowchart.md](pipeline-flowchart.md) | 链路流程图速览 |
| [pipeline-deep-dive.md](pipeline-deep-dive.md) | 各环节详解（流程图 + 代码） |
| [current-implementation-flow.md](current-implementation-flow.md) | L0～L8 链路编号速查 |
| [README.md](../README.md) | 项目概览与目录结构 |
