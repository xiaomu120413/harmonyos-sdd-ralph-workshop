# 阶段 4｜Feature 与 Story 拆解

## 1. Feature 定义

```yaml
feature_id: FW-ACCOUNT-RECONCILE
goal: 系统账号变化后，防火墙系统状态和本地状态收敛到最新账号集合
entry: EnterpriseAdminAbility account event
truth_source: SystemUserProvider.loadAvailableUserIds
done: public/private/custom/removed/empty/concurrent 场景均有证据
```

## 2. 为什么这样拆

Story 不按“一个人写页面、一个人写 Service”拆，也不按文件数量平均切。拆分依据是：

- 每一块能力有独立职责和独立 oracle。
- 高风险平台假设先穿刺。
- 每轮修改范围能放进一个新鲜上下文。
- 失败时知道退回哪一层。
- 后续 Story 可以复用前一 Story 已证明的事实。

## 3. Story 地图

```text
S1 账号真相源与资产对齐
  ↓
S2 删除账号的本地数据清理
  ↓
S3 公共协调器与防火墙 Handler
  ↓
S4 跨进程稳定事实与 UI 运行时消费
  ↓
S5 新增账号稳定快照门
  ↓
S6 custom 模式签名与全链验收
```

## 4. Story 明细

### S1｜读取真实账号集合

- 目标：所有用户列表统一从 `SystemUserProvider` 获取并标准化。
- 允许路径：Provider、权限/签名配置、对应测试和设计文档。
- 禁止：Provider 内下发防火墙策略。
- AC：去重、排序、前台用户标识、失败返回空态；权限配置一致。
- 证据：`system-user-provider.test.ets`、提交 `9ea957d2`。

### S2｜删除账号时清理本地引用

- 目标：从 intent targetUserIds、deployments、用户历史策略中删除失效账号。
- 允许路径：`FirewallLocalRepository`、对应 Service/ViewModel、UT。
- 禁止：对被删除账号调用系统防火墙 API。
- AC：多用户规则只移除该 ID；仅该 ID 的规则删除；空/失败集合不 prune。
- 证据：`local-repository.test.ets`、提交 `09209bb9`。

### S3｜协调账号事件并分发模块 Handler

- 目标：事件只触发；协调器读取完整集合，计算签名/差量，按注册表分发。
- 允许路径：`services/account/**`、`FirewallAccountChangeHandler`、Ability 接入、UT、模块设计。
- 禁止：`EnterpriseAdminAbility` 直接写防火墙；UI 作为 Handler。
- AC：400ms 防抖、single-flight、pending 再跑；public/private/custom/empty 分支明确。
- 证据：`account-change-handler.test.ets`、提交 `94ff17e7`。

### S4｜跨进程传递稳定账号事实

- 目标：系统事件进程完成业务 reconcile 后，把稳定账号事实送到 UI 运行时；页面重读模型数据。
- 允许路径：EventBus、AccountRuntimeService、ApplicationRuntimeManager、ViewModel、UT。
- 禁止：`MainPage` 按路由硬编码业务刷新；弹窗各自读取系统账号。
- AC：EventBus/运行时只有一条消费链；rules/userOptions/policy 使用同一快照刷新。
- 证据：提交 `53751b2e`、`586880a3`、`cecf6d17`。

### S5｜新增账号必须等待稳定快照

- 目标：account-added 只有在完整集合包含触发 ID 后才分发。
- 允许路径：模块设计、专项方案、Coordinator、Coordinator UT。
- 禁止：把 trigger ID 直接 append 到集合；超时仍更新 previous。
- AC：首次 `[100,122]`、后续 `[100,122,123]` 时只分发一次稳定集合；持续旧集合时不分发、不 publish。
- 证据：文档提交 `4b372d0d`；代码/测试提交 `c0c1bc9f`。

### S6｜custom 模式保存已处理账号签名

- 目标：custom 默认 policy 同步成功后保存 `custom + signature`，但不扩规则作用域。
- 允许路径：Handler、模块设计、Handler UT。
- 禁止：把旧 rules 的 targetUserIds 自动加入新用户。
- AC：policy 成功 + 保存签名；保存失败返回 false；intent/deployment 不新增。
- 证据：`account-change-handler.test.ets`、提交 `9c7fb186`。

## 5. Worker Packet 示例（S5）

```yaml
story_id: FW-ACCOUNT-S5-STABLE-SNAPSHOT
goal: account-added 只分发包含 triggerAccountId 的稳定账号快照
inputs:
  - 03-防火墙账号同步RFC.md#核心不变量
  - firewall-account-added-stable-snapshot.md
allowed_paths:
  - docs/03-模块设计/防火墙管理组件设计说明.md
  - entry/src/main/ets/services/account/AccountChangeCoordinator.ets
  - entry/src/test/firewall/account-change-coordinator.test.ets
forbidden:
  - FirewallPage.ets
  - SystemUserProvider 内业务副作用
acceptance:
  - old_then_new_dispatch_once
  - never_visible_dispatch_zero
  - removed_and_manual_read_once
commands:
  - python scripts/check_docs_consistency.py
  - hvigorw test --mode module -p product=default -p module=entry@default
stop:
  - repeated_failure_without_new_evidence
  - need_to_modify_forbidden_path
```

## 6. Ready / Done 门

### Ready

- RFC 已批准，真相源和失败语义明确。
- 依赖 Story 已 PASS。
- 允许/禁止路径已列出。
- RED 测试或手工反证步骤可执行。

### Done

- AC 有逐条证据。
- 文档、代码、测试映射一致。
- Diff 未越界。
- 必需验证门 PASS；未执行门明确 UNKNOWN。
- 结果与下一 Story 写回运行账。

## 7. PPT 截图位

- `PLACEHOLDER`：Story 地图。
- `PLACEHOLDER`：S5 Worker Packet 真实 Markdown 截图。
- `PLACEHOLDER`：对应 Git 提交文件列表。

