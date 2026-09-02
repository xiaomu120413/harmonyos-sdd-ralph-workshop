# GPU Task Cards

任务拆分的源码推导过程见 [`../09-源码调用链与任务拆解.md`](../09-源码调用链与任务拆解.md)，完整 Feature / Story、伪代码和验收点见 [`../13-GPU-Ralph-Story拆分与伪代码验收.md`](../13-GPU-Ralph-Story拆分与伪代码验收.md)。任务卡不是按模块平均分工，而是按可独立验证的风险闭环拆分：hook/decoder、buffer、owner/EndFrame、queue、lifecycle、AVC444、最终验收。

任务由 `ADR-GPU-001` 和最小穿刺结果生成。依赖顺序：

```text
T00 baseline
→ T01 decoder selection
→ T02 one-frame output
→ T03 display owner
→ T04 continuous queue
→ T05 lifecycle recovery
→ T06 AVC444 semantics
→ T07 final acceptance
```

每张卡只允许一个可观察结果；若 Stop 条件触发，写回 `FAIL / UNKNOWN / NEEDS_REPLAN`，不自动扩大修改范围。

每张 Task Card 必须对应 `S0–S7` 中的一张 Story，并继承该 Story 的伪代码不变量和 AC。Task Card 可以进一步收窄源码范围和命令，但不能删除失败分支、放宽验收点，或把后续 Story 合并进当前 Ralph 循环。
