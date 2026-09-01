# 阶段 3｜防火墙账号同步 RFC（课堂摘录版）

> `TEACHING RECONSTRUCTION`：内容来自当前总体 RFC、模块设计、两份 2026-07-02 专项方案和现有代码。正式项目契约仍以 SecurityTool 仓库中的文档为准。

## 1. RFC 目标

定义账号变化后防火墙同步的架构边界、状态模型、数据流、失败语义和验收口径，使后续 Story 可以在不重新猜架构的前提下实施。

## 2. 范围

### 包含

- 账号新增/删除事件接入。
- 最新账号集合读取、签名和 added/removed 计算。
- public/private 模式对最新集合补下发。
- custom 模式同步账号级默认 policy，不扩历史规则作用域。
- 删除账号的本地数据清理。
- 稳定账号事实通知 UI 运行时。

### 不包含

- 远程多设备策略编排。
- 把 custom 历史“全部用户”升级为动态 ALL。
- 在 Provider 内执行防火墙副作用。
- 用 MainPage/UI 强制刷新替代业务同步。
- 本 Feature 内实现权限管理账号同步，仅保留 Handler 扩展点。

## 3. 总体架构

```text
EnterpriseAdminAbility
  └─ account-added / account-removed trigger
       ↓
AccountChangeCoordinator
  ├─ 400ms debounce
  ├─ single-flight + pending
  ├─ SystemUserProvider 读取完整账号集合
  ├─ account-added 稳定快照门
  ├─ signature / added / removed
  └─ AccountChangeHandler registry
       ↓
FirewallAccountChangeHandler
  ├─ pruneUnavailableUsers
  ├─ public/private → applyModeToUsers
  ├─ custom → applyPolicyForMode（不扩规则）
  └─ 成功后保存 mode + signature
       ↓
AccountChangeEventBus / AccountRuntimeService
  └─ ViewModel 重读 userOptions / policy / rules
```

## 4. 状态模型

| 状态/字段 | 所有者 | 作用 |
|---|---|---|
| `pendingRequest` | Coordinator | 合并/保留最新待处理事件 |
| `running` | Coordinator | 保证同一时间只有一个 reconcile |
| `previousUserIds` | Coordinator | 计算 added/removed 与 previous signature |
| `triggerAccountId` | Event request | account-added 稳定条件 |
| `desiredEnabled` | FirewallLocalRepository | 新账号使用的期望防火墙开关状态 |
| `lastAppliedMode` | FirewallLocalRepository | 上次成功处理的主模式 |
| `lastAppliedUserIdsSignature` | FirewallLocalRepository | 上次成功处理的账号集合 |
| rule intents | FirewallLocalRepository | 自定义规则长期业务意图 |
| deployments | FirewallLocalRepository | 本地规则到系统规则 ID 的映射 |

## 5. 核心不变量

1. 事件只触发同步，不充当完整账号真相。
2. 账号新增只有在完整集合包含 `triggerAccountId` 后才允许分发。
3. 空/失败账号集合不能触发 prune 或模式重放。
4. public/private 对最新账号集合生效。
5. custom 只同步账号级默认 policy，不自动扩展旧规则 targetUserIds。
6. 失败不能保存新账号签名，否则下次会把未完成状态误判为已处理。
7. UI 只消费稳定结果，不承担数据修复。

## 6. 场景与结果

| 场景 | 输入 | 系统动作 | 成功状态 | 失败保持 |
|---|---|---|---|---|
| public 新增账号 | stable snapshot 含新 ID | prune → 重放 public policy/规则 | mode=public，signature 更新 | 旧签名、旧可信状态 |
| private 新增账号 | stable snapshot 含新 ID | prune → 重放 private policy/规则 | mode=private，signature 更新 | 旧签名、旧可信状态 |
| custom 新增账号 | stable snapshot 含新 ID | prune → 同步默认 policy | signature 更新，旧规则 scope 不变 | 不扩规则、不保存失败签名 |
| 删除账号 | 最新集合不含被删 ID | prune 本地引用，不调用被删账号系统 API | intents/deployments/history 清理 | 读取失败时全部保留 |
| 新账号一直不可见 | 6 次读取仍无 trigger ID | 不分发、不发布成功 | 状态 INCOMPLETE | previous/签名不更新 |
| 连续多个事件 | debounce 窗口内多事件 | 合并最新 request；串行再跑 | 以最终完整集合收敛 | 不允许并发互相覆盖 |

## 7. 代码映射

| RFC 职责 | 当前文件 |
|---|---|
| 事件入口 | `entry/src/main/ets/enterpriseadminability/EnterpriseAdminAbility.ets` |
| 协调与稳定快照 | `entry/src/main/ets/services/account/AccountChangeCoordinator.ets` |
| 快照模型 | `entry/src/main/ets/services/account/AccountSnapshotModels.ets` |
| 系统账号真相源 | `entry/src/main/ets/services/firewall/providers/SystemUserProvider.ets` |
| 防火墙处理器 | `entry/src/main/ets/services/firewall/FirewallAccountChangeHandler.ets` |
| 本地真相与清理 | `entry/src/main/ets/services/firewall/FirewallLocalRepository.ets` |
| 模式事务 | `entry/src/main/ets/services/firewall/FirewallModeSwitchService.ets` |
| policy 下发 | `entry/src/main/ets/services/firewall/FirewallPolicyService.ets` |
| UI 运行时消费 | `entry/src/main/ets/services/account/AccountRuntimeService.ets` |

### 7.1 代码级时序：新增账号

```text
EnterpriseAdminAbility.onAccountAdded(accountId)
  ├─ ensureKeepAliveAppRegistered(...)
  └─ scheduleAccountReconcile('account-added', accountId)
       ↓
AccountChangeCoordinator.schedule(context, source, triggerAccountId)
  ├─ pendingRequest = latest request
  ├─ clearTimeout(old timer)
  └─ setTimeout(runPending, 400ms)
       ↓
runPending() → runOnce(request)
       ↓
loadStableSnapshot(request)
  ├─ loadSnapshot → SystemUserProvider.loadAvailableUserIds
  │                  └─ getOsAccountLocalIds
  ├─ shouldWaitForAddedAccount
  ├─ 不包含 trigger ID → 每 200ms 重读，最多 5 次
  └─ 仍不可见 → stable=false，停止分发
       ↓ stable=true
buildSnapshot(current/previous/added/removed/signature)
       ↓
dispatchHandlers → FirewallAccountChangeHandler.handle
```

### 7.2 Handler 到系统 API 的调用链

```text
FirewallAccountChangeHandler.handle(snapshot)
  ├─ currentUserIds=[] → skip，避免误 prune
  ├─ FirewallLocalRepository.pruneUnavailableUsers
  ├─ read currentMode + desiredEnabled
  ├─ custom
  │   ├─ FirewallPolicyService.applyPolicyForMode
  │   │   ├─ FirewallSystemRepository.getPolicy
  │   │   │   └─ netFirewall.getNetFirewallPolicy
  │   │   └─ FirewallSystemRepository.setPolicy
  │   │       └─ netFirewall.setNetFirewallPolicy
  │   └─ saveModeApplyState(custom, signature)
  └─ public/private
      ├─ compare lastAppliedMode + lastAppliedUserIdsSignature
      └─ FirewallModeSwitchService.applyModeToUsers
          ├─ createSnapshot(policy/rules/mode/mapping)
          ├─ clearRules → removeNetFirewallRule
          ├─ applyPolicyForMode → setNetFirewallPolicy
          ├─ buildRulesForMode
          ├─ addRule → addNetFirewallRule
          ├─ saveCurrentMode + saveModeApplyState
          └─ any failure → rollbackToSnapshot
```

### 7.3 前台刷新不是业务修复

业务 Handler 执行结束后，Coordinator 调用 `AccountChangeEventBus.publishSnapshot()`。UI 进程通过 `commonEventManager` 收到 `source/triggerAccountId/signature/timestamp`，再由 `ApplicationRuntimeManager` 调用：

```text
refreshFirewallOverviewForAccountChange()
refreshFirewallRulesForAccountChange()
  → SystemUserProvider.loadAvailableUserIds()
  → FirewallRulesViewModel.refreshForAccountChange(context, users)
     → userOptions = users
     → reloadDefaultPolicies
     → reloadRules
```

因此页面是否打开、ViewModel 是否已加载，只影响展示刷新；后台防火墙同步不能依赖此链。

### 7.4 事务与失败保持

| 失败点 | 返回/动作 | 不能宣称或写入 |
|---|---|---|
| 新账号在重试窗口内始终不可见 | `runOnce=false`；不 dispatch、不 publish | 不能宣称账号已同步 |
| prune 本地引用失败 | Handler=false，停止后续模式处理 | 不能继续保存新签名 |
| 读取旧 policy 失败 | `createSnapshot` 失败 | 不能开始破坏性模式切换 |
| clear/set/add/save 任一步失败 | 收集 `failedItems`，尝试 rollback | 不能保存成功状态；回滚也不能假定必成功 |
| custom policy 成功但签名保存失败 | Handler=false | 不能用“policy 已写”替代整体成功 |
| UI refresh 失败 | 记录运行时警告 | 不能反向修改后台业务状态 |

### 7.5 当前实现与 RFC 不变量之间的审阅缺口

RFC 第 6 条“失败不能保存新账号签名”是目标契约，当前实现还存在以下需要单独 Story 收敛的风险：

- `applyModeToUsers()` 在 `saveCurrentMode()` 失败后仍调用 `saveModeApplyState()`。
- `saveModeApplyState()` 分两次写 `lastAppliedMode` 和 `lastAppliedUserIdsSignature`，可能部分成功。
- `rollbackToSnapshot()` 没有恢复 last-applied mode/signature。
- `listRules()` 读取失败回退为 `[]`，系统快照无法区分“无规则”和“读取失败”。
- Handler 返回失败时 Coordinator 仍发布稳定账号事实，且没有自动业务重试；运行时消费链拿不到 Handler 结果。

这意味着当前代码应判定为“核心同步协议已实现，事务性与业务失败可观测性仍需补强”，而不是把 RFC 目标直接当作已完成事实。

## 8. 测试边界

- UT：防抖/重试、稳定快照、Handler 分发、模式分支、prune、签名保存、空列表与失败。
- ohosTest：路由和页面状态恢复，不替代系统账号事件。
- E2E：应用启动、进入防火墙、模式卡/规则页/敏感操作。
- 真机专项：新增/删除系统账号，读取系统账号/防火墙 policy/规则，验证跨进程时序。

## 9. RFC 评审门

- Page / ViewModel / Service / Provider 职责没有反向依赖。
- 每个状态字段有唯一所有者和写入时机。
- 每个失败分支写清“不能改变什么”。
- Story 可以沿职责边界拆分，每个 Story 有独立 oracle。

## 10. PPT 截图位

- **【补充素材】**：总体架构图。
- **【补充素材】**：状态/不变量表。
- **【补充素材】**：RFC 到当前代码的映射表。
