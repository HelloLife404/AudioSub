# AudioSub 时间指标说明

本项目在界面与退出汇总中展示 **三项** 时间指标（不再使用旧的 lat / vis）。

---

## 1. 传输 RTT（`rtt`）

| 项目 | 说明 |
|------|------|
| **含义** | A↔B 之间 WebRTC **ICE 候选对的往返时延**（Round-Trip Time） |
| **数据来源** | `PeerConnection::GetStats()` → `candidate-pair` 统计项中的 `current_round_trip_time` |
| **单位** | 毫秒（ms） |
| **更新方式** | `GetMetrics()` 触发 `GetStats()`；**仅在异步回调**里解析 `current_round_trip_time` 并累计；汇总为会话内样本的**平均 / 峰值** |
| **不包含** | ASR、攒段、字幕生成；仅表示 P2P 连接网络往返 |

**注意**

- 同机联调 RTT 通常很小（约 1～几十 ms）；跨网才有区分度。旧实现曾在回调到达前同步读缓存，且把亚毫秒四舍五入成 0，会误显示「均 0」。
- `wasapi` 对照链路的音频走 DataChannel，但 RTT 仍是整条 PeerConnection 的 ICE 统计，可反映链路状况。
- RTT 是**往返**，不是单向传输时间；单向可粗略理解为 RTT/2。

---

## 2. 出字延迟（`ready`）

| 项目 | 说明 |
|------|------|
| **含义** | B 端：**本段音频开始进入 ASR 缓冲** → **字幕识别完成并可展示** |
| **计算** | `ready_ms = end_ms - start_ms` |
| **起点 `start_ms`** | 该段第一帧进入 `whisper` 累积缓冲的时刻（`pending_start_wall_ms_`） |
| **终点 `end_ms`** | `whisper_full` 完成、产出字幕的时刻（`subtitle_cb` 触发前后） |
| **包含** | RingBuffer 排队、**攒段等待**（如说完后静音 ~0.6s）、**whisper 推理** |
| **不包含** | A→B 网络传输（见 RTT）；A 端开口说话到首包到达 B 的间隔 |

**与 `infer_ms`（仅调试用）的区别**

- `SubtitleSegment::infer_ms` = 仅 `whisper_full` 推理耗时（`steady_clock`），不展示在 UI。
- **`ready_ms` 才是界面上的「出字延迟」**，通常明显大于 `infer_ms`。

**预算建议（界面标 ⚠）**

- 默认参考：平均 ≤ 4000ms（含攒段策略）。

---

## 3. 标注匹配误差（`err`）

| 项目 | 说明 |
|------|------|
| **含义** | 标注发生时刻与字幕时间窗的偏差 |
| **计算** | `MarkMatchError(event_time_ms, start_ms, end_ms)`：落在 `[start_ms, end_ms]` 内为 0，否则为到最近边界的距离（ms） |
| **融合容差** | 匹配标注时还允许 ±1500ms（`SubtitleMarkFuser::kToleranceMs`），与 `err` 统计独立 |
| **预算建议** | ≤ 500ms |

---

## 代码落点速查

| 指标 | 主要文件 |
|------|----------|
| RTT | `client/peer_connection_client.cc`（`GetStats` → `SetRttSampleCallback`）→ `audiosub_engine.cc`（`AddRtt`） |
| ready | `src/asr/whisper_cpp_engine.cc`（`start_ms`/`end_ms`）→ `audiosub_engine.cc`（`AddReady`） |
| err | `audiosub_engine.cc`（`MarkMatchError`）、`include/fusion/subtitle_mark_fuser.h` |
| 展示 | `gui/main_window.cpp`、`client/main.cc`、`client/capi/audiosub_capi.h` |

---

## 展示示例

**顶栏（周期性）：**

```text
传输 RTT: 均 12 / 峰 28 ms (n=40)
出字延迟: 均 2100 / 峰 3800 ms (n=5)
标注误差: 均 0 / 峰 120 ms (n=3)
```

**单条字幕 footer：**

```text
10:01:20–10:01:23 · 出字 2400ms
```
