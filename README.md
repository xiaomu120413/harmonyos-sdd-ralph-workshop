# 使用 AI 进阶能力实现较为复杂的需求

一套面向 HarmonyOS / ArkTS / C++ 工程师的实践课材料：以真实 MDM 需求、FreeRDP AVC420 / AVC444 GPU 送显问题和历史 AI Session 为载体，训练需求拆解、上下文工程、Workflow / Agent 分工、工具调用、长任务协同、实现验证和问题定位。

## 从这里开始

- [完整课程讲师稿（Rich V5 / 双案例工程闭环版）](harmonyos-sdd-ralph-workshop.md)
- [39 页课程结构重构稿](harmonyos-sdd-ralph-workshop-课程结构重构稿.md)
- [媒体资源清单](ASSET_MANIFEST.md)
- [T01 真实 RED/GREEN 证据](evidence/mdm/t01-red-green.md)
- [账号变化跨进程日志证据](evidence/mdm/account-cross-process-log.md)
- [Firewall 运行时回读证据边界](evidence/mdm/firewall-runtime-readback.md)
- [案例二：大代码库快速认知地图](evidence/gpu/01-codebase-map.md)
- [案例二：跨平台调研与最小穿刺](evidence/gpu/02-platform-research-and-spike.md)
- [案例二：任务、验收与排障证据链](evidence/gpu/03-task-acceptance-and-debug.md)
- [案例二独立工程交付包](case-materials/gpu/README.md)
- [案例二：证据状态总表（空白项与媒体边界）](case-materials/gpu/00-证据状态总表.md)
- [案例二：源码调用链与任务拆解主文档](case-materials/gpu/09-源码调用链与任务拆解.md)

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
- 远控案例采用 Context-first + Evidence-first：先用符号级调用链控制 55 万行级代码库的上下文；再区分“OHOS RDPGFX bridge → App GPU compositor”接管路径与“GDI → H264 subsystem”原生回退路径，通过最小能力穿刺、任务拆解和设备证据逐步收口。
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

> 当前状态：源码级调用链与任务拆分已完成文档修订；后续 PPT 以 `case-materials/gpu/09-源码调用链与任务拆解.md` 为 GPU 章节底稿。

主讲稿第 28–38 页已整理为一条可讲、可演示、可审计的案例二证据链：

- 从“用户报告远控视频卡顿”拆出已知事实、工作假设和 `PENDING / UNKNOWN` 指标；
- 用三个源码入口与一条真实 `SurfaceCommand → candidate policy → OH_AVCodec → pending → EndFrame present` 调用链控制上下文预算；
- 对照 FFmpeg / OpenH264、Windows Media Foundation、Android MediaCodec 与 HarmonyOS `OH_AVCodec`，提炼输入输出、生命周期、线程、资源 owner 和 fallback 契约；
- 用 `ADR-GPU-001` 固化调用链、Decision、Deferred 与 Fallback；
- 用 `SP-01..05` 定义最小能力穿刺；当前运行数据未入库的步骤保留 `PENDING`；
- 将实现拆成 `T-GPU-00..07` 八张任务卡，每张卡都有前置条件、修改范围、产物、验收步骤和失败回滚；
- 展示 Planner / Explorer / Implementer / Tool / Reviewer 的交付物式协同；
- 用黑屏失败视频说明 `Stop → Preserve → Locate → Falsify → Repair` 排障循环；
- 用同一 `frameId` 的 Trace Eval + Outcome Eval 证伪“调用成功即结果正确”；
- 补齐生命周期、队列背压、资源 owner、软件回退与长稳五类工程能力；
- 结尾按功能、架构、回退、性能分别输出 `PASS / FAIL / UNKNOWN`，不把未测性能包装为成功；
- 对应文档、任务卡、截图和两段视频均已收进仓库，并在 PPT 页面及备注 `[Sources]` 中建立映射。

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
├── case-materials/gpu/
│   ├── 00-证据状态总表.md
│   ├── 01-问题与基线.md
│   ├── 02-大代码库认知地图.md
│   ├── 03-跨平台实现调研.md
│   ├── 04-ADR-GPU-001-HarmonyOS硬解方案.md
│   ├── 05-最小能力穿刺计划.md
│   ├── 06-工程验收计划.md
│   ├── 07-开发排障复盘.md
│   ├── 08-PPT第28-38页内容映射.md
│   ├── 09-源码调用链与任务拆解.md
│   └── tasks/T-GPU-00..07.md
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
