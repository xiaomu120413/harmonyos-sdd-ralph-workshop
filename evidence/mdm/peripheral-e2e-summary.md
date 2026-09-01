# 外设管理 E2E 结果摘要

源：`C:\Users\mu\Desktop\code\security_tool\scripts\e2e\results\PER-*.json`

提取时间：2026-09-01。以下仅转录结果中的状态、关键步骤与证据边界，原始 JSON 是真源。

| Case | Status | 关键步骤 | 结论边界 |
|---|---|---|---|
| PER-IF-001 | PASS | launch → peripheral page → interface tab → USB 接口可见 | 页面/文本可见 |
| PER-IF-002 | PASS | launch → interface tab → USB 选择器切换 → 恢复 → 行仍可见 | UI 操作往返，不等于系统状态 |
| PER-POL-001 | PASS | launch → policy tab → “还原策略”可见 | 入口可见 |
| PER-POLICY-002 | PASS | storage read-only → restore allow → 行与 allow 可见 | UI 操作与回显 |
| PER-REC-001 | PASS | record tab → “导出记录”可见 | 记录页入口可见 |
| PER-WL-USB-001 | PASS | policy tab → “导出名单”可见 | 名单页入口可见 |
| PER-BL-USB-001 | FAIL | navigation UNKNOWN → tab UNKNOWN → “还原策略”断言 FAIL | 旧运行的页面驱动失败，不直接等于产品缺陷 |

## FAIL → PASS 的课堂用法

`PER-BL-USB-001` 的 primary evidence 是 `peripheral_policy_restore_visible` 断言失败；上游导航和 Tab 点击已经是 UNKNOWN。后续 `PER-POL-001` 中相同目标入口全部 PASS。

这组证据用于说明：

1. 失败报告必须保留；
2. 先判断失败发生在驱动、环境还是业务；
3. 后续 PASS 是新证据，不应覆盖或删除旧 FAIL；
4. 两份结果都没有系统 MDM readback，不能证明还原动作真实清理了系统策略。

## 共同缺口

所有这些 E2E JSON 的 `artifacts` 均为空。课件可以引用 step evidence 和 UI tree，但不能声称已有自动录屏或系统策略截图。

补齐方向：

- 每次运行绑定 runId、commit、device、HAP hash；
- 操作前后读取 global USB、storage policy、disallowed type；
- 录制 USB 实物插拔/可用性；
- 将截图、视频、readback JSON 写入 artifacts。
