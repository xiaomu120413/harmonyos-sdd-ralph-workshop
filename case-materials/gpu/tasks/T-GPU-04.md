# T-GPU-04｜连续播放队列有界

- Requirement：连续视频输入、输出和 present 队列有界，不产生持续增长延迟。
- Allowed：queue policy、bounded wait、drop/backpressure metrics。
- Forbidden：通过无界线程或清 cache 掩盖顺序问题。
- Verify：queue depth、age p95、drop reason、frame order、CPU/FPS。
- Stop：backlog 持续增长或同步越序时回到 first-abnormal。

