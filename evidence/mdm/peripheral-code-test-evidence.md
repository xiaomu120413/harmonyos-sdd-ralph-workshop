# 外设管理代码、测试与历史证据

源工程：`C:\Users\mu\Desktop\code\security_tool`

## 1. 关键代码事实

| 事实 | 锚点 |
|---|---|
| USB UI 入口进入独立全局编排 | `InterfaceControlViewModel.toggleInterface()` → `UsbGlobalPolicyService.setDisabled()` |
| 全局状态由 restrictions 回读 | `PeripheralService.getInterfaceDisabledWithResult(USB)` |
| 默认策略是独立 Preferences key | `PeripheralDevicePolicyRepository.PREF_KEY_USB_DEFAULT_POLICY` |
| 默认非法值回退 allow | `normalizeUsbDefaultPolicy()` |
| 设备意图、在线态、执行态分离 | `UsbDevicePolicyStateRecord.desiredPolicy/present/activePolicy` |
| 默认 allow 也保存资产状态 | `handleConnect()` 的 `shouldSave` |
| 默认 deny 系统成功后才保存 | `dispatchResult?.success === true` |
| 离线设备拒绝动态修改 | `UsbDevicePolicyStateService.setPolicy()` |
| 全局禁用失败恢复显式 deny | `UsbGlobalPolicyService` failure branch |
| 还原先清 EDM，再写本地 allow | `clearAllUsbDeviceTypePolicies()` → `clearAllPolicies()` |
| 系统下发按 baseClass，不是 fingerprint | `toUsbDeviceType()` / `buildUsbDeviceType()` |

## 2. 关键测试事实

| 测试文件 | 关键覆盖 |
|---|---|
| `device-policy-repository.test.ets` | default allow、persisted deny、invalid→allow、save failure |
| `usb-device-policy-state-service.test.ets` | allowlist、deny success/failure、restore cards、cleanup failure |
| `PeripheralPolicyViewModel.test.ets` | state-only candidates、storage conflict、stale read override、restore residue |
| `usb-global-policy-service.test.ets` | global disable/enable、suspend、compensation、snapshot |
| `InterfaceControlViewModel.test.ets` | controlled state、failure stage、Bluetooth marker rollback |

2026-09-01 coverage report 摘要：

| 文件 | 行 | 函数 | 分支 |
|---|---:|---:|---:|
| UsbGlobalPolicyService | 91.67% | 80.00% | 81.82% |
| InterfaceControlViewModel | 88.16% | 100.00% | 64.15% |
| PeripheralPolicyService | 90.91% | 75.00% | 83.33% |
| UsbDevicePolicyStateService | 48.50% | 54.55% | 33.33% |
| PeripheralDevicePolicyDispatchService | 10.46% | 15.00% | 10.71% |

解读：业务编排已有较多自动化覆盖，但系统适配层覆盖低，必须用 MDM 回读和实物矩阵补证。

## 3. 方案演进证据

| Commit | 变化 | 固化的不变量 |
|---|---|---|
| `63dda4b4` | USB 策略状态独立建模 | Trace 不是真源 |
| `a2f0128b` | 策略清理职责收敛 | 清理动作由领域服务负责 |
| `093cb6e4` | 还原时保留卡片 | 还原不是删除 |
| `786e370c` | 修正还原按钮状态 | UI 直接绑定 VM 真状态 |
| `6e7702cd` | 全局 USB 与设备策略协同 | 暂停、补偿、恢复重放 |
| `23c4a046` | 修复默认 allow 设备名单 | allow 也需要资产状态 |
| `0d26c92e` | 启用快照重枚举重试 | 系统重枚举是有界最终一致 |
| `f95c5109` | USB 存储策略状态同步 | 已确认目标避免即时旧回读污染 UI |

## 4. 仍不能由这些证据证明

- 所有 USB 类型都在目标 2in1 上可用；
- fingerprint 对应系统精确单硬件阻断；
- 全局恢复的每个 deny 都在真机成功重放；
- 跨重启、跨升级和长时间运行状态完全一致。

这些结论必须进入系统回读与实物验收，不得由代码审阅或 coverage 推断。
