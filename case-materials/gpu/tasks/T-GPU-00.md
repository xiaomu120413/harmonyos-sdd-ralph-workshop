# T-GPU-00｜保存视频卡顿 before 基线

- Source Scope：仅允许 diagnostics、日志、性能采样和录屏脚本；不得修改 `ohos_rdpgfx_*`、decoder、compositor 与 owner 状态机。
- Requirement：同一设备、服务端、分辨率和视频片段下保存 30 秒 before。
- Allowed：diagnostics、日志、性能采样、录屏脚本。
- Forbidden：修改 decoder、队列、renderer。
- Verify：视频、CPU/FPS、queue、codec path 属于同一 runId。
- PASS：路径和现象均可复现；否则 `UNKNOWN`。
- Stop：无法证明实际 decoder 时，先创建可观测性任务。
