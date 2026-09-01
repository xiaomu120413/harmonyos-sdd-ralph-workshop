# GPU Task Cards

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

