# 媒体资源清单

所有媒体位于 `harmonyos-sdd-workshop-media/`。资源用于教学、复现和验收说明，不应把单张截图或一段流畅视频单独当成系统 PASS。

| 文件 | 类型 | 主要用途 | 注意事项 |
|---|---|---|---|
| `gpu-failure-black-screen-13s.mp4` | 视频 | 第 28/32 页黑屏故障带入 | 证明现象可复现，不直接证明根因 |
| `gpu-failure-black-screen-contact.jpg` | 图片 | 黑屏关键帧讨论 | 需结合 frame trace |
| `gpu-validation-video-playback-16s.mp4` | 视频 | 第 31/38 页动态验证 | 证明可见交互，不单独证明 owner/target 契约 |
| `gpu-validation-video-playback-contact.jpg` | 图片 | 视频、切换、遮挡关键帧 | 用于逐项核对场景覆盖 |
| `gpu-e2e-interaction-public.jpg` | 图片 | 打开内容、页面变化、右键交互 | 已排除连接信息阶段 |
| `gpu-connection-interaction-contact.jpg` | 图片 | 本地 V3 的完整连接与交互联系表 | 含连接信息，仅限 Private 仓库与内部授课 |
| `freerdp-stutter-scenario.jpeg` | 图片 | 建立真实远程 workload | 是场景图，不是卡顿根因证据 |
| `freerdp-frame-pacing.jpeg` | 图片 | frame pacing 讨论 | 单帧不能证明连续帧节奏 |
| `freerdp-render-queue.jpeg` | 图片 | 日志/诊断采集入口 | 含连接信息；不证明队列本身正确 |
| `freerdp-compositor-scale.jpeg` | 图片 | resize、retained/target 尺寸讨论 | 需要尺寸和生命周期日志佐证 |
| `nativebuffer-test-pattern.png` | 图片 | AVC420 NativeBuffer 阶段 | import 成功还需 EGLImage/OES 日志 |
| `rgba-renderer-test-pattern.png` | 图片 | RGBA retained 输出阶段 | dirty rect 还需 rect 外像素断言 |

## 未上传资源

本地 V3 使用的原始场景图和完整连接联系表已同步到 Private 仓库，同时保留 `gpu-e2e-interaction-public.jpg` 供脱敏投屏。原始图片不应转入公开仓库；其他本地长录屏若包含私网地址、用户名或私人窗口，也不应在未审查时上传。

## 建议证据元数据

后续新增截图或视频时，至少同时记录：

```text
runId
device / resolution
codec path
commit / package version
capture time range
PASS / FAIL / UNKNOWN scope
```
