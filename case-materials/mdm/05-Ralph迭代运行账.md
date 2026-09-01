# 阶段 5｜Ralph 迭代运行账

> `TEACHING RECONSTRUCTION`：下面用真实 Git 提交复盘“小 Story → 新上下文 → 执行 → 验证 → 写回”的 Ralph 思想，不声称这些历史提交当时由同名自动 Runner 自动完成。

## 1. Ralph 在本案例中的最小循环

```text
从 Story 队列取一个 Ready 任务
→ 读取 Worker Packet + RFC + 上轮 Progress
→ 只在允许范围内修改
→ 运行该 Story 的确定性验证
→ Reviewer 对照 AC、Diff、证据判定
→ 写回 PASS / FAIL / UNKNOWN、发现和下一任务
→ 新上下文开始下一 Story
```

循环不负责决定业务真相。它依赖外部的 RFC、Story AC、测试、设备状态和停止条件。

## 2. 每轮读取什么

```text
AGENTS.md                    项目硬规则
03-防火墙账号同步RFC.md       业务与架构契约
04-Feature与Story拆解.md      当前任务边界和 AC
progress.md                  已证明/已否定/下一步
evidence/<story-id>/          RED、GREEN、构建、设备证据
```

当前仓库没有把这套课堂文件命名为正式 Ralph Harness，因此培训中必须明确：这里展示的是可迁移运行协议，真实事实来自现有文档、代码、测试和 Git。

## 3. 真实提交映射成迭代账

| 轮次 | 真实提交 | 本轮解决的问题 | 产出/证据 | 下一轮为何存在 |
|---:|---|---|---|---|
| R1 | `9ea957d2` | 账号枚举刷新与权限配置 | Provider、权限、UT | 只有读取，没有账号变化业务同步 |
| R2 | `09209bb9` | 失效账号本地清理 | Repository 清理、UT | 仍缺少统一事件协调入口 |
| R3 | `94ff17e7` | 建立 Coordinator、Handler 和总体 reconcile | 专项方案、account service、Handler UT | 跨进程 UI 消费和规则刷新仍不完整 |
| R4 | `53751b2e` | 跨进程发布账号事实 | EventBus、Handler registry、ViewModel 调整 | 页面规则数据仍要从同一快照收敛 |
| R5 | `586880a3` | 规则展示从账号快照 reconcile | Display service / ViewModel UT | 新增账号事件与列表仍有时序窗口 |
| R6 | `4b372d0d` | 先定义稳定快照行为 | 模块设计 + 专项方案 | 设计通过后才能改 Coordinator |
| R7 | `c0c1bc9f` | 等待 trigger ID 可见 | Coordinator + RED/GREEN UT | custom 分支还需保存处理签名 |
| R8 | `9c7fb186` | custom policy 成功后保存签名 | Handler + UT | 运行时消费职责仍需进一步收口 |
| R9 | `cecf6d17` | 收敛到 ApplicationRuntimeManager | Runtime service + UT + 文档 | 进入整体回归和真机验收 |

## 4. 展开一轮：R6 → R7

### R6 文档轮

- 21:58:55，提交 `4b372d0d`。
- 更新模块设计，新增 `firewall-account-added-stable-snapshot.md`。
- 冻结：等待包含触发 ID、约 1 秒上限、超时不分发、不更新 previous。

### R7 实现轮

- 22:03:53，提交 `c0c1bc9f`。
- 只修改 `AccountChangeCoordinator.ets` 和 `account-change-coordinator.test.ets`。
- RED/GREEN 关注：先旧后新、一直旧、removed 不等待。

本轮不是“加一个重试循环”，而是增加并保护四个代码行为：

1. `runOnce()` 先调用 `loadStableSnapshot()`，`stable=false` 立即停止。
2. `shouldWaitForAddedAccount()` 只对 `account-added + triggerAccountId 不可见` 返回 true。
3. 只有稳定后才更新 `previousUserIds`、分发 Handler、发布 EventBus。
4. UT 同时断言 query、Handler、publish 和 signature，防止旧快照从其他路径泄漏。

这两轮之间只有约 5 分钟，但顺序很重要：先让方案成为可审查契约，再让 AI 进行受控执行。

## 5. progress.md 示例

```yaml
feature: FW-ACCOUNT-RECONCILE
completed:
  - S1 account provider
  - S2 local prune
  - S3 coordinator and handler
current: S5 stable snapshot
facts:
  - account event can arrive before full list visibility
  - provider full list is truth source
rejected:
  - append trigger id manually
  - refresh UI as repair
last_evidence:
  - first query [100,122]
  - second query [100,122,123]
next:
  - implement bounded wait
stop_if:
  - trigger id never visible in device trace
  - fix requires forbidden UI or provider side effect
```

## 6. 每轮 Reviewer 问五个问题

1. 本轮只改变了 Story 允许的行为吗？
2. 测试是否能反证旧实现，而不是迎合新代码？
3. 失败分支有没有保持旧的可信状态？
4. 文档、代码、测试和实际日志是否一致？
5. 当前证据允许 PASS，还是只能 FAIL/UNKNOWN？

## 7. 停止与升级

- 连续两轮没有新增证据：停止，检查规格或工具可见性。
- 需要修改禁止路径：标 `NEEDS_REPLAN`，更新 RFC/Story。
- UT 通过但设备事实不可见：标 `UNKNOWN`，进入设备穿刺，不继续扩代码。
- 发现设计与代码事实冲突：先改文档并重新评审。

## 8. PPT 截图位

- **【补充素材】**：9 个提交组成的迭代时间线。
- **【补充素材】**：`4b372d0d` 与 `c0c1bc9f` 两个真实 Git 详情。
- **【补充素材】**：一轮 progress + evidence 目录。
