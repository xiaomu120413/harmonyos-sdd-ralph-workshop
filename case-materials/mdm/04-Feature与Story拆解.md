# 阶段 4｜Feature 与 Story 拆解

## 1. Feature 定义

**Feature：USB 外设分层策略管理**

管理员可以设置设备级 USB 总控、USB 存储模式、未配置设备默认 allow/deny，并对已识别在线 USB 动态切换黑白名单；系统在全局切换、插拔、失败和还原场景中保持意图、执行态和真实系统状态一致。

优先级属于 Feature 契约，不允许 Story 自己重新解释：

```text
接口管控全局允许/禁止（最高）
  > 已识别设备的显式 allow/deny
    > 未配置设备默认黑/白名单（最低）
```

## 2. 为什么不能按页面拆

按“接口页 / 名单页 / 记录页”拆会把一个事务切断：全局 USB 开关发生在接口页，却必须暂停、恢复并刷新名单页的设备策略。Story 应按“可独立证明的不变量”拆，而不是按 UI 文件拆。

## 3. Story 地图

| Story | 能力 | 主要修改面 | 独立验收点 |
|---|---|---|---|
| S1 | USB 身份与策略状态真源 | Resolver + RDB Repo | SN/弱指纹稳定；Hub 过滤；trace 清理不删策略 |
| S2 | 默认 allow/deny | Preferences Repo + Policy VM | 缺值/坏值回退 allow；切换不影响已有设备 |
| S3 | 首次连接策略 | USB Consumer + State Service | allow 建白名单；deny 成功才建黑名单 |
| S4 | 在线设备动态黑白名单 | PolicyList + VM + Dispatch | 离线拒绝；系统成功后才提交本地状态 |
| S5 | USB 全局禁用/恢复 | Global Service + Interface VM | 暂停、补偿、恢复重放；desired 不变 |
| S6 | 存储策略冲突 | Interface/Policy VM + Dispatch | DISABLED 时 UI 置灰且后端拒绝 |
| S7 | 还原与系统残留清理 | State Service + Dispatch | 先清 EDM，再恢复本地 allow；卡片保留 |
| S8 | 证据闭环 | UT/E2E/设备脚本 | D1–D7 分层、PASS/FAIL/UNKNOWN/PENDING |

### 3.1 每个 Story 的输入、输出与失败证据

| Story | 输入文档 | 必须先出现的 RED | 输出文档/代码 | Done 证据 |
|---|---|---|---|---|
| S1 | 身份规则、事件样本 | 同设备 attach/detach 指纹不一致 | Resolver + State Schema | 身份 UT + 真实 payload 卡 |
| S2 | L3 默认规则 | 默认值变化误改已有设备 | Preferences Repo + 默认策略行 | 持久化 UT + 前后状态对照 |
| S3 | 首次连接状态机 | 默认 allow 设备不进名单 | Consumer/State Service | RDB 状态 + 名单卡片 + 无 deny 调用 |
| S4 | L2 单设备显式规则 | MDM 失败却先改 UI/本地 | VM/Dispatch/受控选择器 | RED→GREEN UT + 系统回读 |
| S5 | L1 全局总控 | 已有 deny 与 restrictions 冲突 | Global Service/补偿 | 编排 UT + restrictions 回读 + 实物 |
| S6 | 存储并行策略 | `9200010/9200007` 被映射成同一失败 | 结果三态 + 冲突提示 | 错误码日志 + 回读 + 重插结果 |
| S7 | 系统残留/还原 | 本地无 deny 时按钮不可用 | EDM 清理 + 本地恢复 | 清理前后系统列表与 RDB |
| S8 | 全链验收 | E2E PASS 被当成设备 PASS | Evidence Pack | 五层矩阵、视频/缺口均登记 |

### 课堂截图卡｜不按页面拆，按不变量拆

> **截图结论（CURRENT）：** 一个 Story = 一个可独立证明的能力 + 一组失败分支 + 一个 Done 证据。

```text
S1 身份/状态真源 ─┐
                      ├→ S3 首次连接 → S4 动态规则 → S5 全局事务 ─┐
S2 默认策略 ──────┘                          └→ S6 冲突 → S7 还原 → S8 证据
```

| 拆分检查 | 通过标准 |
|---|---|
| 范围 | 可以列出允许/禁止文件 |
| 正确性 | 至少先写一个失败测试 |
| 独立性 | 不依赖“顺便把另一页也改了”才能验收 |
| 交接 | 新 Session 仅凭 Worker Packet 能开始 |
| Done | 文档/代码/UT/E2E/设备证据分层登记 |

**不能过门的例子：** “完成外设管理页”同时包含全局、默认、单设备、回退和验收，任何失败都无法定位到一个不变量。

## 4. Story 明细

### S1｜USB 身份与状态真源

**As a** 策略引擎，**I want** 用稳定 fingerprint 保存设备意图与在线/执行态，**so that** 拔插、清记录和跨页面刷新不会丢规则。

验收：

- 有 SN 使用 `USB-SN:<serial>`；无 SN 使用 VID/PID/description 弱指纹。
- Hub 不产生名单状态。
- 状态表包含 `desiredPolicy/present/activePolicy`。
- 清空连接 trace 不删除策略状态。

### S2｜默认策略

- 默认值为 allow。
- 非法持久化值 normalize 为 allow。
- allow→deny 只保存设置，不扫描当前设备。
- 已有设备仍按自己的 `desiredPolicy`。
- 全局 USB 禁用时拒绝修改默认策略。

### S3｜首次连接

- 默认 allow：不调用 deny，保存 `allow/none/present=true`。
- 默认 deny：先下发，成功保存 `deny/deny`。
- deny 下发失败：不新增成功黑名单，但连接诊断仍保留。
- 已有显式策略：忽略默认值。
- USB 存储 DISABLED：跳过设备级 deny，避免策略叠加。

### S4｜动态黑白名单

- 只接受规范 fingerprint，不接受旧 VID/PID 字符串。
- 只允许在线 USB 修改。
- 行级 `updatingKeys` 防重复提交。
- MDM 成功后才保存状态并写快照。
- 失败保持旧值并返回可行动 reasonCode。
- 应用按 fingerprint 保存“单设备意图”，但 Done 证据必须如实记录系统按 `baseClass` 执行的影响范围。

### S5｜全局禁用与恢复

- 禁用前 USB 存储必须为 READ_WRITE。
- 暂停 active deny 失败时不下发全局策略。
- 全局下发失败时恢复已经暂停的 deny。
- 全局禁用不改 `desiredPolicy`，名单仍展示但不可编辑。
- 全局恢复后重放在线显式 deny。
- 重枚举使用 500ms + 空结果时再 1000ms 的有界等待。

### S6｜存储策略冲突

- DISABLED 时 USB_STORAGE 行 `editable=false`。
- 绕过 UI 调用仍由 Dispatch 拒绝。
- 失败提示明确是黑白名单与存储模式冲突，并指向“还原策略”。
- 写入后 UI 使用已确认目标状态刷新，不被短暂旧回读覆盖。
- `9200010` 归为冲突规则；`9200007 + 回读已到目标` 归为部分生效，提示重挂载/重新插拔，而不是回滚成旧策略。

### S7｜还原

- 即使本地无 deny、但 EDM 有残留，按钮也可用。
- 清理 EDM 失败时不改变本地记录。
- 成功后所有卡片 `desired=allow/active=none`，不删除。
- 全局 USB 禁用时禁止还原，避免层级冲突。

### S8｜证据闭环

- 每个 AC 绑定至少一个代码/UT 证据。
- 涉及系统事实必须绑定 MDM 回读。
- 涉及用户价值必须绑定实物插拔行为或视频。
- E2E 报告保留旧 FAIL 与新 PASS。
- 至少加入一次“页面显示允许但系统仍有 baseClass deny”的反例，证明验收不是收集绿色截图。

### 4.9 Story 级伪代码｜先冻结行为，再让 AI 映射到真实类

伪代码不替代 ArkTS 实现，它用于冻结输入、分支、提交顺序和失败语义。Worker Session 可以调整私有函数，但不能改变这里已经评审过的不变量。

#### S1｜身份与状态真源

```text
resolveIdentity(payload):
  if payload is USB_HUB: return SKIP
  fingerprint = serial exists
    ? "USB-SN:" + serial
    : weakFingerprint(vendorId, productId, description, baseClass)
  return { fingerprint, baseClass, fingerprintType }

onDetach(fingerprint):
  state = policyStateRepo.find(fingerprint)
  if state exists: policyStateRepo.save(state with present=false)
  // 保留 desiredPolicy；不因拔出删除长期意图
```

#### S2｜默认策略

```text
setDefaultPolicy(next):
  require usbGloballyDisabled == false
  normalized = next in [allow, deny] ? next : allow
  preferences.save("usb_default_policy", normalized)
  // 禁止枚举当前设备；禁止批量改写已有 desiredPolicy
```

#### S3｜首次连接

```text
handleConnect(identity, defaultPolicy):
  existing = policyStateRepo.find(identity.fingerprint)
  if existing:
    return markPresentAndApplyExisting(existing)

  desired = normalize(defaultPolicy)
  if desired == allow:
    save { desired=allow, active=none, present=true }
    return SUCCESS

  if identity is USB_STORAGE and storagePolicy == DISABLED:
    save diagnostic only
    return CONFLICT

  result = dispatch(deny, identity.baseClass)
  if result failed: return result without fake deny record
  save { desired=deny, active=deny, present=true }
```

#### S4｜在线设备动态黑白名单

```text
setDevicePolicy(fingerprint, target):
  state = policyStateRepo.find(fingerprint)
  require state exists and state.present == true
  require no storage/global conflict

  result = dispatch(target, state.baseClass)
  if result failed:
    return reasonCode; keep old state and controlled UI value

  save state with desired=target,
    active=(target == deny ? deny : none)
  append policy snapshot; notify observers
  return SUCCESS
```

#### S5｜全局 USB 事务

```text
setGlobalUsbDisabled(disallow):
  if disallow:
    snapshot = states where present && active==deny
    suspended = suspend(snapshot)
    if suspended failed: compensate(suspended); return FAIL
    result = restrictions.set("usb", disabled)
    if result failed: compensate(suspended); return FAIL
    commit active=none; keep desired unchanged
    return SUCCESS

  result = restrictions.set("usb", enabled)
  if result failed: return FAIL
  online = boundedReEnumerate(500ms, then 1000ms if empty)
  replay deny for online where desired==deny
  return SUCCESS or PARTIAL
```

#### S6｜存储策略冲突与部分生效

```text
setStoragePolicy(target):
  if target conflicts with remaining device-type deny:
    return CONFLICT_9200010 with restore action

  apiResult = usbManager.setStoragePolicy(target)
  readback = usbManager.getStoragePolicy()
  if apiResult success and readback == target: return SUCCESS
  if apiResult failed and readback == target:
    return PARTIAL_APPLIED; ask user to remount/replug
  return FAIL; keep previous confirmed UI state
```

#### S7｜还原与系统残留清理

```text
clearAllPolicies():
  require usbGloballyDisabled == false
  systemResult = dispatch.clearAllUsbDeviceTypePolicies()
  if systemResult failed: return FAIL without changing local records

  for each policy state:
    save desired=allow, active=none; keep asset card
  notify policy observers
  return SUCCESS
```

#### S8｜证据闭环

```text
accept(case):
  evidence = collect(D1_doc, D2_code, D3_UT, D4_build,
                     D5_UI, D6_systemReadback, D7_physical)
  if required oracle unavailable: return UNKNOWN or PENDING
  if any required oracle failed: return FAIL
  if system applied but runtime not converged: return PARTIAL_APPLIED
  return PASS
```

伪代码评审通过的标准：每个失败分支都写清“什么不提交”，每次成功都写清“提交到哪个真源”，每个系统能力都能指向独立 oracle。

### 4.10 真实问题如何拆回 Story，而不是塞进一个 Bug

| 真实问题 | 不能怎么修 | 正确归属 | 为什么 |
|---|---|---|---|
| USB 接口禁用其实只改默认值 | 只改页面文案 | S2 + S5 | 两个作用域和两个真源必须拆开 |
| 默认 allow 不建名单卡片 | 在 View 里临时拼卡片 | S1 + S3 | 资产身份和业务状态必须先落真源 |
| HID deny 残留导致 `9200010` | 捕获异常后显示“管理员未激活” | S6 + S7 | 是系统冲突与残留清理问题 |
| `9200007` 但回读已是读写 | 无条件回滚 UI | S6 | 下发结果和运行态切换是两阶段 |
| Trace 已入库但页面未显示 | Tab 切换时手工刷新 | S8/记录链 | 应修写入通知/状态提交，不让 View 补救 |
| 设备拔出后仍像在线 | 只把整行颜色改灰 | S1 | 必须先修 `present` 真源与启动对账 |

这样拆的价值是：每个问题都有一个最早应该失败的层。修复若落在更上层，通常只是让症状暂时消失。

## 5. Worker Packet 示例｜S5 全局 USB

```markdown
# Worker Packet: S5 USB 全局禁用与恢复

## Goal
实现全局 USB 总控与设备显式策略的协同，不丢失 desiredPolicy。

## Read first
- docs/03-模块设计/外设管理组件设计说明.md §2.1
- docs/superpowers/plans/2026-07-11-peripheral-usb-global-control-story.md
- InterfaceControlViewModel.ets
- UsbGlobalPolicyService.ets
- UsbDevicePolicyStateService.ets

## Invariants
1. global > desired > default
2. 全局禁用不改 desired
3. 系统成功后才提交 active
4. 失败必须补偿已暂停 deny

## Allowed files
- services/peripheral/interface-control/UsbGlobalPolicyService.ets
- services/peripheral/device-policy/UsbDevicePolicyStateService.ets
- viewmodels/peripheral/interface-control/InterfaceControlViewModel.ets
- 对应测试
- 外设模块设计文档

## Forbidden
- 不把事务写进 View
- 不新增本地 globalDisabled 持久化
- 不把 trace 当策略真源
- 不改 PPT 或其它业务模块

## Tests
- 禁用成功
- 存储模式冲突
- suspend 失败
- 全局下发失败并恢复 deny
- 启用后重放 present deny
- 部分重放失败

## Done evidence
- UT 结果
- build 结果
- 设备系统回读（若环境可用）
- 未执行项标 PENDING
```

### 5.1 Worker Packet 中必须附真实反例

S5 不能只写抽象规则，还要附一张反例卡：

```markdown
## Counterexample
前置：HUAWEI Wired Keyboard 曾保存 desired=deny，系统残留 baseClass=3。
动作：接口管控从 USB 禁止切回允许。
观察：restrictions 已启用，但 HID 仍被拒绝；存储策略设置可能返回 9200010。

## Required judgement
- 全局启用不是“所有设备最终允许”。
- 必须重放/清理 L2 规则并记录结果。
- 同 baseClass 影响面必须进入 PENDING 实物矩阵。
```

没有反例的 Worker Packet 很容易让新 Session 只验证 happy path，然后重复旧错误。

S5 在本章用于讲解 Packet 的结构；S1–S8 可直接下发给新 Session 的完整工作包、真实类/测试映射和模块变更账，统一见 `15-Story工作包与模块变更账.md`。这样本章保持“如何拆”的主线，细化文档负责“拆完以后具体交给 AI 什么”。

## 6. 依赖与并行边界

```text
S1 ─┬─> S3 ─> S4 ─┬─> S5 ─┐
    └─> S2 ────────┘       ├─> S8
S4 ────────────────> S6 ─> S7 ─┘
```

- S1/S2 可部分并行：状态库与 Preferences 相互独立。
- S3 依赖身份/状态真源和默认策略。
- S5 必须在设备策略语义稳定后做，否则无法定义暂停/恢复。
- S8 从第一轮就建证据表，但最终验收依赖所有业务 Story。

## 7. Ready / Done 门

### Ready

- Story 只包含一个可证明能力。
- 真源、优先级和失败语义已引用 RFC。
- 修改文件和禁止文件明确。
- 至少一个失败测试先写出。
- 设备依赖和权限已知。

### Done

- 代码与模块设计一致。
- 所有系统调用都有成功/失败分支。
- UT 不只测 happy path。
- UI 状态来自回读或已确认提交，不是无条件乐观更新。
- 证据写入运行账，未跑设备项标 PENDING。
- 下一 Session 可仅凭 Worker Packet 和 progress 继续。
