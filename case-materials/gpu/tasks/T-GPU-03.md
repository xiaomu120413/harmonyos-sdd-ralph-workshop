# T-GPU-03｜一帧进入正确显示 Owner

- Source Scope：`ohos_rdpgfx_avc444_policy.c::ohos_rdpgfx_record_avc420_gpu_candidate`、`Avc420GpuCompositor::OnSurfaceCommand`、`ProcessCommand`、`PresentEndFrame` 与 `PresentQueuedUpdate`。
- Requirement：T02 输出沿既有 bridge 到达唯一 render owner，并在正确边界 present。
- Allowed：bridge、owner、present diagnostics 与最小接线。
- Forbidden：性能优化、AVC444 LC、输入系统。
- Verify：同一 frameId 的 decoder output、owner claim、target generation、EndFrame present。
- Stop：GDI/GPU 双写或旧 target 未拒绝时判 FAIL。
- Exit Gate：必须看到 callbackReady、owner claim、pendingFrameId、matched EndFrame、PresentComposite 属于同一 frame；decode 后直接 swap 不得判 PASS。
