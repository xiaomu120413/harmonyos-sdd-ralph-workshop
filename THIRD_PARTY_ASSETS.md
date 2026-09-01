# 第三方图片来源与使用记录

本文件记录课程中引用的第三方视觉素材。它解决“图片从哪里来、用于讲什么、公开发布前需要检查什么”，不代表仓库获得了素材的再许可。

## Anthropic 官方工程配图

| 本地文件 | 原始图片 URL | 文章页面 | 课程映射 |
|---|---|---|---|
| `harmonyos-sdd-workshop-media/anthropic/autonomous-agent-loop.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/58d9f10c985c4eb5d53798dea315f7bb5ab6249e-2401x1000.png) | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | 第 7 页：给 Loop 增加环境反馈与 Stop |
| `harmonyos-sdd-workshop-media/anthropic/coding-agent-flow.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/4b9a1f4eb63d5962a6e1746ac26bbc857cf3474f-2400x1666.png) | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | 第 4 页：从 Prompt 进入行动—观察—修正循环 |
| `harmonyos-sdd-workshop-media/anthropic/prompt-vs-context-engineering.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/faa261102e46c7f090a2402a49000ffae18c5dd6-2292x1290.png) | [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | 第 17 页：上下文筛选 |
| `harmonyos-sdd-workshop-media/anthropic/augmented-llm.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/d3083d3f40bb2b6f477901cc9a240738d3dd1371-2401x1000.png) | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | 第 25 页：Retrieval / Tools / Memory |
| `harmonyos-sdd-workshop-media/anthropic/evaluator-optimizer.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/14f51e6406ccb29e695da48b17017e899a6119c7-2401x1000.png) | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | 第 27 页：独立评审闭环 |
| `harmonyos-sdd-workshop-media/anthropic/eval-quality-layers.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/b77b8dbb7c2e57f063fbc8a087a853d5809b74b0-4584x2580.png) | [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) | 第 39 页：多层质量防线 |
| `harmonyos-sdd-workshop-media/anthropic/agent-evaluation-components.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/0205b36f9639fc27f2f6566f73cb56b06f59d555-4584x2580.png) | [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) | 附录 C10.4：Eval 对象映射 |
| `harmonyos-sdd-workshop-media/anthropic/orchestrator-workers.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/8985fc683fae4780fb34eab1365ab78c7e51bc8e-2401x1000.png) | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | 附录 D4：动态拆分与并行协同 |
| `harmonyos-sdd-workshop-media/anthropic/system-prompt-calibration.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/0442fe138158e84ffce92bed1624dd09f37ac46f-2292x1288.png) | [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | 第 10/17 页候选：系统提示词约束强度校准 |

## Anthropic 官方产品界面与产品矩阵

| 本地文件 | 原始图片 / 文档 URL | 课程映射 | 处理方式 |
|---|---|---|---|
| `harmonyos-sdd-workshop-media/methodology/anthropic-chat-code-cowork-matrix.png` | [Claude 产品矩阵 PDF](https://www-cdn.anthropic.com/files/4zrzovbb/website/34783bca828d7fa331f515ced26f1c9232151b2c.pdf) | 讲师备课：提炼三种通用工作形态 | 从官方 PDF 第 7 页渲染；为保持厂商中立，不直接进入正文 |
| `harmonyos-sdd-workshop-media/methodology/anthropic-claude-code-ide.png` | [Anthropic CDN](https://www-cdn.anthropic.com/images/4zrzovbb/website/3613f360926fae004521197488623465eb0cd751-1920x1035.png) | 第 3 页：仓库、终端、Diff、测试进入执行环境 | 保持真实界面，不重画终端内容 |
| `harmonyos-sdd-workshop-media/methodology/anthropic-cowork-browser.webp` | [Anthropic CDN](https://cdn.prod.website-files.com/6889473510b50328dbb70ae6/6a8f16a6d44c900b29172541_aacbc99c7782f29da95a30ed84ff0054_260826-Cowork-InAppBrowser-4K-Thumbnail_003.webp) | 第 3 页：目标委托、跨工具与较长任务 | 保持真实界面，裁切时保留任务区与执行区关系 |

## 使用边界

- 课程正文必须同时保留文章来源和中文读图说明，不能让图片看起来像本项目原创。
- 图片用于方法论讲解，不得作为 HarmonyOS、MDM、FreeRDP 或设备行为的事实证据。
- 当前 GitHub 仓库为 Private，定位为内部课程素材库。
- 转为公开仓库、公开课录像、商业培训或二次出版前，必须重新确认 Anthropic 的版权、品牌和再分发要求；无法确认时，改用自行绘制并经版权审查的中文图。
- 原有工程配图下载日期：2026-08-31（Asia/Shanghai）；产品矩阵与界面补充日期：2026-09-01（Asia/Shanghai）。

## 基于公开课程模型的自绘图

`harmonyos-sdd-workshop-media/methodology/ai-fluency-neutral.svg` 及其 PNG 导出版由本课程重新绘制，用于第 5 页“复杂需求中的 AI 熟练度成长模型”，方法参考 [Getting good at Claude: A research-backed curriculum](https://academy.claude.com/tutorials/getting-good-at-claude-a-research-backed-curriculum)。

自绘版没有复制原图的产品分区与功能节点，也不突出任何具体产品；仅保留以下可迁移教学结构：

- 先教授能带动后续能力的入口动作：对话式工具强调迭代，执行型与异步型 Agent 强调澄清目标；
- 描述能力从单次输入走向可复用与持久化配置；
- 辨别、质疑和验证必须贯穿每个层级并反复训练。

在公开课件中使用自绘图时，仍建议在讲师备注或资源页保留方法来源链接。

## 用户提供、来源待补的概念图

| 本地文件 | 课程映射 | 当前使用边界 |
|---|---|---|
| `harmonyos-sdd-workshop-media/methodology/user-prompt-context-harness-loop.png` | 第 3 页：Prompt、Context、Harness、Loop 的教学演进 | 只作为课堂组织视角，不能表述为统一行业标准；原图为 2:3，PPT 需裁切，不能整图缩小 |
| `harmonyos-sdd-workshop-media/methodology/user-naive-loop-engineering.png` | 第 7 页：让学员找出自主循环仍缺什么 | 按反例使用；图中的“自主完成、稳定可靠”不是课程结论，必须补外部事实、独立 AC、Stop、Escalate、权限边界和回滚点 |

两张图由用户在 2026-09-01 当前任务中提供，尚无原始页面、作者或授权信息。内部备课可暂存；公开发布、录像、商业培训或二次分发前必须补齐来源与使用许可，无法补齐时用可追溯的官方图或课程自有素材替换。
