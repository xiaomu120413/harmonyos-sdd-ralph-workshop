# 媒体资源清单

所有媒体位于 `harmonyos-sdd-workshop-media/`。资源用于教学、复现和验收说明，不应把单张截图或一段流畅视频单独当成系统 PASS。

| 文件 | 类型 | 主要用途 | 注意事项 |
|---|---|---|---|
| `gpu-failure-black-screen-13s.mp4` | 视频 | 第 28/32 页黑屏故障带入 | 证明现象可复现，不直接证明根因 |
| `gpu-failure-black-screen-contact.jpg` | 图片 | 黑屏关键帧讨论 | 需结合 frame trace |
| `gpu-validation-video-playback-16s.mp4` | 视频 | 第 31/38 页动态验证 | 证明可见交互，不单独证明 owner/target 契约 |
| `gpu-validation-video-playback-contact.jpg` | 图片 | 视频、切换、遮挡关键帧 | 用于逐项核对场景覆盖 |
| `gpu-e2e-interaction-public.jpg` | 图片 | 打开内容、页面变化、右键交互 | 已排除连接信息阶段 |
| `nativebuffer-test-pattern.png` | 图片 | AVC420 NativeBuffer 阶段 | import 成功还需 EGLImage/OES 日志 |
| `rgba-renderer-test-pattern.png` | 图片 | RGBA retained 输出阶段 | dirty rect 还需 rect 外像素断言 |

## 未上传资源

未脱敏的连接信息联系表没有进入仓库，已由 `gpu-e2e-interaction-public.jpg` 替代。带私网地址、用户名、完整浏览器标签/书签的原始截图也保留为本地素材，不进入 Git 历史。其他本地长录屏若包含私网地址、用户名或私人窗口，也不应在未审查时上传。

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
