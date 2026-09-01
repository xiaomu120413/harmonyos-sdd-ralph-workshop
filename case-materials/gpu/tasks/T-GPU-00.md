# T-GPU-00｜保存 CPU 路径与视频卡顿 before 基线

- Source Scope：仅允许 diagnostics、日志、性能采样和录屏脚本；不得修改 `ohos_rdpgfx_*`、decoder、compositor 与 owner 状态机。
- Requirement：同一设备、服务端、分辨率和视频片段下保存 30 秒 before，并把案例已知的 CPU/软件路径转换成可展示的当前 run 日志。
- Allowed：diagnostics、日志、性能采样、录屏脚本。
- Forbidden：修改 decoder、队列、renderer。
- Verify：视频、CPU/FPS、queue、codec path 属于同一 runId。
- PASS：卡顿现象、CPU/software path 与基线指标在同一 runId 中可复现；缺任一证据则 `PENDING/UNKNOWN`。
- Stop：无法证明实际 decoder 时，先创建可观测性任务。
