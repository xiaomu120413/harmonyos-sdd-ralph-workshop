# 案例二视频槽位与拍摄脚本

## `gpu-cpu-stutter-before.mp4`（PENDING）

> 这是历史预留文件名，不代表 CPU 路径已经证实。未取得 path 日志前，页面标题只写“视频卡顿 before”。

- 20–30 秒，固定设备、服务端、网络、分辨率和同一视频片段。
- 画面同时出现远端视频、路径 diagnostics、CPU/FPS 或性能采样。
- 开头先展示 `software/OpenH264/FFmpeg fallback` 等真实路径证据；如果无法证明路径，只能命名为“卡顿现象”，不能写“CPU 解码导致”。
- 保留原始音频或时间码，便于观察画面节奏与丢帧。

## `gpu-hwdecode-after.mp4`（PENDING）

- 使用与 before 完全相同的场景和录制长度。
- 开头展示 `OHOS-AVCodec`、decoder name、codecId、分辨率。
- 结尾展示 CPU/FPS 采样摘要以及交互动作。
- 视频只证明可见现象；完整结论还必须绑定 `runtime-path.log`、性能 CSV 和 reviewer verdict。

## 可直接使用的现有素材

- `gpu-failure-black-screen-13s.mp4`：开发失败现象，不等于 CPU 卡顿基线。
- `gpu-validation-video-playback-16s.mp4`：可见播放/交互，不等于性能 A/B。
- `freerdp-stutter-scenario.jpeg`：卡顿场景静态说明，不能证明持续时间与 CPU 路径。

空白字段和当前媒体边界见 `../case-materials/gpu/00-证据状态总表.md`。
