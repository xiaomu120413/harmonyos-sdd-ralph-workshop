# T-GPU-06｜AVC444 帧语义闭合

- Requirement：按 LC 更新 luma/chroma retained state，使用单一 decoder 状态并在 EndFrame present。
- Allowed：AVC444 专用 decoder/compositor/policy 与 frame trace。
- Forbidden：直接套用 AVC420 dirty-rect 或双 decoder 普通视频假设。
- Verify：LC、stream 顺序、readiness、single decoder、command consumed、EndFrame。
- Stop：任一 retained 前提未知时不 suppress 原生 GDI。

