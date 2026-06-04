# AudioSub 当前实现链路梳理

本文是当前主干实现的**链路速查**（L0~L8），目标是帮助你快速回答三件事：

1. 每条链路从哪里进入、在哪里结束；
2. 中间经过哪些模块；
3. 关键代码在什么文件/函数。

---

## 链路编号约定（用于对照代码注释）

- `L0` 启动与应用层编排链路（参数、对象、主循环）
- `L1` 信令传输链路（TCP + JSON 行协议）
- `L2` 信令服务器 relay 链路（join/peer_ready/转发/离开）
- `L3` WebRTC 初始化链路（线程、Factory、PeerConnection）
- `L4` 协商链路（Offer/Answer/ICE/DataChannel）
- `L5` 发送音频链路（A 端 `/talk on`）
- `L6` 接收 PCM 链路（B 端 `OnTrack -> AudioSink -> 回调`)
- `L8` 退出清理链路（Close/线程回收）

---

## L0 启动与应用层编排链路

**入口**
- CLI：`client/main.cc` `main()` → `AudiosubEngine::Start()`
- GUI：`gui/main.cpp` → `audiosub_capi` → 同上引擎

**结束**
- CLI：进入 `std::getline` 命令循环；GUI：事件循环 + 引擎后台线程

**关键职责**
- 解析 `--id` / `--host` / `--port` / `--audio-path`；
- 引擎内创建并接线 `PeerConnectionClient`、`SignalingClient`、ASR、融合器；
- 注册状态/字幕/标注回调；B 端启动 ASR 消费线程。

> 如何运行见 [usage-guide.md](usage-guide.md)；指标定义见 [latency-metrics.md](latency-metrics.md)。

---

## L1 信令传输链路（客户端）

**入口**
- `client/signaling_client.cc` `SignalingClient::Connect()`

**结束**
- `DispatchLine()` 调应用层 `handler_` 回调

**关键职责**
- 用 Winsock 建立 TCP；
- 发第一条 `hello`（声明 A/B 身份）；
- 后台线程 `RecvLoop()` 按 `\n` 拆 JSON；
- 解析后交给应用层处理。

---

## L2 信令 relay 链路（服务器）

**入口**
- `signaling/server.py` `handle()`

**结束**
- 转发给对端；断线时发 `peer_left`

**关键职责**
- 校验第一条 `hello`；
- `Room.join()` 维护 A/B；
- 双向发送 `peer_ready`；
- 透明转发 `offer/answer/candidate/...`；
- 清理连接与离线通知。

---

## L3 WebRTC 初始化链路

**入口**
- `client/peer_connection_client.cc` `PeerConnectionClient::Initialize()`

**结束**
- `pc_` 构建成功可用于协商

**关键职责**
- 初始化 SSL；
- 初始化 COM + ADM（音频设备模块）；
- 启动 `network/worker/signaling` 三线程；
- 创建 `PeerConnectionFactory`；
- 创建 `PeerConnection`。

---

## L4 协商链路（SDP + ICE + DataChannel）

**入口**
- A 收到 `peer_ready` 后 `CreateOfferAndDataChannel()`
- B 收到 `offer` 后 `SetRemoteSdp + CreateAnswer()`

**结束**
- 双端 `dc:open`，可收发 DataChannel 文本

**关键职责**
- `OnLocalSdpReady()` 将本地 SDP 回调给应用层并发信令；
- `OnIceCandidate()` 生成本地 candidate 回调给应用层；
- 收到信令后 `SetRemoteSdp()/AddRemoteIceCandidate()` 回喂 WebRTC；
- B 在 `OnDataChannel()` 绑定通道 observer。

---

## L5 发送音频链路（A 端讲话）

**入口**
- A 输入 `/talk on`

**结束**
- 本地音频轨启用；ADM 处于 recording 状态

**关键职责**
- `EnableLocalAudio()` 在协商前添加本地音轨；
- `SetLocalAudioEnabled(true)`：
  - 优先 `InitRecording/StartRecording`；
  - 若设备要求 AEC 路径，先 `InitPlayout/StartPlayout` 再重试；
  - 最终 `local_audio_track_->set_enabled(true)`。

---

## L6 接收 PCM 链路（B 端）

**入口**
- `OnTrack()` 收到远端音轨

**结束**
- 应用层 `SetRemoteAudioFrameCallback` 收到 `PcmFrame`

**关键职责**
- 在 `OnTrack()` 给远端 `AudioTrack` 绑定 `RemoteAudioSink`；
- `RemoteAudioSink::OnData()` 把 WebRTC PCM 转成统一 `core::PcmFrame`；
- `DeliverRemoteAudioFrame()` 回调 `AudiosubEngine`；
- 引擎推入 `remote_audio_buffer_` / `remote_audio_asr_buffer_`，ASR 线程经 `AsrAudioConverter` 喂给 whisper。

---

## L8 退出清理链路

**入口**
- 用户 `/quit` 或对象析构

**结束**
- socket、DataChannel、PeerConnection、ADM、线程全部释放

**关键职责**
- `AudiosubEngine::Stop()`：先停信令，再 `pc_.Close()`；
- `Close()` 中按顺序关 DC -> PC -> ADM -> 线程；
- 卸载 remote sink，防止回调访问悬空对象；
- 关闭 ring buffer 让监控线程退出并 `join`。

