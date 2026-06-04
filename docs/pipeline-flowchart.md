# AudioSub 流程图说明

> 本文按「每条链路一章 + 一张流程图 + 要点补充」的方式讲清整个系统。
> **如何在本机跑起来**见 [usage-guide.md](usage-guide.md)。
> 链路编号见 [current-implementation-flow.md](current-implementation-flow.md)；
> 逐环节代码细节见 [pipeline-deep-dive.md](pipeline-deep-dive.md)。
>
> 先记住三条总约定：
> - 讲话方恒为 **A**（WebRTC 发起方 offerer），**B** 是接收 + 识别（ASR）方。
> - 信令走 **TCP + 一行一条 JSON**，只用于交换 SDP/ICE，连通后音频不再经过它。
> - 音频有两条链路：**默认 WebRTC ADM 音轨（走 RTP）**，对照 **WASAPI 直采（走 DataChannel）**；
>   CLI 默认 `webrtc`，用 `--audio-path wasapi` 切换。

---

## 全局总览

```mermaid
flowchart LR
    subgraph A[A 端 · 讲话方]
        MIC[麦克风] --> CAP[采集<br/>WebRTC ADM 或 WASAPI]
        CAP --> SEND[发送<br/>RTP 或 DataChannel]
        NOTE[标注输入] --> DC1[DataChannel 文本]
    end
    SIG[(信令服务器<br/>仅 SDP/ICE)]
    P2P((WebRTC P2P))
    subgraph B[B 端 · 识别方]
        RECV[收音频] --> RB[(RingBuffer)]
        RB --> CONV[转 16k/mono] --> ASR[whisper 识别]
        ASR --> FUSE[字幕↔标注融合]
        DC2[收标注] --> FUSE
        FUSE --> BACK[回传字幕]
    end
    A -. 协商期 .-> SIG -. 转发 .-> B
    SEND --> P2P --> RECV
    DC1 --> P2P --> DC2
    BACK --> P2P --> A
```

---

## 一、A 和 B 是怎么建立 P2P 连接的（信令 + 媒体）

### 为什么需要信令服务器

- WebRTC 是点对点的，但两端一开始互不知道对方的存在、地址、支持的编解码，必须有个双方都能先连上的「中间人」帮它们对暗号。
- WebRTC 故意不规定中间人怎么实现，用 HTTP、WebSocket 都行；本项目用最简单的 **TCP 长连接 + 每行一条 JSON**。
- 中间人只在「建立连接」阶段传几条小消息，**P2P 一旦通了，音频/数据完全不经过它**。

**需要交换的信息**

- **SDP（会话描述）**：一大段文本，描述「我支持哪些编解码、用什么传输参数」。发起方发的叫 **Offer**，应答方回的叫 **Answer**，一来一回双方就对齐了媒体能力。
- **ICE candidate（网络地址候选）**：每端列出「我可能能被连上的地址」（内网 IP、经路由器看到的公网 IP、中转地址等），一条条发给对方；两边两两配对去试，试通一条 P2P 通道就建起来了。本项目配了 Google 的 STUN 帮忙发现公网 IP（同机/同局域网用不到）。

### 信令传输层

```mermaid
flowchart TD
    C[A/B 各自 TCP 连上服务器] --> HELLO[发 hello 声明身份]
    HELLO --> JOIN[服务器 join 房间]
    JOIN --> RDY{两端都到齐?}
    RDY -- 是 --> PR[双向发 peer_ready]
    PR --> FWD[此后每行 JSON 原样转发给对端]
    FWD --> LEFT[任一端断开 → 发 peer_left]
```

- 服务器（`signaling/server.py`）**不解析**业务字段，offer/answer/candidate 都走同一段透明转发。
- 客户端（`client/signaling_client.cc`）主线程发、后台线程收；TCP 是字节流，需自己按 `\n` 拆行处理粘包/拆包；`Send()` 加锁防多线程写交错。

### 协商时序（一份 Offer 同时谈妥音频管 + 数据管）

```mermaid
sequenceDiagram
    participant A as A (offerer)
    participant S as 信令服务器
    participant B as B (answerer)
    A->>S: hello(A)
    B->>S: hello(B)
    S-->>A: peer_ready
    S-->>B: peer_ready
    Note over A: A 收到 peer_ready 才主动发起
    A->>A: 加音轨 + 建 DataChannel + CreateOffer
    A->>S: offer (含 m=audio + m=application)
    S-->>B: offer
    B->>B: SetRemoteSdp + CreateAnswer<br/>OnTrack / OnDataChannel 到达
    B->>S: answer
    S-->>A: answer
    par ICE 双向交换
        A->>S: candidate
        S-->>B: candidate
        B->>S: candidate
        S-->>A: candidate
    end
    Note over A,B: ICE 选通地址 + DTLS 握手 → pc:connected & dc:open
```

- **A 当 offerer 是约定**：只有 `id=="A"` 在收到 `peer_ready` 时发起，避免双方同时发 Offer（glare）。
- **DataChannel 必须在 CreateOffer 之前建**，否则 Offer 里没有数据通道的 `m=` section，B 不会触发 `OnDataChannel`。
- SDP 是**异步**生成：`CreateOffer` → `OnLocalSdpReady` → 先 `SetLocalDescription`，再把 SDP 经信令发给对端。
- **加密**：媒体用 SRTP、数据用 DTLS，密钥在 DTLS 握手时协商；`Initialize()` 里必须先 `InitializeSSL()`。
- **判断通了**：看到 `pc:connected`（`OnConnectionChange`）和 `dc:open`（DataChannel `OnStateChange`）即成功。

> 关键代码：`peer_connection_client.cc` 的 `CreateOfferAndDataChannel / CreateAnswer / OnLocalSdpReady / OnIceCandidate / OnDataChannel / OnTrack`；接线在 `audiosub_engine.cc` 的信令 handler。

---

## 二、WebRTC 初始化（线程 / COM / ADM / Factory）

```mermaid
flowchart TD
    SSL[InitializeSSL] --> COM[COM 初始化 kMTA]
    COM --> ADM[创建 ADM 音频设备模块<br/>逐个尝试 audio layer]
    ADM --> OBS[挂 AdmCaptureObserver<br/>拿本地麦克风 PCM]
    OBS --> T[启动三线程<br/>network / worker / signaling]
    T --> FAC[CreatePeerConnectionFactory<br/>内建 Opus 编解码]
    FAC --> PC[CreatePeerConnection<br/>STUN + UnifiedPlan]
```

- **三个 WebRTC 线程**：`network`（必须带 SocketServer 跑 socket I/O）、`worker`（编解码/媒体）、`signaling`（PC 状态机），让 WebRTC 不阻塞业务线程。
- **COM 必须 MTA**：Windows 下 ADM 走 Core Audio 需要 COM；这也是 Qt GUI 的坑（Qt 主线程是 STA，所以 GUI 把初始化放后台线程）。
- **Factory 自带 Opus 编解码工厂**，默认 WebRTC 音轨链路靠它编解码；WASAPI 对照链路不依赖它。

> 关键代码：`peer_connection_client.cc` `Initialize()`。

---

## 三、A 端音频采集与发送（两条链路）

| | 默认链路 `webrtc` | 对照链路 `wasapi` |
|---|---|---|
| 采集 | WebRTC ADM | 自写 WASAPI 直采 |
| 处理 | 默认 3A（降噪/AEC/AGC） | **无 3A** |
| 编码 | Opus | 不编码，raw PCM |
| 传输 | RTP 媒体流（SRTP） | DataChannel 二进制帧 |
| B 端入口 | `OnTrack → RemoteAudioSink::OnData` | `OnMessage(binary) → HandlePcmDataChannel` |

> 不管走哪条，`Initialize()` 都会创建 ADM，`EnableLocalAudio()` 也都会挂一条 WebRTC 音轨；区别只在「发给 B 的语音实际走 RTP 还是 DataChannel」。

### 3.1 默认链路：WebRTC ADM 音轨（RTP）

```mermaid
flowchart LR
    MIC[麦克风] --> ADM[WebRTC ADM 采集]
    ADM --> A3A[默认 3A<br/>降噪/AEC/AGC]
    A3A --> TRK[AudioSource → AudioTrack]
    TRK --> SW{talk 开?<br/>set_enabled}
    SW -- 否 --> MUTE[发静音]
    SW -- 是 --> OPUS[Opus 编码] --> RTP[(RTP 媒体流<br/>worker 编码 → network 发)]
```

- `EnableLocalAudio()` 用 `CreateAudioSource + CreateAudioTrack + AddTrack`，让首个 Offer 就带上 `m=audio`。
- `SetLocalAudioEnabled(true/false)`（GUI 的「开始说话」/CLI 的 `/talk`）用 `set_enabled` 控制是否发有效语音；内部还兼容了某些设备需先 `StartPlayout` 再 `StartRecording` 的问题。
- 编码在 worker 线程、发包在 network 线程。
- **代价**：3A 在同机调试下会把弱人声当噪声削掉 —— 这正是要保留下面 WASAPI 链路的原因。

### 3.2 对照链路：WASAPI 直采 → DataChannel

```mermaid
flowchart LR
    MIC[默认录音设备] --> GMF[GetMixFormat<br/>常见 48k/2ch/float32]
    GMF --> POLL[轮询 GetBuffer]
    POLL --> MONO[多声道平均混 mono]
    MONO --> I16[float → int16]
    I16 --> GAIN[软增益 2x + 防削波]
    GAIN --> CB{talk 开?}
    CB -- 是 --> PACK[拼 PcmDcHeader 16B + raw PCM]
    PACK --> DC[(DataChannel binary 帧)]
    CB -- 否 --> SKIP[采集照跑但不发送]
```

- 采集线程（`wasapi_mic_capture.cc`）：`CoInitializeEx(MTA)` → 默认 `eCapture` 设备 → `IAudioClient` 共享模式 → 轮询 `GetBuffer`，把任意声道/位深统一**混 mono、转 int16、施 2x 软增益**后回调。采样率跟系统 mix format 走（通常 48000）。
- 打包协议：每个二进制帧前缀 16 字节定长头 `PcmDcHeader`（magic="PCM1" / sample_rate / channels / bits / sample_count），raw PCM **不经任何编码**原样塞进 DataChannel，保真喂给 ASR。
- **为什么保留**：同机调试无真实回声参考，WebRTC 3A 会把弱人声削成底噪，导致 whisper「幻觉」出「谢谢观看」之类短语；WASAPI 绕开 3A 可对照验证。

> 关键代码：`peer_connection_client.cc` 的 `EnableLocalAudio / SetLocalAudioEnabled / SendPcmDataChannel`；`wasapi_mic_capture.cc` 的 `CaptureThreadMain`。

---

## 四、B 端如何从 WebRTC 收集音频（RingBuffer + 存取线程）

### 4.1 两条链路对应两个收包入口

```mermaid
flowchart TD
    subgraph 默认链路
        OT[OnTrack 远端音轨] --> SINK[RemoteAudioSink::OnData<br/>解码后的 PCM]
    end
    subgraph 对照链路
        OM[OnMessage binary=true] --> HP[HandlePcmDataChannel<br/>校验 magic / 解头]
    end
    SINK --> CB[remote_audio_frame_cb_]
    HP --> CB
    CB --> PUSH[Push 进 ring buffer]
```

- 两条链路最终都汇到同一个回调 `remote_audio_frame_cb_`，下游处理完全一致。

### 4.2 PcmRingBuffer：线程安全、定容、丢旧帧

```mermaid
flowchart LR
    PROD[生产者：网络回调线程<br/>不能阻塞] -->|Push| Q[(deque + mutex + cv)]
    Q -->|满了先 pop_front 丢最旧帧| Q
    Q -->|WaitPop 阻塞等数据| CONS[消费者：ASR 线程<br/>慢，按自己节奏取]
```

- `Push`：满了就丢最旧一帧（宁可丢历史也不阻塞采集/网络回调），再 `push_back + notify_one`。
- `WaitPop`：`cv.wait` 直到「有数据」或「已关闭」；关闭且空时返回空，消费线程据此退出。
- **作用**：把不能阻塞的生产者和计算重的消费者解耦。

### 4.3 存取线程模型（在 `audiosub_engine.cc` 接线）

```mermaid
flowchart LR
    CB[remote_audio_frame_cb_] -->|Push| RB1[(remote_audio_buffer_ 128)]
    CB -->|Push| RB2[(remote_audio_asr_buffer_ 256)]
    RB1 --> W1[remote_audio_worker_<br/>持续抽干]
    RB2 --> W2[asr_worker_]
    W2 --> CONV[转 16k/mono] --> WHIS[whisper PushAudio]
```

- 远端 PCM 同时 Push 进两个缓冲：`remote_audio_buffer_`（监视/扩展）和 `remote_audio_asr_buffer_`（喂 ASR）。
- 退出时 `Close()` 缓冲 → 各线程 `WaitPop` 返回空 → `join`。

> 关键代码：`pcm_ring_buffer.cc`、`audiosub_engine.cc` 的回调接线与三个 worker 线程。

---

## 五、ASR：从 PCM 到字幕

```mermaid
flowchart TD
    PUSH[PushAudio] --> CHK{16k/mono?}
    CHK -- 否 --> DROP[丢弃]
    CHK -- 是 --> ACC[累积 + 记段起始 wall-clock]
    ACC --> FLUSH{攒满 4s 或<br/>说话后静音≥0.6s?}
    FLUSH -- 否 --> ACC
    FLUSH -- 是 --> EN{整段能量够?}
    EN -- 否 --> DROP2[丢弃挡幻觉]
    EN -- 是 --> INF[whisper_full 推理<br/>infer_ms]
    INF --> NSP{no_speech_prob 高?}
    NSP -- 是 --> DROP3[丢弃幻觉]
    NSP -- 否 --> POST[繁→简 + 黑名单 + 去重] --> EMIT[输出字幕]
```

- 输入必须 16k/mono/16bit，否则丢弃。
- **三道幻觉防线**：整段能量门限 → whisper `no_speech_prob` → 关键词黑名单（「谢谢观看/点赞」等）。
- 字幕 `start_ms` / `end_ms` 为段首帧进缓冲与出字完成的 Unix 毫秒；**`ready_ms = end_ms - start_ms`** 为界面「出字延迟」（含攒段 + 推理）。

> 关键代码：`src/asr/whisper_cpp_engine.cc`。

---

## 六、标注通道（A → B）

```mermaid
flowchart LR
    IN[A 端输入标注] --> SEND[SendNote<br/>seq++ / event_time_ms=Unix毫秒]
    SEND --> JSON[annotation JSON 文本帧] --> DC[(DataChannel binary=false)]
    DC --> RECV[B: OnMessage text → 按 type 分流]
    RECV --> FUSE[交给融合器 AddMark]
```

- 标注与回传字幕都走**同一条 DataChannel 的文本帧**，用 JSON 的 `type` 区分。
- 同一通道：`binary=true` 是 WASAPI PCM，`binary=false` 是这套 JSON 协议，互不干扰。

> 关键代码：`include/proto/dc_message.h`；`audiosub_engine.cc` 的 `SendNote / HandlePeerMessage`。

---

## 七、字幕与标注融合（统一时间轴对齐）

```mermaid
flowchart TD
    MARK[AddMark 网络线程<br/>按 seq 去重] --> STORE[(entries_)]
    SUB[一条字幕产出 ASR 线程] --> F[Fuse 遍历 entries_]
    STORE --> F
    F --> M{标注时刻落在<br/>字幕 start~end ±1500ms?}
    M -- 是 --> ATTACH[挂到该字幕 marks]
    M -- 否 --> ORPH[早于本段且没认领 → 孤儿单独展示]
```

- **对齐判据**：标注时刻落在 `[字幕start−容差, 字幕end+容差]`（容差 1500ms）就归到该字幕。
- **孤儿标注**：早于当前字幕起点又没被认领的，未来字幕只会更晚，判为无归属单独抛出，保证标注不丢。

> 关键代码：`include/fusion/subtitle_mark_fuser.h`。

---

## 八、B → A 字幕回传

```mermaid
flowchart LR
    PROD[B 产出增强字幕] --> PACK[打包 type=subtitle JSON<br/>含 index/start/end/ready_ms/text/marks]
    PACK --> DC[(DataChannel)] --> A[A: HandlePeerMessage<br/>还原成 remote 字幕给 UI]
```

- 这样 **A 也能看到自己说的话被识别成的字幕**；GUI 里 A 的字幕显示在右侧（「我说的」），B 显示在左侧（「对端 A 说的」）。

> 关键代码：`audiosub_engine.cc` 的 `OnSubtitleSegment`（发）/ `HandlePeerMessage`（收）。

---

## 九、指标观测

| 指标 | 含义 |
|------|------|
| 传输 RTT | WebRTC ICE `current_round_trip_time` |
| 出字延迟 | `ready_ms = end_ms - start_ms`（B 端收段→出字幕） |
| 标注误差 | `MarkMatchError` |

运行步骤见 [usage-guide.md](usage-guide.md)；指标详见 [latency-metrics.md](latency-metrics.md)。

---

## 十、退出清理

```mermaid
flowchart LR
    QUIT[/quit 或析构/] --> SIG[signaling.Close 停 recv 线程]
    SIG --> PC[pc.Close：DC → PC → ADM → 三线程]
    PC --> RB[关闭 ring buffer]
    RB --> JOIN[join asr/local/remote worker]
    JOIN --> ORPH[CollectRemainingOrphans 兜底]
```

- 顺序很重要：先停信令，再关 WebRTC（先卸 remote sink 防回调访问悬空对象），再 `Close` ring buffer 让消费线程从 `WaitPop` 返回并 `join`。

> 关键代码：`peer_connection_client.cc` `Close()`；`audiosub_engine.cc` `Stop()`。

---

## 附：关键文件索引

| 关注点 | 文件 |
|--------|------|
| 总接线 / 线程 / 缓冲 | `client/audiosub_engine.cc` |
| 信令服务器 | `signaling/server.py` |
| 信令客户端 | `client/signaling_client.{h,cc}` |
| WebRTC / 协商 / 音频收发 | `client/peer_connection_client.{h,cc}` |
| WASAPI 采集 | `client/wasapi_mic_capture.{h,cc}` |
| 环形缓冲 | `include/audio/pcm_ring_buffer.h`, `src/audio/pcm_ring_buffer.cc` |
| 格式转换 | `src/audio/asr_audio_converter.cc` |
| ASR | `src/asr/whisper_cpp_engine.cc` |
| 标注协议 / 融合 | `include/proto/dc_message.h`, `include/fusion/subtitle_mark_fuser.h` |
