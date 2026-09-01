# 阶段 4｜Feature 与 Story 拆解

## 1. Feature 定义

**Feature：USB 外设分层策略管理**

管理员可以设置设备级 USB 总控、USB 存储模式、未配置设备默认 allow/deny，并对已识别在线 USB 动态切换黑白名单；系统在全局切换、插拔、失败和还原场景中保持意图、执行态和真实系统状态一致。

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
