# T-GPU-04｜连续播放队列有界

- Source Scope：`Avc420GpuCompositor::EnqueueSurfaceCommand`、worker backlog/compaction、EndFrame coalescing 与队列指标；不得用无界线程替代 backpressure。
- Requirement：连续视频输入、输出和 present 队列有界，不产生持续增长延迟。
- Allowed：queue policy、bounded wait、drop/backpressure metrics。
- Forbidden：通过无界线程或清 cache 掩盖顺序问题。
- Verify：queue depth、age p95、drop reason、frame order、CPU/FPS。
- Stop：backlog 持续增长或同步越序时回到 first-abnormal。
- Exit Gate：至少证明 queue depth、command age p95、drop/compaction reason 与 frame order；仅看平均 FPS 不足以进入 T05。
