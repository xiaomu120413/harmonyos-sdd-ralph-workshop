# 使用 AI 进阶能力实现较为复杂的需求

一套面向 HarmonyOS / ArkTS / C++ 工程师的实践课材料：以真实 MDM 需求、FreeRDP AVC420 / AVC444 GPU 送显问题和历史 AI Session 为载体，训练需求拆解、上下文工程、Workflow / Agent 分工、工具调用、长任务协同、实现验证和问题定位。

## 从这里开始

- [完整课程讲师稿（Rich V5 / 双案例工程闭环版）](harmonyos-sdd-ralph-workshop.md)
- [媒体资源清单](ASSET_MANIFEST.md)
- [T01 真实 RED/GREEN 证据](evidence/mdm/t01-red-green.md)
- [账号变化跨进程日志证据](evidence/mdm/account-cross-process-log.md)
- [Firewall 运行时回读证据边界](evidence/mdm/firewall-runtime-readback.md)
- [案例二：大代码库快速认知地图](evidence/gpu/01-codebase-map.md)
- [案例二：跨平台调研与最小穿刺](evidence/gpu/02-platform-research-and-spike.md)
- [案例二：任务、验收与排障证据链](evidence/gpu/03-task-acceptance-and-debug.md)
- [原始大纲](archive/harmonyos-sdd-ralph-workshop-original.md)
- [Rich V2 归档](archive/harmonyos-sdd-ralph-workshop-rich-v2.md)

## 课程主线

```text
模糊需求
→ 可测试规格
→ 有边界任务
→ RED / GREEN
→ 设备与系统事实
→ PASS / FAIL / UNKNOWN
```

- MDM 主实践采用 Feature-first：先冻结多用户、状态、事务和失败语义，再实现。
- 远控案例采用 Context-first + Evidence-first：先用分层地图控制 55 万行级代码库的上下文，再沿其他平台的 H.264 子系统契约映射 HarmonyOS `OH_AVCodec`，通过最小能力穿刺、任务拆解和设备证据逐步收口。
- MCP 负责构建、安装、设备操作、日志、截图与证据保存，不负责替代工程判断。

每项能力都按同一教学闭环展开：

```text
真实问题 → AI 初判 → 反证 → 学员操作 → 验收 → 迁移规则
```

## 六项 AI 进阶能力

- 需求澄清与边界冻结：先把歧义、状态和失败语义变成可测试规格。
- 上下文工程：只加载当前任务所需的最小高信号材料，用结构化记录跨会话交接。
- Workflow / Agent 分工：确定步骤固化为工作流，开放判断交给 Agent。
- 工具契约与环境事实：让 MCP 返回原因码、对象标识和证据路径，而不只是大段原始输出。
- Eval 与独立反证：同时检查执行轨迹与最终系统状态，输出 `PASS / FAIL / UNKNOWN`。
- 长任务 Harness 与协同：通过小任务、progress、Git checkpoint、证据包和 Reviewer 持续推进。

课程不把“更多 Agent”视为进阶：单文件低风险任务优先单 Agent，确定步骤固化为 Workflow，跨进程/事务/GPU 等高风险任务才增加 Planner、Implementer 与独立 Reviewer。

方法论参考 Anthropic 的 [Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents)、[Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)、[Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) 与 [Harness Design for Long-Running Application Development](https://www.anthropic.com/engineering/harness-design-long-running-apps)，并在主文档附录 I 中给出逐项工程映射。

附录轻量参考 Anthropic Academy 的研究型 AI Fluency 课程模型，用于提醒讲师“先澄清并迭代、每一步训练辨别”；它不构成课程主方法论。课程仍围绕需求拆解、开发、验证、问题定位与协同闭环展开，不围绕任何具体产品功能。

## GPU 章节亮点

第 28–38 页包含：

- 黑屏故障录屏与逐帧观察；
- AVC420 / AVC444 双路径和两层三套 OH_AVCodec 接入；
- `frameId` 同帧 trace 与最早异常定位；
- 8×4 NV12/NV21 plane math；
- AVC420 dirty rect、可信背景与 retained RGBA FBO；
- 白帧 readiness gate 复盘；
- worker queue、EndFrame、LC readiness 与 Surface 生命周期；
- 人—AI—MCP—Reviewer 协同脚本；
- 8 张 Anthropic 官方方法论配图及逐图中文讲解；
- 6 组真实 Session 证据卡：延迟补丁、跨进程、MCP 四层结果、乐观 UI、截图误判与修改越界；
- 四线闭环审计：方法论、具体需求、演示证据和问题处理必须在同一案例链路相遇；
- 案例二完整工程叙事：30 万行级以上开源库背景、CPU 视频卡顿、代码认知、跨平台调研、HM 方案、最小穿刺、任务验收、开发排障与最终证据；
- 28、45、60–75 分钟三种讲授模式。

## 目录

```text
.
├── README.md
├── harmonyos-sdd-ralph-workshop.md
├── ASSET_MANIFEST.md
├── THIRD_PARTY_ASSETS.md
├── evidence/mdm/
│   ├── t01-red-green.md
│   ├── account-cross-process-log.md
│   └── firewall-runtime-readback.md
├── evidence/gpu/
│   ├── 01-codebase-map.md
│   ├── 02-platform-research-and-spike.md
│   └── 03-task-acceptance-and-debug.md
├── archive/
│   ├── harmonyos-sdd-ralph-workshop-original.md
│   └── harmonyos-sdd-ralph-workshop-rich-v2.md
└── harmonyos-sdd-workshop-media/
    ├── anthropic/*.png
    ├── methodology/*.png / *.svg
    ├── mdm/*.jpeg
    ├── *.jpeg / *.jpg / *.png
    └── *.mp4
```

## 使用建议

1. 先阅读主文档的“课程定位”和 39 页大纲。
2. 保持原 120 分钟时，GPU 段使用附录 K 的 28 分钟核心迁移版。
3. 技术分享或半天 workshop，可使用 45 分钟演示版或 60–75 分钟分组实操版。
4. 公开投屏前再次检查设备、账号、地址和私人窗口信息。
5. 视频与截图只作可见行为证据；最终结论仍按日志、系统状态和验收契约判定。

## 内容说明

本仓库暂不附加开源许可证。课程案例包含项目实现事实和内部教学编排，默认仅按仓库访问权限使用；如需公开发布或再分发，请先完成代码归属、媒体版权和脱敏审查。
