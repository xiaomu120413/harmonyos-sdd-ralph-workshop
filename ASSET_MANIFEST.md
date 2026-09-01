# 媒体资源清单

所有媒体位于 `harmonyos-sdd-workshop-media/`。资源用于教学、复现和验收说明，不应把单张截图或一段流畅视频单独当成系统 PASS。

| 文件 | 类型 | 主要用途 | 注意事项 |
|---|---|---|---|
| `mdm/firewall-domain-rule-created.jpeg` | 真机截图 | 第 1 页建立“UI 可见不等于系统事实”的开场冲突 | 能证明域名规则出现在页面；不能证明所有账号的系统 policy/rules 已正确下发 |
| `gpu-failure-black-screen-13s.mp4` | 视频 | 第 28/32 页黑屏故障带入 | 证明现象可复现，不直接证明根因 |
| `gpu-failure-black-screen-contact.jpg` | 图片 | 黑屏关键帧讨论 | 需结合 frame trace |
| `gpu-validation-video-playback-16s.mp4` | 视频 | 第 31/38 页动态验证 | 证明可见交互，不单独证明 owner/target 契约 |
| `gpu-cpu-stutter-before.mp4` | 视频槽位 / PENDING | 第 28 页案例二原始问题：视频经 CPU 路径播放卡顿 | 当前仓库没有能同时证明卡顿、CPU 占用和协商路径的原始视频；拍摄脚本见 `harmonyos-sdd-workshop-media/VIDEO_TODO.md` |
| `gpu-hwdecode-after.mp4` | 视频槽位 / PENDING | 第 38 页硬解方案最终 A/B 验收 | 必须与 before 使用同一设备、分辨率、服务端和片段，并同时保存性能采样与路径日志 |
| `gpu-validation-video-playback-contact.jpg` | 图片 | 视频、切换、遮挡关键帧 | 用于逐项核对场景覆盖 |
| `gpu-e2e-interaction-public.jpg` | 图片 | 打开内容、页面变化、右键交互 | 已排除连接信息阶段 |
| `gpu-connection-interaction-contact.jpg` | 图片 | 本地 V3 的完整连接与交互联系表 | 含连接信息，仅限 Private 仓库与内部授课 |
| `freerdp-stutter-scenario.jpeg` | 图片 | 建立真实远程 workload | 是场景图，不是卡顿根因证据 |
| `freerdp-frame-pacing.jpeg` | 图片 | frame pacing 讨论 | 单帧不能证明连续帧节奏 |
| `freerdp-render-queue.jpeg` | 图片 | 日志/诊断采集入口 | 含连接信息；不证明队列本身正确 |
| `freerdp-compositor-scale.jpeg` | 图片 | resize、retained/target 尺寸讨论 | 需要尺寸和生命周期日志佐证 |
| `nativebuffer-test-pattern.png` | 图片 | AVC420 NativeBuffer 阶段 | import 成功还需 EGLImage/OES 日志 |
| `rgba-renderer-test-pattern.png` | 图片 | RGBA retained 输出阶段 | dirty rect 还需 rect 外像素断言 |

## Anthropic 方法论配图

以下图片位于 `harmonyos-sdd-workshop-media/anthropic/`，均来自 Anthropic 官方工程文章。它们用于解释方法，不作为 HarmonyOS 项目事实或产品验收证据。

| 文件 | 对应页面 | 课堂用途 | 官方来源 |
|---|---|---|---|
| `autonomous-agent-loop.png` | 第 3 页 | Human / Agent / Environment / Stop | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `coding-agent-flow.png` | 第 13 页 | 澄清、上下文、搜索、修改与测试时序 | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `prompt-vs-context-engineering.png` | 第 17 页 | 从单轮 Prompt 转向动态上下文筛选 | [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) |
| `augmented-llm.png` | 第 25 页 | Retrieval / Tools / Memory 的职责边界 | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `evaluator-optimizer.png` | 第 27 页 | 实现与独立评估的反馈闭环 | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `eval-quality-layers.png` | 第 39 页 | 多层验证共同拦截漏检 | [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) |
| `agent-evaluation-components.png` | 附录 I4 | Task / Trial / Trajectory / Outcome / Grader | [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) |
| `orchestrator-workers.png` | 附录 J4 | 解释什么时候适合动态拆分和并行协同 | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |

## 文本证据资产

| 文件 | 对应页面 | 状态 | 用途 |
|---|---:|---|---|
| `evidence/mdm/t01-red-green.md` | 15–17 | READY | 真实 RED/GREEN、commit、命令、用例结果与退出码陷阱；课前可直接转图 |
| `evidence/mdm/account-cross-process-log.md` | 18、24 | READY / eventId GAP | 真机日志已脱敏；明确历史无统一 eventId，不补造四段完整链路 |
| `evidence/mdm/firewall-runtime-readback.md` | 25–26 | READY / bridge TARGET | 证明 App 内部 getter 已存在、验收 bridge 缺失、系统结论仍为 UNKNOWN |
| `evidence/gpu/01-codebase-map.md` | 28–29 | READY | 记录 55.9 万行级源码规模、分层地图、入口和按需上下文策略 |
| `evidence/gpu/02-platform-research-and-spike.md` | 30–32 | READY / device evidence GAP | 对照软件、Windows、Android 与 HarmonyOS H.264 后端，并定义最小穿刺证据 |
| `evidence/gpu/03-task-acceptance-and-debug.md` | 33–38 | READY / performance video GAP | 任务树、验收矩阵、AI 正误判定、故障处理与最终证据包 |

版权归原作者 / Anthropic 所有。当前仓库仅按内部教学参考保存；公开发布、再分发或商业课件使用前，应重新确认授权、品牌与版权要求。详细记录见 `THIRD_PARTY_ASSETS.md`。

## 厂商中立方法论主视觉

| 文件 | 类型 | 主要用途 | 说明 |
|---|---|---|---|
| `methodology/ai-fluency-neutral.png` | PNG | 附录 I5 与可选讲师参考 | 16:9 中文投屏版，不出现具体 AI 产品 |
| `methodology/ai-fluency-neutral.svg` | SVG | PPT、讲义和后续可编辑版本 | 本课程自绘；方法参考 Anthropic Academy 的 AI Fluency Curriculum |

这张图保留“入口动作、描述能力耐久度、辨别力贯穿每一层”的教学结构，去除了具体产品名称和功能罗列。图片本身为课程自绘，文章来源仍应在讲义或备注中保留。

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
