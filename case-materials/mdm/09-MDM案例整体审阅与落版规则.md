# MDM 案例整体审阅与落版规则

## 1. 整体评价

当前 P9–25 已经形成完整学习曲线：先管理输入，再做决策，随后用 RFC 固化系统不变量，用 Story/Ralph 控制执行，最后用 E2E 和系统事实验收。代码层也能从 `onAccountAdded()` 穿刺到 `netFirewall`，再从 Case JSON 穿刺到 `CaseResult`。

最有培训价值的不是“方案最终跑通”，而是三次认知升级：

1. AI 修正版可能把产品模式和平台规则类型混为一谈，说明专业表述不等于正确决策。
2. 账号事件先于完整列表可见，说明事件不等于事实，必须用稳定条件穿刺。
3. 816 个 UT 全绿后仍能从 RFC 反查出 apply-state 原子性和回滚缺口，说明 AI 完成不等于交付。

## 2. 当前已达到的深度

- 需求层：有原始 Excel 坐标、AI 修正版冲突、PRD 决策和代码/test 映射。
- 方案层：有四方案、失败模式、稳定快照穿刺和 ADR 重开条件。
- 架构层：有后台业务闭环、前台展示闭环、系统 API 和状态写入门。
- 实现层：有函数契约、public/private/custom 分支、快照与补偿式回滚。
- 迭代层：有九个真实 Git 提交、R6/R7 文档先行和 Reviewer verdict。
- 验收层：有 816/816、E2E 三态、证据边界和设备 UNKNOWN。

## 3. 仍需保留为占位的真实证据

以下内容当前没有足够本地事实，不能用重绘图或合成日志冒充：

- Excel 原始第 4 行与 AI 修正版第 4 行的并排截图。
- 真机 `onAccountAdded(123)` 与旧账号集合的同时间轴原始日志。
- HAP 构建、签名、安装、管理员激活的同一 run 证据。
- 新账号最终 `getNetFirewallPolicy/getNetFirewallRules` readback。
- S7 原子提交与 rollback 缺口的 RED/GREEN 测试结果。

这些位置应明确写 **【补充素材】**；证据结论仍可为 `UNKNOWN`，后续补真实截图、视频或报告；不能为了版面完整而生成伪证据。

## 4. 页内、备注、现场三层信息分工

### 页内可见内容

- 一句结论型标题。
- 一张主视觉：真实截图、代码 Diff、调用链或结果表四选一。
- 最多 3–5 个判断点。
- 一个明确输出或 Gate。

### 讲师备注

- 完整函数调用链。
- 输入、输出、副作用和失败语义。
- 现场提问、错误答案及追问。
- 来源路径、提交、命令和证据边界。

### 现场操作

- 打开真实文件或报告。
- 沿固定搜索锚点走 2–5 个函数。
- 让学员留下 YAML、ADR、Worker Packet、Reviewer verdict 或 acceptance result。
- 没有设备时做静态路由，不把设备结论写成 PASS。

## 5. P9–25 推荐落版

| 页 | 页面主视觉 | 页内只讲 | 备注/现场展开 |
|---:|---|---|---|
| 9 | 六阶段证据链 | 六步各产出什么 | Gate 与全课导航 |
| 10 | 真实资产地图 | 每类资产回答/不能证明什么 | 资产登记实操 |
| 11 | C4 四联证据 | 主模式与规则类型不是一回事 | `FirewallPresetMode`、`buildRulesForMode` |
| 12 | 需求卡渲染 | 来源→决策→实现→测试 | claim scope 修正练习 |
| 13 | 四方案失败对比 | 为什么选 Coordinator+Snapshot+Handler | 用失败场景攻击方案 |
| 14 | old→new 时间线 + 小段代码 | 等的是业务条件，不是固定 sleep | `loadStableSnapshot` 穿刺 |
| 15 | ADR 摘录 | 选择、拒绝、失败、重开 | 新证据何时推翻旧决策 |
| 16 | 六条不变量 | 失败时不能改变什么 | invariant→function→counterexample |
| 17 | 后台/前台双闭环 | 业务同步与 UI 刷新分离 | 4 文件调用链和系统 API |
| 18 | Story 依赖图 | 按独立 oracle 拆分 | S1–S6 函数表；S7–S9 作为审阅结果 |
| 19 | S5 Worker Packet | 输入、范围、AC、stop | 把 AC 写成调用次数和状态 |
| 20 | 一轮 Ralph 闭环 | Read→Inspect→Verify→Review→Write back | 双人角色实操 |
| 21 | 真实 Git 时间线 | 每轮暴露下一断点 | 不逐个念 Diff，点函数责任 |
| 22 | R6/R7 Diff + 缺口卡 | 文档契约如何变成测试；如何发现未覆盖契约 | 原子提交反例设计 |
| 23 | 讲师 MCP 原页 | 工具接入验证链 | 保持预留，不重复制作 |
| 24 | E2E Runner 重绘图 | Case/Flow/Driver/Assertion/Artifact | 五文件静态穿刺 |
| 25 | CaseResult + 截图 + 分层判定 | UI PASS、系统事实 UNKNOWN、overall INCOMPLETE | 有设备执行；无设备读报告 |

## 6. 最需要避免的落版问题

1. 把第 17 页所有函数名、状态表和系统 API 同时塞到页面；页面只留双闭环，代码在现场打开。
2. 第 21 页做成九行提交清单；应突出 R5 发现时序窗口、R6 文档、R7 实现这条因果链。
3. 第 22 页只赞美“文档先行”；真正高潮应是用 RFC 反查出当前实现仍有原子性缺口。
4. 第 24 页只放架构图；必须实际走一遍 `rule_create.json → ACTION_TEMPLATES → AssertionExecutor`。
5. 第 25 页用截图宣布 PASS；截图只能证明 UI，最终结论必须分层。

## 7. 课程结束时学员应能迁移的判断框架

面对新的复杂需求，学员应能独立回答：

```text
输入事实来自哪里？
事件、缓存、UI 和系统 getter 谁是真相源？
关键未知能否先做最小穿刺？
RFC 中哪些不变量必须转成反例测试？
Story 是否有独立 oracle 和禁止路径？
AI 的完成声明对应哪一层证据？
当前应该给 PASS、FAIL、UNKNOWN 还是 NEEDS_REPLAN？
```

如果课程只能让学员记住 Ralph、MCP 或几个 Prompt 名称，就没有达到目标；如果学员能沿上述问题检查另一个真实工程，才算实现举一反三。
