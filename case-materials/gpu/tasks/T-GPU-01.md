# T-GPU-01｜真机选择 OHOS Hardware Decoder

- Source Scope：`app/common/src/main/cpp/channels/rdpgfx_pipeline.cpp::InstallRdpgfxDiagnosticsHooks`；`third_party/FreeRDP/client/OHOS/ohos_rdpgfx_bridge.c::freerdp_ohos_rdpgfx_bridge_attach`；`avc420_gpu_compositor_internal.cpp` 中 OH_AVCodec capability/create/configure/prepare/start。
- Requirement：OHOS RDPGFX GPU compositor 在目标设备实际创建并使用硬件类别 `OH_AVCodec` decoder；原生 `H264_CONTEXT_SUBSYSTEM / GDI` 仍保持可回退。
- Allowed：build flags、subsystem registration、capability/selection logs。
- Forbidden：修改合成、Surface owner、AVC444。
- RED/Probe：当前运行日志无法证明 decoder 或显示 software fallback。
- Verify：decoder name、capability、configure/start 与 fallback reason。
- Stop：设备不提供硬件 H.264 能力则返回 `UNKNOWN/UNSUPPORTED`。
- Exit Gate：必须同时证明 bridge 符号进入 arm64/HAP 产物、attach 成功、真机 decoder identity 为硬件类别；只看到 API 调用或编译成功不能进入 T02。
