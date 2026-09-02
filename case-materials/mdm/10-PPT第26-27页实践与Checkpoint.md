## 第 26 页｜实践：把外设策略验收写成可执行系统契约

### PPT 内容

学员任选一个场景，写完整 Case：

```text
A 默认 allow 新设备
B 默认 deny 新设备
C 全局禁用/恢复
D 还原系统残留
E 同 baseClass 双设备影响面
F 全局恢复后显式 deny 重放
G 9200007：API 报错但回读已改变
H Trace 已入库但当前页面未刷新
```

Case 必须包含：

```yaml
precondition:
  admin: activated
  global_usb: enabled
  storage_policy: read_write
  local_policy_state: baseline
action:
  - set_default_deny
  - attach_usb_device
oracles:
  ui: device appears in blacklist
  local: desired=deny, active=deny, present=true
  system: disallowed usb type is readable
  physical: device cannot be used
cleanup:
  - restore_all_policies
result: PENDING
```

### 怎么做 / 怎么判断 / 不对怎么办

- 怎么做：先写预置和清理，再写动作，最后定义四栏 oracle。
- 怎么判断：换一台设备、换一个 Session 仍能重复执行，并能区分 PASS/PARTIAL_APPLIED/FAIL/UNKNOWN。
- 不对怎么办：无法系统回读时标 UNKNOWN；清理失败时停止下一 Case，避免污染。

### 讲师备注

- 要求学员特别写出“同类型第二台设备”，验证平台 baseClass 粒度。
- 要求学员先写出冻结规则：全局接口管控 > 单设备显式规则 > 默认黑白名单；USB 存储读写策略并行判断。
- 对 `9200007` 不允许只看 API 返回值，至少同时检查 getter、挂载状态和重插结果。
- 强调视频是实物证据，但视频里还要同步出现操作、设备和时间/Case ID。

### 文档 / 截图

- 文档：`case-materials/mdm/06-测试验收报告.md#5-真机验收-runbook`
- **【需用户补充 U02｜视频占位】**：默认 deny 插入 U 盘，UI/系统回读/实物行为同屏。

---

## 第 27 页｜MDM Checkpoint：交付的不是代码，而是一条可复核证据链

### PPT 内容

```text
需求卡
  → 冲突清单
  → 穿刺与 ADR
  → MVVM / 策略 RFC
  → Story + Worker Packet
  → Ralph 运行账
  → UT / E2E / 系统回读 / 实物视频
  → 验收矩阵
```

本案例当前状态：

| 维度 | 结论 |
|---|---|
| 架构、规则、代码映射 | PASS |
| 关键分支 UT | PASS / 有覆盖缺口 |
| 页面 E2E | PASS（保留一份旧 FAIL） |
| 系统回读 | **[需用户补充 U01–U08] PENDING** |
| 实物 USB 矩阵 | **[需用户补充 U01–U08] PENDING** |

学员应能回答：

1. AI 为什么这样拆是对的？
2. 哪个状态是谁的真源？
3. 系统失败后怎样补偿或回退？
4. 哪份证据能证明哪层事实？
5. 证据不足时为什么要停止并升级？

### 怎么做 / 怎么判断 / 不对怎么办

- 怎么做：从验收矩阵任取一行，反向追到 Story、RFC、代码和需求。
- 怎么判断：链路任一跳都能打开真实文件或结果。
- 不对怎么办：断链处就是下一项工作，不用一句“整体完成”覆盖。

### 讲师备注

- 这一页把“复杂需求交付”收口到可复核，而不是代码量。
- 然后切入案例二：同样的方法如何进入 55 万行级 FreeRDP 与 GPU 性能问题。

### 文档 / 截图

- 文档：`case-materials/mdm/00-外设管理证据状态总表.md`
- 文档：`case-materials/mdm/09-MDM案例整体审阅与落版规则.md`
- **【可本地生成 L01/L02/L04】**：六阶段文档目录、关键代码、E2E FAIL→PASS 组成证据墙；U01–U08 视频位按 `16-用户补充素材清单.md` 保持明显占位。

---
