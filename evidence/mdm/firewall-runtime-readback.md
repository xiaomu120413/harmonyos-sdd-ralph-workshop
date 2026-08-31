# Firewall Runtime Readback：已有能力与验收缺口

> 用途：第 25–26 页。纠正“仓库没有 getter”的表述：内部系统读取已经存在，缺的是面向自动验收的只读结构化 bridge。

## 代码事实：内部 system read 已存在

`entry/src/main/ets/services/firewall/FirewallSystemRepository.ets` 已直接调用系统接口：

```text
getPolicy(userId)  -> netFirewall.getNetFirewallPolicy(userId)
listRules(userId)  -> netFirewall.getNetFirewallRules(userId, requestParam)
```

`FirewallPolicyService.loadOverviewState(context)` 也已组合系统账号、系统策略与本地状态供业务页面读取。因此不能再把缺口写成“没有 system getter”。

## 真正缺失：验收侧结构化回读

当前 `scripts/e2e/tools/bridge_action_map.json` 有：

```text
firewall.toggle_status -> toggle_firewall
```

但没有：

```text
firewall.get_runtime_state
```

即自动化 Runner 还不能获得“账号快照 + 每用户 policy + rules + local apply record + 原始读取错误”的统一 JSON，也无法仅凭 UI 树对系统最终状态作 PASS 判定。

## `FW-STATUS-001` 当前真实证据

```text
tool action       toggle_firewall        PASS
UI before/after   false -> true           PASS
evidence source   ui_tree                 PRESENT
internal getter   getPolicy/listRules     PRESENT in app code
acceptance bridge get_runtime_state       MISSING
system verdict                             UNKNOWN
```

用例明确设置 `allow_unknown: true`，并备注：未来主断言应迁移到 service-backed firewall status read。当前 case-level PASS 只表示自动化流程按允许 UNKNOWN 的契约完成，不等于每个账号的系统 policy/rules 已被回读证明。

## 工具职责边界

通用设备 MCP 负责原子能力：构建、安装、启动、UI tree/action、日志和截图；业务流程与 PASS/FAIL/UNKNOWN 判定留在业务 E2E adapter / Runner。建议的最小改动是增加测试专用、只读 bridge，复用现有 `FirewallSystemRepository`，而不是把防火墙业务语义硬编码进通用 MCP。

bridge 至少返回：

```json
{
  "result": "PASS|FAIL|UNKNOWN",
  "accounts": [100, 112, 123],
  "signature": "100,112,123",
  "users": [
    {"id": 123, "policy": {"isOpen": true}, "ruleCount": 4, "readError": null}
  ],
  "failures": [],
  "evidence": {"device": "...", "commit": "...", "timestamp": "..."}
}
```

读取失败不能被 repository fallback 抹成空数组后判 PASS；必须在 bridge 中保留 per-user 原始错误并把最终结论降为 `UNKNOWN`。

## 可迁移规则

| 层 | 当前状态 | 证据 |
|---|---|---|
| App 内部系统读取 | PRESENT | `getPolicy` / `listRules` |
| UI 状态读取 | PRESENT | E2E `ui_tree` false → true |
| 自动验收业务回读 | MISSING | action map 无 `get_runtime_state` |
| 系统最终结论 | UNKNOWN | 没有结构化 per-user readback |

不要用“内部有 getter”推导“验收已闭环”，也不要用“验收 bridge 缺失”反推“系统接口不存在”。
