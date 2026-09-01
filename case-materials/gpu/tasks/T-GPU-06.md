# T-GPU-06｜AVC444 帧语义闭合

- Source Scope：`ohos_rdpgfx_avc444_policy.c::ohos_rdpgfx_record_avc444_gpu_candidate`、AVC444 compositor/decoder policy、LC/stream retained state 与 EndFrame；不得直接复用 AVC420 的单流 dirty-rect 假设。
- Requirement：按 LC 更新 luma/chroma retained state，使用单一 decoder 状态并在 EndFrame present。
- Allowed：AVC444 专用 decoder/compositor/policy 与 frame trace。
- Forbidden：直接套用 AVC420 dirty-rect 或双 decoder 普通视频假设。
- Verify：LC、stream 顺序、readiness、single decoder、command consumed、EndFrame。
- Stop：任一 retained 前提未知时不 suppress 原生 GDI。
- Exit Gate：同一 frame trace 必须包含 LC、两个 bitstream 的处理顺序、single-decoder identity、retained readiness、consumed 与 matched EndFrame。
