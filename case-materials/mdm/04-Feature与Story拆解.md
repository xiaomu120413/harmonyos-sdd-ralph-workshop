# 阶段 4｜Feature 与 Story 实施规格

## 1. Feature 契约

### 1.1 Feature

**USB 外设分层策略管理**

在企业管理员激活的 HarmonyOS 2in1 设备上，管理员可以设置设备级 USB 全局总控、USB 存储访问模式、未配置设备默认 allow/deny，并对已识别的在线 USB 设备设置显式 allow/deny。系统必须在全局切换、设备插拔、系统调用失败、策略冲突和还原场景中，分别维护管理员意图、当前执行态、系统回读和真实设备行为。

### 1.2 业务优先级

```text
L1 设备级 USB 全局总控（最高）
  > L2 已识别设备的显式 allow / deny
    > L3 未配置设备默认 allow / deny（最低）

USB 存储访问模式是并行的设备类型策略，不得混入上述三层优先级。
```

### 1.3 Feature 级不变量

1. 全局 USB 状态只相信 `restrictions.getDisallowedPolicy(..., 'usb')` 回读。
2. `desiredPolicy` 表示长期意图，`activePolicy` 表示设备级 deny 是否已真实下发，两者不能合并。
3. 全局禁用不得覆盖 `desiredPolicy`；全局恢复后只重放在线显式 deny。
4. 没有显式记录的新设备才读取默认策略；已有设备忽略默认值变化。
5. 默认 allow 首次连接写入 `allow/none/present=true`，使设备成为可管理资产；默认 deny 只有系统下发成功后才写入 `deny/deny/present=true`。
6. 系统调用失败、回读不一致或实物行为未收敛时，不得显示或记录假成功。
7. 应用按 fingerprint 保存设备意图，但当前 MDM 下发可能按 `baseClass` 影响同类型设备，验收必须明确这一执行粒度。

### 1.4 范围

**包含：**

- USB 全局禁用与恢复事务。
- USB 存储 `READ_WRITE / READ_ONLY / DISABLED` 策略。
- SN 强指纹与无 SN 弱指纹。
- 未配置设备默认 allow/deny。
- 在线设备显式 allow/deny、离线拒绝、还原与系统残留清理。
- UI、RDB、Preferences、MDM 回读、日志和实物行为的分层验收。

**不包含：**

- USB 充电能力控制。
- 把 HDC、MTP、USB 转串口和外置存储卡合并成一个策略。
- 蓝牙设备黑白名单扩展。
- 未经双设备验证便声明 SN 级物理隔离。
- 为了获得绿色测试而自动清理不属于本应用管理的系统策略。

## 2. Story 编写标准

每个 Story 都必须是一个可独立证明的业务能力，不按页面或文件数量拆分。以下字段缺失时，Story 不满足 Ready：

| 字段 | 必须回答的问题 |
|---|---|
| Goal | 本 Story 独立交付什么用户价值或工程不变量？ |
| Problem | 当前行为为什么不正确，最早在哪一层出现？ |
| Scope / Non-goal | 本轮允许改变什么，明确不处理什么？ |
| Read first | 新 Session 开工前必须读取哪些设计和代码切片？ |
| State / Contract | 输入、输出、状态变化和系统调用顺序是什么？ |
| RED | 修改代码前，哪条测试或反例必须先失败？ |
| Tasks | 文档、实现、测试和证据按什么顺序落地？ |
| AC | Given / When / Then 是否明确，能否绑定独立 oracle？ |
| Failure semantics | 失败时哪些状态绝对不能提交？ |
| Evidence / Done | 通过需要哪些原始结果，哪些项允许标 PENDING？ |

统一状态只使用：`PASS / FAIL / PARTIAL / UNKNOWN / PENDING`。`E2E PASS` 只能证明它实际观察到的 UI 路径，不自动升级为 MDM 或实物 PASS。

## 3. Story 地图与依赖

| Story | 独立能力 | 上游 | 主要交付物 | 当前判定 |
|---|---|---|---|---|
| S1 | USB 身份与策略状态真源 | RFC 身份规则 | Resolver、状态模型、Repository | PARTIAL |
| S2 | 未配置设备默认策略 | S1 状态边界 | Preferences Repo、默认策略入口 | PARTIAL |
| S3 | 首次连接状态机 | S1、S2 | Consumer、State Service | PARTIAL |
| S4 | 在线设备动态黑白名单 | S1、S3 | Policy VM、State Service、Dispatch | PARTIAL |
| S5 | USB 全局禁用与恢复事务 | S1、S4 | Global Service、Interface VM | PARTIAL |
| S6 | USB 存储冲突与部分生效 | S4、S5 | 结果模型、错误映射、前后端门禁 | PARTIAL |
| S7 | 还原与系统残留清理 | S4、S6 | Clear transaction、可重试状态 | PARTIAL |
| S8 | 分层证据闭环 | S1–S7 | Evidence Pack、验收矩阵 | PARTIAL |

```text
S1 ─┬─> S3 ─> S4 ─┬─> S5 ─┐
    └─> S2 ────────┘       ├─> S8
S4 ────────────────> S6 ─> S7 ─┘
```

S1 与 S2 可以在状态契约冻结后并行；S3 依赖身份与默认策略；S5 必须等待显式策略语义稳定；S8 从第一轮建立证据表，但最终结论依赖 S1–S7。

---

## 4. S1｜USB 身份与策略状态真源

### 4.1 Story 卡

| 项 | 内容 |
|---|---|
| Goal | 稳定识别 USB 资产，并把长期意图、在线态、执行态分开保存 |
| Problem | 同一设备可能因事件字段变化生成不同身份；连接记录曾被误当成策略真源；拔出后策略意图可能丢失 |
| Read first | RFC §5.3、§6.2、§9；`UsbDeviceIdentityResolver.ets`、`UsbDevicePolicyStateRepository.ets`、`PeripheralConnectionRecordUsbConsumer.ets` |
| Allowed | 身份解析器、USB 状态模型与仓储、USB Consumer、对应测试、模块设计文档 |
| Forbidden | View 拼 fingerprint；从 trace 反推策略；设备离线时删除 `desiredPolicy` |
| RED | 同一无 SN 设备 attach/detach 得到两个 fingerprint；清 trace 后名单消失；Hub 被写成设备资产 |

### 4.2 状态契约

| 字段 | 来源 | 含义 | 更新时机 |
|---|---|---|---|
| `fingerprint` | IdentityResolver | 业务唯一键 | 首次解析设备事件 |
| `fingerprintType` | IdentityResolver | `serial` 或 `weak` | 与 fingerprint 同时生成 |
| `baseClass` | USB payload | MDM 实际下发粒度 | 首次建档并随有效事件校验 |
| `desiredPolicy` | Policy State RDB | 管理员长期 allow/deny 意图 | 首次建档或显式修改成功后 |
| `activePolicy` | Policy State RDB | 当前设备级 deny 是否已下发 | 系统操作成功或补偿成功后 |
| `present` | attach/detach + 对账 | 当前是否在线 | 事件处理或启动对账后 |

身份生成规则：

```text
if baseClass == USB_HUB:
  return SKIP
if serial is valid:
  fingerprint = "USB-SN:" + normalize(serial)
else:
  fingerprint = "USB-WEAK:" + hash(vid, pid, normalizedDescription)
return fingerprint + baseClass + fingerprintType
```

### 4.3 实施任务

1. 先在模块设计中冻结强指纹、弱指纹、Hub 过滤和离线保留规则。
2. 为强指纹、弱指纹稳定性、非法 VID/PID、Hub 和 description 变化补 RED 测试。
3. Resolver 只负责身份，不读写页面状态和策略。
4. Repository 以 fingerprint 为主键保存 `desired/active/present`，不耦合 fingerprint 算法。
5. detach 只提交 `present=false`；清理 trace 不调用策略状态 Repository。
6. 若启动时缺少 `getDevices()` 在线对账，记录为工程开放项，不用 UI 刷新掩盖。

### 4.4 验收标准

| AC | Given | When | Then | Oracle |
|---|---|---|---|---|
| S1-AC01 | USB 设备有稳定 SN | 连续 attach、detach、再次 attach | 三次 fingerprint 完全一致，类型为 `serial` | Resolver UT + 原始 payload |
| S1-AC02 | USB 设备无 SN | 使用相同 VID/PID/description 重复连接 | 生成同一个弱指纹，类型为 `weak` | Resolver UT + RDB 主键 |
| S1-AC03 | 输入为 USB Hub | 处理 attach | 不写 trace 资产、不写策略状态、不触发 MDM 下发 | Consumer UT + Repository 调用次数 |
| S1-AC04 | 已有 `desired=deny` 的设备在线 | 收到 detach | 只改 `present=false`，保留 desired，active 按回退结果更新 | RDB 前后 diff |
| S1-AC05 | trace 与策略状态同时存在 | 执行“清空连接记录” | trace 清空，策略状态行完整保留 | 两张表的前后快照 |
| S1-AC06 | 应用启动前错过 detach | 初始化外设模块 | 以 `getDevices()` 修正 present；能力未实现时标 PENDING | 设备枚举结果 + RDB |

### 4.5 失败语义与 Done

- 身份字段不足时允许记录诊断事件，但不得创建可操作的策略状态。
- 状态写入失败不得伪造成名单卡片；必须返回可追踪错误。
- Done 需要 Resolver/State Service UT、RDB 快照和真实 attach/detach payload；S1-AC06 没有设备结果时整体最多为 `PARTIAL`。

---

## 5. S2｜未配置设备默认 allow / deny

### 5.1 Story 卡

| 项 | 内容 |
|---|---|
| Goal | 默认策略只决定“尚无 fingerprint 状态的新设备”首次接入行为，不批量修改已有设备 |
| Problem | 原 USB 接口开关曾混用 `usb_default_policy`，导致全局总控和默认规则语义冲突 |
| Read first | RFC §6.3；`PeripheralDevicePolicyRepository.ets`、`PeripheralPolicyViewModel.setUsbDefaultPolicy()`、`PolicyList.ets` |
| Allowed | 默认策略 Repository、Policy VM、默认策略行、Preferences 适配、对应测试 |
| Forbidden | 切换默认值时枚举 USB、调用 MDM、修改已有 `desiredPolicy`、生成设备策略快照 |
| RED | 非法值读成 deny；allow→deny 后现有 allow 设备被改写；保存失败但 UI 显示成功 |

### 5.2 行为契约

```text
getDefaultPolicy():
  raw = preferences.get("usb_default_policy")
  return raw in [allow, deny] ? raw : allow

setDefaultPolicy(next):
  require globalUsbDisabled == false
  require next in [allow, deny]
  persist next
  after persistence PASS: update ViewModel state and notify
  never enumerate devices
  never dispatch MDM policy
```

### 5.3 实施任务

1. 将默认策略入口固定在“黑白名单”主卡顶部，文案说明只影响未配置设备。
2. Repository 对缺值和非法值统一回退 allow。
3. ViewModel 在全局禁用时拒绝保存，不能只依赖控件置灰。
4. 保存成功后更新内存并通知；保存失败保持原值。
5. 测试保存前后所有已有 fingerprint 行完全一致。

### 5.4 验收标准

| AC | Given | When | Then | Oracle |
|---|---|---|---|---|
| S2-AC01 | Preferences 无该 key | 加载黑白名单页 | 默认值为 allow | Repository UT + Preferences dump |
| S2-AC02 | Preferences 为非法字符串 | 加载默认策略 | normalize 为 allow，并记录诊断 | Repository UT + 日志 |
| S2-AC03 | 已有 A=allow、B=deny | 默认 allow 改为 deny | 只改变 Preferences；A/B 状态、MDM 列表均不变 | RDB diff + Dispatch 零调用 |
| S2-AC04 | 全局 USB 已禁用 | 尝试修改默认值 | UI 置灰；绕过 UI 调 VM 也失败；Preferences 不变 | VM UT + Preferences 前后值 |
| S2-AC05 | Preferences 保存失败 | 用户选择新值 | 控件回到旧值，返回 reasonCode，不通知成功 | VM UT + UI 状态 |

### 5.5 Done

Repository、ViewModel 测试通过；有专用 UI Case 证明入口、说明和禁用态；没有专用 UI 结果时本 Story 为 `PARTIAL`，不能由名单页“可打开”代替。

---

## 6. S3｜首次连接状态机

### 6.1 Story 卡

| 项 | 内容 |
|---|---|
| Goal | 将 USB attach 转成可信资产状态；默认 deny 只有系统下发成功后才形成 active deny |
| Problem | 默认 allow 设备曾可用但不进入资产列表；默认 deny 失败可能留下假黑名单；已有显式意图可能被默认值覆盖 |
| Read first | RFC §9；`PeripheralConnectionRecordUsbConsumer.ets`、`UsbDevicePolicyStateService.handleConnect()`、Dispatch 与 State Repository |
| Allowed | USB Consumer、State Service/Repository、Dispatch 接口、对应测试 |
| Forbidden | View 监听 USB 事件；从 trace 反推策略；先写 deny 再调用系统 |
| RED | 默认 allow 不建状态；deny 失败仍写 `active=deny`；已有 allow 被新默认 deny 覆盖 |

### 6.2 决策表

| 全局 USB | 已有状态 | 规则 | 动作 | 状态提交 |
|---|---|---|---|---|
| 禁用 | 任意 | 任意 | 不额外下发设备规则 | 不因本次连接覆盖显式意图 |
| 启用 | 是 | desired=allow | 忽略默认值，不下发 deny | `present=true`，保留 allow/none |
| 启用 | 是 | desired=deny | 确保类型 deny 下发 | 成功后 `deny/deny/present=true` |
| 启用 | 否 | default=allow | 不下发 deny | 新建 `allow/none/present=true` |
| 启用 | 否 | default=deny | 先下发类型 deny | 仅成功后新建 `deny/deny/present=true` |

### 6.3 核心伪代码

```text
handleConnect(identity):
  require identity is valid and not HUB
  existing = stateRepo.find(identity.fingerprint)

  if globalUsbDisabled:
    update existing.present when existing exists
    append connection fact
    return GLOBAL_BLOCKED

  desired = existing?.desiredPolicy ?? defaultPolicyRepo.get()
  if desired == allow:
    save allow / none / present=true
    append allow connection fact
    return PASS

  if identity is USB_STORAGE and storagePolicy == DISABLED:
    append conflict diagnostic
    return CONFLICT without fake deny state

  result = dispatch deny by baseClass
  if result failed:
    append failure diagnostic
    return FAIL without active=deny
  save deny / deny / present=true
  append deny connection fact
```

### 6.4 实施任务

1. Consumer 解析事件和身份后，把状态决策交给 State Service。
2. State Service 按“已有显式状态优先、默认规则兜底”选择 desired。
3. allow 分支不调用 deny，但必须创建或更新可管理状态。
4. deny 分支先 dispatch，PASS 后再 upsert。
5. 存储冲突和下发失败只写诊断事实，不写成功执行态。
6. 完成状态提交后再通知名单 ViewModel；通知失败不得回滚已确认的系统和 RDB 事实。

### 6.5 验收标准

| AC | Given | When | Then | Oracle |
|---|---|---|---|---|
| S3-AC01 | 新设备无状态，默认 allow | attach | 不调用 deny；新建 allow/none/present=true；白名单可见 | Dispatch 零调用 + RDB + UI |
| S3-AC02 | 新设备无状态，默认 deny | attach 且 MDM 成功 | 先下发，再新建 deny/deny/present=true | 调用顺序 + RDB + MDM 回读 |
| S3-AC03 | 新设备默认 deny | MDM 下发失败 | 不创建成功黑名单；保留失败诊断 | RDB 无目标行 + trace |
| S3-AC04 | 已有 desired=allow，默认已改 deny | 再次 attach | 忽略默认值，仍为 allow | RDB + Dispatch 零 deny |
| S3-AC05 | 全局 USB 禁用 | 任意设备 attach | 不叠加设备 deny，不改 desired | State Service UT + MDM 调用次数 |
| S3-AC06 | 存储策略 DISABLED | 新 USB_STORAGE 默认 deny | 返回冲突，不叠加类型 deny，不写 active=deny | Dispatch UT + RDB |

### 6.6 Done

必须同时提供 State Service UT、调用顺序、RDB 行和至少一组真实插入结果。只有 UT 时可判代码逻辑 PASS，但 Story 总体仍为 `PARTIAL`。

---

## 7. S4｜在线设备动态黑白名单

### 7.1 Story 卡

| 项 | 内容 |
|---|---|
| Goal | 在线设备规则修改保持“校验 → 系统执行 → 本地提交 → 刷新”的顺序，失败不假成功 |
| Problem | 离线设备可能仍可编辑；MDM 失败后 UI/RDB 可能提前变化；应用的 fingerprint 粒度容易被误写成系统隔离粒度 |
| Read first | RFC §6.4、§6.5；`PolicyList.ets` → `PeripheralPolicyViewModel.setDevicePolicy()` → `UsbDevicePolicyStateService.setPolicy()` → Dispatch |
| Allowed | Policy VM、State Service、Dispatch、受控选择器、对应测试 |
| Forbidden | View 直接写 RDB/MDM；无条件乐观更新；隐藏 baseClass 影响面 |
| RED | 离线仍可改；MDM 失败但值已变；同一行可重复提交；A 的规则影响 B 却宣称 SN 级隔离 |

### 7.2 提交顺序

```text
PolicyList emits fingerprint + target
  -> VM checks admin/global/present/storage/updatingKey
  -> StateService loads authoritative row
  -> Dispatch applies allow or deny by baseClass
     -> FAIL: keep old desired/active and return reasonCode
     -> PASS: upsert desired/active
  -> append policy snapshot
  -> reload records from State Repository
```

### 7.3 实施任务

1. 行组件只提交 fingerprint 与目标值，不直接维护成功态。
2. VM 使用 `updatingKeys` 做行级防重入，并重复校验在线、全局和存储门禁。
3. State Service 只接受规范 fingerprint，拒绝旧 VID/PID 拼接 ID。
4. Dispatch 负责 MDM 参数映射与 reasonCode，不把错误吞成布尔值。
5. 系统 PASS 后才更新 `desired/active`；快照失败作为审计副作用单独报告。
6. 双设备验证系统按 baseClass 的影响面，并将结论写入证据包。

### 7.4 验收标准

| AC | Given | When | Then | Oracle |
|---|---|---|---|---|
| S4-AC01 | 设备 present=false | 选择 allow 或 deny | 前端不可选；直接调 VM 也拒绝；状态不变 | UI + VM UT + RDB diff |
| S4-AC02 | 在线 allow 设备 | 改为 deny 且系统成功 | Dispatch PASS 后保存 deny/deny，页面回读 deny | 调用顺序 + RDB + MDM |
| S4-AC03 | 在线 deny 设备 | 改为 allow 且系统成功 | 移除系统 deny 后保存 allow/none | MDM 前后列表 + RDB |
| S4-AC04 | 任一系统操作失败 | 提交新规则 | 控件回到旧值，RDB 不变，reasonCode 可行动 | VM UT + UI 状态 |
| S4-AC05 | 同一行请求未结束 | 再次点击 | 第二次请求被拒绝，不发生重复下发 | Dispatch 调用次数 |
| S4-AC06 | A、B 为同 baseClass 两台设备 | 只对 A 设置 deny | 记录 B 是否受影响；结果不得扩大成 SN 级隔离结论 | 两台设备视频 + MDM 回读 |

### 7.5 Done

代码与 UT 需证明失败不提交；D6/D7 必须覆盖一台设备的真实 allow/deny 和两台同 baseClass 影响面。缺双设备结果时隔离粒度为 `UNKNOWN/PENDING`。

---

## 8. S5｜USB 全局禁用与恢复事务

### 8.1 Story 卡

| 项 | 内容 |
|---|---|
| Goal | 全局闸门切换时暂停和恢复显式 deny，保留长期意图，并对中途失败执行补偿 |
| Problem | restrictions USB 与设备类型 deny 可能冲突；直接下发会造成全局状态、active 和 UI 不一致 |
| Read first | RFC §7、§8；`InterfaceControlViewModel.toggleInterface()`、`UsbGlobalPolicyService.setDisabled()`、State Service、`PeripheralService` |
| Allowed | Global Service、Interface VM、State Service、对应测试 |
| Forbidden | 改写 desired；在 View 编排事务；持久化本地 global 镜像替代系统回读；无限等待设备枚举 |
| RED | suspend 第 N 项失败后未补偿；全局下发失败但 active 已清；恢复后未重放在线 deny；空枚举无限等待 |

### 8.2 禁用事务

```text
disableGlobalUsb():
  require storagePolicy == READ_WRITE
  before = restrictions.get("usb")
  if before == disabled: return IDEMPOTENT_PASS

  snapshot = present states where active == deny
  suspended = suspend one by one
  if any suspend failed:
    compensate previously suspended items
    return FAIL

  result = restrictions.set("usb", disabled)
  readback = restrictions.get("usb")
  if result failed or readback != disabled:
    compensate suspended items
    return FAIL

  keep every desired unchanged
  publish global disabled and refresh policy editability
```

### 8.3 恢复事务

```text
enableGlobalUsb():
  result = restrictions.set("usb", enabled)
  readback = restrictions.get("usb")
  if result failed or readback != enabled: return FAIL

  online = enumerate after 500ms
  if online is empty: enumerate once more after 1000ms
  replay deny for online states where desired == deny
  update active only for successful items
  return PASS or PARTIAL with failed fingerprints
```

### 8.4 实施任务

1. 模块设计先冻结事务快照、补偿范围、回读规则与部分成功模型。
2. Global Service 成为唯一事务编排者；原子系统读写保留在 `PeripheralService`。
3. 禁用前校验存储策略并暂停 active deny。
4. 任一暂停或全局下发失败时，只补偿本轮已改变的项目。
5. 全局成功后保持 desired，刷新名单为只读。
6. 恢复后有界枚举并逐项重放 deny；失败项保持 active=none。
7. ViewModel 根据完整结果显示成功、失败或部分失败，不自行猜测。

### 8.5 验收标准

| AC | Given | When | Then | Oracle |
|---|---|---|---|---|
| S5-AC01 | USB 已全局禁用 | 再次禁用 | 幂等成功，不重复暂停或写状态 | Service UT + 调用次数 |
| S5-AC02 | 存储为 READ_ONLY/DISABLED | 禁用全局 USB | 主事务不开始，状态全部不变 | UT + restrictions 前后值 |
| S5-AC03 | 有多个 active deny | suspend 中途失败 | 停止主操作，补偿之前成功项，desired 不变 | 编排 UT + RDB diff |
| S5-AC04 | suspend 全部成功 | restrictions 下发或回读失败 | 重放已暂停项，全局 UI 不显示禁用成功 | UT + getter + UI |
| S5-AC05 | 全局禁用成功 | 查看名单和存储策略 | desired 保留；active 收敛 none；细粒度控件全部禁用 | RDB + UI + getter |
| S5-AC06 | 全局禁用且有在线 desired deny | 恢复 USB | 全局先启用，再重放在线 deny | 调用顺序 + MDM 回读 |
| S5-AC07 | 多个 deny 中部分重放失败 | 恢复 USB | 全局保持启用；成功项 active=deny，失败项 active=none；返回 PARTIAL | UT + RDB + UI 提示 |

### 8.6 Done

必须有编排 UT、restrictions 回读、设备类型规则回读和实物行为。`PER-IF-002 PASS` 只属于 UI 往返证据，不能单独完成 S5。

---

## 9. S6｜USB 存储策略冲突与部分生效

### 9.1 Story 卡

| 项 | 内容 |
|---|---|
| Goal | 区分策略冲突、API 失败、系统已写入但运行态未收敛，不把不同状态压成一个错误提示 |
| Problem | `9200010` 曾被误判为管理员未激活；`9200007` 可能在 getter 已到目标时仍被当成完全失败 |
| Read first | RFC §6.5、§12；`InterfaceControlViewModel.setUsbStoragePolicy()`、`PeripheralService.setUsbStoragePolicyWithCleanupResult()`、Dispatch |
| Allowed | PeripheralService、两个子 VM、Dispatch、错误映射、对应测试 |
| Forbidden | 只看异常码；用 UI 值代替 getter；吞掉重新挂载或重新插拔要求 |
| RED | 9200010 映射错误；9200007 + getter target 时 UI 回滚；DISABLED 时存储设备仍可改 allow/deny |

### 9.2 结果模型

| API 结果 | Getter | 实物运行态 | 判定 | UI 行为 |
|---|---|---|---|---|
| success | target | 已收敛 | PASS | 提交目标状态 |
| fail 9200010 | old/unknown | 未执行 | CONFLICT | 保持旧值并指向还原策略 |
| fail 9200007 | target | 未验证 | PARTIAL_APPLIED | 显示目标回读，要求重挂载/重插 |
| success/fail | not target | 未收敛 | FAIL | 保持最后确认值 |
| 任意 | getter 不可用 | 未验证 | UNKNOWN | 不猜状态，要求补证据 |

### 9.3 实施任务

1. Service 返回结构化结果：API 码、getter 值、目标值、冲突项和建议动作。
2. Dispatch 在存储 DISABLED 时拒绝 USB_STORAGE 的 allow 与 deny，防止绕过 UI。
3. Policy VM 根据已确认的目标存储状态刷新名单可编辑性。
4. 错误映射区分管理员、冲突、部分生效和未知。
5. 记录 mount/unmount 日志及重新插拔结果，不能只保留 Toast。

### 9.4 验收标准

| AC | Given | When | Then | Oracle |
|---|---|---|---|---|
| S6-AC01 | 存在设备类型 deny 冲突 | 设置存储策略 | 返回 9200010 冲突语义，提示可执行还原，状态不假成功 | reasonCode UT + MDM 列表 |
| S6-AC02 | API 返回 9200007 | getter 已是 target | 标记 PARTIAL_APPLIED，不回滚为旧值 | Service UT + getter |
| S6-AC03 | PARTIAL_APPLIED | 未重挂载/重插 | 不判最终 PASS，提示完成运行态验证 | UI + 验收记录 |
| S6-AC04 | 存储策略 DISABLED | USB_STORAGE 行改 allow/deny | UI 置灰；直接调后端同样拒绝 | UI + Dispatch UT |
| S6-AC05 | 存储策略恢复 READ_WRITE | 刷新名单 | 只解除相应门禁，不覆盖 desired | VM UT + RDB diff |
| S6-AC06 | getter 不可用 | API 返回异常 | 判 UNKNOWN/FAIL，不把异常码映射成成功 | Service UT + 日志 |

### 9.5 Done

需要错误映射 UT、系统 getter、hilog 和实物重新插拔结果。只有冲突日志或 UI Case 时判 `PARTIAL`。

---

## 10. S7｜还原与系统残留清理

### 10.1 Story 卡

| 项 | 内容 |
|---|---|
| Goal | 先清理 EDM 类型规则，再把本地意图恢复为 allow；系统清理失败时不伪造本地成功 |
| Problem | 本地无 deny 时可能隐藏还原入口；只改本地会留下 EDM 残留；直接删除卡片会丢失资产 |
| Read first | RFC §11；`PeripheralPolicyViewModel.clearAllPolicies()` → `UsbDevicePolicyStateService.clearAllPolicies()` → Dispatch |
| Allowed | Policy VM、State Service、Dispatch、State Repository、对应测试 |
| Forbidden | 先改本地再清系统；删除资产卡片；全局 USB 禁用时强行还原 L2 |
| RED | 本地无 deny 但系统有残留时按钮不可用；EDM 清理失败却全部改 allow；还原后资产卡消失 |

### 10.2 事务顺序

```text
clearAllPolicies():
  require globalUsbDisabled == false
  systemRules = dispatch.readAllUsbDeviceTypePolicies()
  if local has deny or systemRules has deny: restore entry is enabled

  systemResult = dispatch.clearAllUsbDeviceTypePolicies()
  if systemResult failed:
    keep every local row unchanged
    return FAIL

  localResult = stateRepo.upsertAll(desired=allow, active=none, keep identity)
  if localResult failed:
    return RETRYABLE_INCONSISTENT
  notify and reload records
  return PASS
```

### 10.3 实施任务

1. `hasRestorableRecords` 同时读取本地 deny/active 与 EDM 残留类型规则。
2. 全局 USB 禁用时禁用还原入口，后端再次拒绝。
3. 先清 EDM；失败时不修改任何本地状态。
4. 系统成功后批量保存 allow/none，保留 fingerprint、名称和 present。
5. 本地批量保存失败返回可重试中间态，并要求重新同步，不显示完全成功。

### 10.4 验收标准

| AC | Given | When | Then | Oracle |
|---|---|---|---|---|
| S7-AC01 | 本地全部 allow，但 EDM 有残留 deny | 刷新名单页 | 还原按钮可用 | MDM 列表 + VM 状态 |
| S7-AC02 | EDM 清理失败 | 点击还原 | 本地 desired/active 完全不变，显示失败 | UT + RDB diff |
| S7-AC03 | EDM 清理成功 | 本地批量保存成功 | 所有卡片变 allow/none，资产卡和 present 保留 | MDM + RDB + UI |
| S7-AC04 | EDM 成功、本地保存失败 | 执行还原 | 返回可重试不一致，不显示 PASS | Service UT + 错误记录 |
| S7-AC05 | 全局 USB 禁用 | 尝试还原 | 前后端均拒绝，不修改 L2 状态 | UI + VM/Service UT |

### 10.5 Done

需要清理前后 EDM 列表、RDB diff、UI 卡片和实物重插。`PER-POL-001` 或“还原策略”按钮可见只能证明入口存在。

---

## 11. S8｜分层证据闭环

### 11.1 Story 卡

| 项 | 内容 |
|---|---|
| Goal | 为每个业务结论选择适配的独立 oracle；证据不足时明确输出 UNKNOWN/PENDING |
| Problem | build PASS、UI PASS 和 AI 完成声明曾被错误汇总为业务交付完成 |
| Read first | `00-外设管理证据状态总表.md`、`06-测试验收报告.md`、`12-附录E外设验收手册.md`、E2E 原始 JSON |
| Allowed | 测试、E2E Case、Reporter、evidence、验收文档、必要可观测性改动 |
| Forbidden | 为绿色结果改变产品语义；删除旧 FAIL；伪造日志、截图、系统回读或实物结论 |
| RED | E2E 仅看到 UI 却宣称 MDM 生效；旧 FAIL 被覆盖；设备缺失时写“基本通过” |

### 11.2 证据层级

| 层 | 证明内容 | 不能证明 |
|---|---|---|
| D1 文档 | 需求、边界、AC 已冻结 | 代码已实现 |
| D2 代码 | 调用链和提交顺序存在 | 分支运行正确 |
| D3 UT | 纯逻辑、失败与补偿分支 | 系统真实执行 |
| D4 Build | 工程可编译、权限和签名链可用 | 用户价值成立 |
| D5 UI/E2E | 页面路径、控件和可见状态 | MDM、RDB、硬件行为正确 |
| D6 系统/本地回读 | restrictions、类型规则、RDB/Preferences 事实 | 实物一定可用或不可用 |
| D7 实物 | 挂载、输入、重插和双设备影响 | 所有代码分支均正确 |

### 11.3 每条验收 Case 的固定结构

```yaml
case_id: USB-G-xx
precondition:
  ui: 初始页面状态
  local: Preferences和RDB快照
  system: restrictions、storage和type rules回读
action: 一个原子动作
observations:
  ui: 页面结果和reasonCode
  local: 提交后的状态diff
  system: MDM getter结果
  physical: 挂载、输入或双设备行为
cleanup: 恢复基线并再次回读
verdict: PASS | FAIL | PARTIAL | UNKNOWN | PENDING
artifacts: 原始JSON、日志、截图或视频路径
```

### 11.4 验收标准

| AC | Given | When | Then | Oracle |
|---|---|---|---|---|
| S8-AC01 | 一个 Story 有多条 AC | 建立验收矩阵 | 每条 AC 至少绑定代码/UT，系统事实绑定 D6，用户价值绑定 D7 | acceptance matrix |
| S8-AC02 | E2E 状态 PASS | 汇总结论 | 只把实际检查的 UI 路径标 PASS，不自动提升 D6/D7 | 原始 E2E JSON |
| S8-AC03 | 新运行由 FAIL 变 PASS | 归档结果 | 同时保存旧 FAIL、新 PASS、修复提交和环境差异 | evidence 目录 |
| S8-AC04 | 必需 getter 或设备不可用 | 判定 Case | 输出 UNKNOWN/PENDING，不使用“基本通过” | 验收报告 |
| S8-AC05 | 清理失败导致环境污染 | 执行下一 Case | 停止套件并记录污染，不继续制造不可信结果 | runner/report |
| S8-AC06 | UI、RDB、MDM、实物证据冲突 | 分析失败 | 以最早分叉层定位，不用最高层截图覆盖底层冲突 | 时间线 + 原始证据 |

### 11.5 Done

1. S1–S7 的所有 AC 都进入验收矩阵。
2. 每条 Case 有前置、动作、四层观察、清理和原始附件。
3. 当前缺少的 D6/D7 明确登记责任人和补充方法。
4. 结论能从报告反向定位到测试、代码、系统回读和文件路径。
5. Evidence Pack 不覆盖旧失败，不把索引当证据正文。

---

## 12. Story Ready / Done 总门

### 12.1 Ready

- [ ] 只包含一个可独立证明的能力。
- [ ] 引用 RFC 中的真源、优先级、状态和失败语义。
- [ ] 明确 Allowed / Forbidden 路径，避免跨模块扩散。
- [ ] 至少一个 RED 测试或真实反例可复现。
- [ ] 每个 AC 都有 Given / When / Then 和独立 oracle。
- [ ] 权限、企业管理员、设备和签名依赖已登记。

### 12.2 Done

- [ ] 先更新模块设计，再提交实现。
- [ ] 所有系统调用均有成功、失败、回读不一致和必要补偿分支。
- [ ] UT 覆盖失败路径，不只覆盖 happy path。
- [ ] UI 状态来自系统回读或已确认提交，不做无条件乐观更新。
- [ ] Story 的每条 AC 在证据矩阵中有结果；未执行项为 PENDING。
- [ ] progress 记录本轮假设、RED、修改、验证、剩余未知和下一步。
- [ ] 新 Session 仅凭本 Story、RFC 与 progress 即可继续，不依赖口头补充。

## 13. 文档与提交顺序

每个 Story 的最小提交链为：

```text
1. docs(story): 冻结 Story、AC、失败语义和验证计划
2. test(story): 提交能够复现问题的 RED
3. feat/fix(story): 提交最小实现
4. test(story): 提交 GREEN、回归与设备脚本
5. docs(evidence): 回填结果、限制和证据路径
```

不得先写代码再用文档解释现状；不得把多个 Story 混入一个无法回滚和无法独立验收的提交。
