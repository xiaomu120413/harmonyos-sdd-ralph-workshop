# 案例二证据 01｜55 万行级 FreeRDP 代码认知地图

> 目标：演示 AI 面对大型开源库时如何快速建立“足够完成当前任务”的认知，而不是把整个仓库塞进上下文。

## 1. 规模事实

2026-09-01 对本地 `harmony/third_party/FreeRDP` 快照进行只读统计：

```text
统计扩展名：c / h / cpp / hpp / cmake / java / kt
源文件数量：2014
近似代码行：559,355
```

课程可以表述为“30 万行级以上开源库”或更精确地说“本地快照约 55.9 万行”。这个数字只用来说明搜索空间，不代表 AI 需要读取全部文件。

## 2. 先把问题改写成五个可回答的问题

原始问题：播放远端视频时走 CPU 解码，画面明显卡顿。

AI 第一次只回答：

1. H.264 数据从哪个协议回调进入？
2. 当前实际选择了哪个 decoder subsystem？
3. 解码结果以什么格式、stride 和 plane 交给后续链路？
4. 哪个组件拥有最终显示 Surface，何时 present？
5. 如何证明设备运行的是新路径而不是 fallback？

这些问题决定搜索关键词。没有证据前，不直接问“如何重构 FreeRDP 硬解”。

## 3. 四层地图

| 层 | 只读入口 | 本轮需要回答 | 暂不展开 |
|---|---|---|---|
| 协议语义 | `libfreerdp/gdi/gfx.c`、`libfreerdp/codec/h264.c` | AVC420/444 命令如何进入、原生 fallback 是什么 | 音频、文件、打印等无关 channel |
| 平台 codec | `h264_ffmpeg.c`、`h264_openh264.c`、`h264_mf.c`、`h264_mediacodec.c`、`h264_ohos_*` | subsystem 的 Init/Decompress/Uninit 契约 | 编码端与当前客户端无关能力 |
| OHOS 适配 | `client/OHOS/ohos_rdpgfx_*`、`app/common/.../rdpgfx_pipeline.*` | FreeRDP 与 App/NativeWindow 如何交接 | ArkTS 页面样式和其他业务 UI |
| 输出与生命周期 | `surface_bridge.*`、`render_output_owner.*`、`avc420_*`、`avc444_*` | Surface、owner、queue、EndFrame、resize 如何闭合 | 还未触发的问题分支 |

## 4. 上下文预算

每轮上下文只保留：

```text
任务问题 1 条
当前调用链 1 条
关键接口/结构 3–7 个
已证实事实 5–10 条
待证假设 1 条
下一条最小命令 1 条
```

大段源码不复制到聊天。只保存 `path:line + 为什么相关 + 当前结论`，需要时重新检索原文件。

示例 evidence index：

| ID | 代码锚点 | 事实 | 可信度 |
|---|---|---|---|
| C01 | `libfreerdp/codec/h264.c` | 上层通过 `H264_CONTEXT_SUBSYSTEM` 调用平台后端 | CODE FACT |
| C02 | `libfreerdp/codec/h264_mediacodec.c` | Android 已有 MediaCodec 后端，可参考契约和生命周期 | CODE FACT |
| C03 | `libfreerdp/codec/h264_mf.c` | Windows 已有 Media Foundation 后端 | CODE FACT |
| C04 | `libfreerdp/codec/h264_ohos_decoder.c` | OHOS 后端选择硬件 H.264 decoder 并实现 subsystem | CODE FACT |
| C05 | `client/OHOS/README.md` | AVC420 buffer 模式与 AVC444 GPU compositor 有不同输出策略 | CODE FACT |
| C06 | 设备路径日志 | 本次运行是否真正选择 OHOS hardware decoder | RUNTIME REQUIRED |

## 5. AI 勘察 Prompt

```text
只读任务。不要修改代码，也不要总结整个仓库。
问题：远端 H.264 视频当前走 CPU 路径并卡顿。

请输出：
1. 从 RDPGFX SurfaceCommand 到 H.264 decoder，再到最终 present 的调用链；
2. 每一段只列 path:line、输入、输出、owner；
3. 当前平台后端与其他平台后端的共同接口；
4. 仍缺的运行时证据；
5. 下一轮最多需要打开的 5 个文件。

禁止：提出最终重构、批量阅读无关目录、把文件名推断成运行时事实。
```

## 6. 本页验收

- AI 能在不读取全库的前提下画出一条可定位调用链。
- 每个结论都有代码锚点或显式标记 `UNKNOWN`。
- 下一轮上下文由问题驱动，不把历史聊天整体带入。

