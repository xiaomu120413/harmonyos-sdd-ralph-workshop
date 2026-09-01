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
2. 当前 command 是被 GPU compositor 消费，还是交回原生 GDI/H.264 subsystem？
3. 解码结果以什么格式、stride 和 plane 交给后续链路？
4. 哪个组件拥有最终显示 Surface，何时 present？
5. 如何证明设备运行的是新路径而不是 fallback？

这些问题决定搜索关键词。没有证据前，不直接问“如何重构 FreeRDP 硬解”。

## 3. 四层地图

| 层 | 只读入口 | 本轮需要回答 | 暂不展开 |
|---|---|---|---|
| 原生 GDI/fallback | `libfreerdp/gdi/gfx.c`、`libfreerdp/codec/h264.c` | AVC420/444 如何分发，original callback 最终去哪里 | 音频、文件、打印等无关 channel |
| 平台 decoder 对照 | `h264_ffmpeg.c`、`h264_openh264.c`、`h264_mf.c`、`h264_mediacodec.c`、`h264_ohos_*` | Init/Input/Output/Uninit 契约；不把它误当成完整 GPU 方案 | 编码端与当前客户端无关能力 |
| OHOS hook/policy | `ohos_rdpgfx_bridge.c`、`ohos_rdpgfx_surface.c`、`ohos_rdpgfx_avc444_policy.c`、`rdpgfx_pipeline.cpp` | 保存/替换回调、校验、consumed 与 fallback | ArkTS 页面样式和其他业务 UI |
| GPU 输出/生命周期 | `avc420_gpu_compositor*`、`render_output_owner.*`、`surface_bridge.*` | OH_AVCodec、native buffer、worker、retained state、EndFrame、resize | 还未触发的问题分支 |

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
| C01 | `ohos_rdpgfx_bridge.c::freerdp_ohos_rdpgfx_bridge_attach` | 先保存 original callbacks，再替换 StartFrame/SurfaceCommand/EndFrame | CODE FACT |
| C02 | `ohos_rdpgfx_surface.c::ohos_rdpgfx_surface_command` | AVC420/444 未 consumed 时调用保存的 original SurfaceCommand | CODE FACT |
| C03 | `ohos_rdpgfx_avc444_policy.c::ohos_rdpgfx_record_avc420_gpu_candidate` | GPU 路径沿原生顺序校验；callback 未 ready 时保留 GDI | CODE FACT |
| C04 | `avc420_gpu_compositor_internal.cpp::ProcessCommand` | App GPU 路径直接调用 OH_AVCodec、native import 与 retained composite | CODE FACT |
| C05 | `PresentQueuedUpdate` / `PresentEndFrame` | pendingFrameId 只有在 matched EndFrame 才 present | CODE FACT |
| C06 | `libfreerdp/gdi/gfx.c::gdi_SurfaceCommand` | original GDI 仍分发 AVC420/444，并进入 H.264 subsystem | CODE FACT |
| C07 | 设备 frame trace | 本次运行是否真正经过目标 decoder→buffer→owner→EndFrame | RUNTIME REQUIRED |

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

完整源码流程、分支条件与任务推导见 [`../../case-materials/gpu/09-源码调用链与任务拆解.md`](../../case-materials/gpu/09-源码调用链与任务拆解.md)。
