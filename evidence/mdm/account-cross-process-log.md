# 账号变化跨进程问题：真实日志证据

> 用途：第 18、24 页课堂回放。日志已去除设备标识和无关系统噪声；账号 ID 保留为本次问题中的业务关联值。

## 原始问题链

历史 Session 中，问题依次收敛为：

1. 新增用户后全局策略已下发，但自定义模式 UI、出入站规则没有刷新。
2. 延迟 2 秒后仍拿不到预期结果，否定“只是系统刷新慢”。
3. 用户要求增加日志，不允许继续猜测。
4. 最终指出 Extension 与 UI 是两个进程，进程内静态传递无效，应恢复跨进程事件。
5. 接入 CommonEvent 后黑/白名单卡片刷新，但出入站规则和操作记录仍未刷新，形成局部 GREEN。

## 2026-07-02 16:56:03 真机日志

```text
16:56:03.199 enterpriseAdmin/EnterpriseAdminAbility
  Account removed: 115
16:56:03.204 AccountChangeCoordinator
  schedule | source=account-removed, trigger=115, handlerCount=1
16:56:03.607 SystemUserProvider
  getOsAccountLocalIds raw=[100,114]
16:56:03.611 SystemUserProvider
  loadAvailableUserIds success: count=2, selectedUserId=100, finalIds=100,114
16:56:03.611 AccountChangeCoordinator
  account snapshot reconciled | source=account-removed, count=2, signature=100,114
16:56:03.612 AccountChangeCoordinator
  account snapshot detail | previous=, current=100,114, added=100,114, removed=, handlers=1
16:56:03.612 AccountChangeCoordinator
  dispatch handler start | name=firewall, signature=100,114
16:56:03.612 FirewallAccountChangeHandler
  handle snapshot | source=account-removed, current=100,114, added=100,114,
  removed=, signature=100,114
16:56:03.619 FirewallAccountChangeHandler
  reconcile mode | mode=custom, desiredEnabled=true
16:56:03.628 AccountChangeCoordinator
  dispatch handler finish | name=firewall, success=true
```

## 这段日志能证明什么

| 问题 | 证据 | 判定 |
|---|---|---|
| Extension 是否收到账号删除事件 | `Account removed: 115` | YES |
| 是否重新读取系统账号 | `raw=[100,114]` | YES |
| Coordinator 是否生成并下发快照 | `signature=100,114`、`dispatch handler start` | YES |
| Firewall Handler 是否完成本次调用 | `finish ... success=true` | YES |
| diff 是否拥有可信旧基线 | `previous=`，把 100/114 都算成 added | NO |
| UI 进程是否收到并刷新全部消费者 | 本链路没有 UI 进程日志；后续仅卡片刷新 | PARTIAL / UNKNOWN |

最早能确认的异常不是“事件没来”，而是两个独立边界：首次基线为空导致 diff 语义失真；Extension 内的成功也不能证明另一个进程中的 UI 消费完成。

## 相关性边界：历史日志没有 eventId

历史证据没有统一 `eventId`，不能在课件中补造。当前只能用下面的组合关联同一链路：

```text
time window + source=account-removed + trigger=115
+ signature=100,114 + process=enterpriseAdmin
```

显式 `eventId/runId` 是下一版可观测性任务，而不是已经存在的事实。课堂应把真实日志讲成“两进程、四阶段”：事件接收、系统事实读取、业务 reconcile、UI 消费；其中前三阶段有历史证据，第四阶段只有局部结果与缺口。

## 可迁移规则

- `eventReceived`、`businessSucceeded`、`factPublished`、`uiRefreshed` 必须分别记录。
- 进程内单例、静态变量和回调不能作为跨进程事实通道。
- CommonEvent 解决的是传递，不自动完成规则重放、记录刷新或事务补偿。
- 没有相关 ID 时明确标记遥测缺口，不用时间相近伪造确定因果。

