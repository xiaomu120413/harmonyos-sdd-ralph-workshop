# ADR-GPU-001｜HarmonyOS H.264 硬解接入方案

- 状态：Accepted for spike
- 适用：FreeRDP HarmonyOS client
- 依据：代码认知地图、跨平台实现调研、当前 OHOS adapter

## Context

软件 H.264 路径可作为正确性 fallback，但用户场景出现视频卡顿。目标是在不破坏 FreeRDP 协议语义和原生回退的前提下接入 HarmonyOS 硬件 decoder。

## Decision

1. 在 `H264_CONTEXT_SUBSYSTEM` 契约内接入 `OH_AVCodec`。
2. 第一条穿刺选择 AVC420 buffer 硬解，不直接实现完整 AVC444 GPU compositor。
3. 只有当前 command 的 decode、layout、state 和 pending present 完成后，才允许新路径消费该 command。
4. 失败必须显式记录原因并回到软件/原生 GDI；禁止静默黑屏。
5. App 层只保留 UI、权限、NativeWindow 与生命周期交接；FreeRDP OHOS 层拥有协议和 codec policy。

## 方案调用链

```text
RDPGFX SurfaceCommand
→ FreeRDP H264 subsystem
→ OH_AVCodec hardware decoder
→ validated AVBuffer / decoded planes
→ AVC420 composition or AVC444 retained state
→ RenderOutputOwner
→ EndFrame present
```

## Alternatives

| 方案 | 结论 | 原因 |
|---|---|---|
| 在 App 旁路收到 H.264 字节直接送 Surface | Reject | 绕开 dirty rect、AVC444 LC、EndFrame 和 fallback |
| 先实现完整 zero-copy AVC444 | Defer | 同时引入 codec、plane、owner、surface、shader 变量 |
| 永久使用软件解码 | Keep as fallback | 正确性可兜底，但不能满足性能目标 |

## Deferred

- AVC444 LC / 单 decoder / retained state。
- zero-copy 优化。
- resize、后台、重连的完整长稳。
- 多设备 decoder 兼容矩阵。

## 进入实现的证据门

只有最小穿刺 `SP-01..05` 全部有结论，才能把方案拆成完整任务；否则 ADR 仍是方向，不是已完成设计。

