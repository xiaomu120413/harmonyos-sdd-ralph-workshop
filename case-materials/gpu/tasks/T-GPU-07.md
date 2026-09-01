# T-GPU-07｜完整 A/B 与工程验收

- Source Scope：测试、脚本、日志、性能 CSV、录屏和验收报告；进入本任务后生产代码冻结，发现缺陷生成新任务，不在验收脚本中顺手修实现。
- Requirement：使用 T00 同一场景验证路径、性能、画质、稳定、交互和 fallback。
- Allowed：测试、脚本、日志、性能 CSV、录屏、报告。
- Forbidden：为了通过验收继续修改生产代码；发现问题应生成新任务。
- Verify：`AC-BUILD/PATH/420/444/PERF/STABLE/INPUT/FALLBACK/SCOPE`。
- PASS：关键维度独立证据完整；缺性能或长稳时整体不能写“全部完成”。
- Stop：任何证据不能关联同一 commit/runId 时标 `UNKNOWN`。
