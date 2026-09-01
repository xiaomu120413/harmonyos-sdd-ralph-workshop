# T-GPU-02｜AVC420 一帧合法输出

- Source Scope：`app/common/src/main/cpp/surface/avc420_gpu_compositor_internal.cpp` 中 input/output buffer、PTS、output description、`OH_AVBuffer_GetNativeBuffer`；不修改 owner、EndFrame 与 queue policy。
- Requirement：一个已知 AVC420 sample 产生可验证 output。
- Allowed：OHOS codec buffer adapter、format/stride/plane diagnostics。
- Forbidden：AVC444、zero-copy、队列重构。
- Verify：frameId、input bytes、output size/format/stride/planes、bounded wait。
- PASS：output 合法且能关联输入；callback 成功但内容不明为 `UNKNOWN`。
- Stop：输出格式不在支持集合时保留原始描述并 fallback。
- Exit Gate：输入 bytes、input PTS、output PTS、size/format 与 native buffer 必须可关联到同一 frameId；“有 output callback”不足以进入 T03。
