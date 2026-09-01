# 阶段 5｜Ralph 迭代运行账

> Ralph 在这里不是“让 AI 一直跑”，而是每轮只收敛一个可验证未知：读取外部记忆 → 实现/修正 → 独立验证 → 记录证据 → 决定停止、继续或升级。

## 1. 单轮协议

```text
READ
  RFC + 当前 Story + progress + 上轮失败证据
PLAN
  一个未知、最小文件面、一个独立 oracle
IMPLEMENT
  文档先行（若行为变更）+ 代码 + 测试
VERIFY
  UT → build → E2E/UI → 系统回读 → 实物
RECORD
  结论、证据路径、剩余未知、下一轮入口
STOP / CONTINUE / ESCALATE
```

每轮最多解决一个主失败。如果同时出现系统能力、UI 回显和数据迁移三个问题，拆成三轮。

## 2. 真实演进账

| 轮次 | 发现的未知/错误 | 证据 | 修正 | 新不变量 |
|---|---|---|---|---|
| R1 | USB Hub 被当成可管理设备 | 运行时 payload/baseClass | 过滤 0x09 | Hub 不进 trace/名单/策略 |
| R2 | “USB 接口”被当成默认策略入口 | 旧 2026-05-14 计划与需求冲突 | 后续恢复为 restrictions 全局总控，默认策略移到名单页 | 全局与默认分离 |
| R3 | 名单从连接记录反推，清 trace 会丢候选 | 代码路径与清理行为 | 建 `usb_device_policy_states` | Trace ≠ Policy State |
| R4 | 拔出 deny 设备后系统类型规则残留 | 同类型 MDM 下发语义 | `activePolicy` + 同 baseClass 在线计数 + 内部 allow | desired 与 active 分离 |
| R5 | “还原”删除卡片，资产和意图丢失 | UI/状态测试 | 保留记录，恢复 allow/none | 还原不是删除 |
| R6 | 还原按钮只看本地 deny，无法清 EDM 残留 | 系统残留场景 | `hasDisallowedUsbDeviceTypePolicies()` 也计入 | 本地空不等于系统空 |
| R7 | 全局 USB 与单设备 deny 直接叠加，恢复易冲突 | 最小穿刺 | `UsbGlobalPolicyService` 暂停/补偿/重放 | 全局事务独立编排 |
| R8 | 全局禁用/启用没有留下真实在场设备策略证据 | trace 审阅 | 禁用前捕获；启用后重枚举写快照 | 不能从历史推断“当前在场” |
| R9 | 默认 allow 设备未进入白名单 | UT/页面空列表 | allow 首次连接也保存状态，不下发 deny | allow 也需要显式业务记录 |
| R10 | USB 存储写入后立即回读旧值导致名单仍可编辑 | 真机/状态同步问题 | 成功后用已确认目标状态刷新 | 已提交目标可作为短期 UI 刷新依据 |

对应真实提交包括：

- `63dda4b4 refactor peripheral usb policy state`
- `a2f0128b refactor peripheral policy state cleanup`
- `093cb6e4 fix peripheral policy restore cards`
- `786e370c fix peripheral policy restore state`
- `6e7702cd feat(peripheral): coordinate usb global disable with per-device policies`
- `23c4a046 fix(peripheral): restore usb policy records`
- `0d26c92e fix(peripheral): retry usb enable snapshots`
- `f95c5109 fix(peripheral): sync USB storage policy state`

## 3. 课堂截图卡｜五个转折如何变成工程不变量

> **截图结论（CURRENT）：** 迭代的价值不是“AI 多跑几轮”，而是每个反例都被写回设计、测试和下一轮的约束。

| 症状/反例 | 根因 | 固化的新不变量 | 下一轮 oracle |
|---|---|---|---|
| USB 接口变成默认策略 | 两种作用域共用入口 | 全局与默认分离 | restrictions 回读 + 首插策略 |
| 清 trace 后名单丢失 | 把历史事件当策略真源 | Trace ≠ Policy State | 清 trace 后规则仍在 |
| 拔出后 deny 残留 | `desired=active` 且忽略类型粒度 | 意图/执行态/在线态分离 | 同 baseClass 双设备 |
| 还原后卡片消失 | 把还原等于删除 | 先清 EDM，卡片恢复 allow/none | 系统残留 + UI 对照 |
| 默认 allow 设备不在名单 | 把“不下发 deny”当作“不需记录” | allow 也是管理意图 | RDB 记录 + Policy VM 卡片 |

**截图时要保留的链条：** 症状 → 根因 → 新不变量 → 下一轮 oracle。只截 commit 列表不能说明为什么 AI 修对了。

## 4. 展开一轮｜R7 全局 USB 协同

### 4.1 失败假设

最初直觉：先下发全局禁用，恢复时再按名单重新下发 deny 即可。

### 4.2 反例

- 系统已有按类型 deny 时，全局 restrictions 可能发生冲突。
- 如果先清 deny 后全局下发失败，原黑名单真实执行态被破坏。
- 如果禁用时把 `desiredPolicy` 改成 allow，恢复后无法知道谁应继续 deny。

### 4.3 红灯测试

- suspend 第二个策略失败时，第一个必须补偿为 deny。
- 全局下发失败时，所有在线显式 deny 被恢复。
- 全局成功时 desired 不变、active 归 none。
- 全局恢复时只重放 `present && desired=deny`。

### 4.4 实现

新增 `UsbGlobalPolicyService`，把事务从接口 ViewModel 中抽离；State Service 提供 suspend/restore/compensate 原语；父 VM 只在成功后刷新名单。

### 4.5 验证

- UT：编排分支与状态服务分支。
- 代码审阅：ViewModel 不直接访问 Policy VM 内部状态做事务。
- E2E：`PER-IF-002` 证明 UI 往返可执行。
- 系统/实物：完整矩阵仍为 PENDING。

### 4.6 下一轮未知

全局恢复后 USB 重新枚举存在延迟，立即捕获可能得到空集合，于是进入 R8 的有界重试。

## 5. 展开一轮｜R9 默认 allow 漏记录

### 症状

默认 allow 时设备可用，但黑白名单为空。若只看“设备能用”，很容易判为完成。

### 根因

早期逻辑把“没有下发 deny”误等同于“不需要保存状态”，导致允许设备没有业务身份记录。

### 修正

```ts
const shouldSave = desiredPolicy === 'allow' ||
  existing !== null ||
  dispatchResult?.success === true
```

### 独立 oracle

- 新 allow 设备不触发 dispatch；
- RDB 出现 `desired=allow/active=none/present=true`；
- Policy VM 能构建白名单记录。

这轮特别适合课堂讲：“功能可用”不等于“需求完成”，因为验收标准还要求资产可管理。

## 6. progress.md 示例

```markdown
# Progress: USB layered policy

## Current story
S5 USB global disable/restore

## Completed
- global state reads restrictions usb
- suspend active deny with compensation
- preserve desiredPolicy during global disable
- restore present explicit deny after enable

## Evidence
- UT: usb-global-policy-service.test.ets PASS
- UT: usb-device-policy-state-service.test.ets PASS
- E2E: PER-IF-002 PASS (UI roundtrip only)
- device system readback: PENDING

## Known limits
- MDM deny dispatch is baseClass-level, not serial-level
- partial deny replay after global enable logs warning; device matrix pending

## Next
Run two same-class USB devices and record system readback + video.

## Stop reason
Code loop stops; remaining work requires physical device matrix.
```

## 7. 每轮 Reviewer 必问

1. 本轮改变的是意图、执行态、UI 状态还是系统真源？
2. 系统调用失败时，哪个字段保持旧值，哪个副作用需要补偿？
3. 测试的 fake 是否和真实系统 API 粒度一致？
4. E2E 的 PASS 到底断言了什么，是否被过度解读？
5. 下一轮的未知能否通过一个更小穿刺解决？

## 8. 停止与升级条件

### Stop

- Story 的 AC 均有对应证据。
- 重复运行结果稳定。
- 没有新增未知。
- 剩余项明确移交给后续 Story 或实物验收。

### Escalate

- 三轮都遇到同一系统冲突且无新证据。
- 需要真实 USB 设备、企业管理员权限或系统日志才能继续。
- 系统 API 粒度无法满足产品“单硬件设备”语义，需要产品/架构决策。
- 补偿失败后系统与本地可能不一致，需要人工清理与重新同步。

升级不是失败；在没有设备证据时继续让 AI 改代码，才是失控循环。
