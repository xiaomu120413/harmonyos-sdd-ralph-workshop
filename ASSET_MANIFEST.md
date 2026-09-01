# 媒体资源清单

所有媒体位于 `harmonyos-sdd-workshop-media/`。资源用于教学、复现和验收说明，不应把单张截图或一段流畅视频单独当成系统 PASS。

| 文件 | 类型 | 主要用途 | 注意事项 |
|---|---|---|---|
| `e2e/peripheral-policy-current.png` | 真机截图 | 第 1/6/8/9 页建立“外设策略 UI 可见不等于 MDM 与实物事实”的冲突 | 证明外设策略页面在该次运行中的可见状态；系统策略事实仍需 restrictions/usbManager 回读和 USB 实物证据 |
| `mdm/firewall-domain-rule-created.jpeg` | 历史真机截图 / LEGACY | 不再用于当前 MDM 主案例 | 保留旧案例资产，不进入外设管理页稿 |
| `mdm/firewall-duplicate-rule-failure.jpeg` | 历史真机截图 / LEGACY | 不再用于当前 MDM 主案例 | 保留旧案例资产，不进入外设管理页稿 |
| `e2e/security-log-menu-open.jpeg` | 真机截图 | 可观测性、审计日志与菜单交互样例 | 含真实审计日志行，投屏前仍应检查设备名、账号和时间信息 |
| `e2e/e2e-runner-current-reference.jpg` | 架构参考图 | E2E 两页预留内容的结构输入 | 用户提供的当前版本；结构可用但版式需重绘，不作为最终视觉或运行事实证据 |
| `gpu-failure-black-screen-13s.mp4` | 视频 | 第 28/32 页黑屏故障带入 | 只证明该媒体中可见黑屏，不直接证明根因或与卡顿报告属于同一 run |
| `gpu-failure-black-screen-contact.jpg` | 图片 | 黑屏关键帧讨论 | 需结合 frame trace |
| `gpu-validation-video-playback-16s.mp4` | 视频 | 第 31/38 页媒体观察 | 只证明该媒体中可见播放/部分交互；未绑定 commit/runId，不证明硬解、性能或 owner/target 契约 |
| `gpu-cpu-stutter-before.mp4` | 视频槽位 / PENDING | 第 28 页：待采集用户报告的卡顿现象；文件名中的 CPU 不是已证事实 | 当前仓库没有能同时证明卡顿、CPU 占用和协商路径的原始视频；拍摄脚本见 `harmonyos-sdd-workshop-media/VIDEO_TODO.md` |
| `gpu-hwdecode-after.mp4` | 视频槽位 / PENDING | 第 38 页硬解方案最终 A/B 验收 | 必须与 before 使用同一设备、分辨率、服务端和片段，并同时保存性能采样与路径日志 |
| `gpu-validation-video-playback-contact.jpg` | 图片 | 视频、切换、遮挡关键帧 | 用于逐项核对场景覆盖 |
| `gpu-e2e-interaction-public.jpg` | 图片 | 打开内容、页面变化、右键交互 | 只证明关键帧中可见这些结果，不证明完整输入路径 |
| `gpu-connection-interaction-contact.jpg` | 图片 | 本地 V3 的完整连接与交互联系表 | 含连接信息；当前仓库为 Public，公开使用前必须再次审查与脱敏 |
| `freerdp-stutter-scenario.jpeg` | 图片 | 建立真实远程 workload | 是场景图，不是卡顿根因证据 |
| `freerdp-frame-pacing.jpeg` | 图片 | frame pacing 讨论 | 单帧不能证明连续帧节奏 |
| `freerdp-render-queue.jpeg` | 图片 | 日志/诊断采集入口 | 含连接信息；不证明队列本身正确 |
| `freerdp-compositor-scale.jpeg` | 图片 | resize、retained/target 尺寸讨论 | 需要尺寸和生命周期日志佐证 |
| `nativebuffer-test-pattern.png` | 图片 | AVC420 NativeBuffer 阶段 | import 成功还需 EGLImage/OES 日志 |
| `rgba-renderer-test-pattern.png` | 图片 | RGBA retained 输出阶段 | dirty rect 还需 rect 外像素断言 |

案例二媒体与空白证据的统一状态见 `case-materials/gpu/00-证据状态总表.md`。

## Anthropic 方法论配图

以下图片位于 `harmonyos-sdd-workshop-media/anthropic/`，均来自 Anthropic 官方工程文章。它们用于解释方法，不作为 HarmonyOS 项目事实或产品验收证据。

| 文件 | 对应页面 | 课堂用途 | 官方来源 |
|---|---|---|---|
| `autonomous-agent-loop.png` | 第 7 页 | 给受控 Ralph 解释环境反馈与 Stop | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `coding-agent-flow.png` | 第 4 页 | 从 Prompt 进入行动、观察与修正循环 | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `prompt-vs-context-engineering.png` | 第 17 页 | 从单轮 Prompt 转向动态上下文筛选 | [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) |
| `augmented-llm.png` | 第 25 页 | Retrieval / Tools / Memory 的职责边界 | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `evaluator-optimizer.png` | 第 27 页 | 实现与独立评估的反馈闭环 | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `eval-quality-layers.png` | 第 39 页 | 多层验证共同拦截漏检 | [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) |
| `agent-evaluation-components.png` | 附录 I4 | Task / Trial / Trajectory / Outcome / Grader | [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) |
| `orchestrator-workers.png` | 附录 J4 | 解释什么时候适合动态拆分和并行协同 | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) |
| `system-prompt-calibration.png` | 第 10/17 页候选 | 解释系统提示词应在约束强度、可复用性与自主性之间校准 | [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) |

## Anthropic 产品方式与趋势配图

以下图片位于 `harmonyos-sdd-workshop-media/methodology/`，用于讲师研究工作形态和素材溯源。为保持厂商中立，它们不直接进入第 2–3 页正文，也不作为本地项目运行证据。

| 文件 | 对应页面 | 课堂用途 | 官方来源 |
|---|---|---|---|
| `anthropic-chat-code-cowork-matrix.png` | 讲师备课 / 来源存档 | 研究对话式协作、仓库内 Agent、异步任务代理三种任务形态；不进入正文 | [Claude 产品矩阵 PDF](https://www-cdn.anthropic.com/files/4zrzovbb/website/34783bca828d7fa331f515ced26f1c9232151b2c.pdf) |
| `anthropic-claude-code-ide.png` | 第 3 页备选 / 讲师参考 | 展示 AI 进入仓库与终端后的真实执行界面 | [Anthropic 官方 CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/3613f360926fae004521197488623465eb0cd751-1920x1035.png) |
| `anthropic-cowork-browser.webp` | 第 3 页备选 / 讲师参考 | 展示目标委托和跨工具长任务的真实界面 | [Claude Cowork](https://claude.com/product/cowork) |

## 文本证据资产

| 文件 | 对应页面 | 状态 | 用途 |
|---|---:|---|---|
| `case-materials/mdm/00-外设管理证据状态总表.md` | 9–27 | READY | 当前外设结论、证据等级、E2E 结果与必须补拍的 D6/D7 证据 |
| `evidence/mdm/peripheral-e2e-summary.md` | 23–25 | READY / artifacts GAP | 外设 E2E PASS 与旧 FAIL 的自包含摘要，明确 UI 证据边界 |
| `evidence/mdm/peripheral-code-test-evidence.md` | 10–25 | READY / device GAP | 关键代码锚点、覆盖率、提交演进与不能证明的事实 |
| SecurityTool `scripts/e2e/results/PER-*.json` | 23–25 | READY / artifacts GAP | 多条外设 UI E2E PASS 与一条旧 FAIL；JSON 的 artifacts 为空，不补造截图或视频 |
| SecurityTool `entry/.test/default/outputs/test/reports/coverageReport.json` | 21–25 | READY | 2026-09-01 本地覆盖率；只证明执行面，不等于系统验收 |
| `evidence/mdm/t01-red-green.md` | LEGACY | 历史旧案例证据，不进入当前外设主线 |
| `evidence/mdm/account-cross-process-log.md` | LEGACY | 历史旧案例证据，不进入当前外设主线 |
| `evidence/mdm/firewall-runtime-readback.md` | LEGACY | 历史旧案例证据，不进入当前外设主线 |
| `evidence/gpu/01-codebase-map.md` | 28–29 | READY | 记录 55.9 万行级源码规模、分层地图、入口和按需上下文策略 |
| `evidence/gpu/02-platform-research-and-spike.md` | 30–32 | READY / device evidence GAP | 对照软件、Windows、Android 与 HarmonyOS H.264 后端，并定义最小穿刺证据 |
| `evidence/gpu/03-task-acceptance-and-debug.md` | 33–38 | READY / performance video GAP | 任务树、验收矩阵、AI 正误判定、故障处理与最终证据包 |

版权归原作者 / Anthropic 所有。当前仓库仅按内部教学参考保存；公开发布、再分发或商业课件使用前，应重新确认授权、品牌与版权要求。详细记录见 `THIRD_PARTY_ASSETS.md`。

## 厂商中立方法论主视觉

| 文件 | 类型 | 主要用途 | 说明 |
|---|---|---|---|
| `methodology/ai-fluency-neutral.png` | PNG | 第 5 页：产品形态演进后，人需要提升的复杂需求能力 | 16:9 中文投屏版，不出现具体 AI 产品；正文只使用一次 |
| `methodology/ai-fluency-neutral.svg` | SVG | PPT、讲义和后续可编辑版本 | 本课程自绘；方法参考 Anthropic Academy 的 AI Fluency Curriculum |
| `methodology/user-prompt-context-harness-loop.png` | 用户提供概念图 | 第 3 页：Prompt → Context → Harness → Controlled Loop | 原图 2:3；落版必须裁切并重写可见结论；来源与授权待补，不作为行业标准证据 |
| `methodology/user-naive-loop-engineering.png` | 用户提供概念图 | 第 7 页：课堂找错案例 | 故意按“表面完整但工程不闭合”讲；来源与授权待补，不作为 Ralph 官方结构 |

其中 `ai-fluency-neutral` 保留“入口动作、描述能力耐久度、辨别力贯穿每一层”的教学结构，去除了具体产品名称和功能罗列。第 5 页讲师备注必须补充原文边界：对话式工具的入口动作是迭代，执行型与异步型 Agent 的入口动作是澄清目标。图片本身为课程自绘，文章来源仍应在讲义或备注中保留。两张 `user-*` 图片的来源状态和公开使用边界见 `THIRD_PARTY_ASSETS.md`。

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
