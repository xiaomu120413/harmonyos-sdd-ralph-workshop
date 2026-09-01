# 案例二证据 02｜跨平台调研、HarmonyOS 对接方案与最小穿刺

> 目标：让 AI 从成熟平台学习“接口契约”，再映射 HarmonyOS 能力；不照抄平台代码，也不一开始实现完整 AVC420/AVC444 GPU 链路。

## 1. 先看仓库已经接受的扩展方式

FreeRDP H.264 层通过 subsystem 隔离平台实现。当前源码可定位到：

| 路径 | 平台/作用 | 可借鉴的契约 |
|---|---|---|
| `h264_ffmpeg.c`、`h264_openh264.c` | 通用软件解码 | 正确性 fallback、YUV 输出语义 |
| `h264_mf.c` | Windows Media Foundation | 平台硬解的创建、输入、输出和释放 |
| `h264_mediacodec.c` | Android MediaCodec | 移动平台 codec 生命周期和 buffer 交互 |
| `h264_ohos_decoder.c` + `h264_ohos_decoder_buffer.c` | HarmonyOS `OH_AVCodec` | 硬件能力选择、buffer 输出、格式/stride/plane 处理 |

研究重点不是逐行翻译 MediaCodec，而是抽出平台无关契约：

```text
create → configure → callback/buffer registration → start
input compressed sample → bounded wait/output callback
validate format/size/stride/planes → publish decoded state
flush/reset/stop/destroy
failure → explicit fallback or explicit UNKNOWN
```

## 2. HarmonyOS 能力映射

| 平台能力问题 | HarmonyOS 对接点 | 必须保存的证据 |
|---|---|---|
| 是否有硬件 H.264 decoder | `OH_AVCodec_GetCapabilityByCategory(..., HARDWARE)` | decoder name + capability result |
| 如何配置 | `OH_AVFormat_CreateVideoFormat` + configure/prepare/start | width/height/pixel format/return code |
| 如何输入 | AVBuffer / input callback | frameId、bytes、timestamp、queue result |
| 如何取得输出 | buffer callback + output description | output index、format、stride、plane layout |
| 如何显示 | buffer 合成路径或 NativeWindow/GL path | owner、target generation、present boundary |
| 失败怎么办 | 软件 fallback 或保留原生 GDI | fallback reason，不能静默宣称硬解成功 |

## 3. Architecture Decision

```text
ADR-GPU-001
Decision:
  先在 FreeRDP 既有 H264_CONTEXT_SUBSYSTEM 契约内接入 OH_AVCodec，
  不绕开协议语义另建一套“收到字节就直接画”的旁路。

First slice:
  AVC420 单路 buffer 硬解，一帧输入、一帧输出、可回退。

Deferred:
  AVC444 luma/chroma 状态、GPU 合成、零拷贝、队列优化、resize 长稳。

Why:
  先证明 codec 能被选择且产出正确帧，再扩展所有权和持续显示；
  否则黑屏时无法区分 codec、format、composition、owner 和 present。
```

## 4. 最小能力穿刺

穿刺不是“做一个无法复用的 demo”，而是沿最终架构走通最短真实路径：

```text
FreeRDP H.264 subsystem selected
  → OH_AVCodec hardware decoder created
  → known AVC420 sample queued
  → one decoded output returned
  → width/height/stride/planes validated
  → one frame reaches the existing output boundary
  → failure can return to software/native path
```

### Spike 验收点

| ID | 验收 | PASS 证据 | 常见假绿 |
|---|---|---|---|
| SP-01 | 能编译和链接 | OHOS arm64 build + symbol/link output | 只编了 App，没编新 backend |
| SP-02 | 真机选中硬件 decoder | capability 与 decoder name 日志 | CMake 开关打开但运行时 fallback |
| SP-03 | 一帧有合法输出 | frameId、尺寸、stride、planes、输出状态 | callback 返回但 buffer 无效 |
| SP-04 | 一帧可见且路径一致 | 输入→解码→owner→present 同一 run/frame 证据 | 截图来自旧 GDI 路径 |
| SP-05 | 失败可控 | 注入 configure/input/output 失败后明确 fallback | 静默吞错、停在黑屏 |

## 5. 为什么不能直接宣布方案正确

最小穿刺只证明“平台能力与接口方向成立”，不证明：

- 连续视频不卡顿；
- CPU 占用达到目标；
- AVC444 语义正确；
- resize、切后台、断线重连稳定；
- GDI 与 GPU 不会争抢输出；
- 所有设备的 decoder 行为一致。

这些内容必须进入后续任务和验收矩阵，而不是从一帧成功外推。
