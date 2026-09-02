# 外设管理真实 E2E 运行归档（2026-09-02）

这份归档保留同一轮验证中的失败、诊断、修复和复验，不用最终 PASS 覆盖第一次失败。原始 JSON 是结论真源，本页只负责建立可复核索引。

## 1. 运行对象

| 项目 | 值 |
|---|---|
| SecurityTool 基线提交 | `714d68e2038ef6570b82dee866697a764329b273` |
| E2E 桥接修复提交 | `836ddffd8fdea12c15edbcf5f4325c05f5b238b9` |
| 分支 | `codex/log-import-large-archive` |
| 应用 | `com.huawei.securitytool` / `1.0.7` / `1000007` |
| 设备 | `HAD-W32` / `3QC0124C11000711` |
| 执行后端 | `real_bridge` + `real_harmonyos_mcp_backend.py` |
| MCP 包 | `harmonyos-dev-mcp 0.9.1` |
| 签名 HAP SHA-256 | `6C2357A7B2214AD543658E3B3771770E5040C4DFD8F6BF288132D1D9221B1DF8` |

运行前已完成：用例资产校验、文档一致性校验、当前代码构建、HAP 签名与安装、企业管理员激活。构建第一次出现 `spawn java ENOENT`，按项目规范切换到 DevEco Studio 自带 JBR 后构建成功。

## 2. 执行链路

```text
case JSON
  -> Python E2E runner
  -> harmonyos_mcp_bridge.py
  -> real_harmonyos_mcp_backend.py
  -> harmonyos-dev-mcp 0.9.1
  -> HDC / UiTest
  -> HAD-W32 真机上的 SecurityTool
```

核心命令：

```powershell
$env:HARMONYOS_E2E_MCP_BRIDGE="python scripts\e2e\bridges\harmonyos_mcp_bridge.py"
$env:HARMONYOS_E2E_MCP_BACKEND_MODULE="scripts\e2e\bridges\real_harmonyos_mcp_backend.py"
python scripts/e2e/run_e2e.py --adapter security_tool --suite peripheral --device-id 3QC0124C11000711 --output-dir <run-dir>
```

## 3. 第一次真实运行：8 FAIL

- 时间：2026-09-02 10:45:24—10:48:30（Asia/Shanghai）
- 结果：`0 PASS / 8 FAIL / 0 UNKNOWN`
- 共同失败阶段：`open_peripheral_manage`
- 共同失败信息：`Failed to click sidebar item sidebar-nav-peripheral-manage`

UI 树已经找到外设管理侧栏节点和坐标，但点击结果为空。继续检查本机 MCP 工具清单后确认：桥接仍调用 `click_element / wait_element / find_element`，而 `harmonyos-dev-mcp 0.9.1` 已公开为 `click / wait_for_element / find_elements`。因此这是测试基础设施兼容问题，不能直接判定为产品缺陷。

原始证据位于 [`01-initial-fail`](./01-initial-fail/)。

## 4. 修复与单条复验

修复只发生在 E2E 桥接层：

1. 增加旧动作名到 0.9.x 新动作名的映射；
2. 将 `find_elements` 的首个结果补齐为旧桥接所需的 `element / first_match`；
3. 不修改 MDM 业务代码和模块验收口径。

单跑 `PER-IF-001` 后得到 `PASS`，再启动完整套件复验。

## 5. 最终真实运行：8 PASS

- 时间：2026-09-02 10:51:40—11:01:24（Asia/Shanghai）
- 总耗时：9 分 44 秒
- 结果：`8 PASS / 0 FAIL / 0 UNKNOWN`
- 汇总 SHA-256：`3DD4E3DA892B674D4F1851B83AEF8418E33974AAD71D02BEA76E34BE84AC0C0C`

| Case | 结果 | 耗时 | Step | UNKNOWN Step | 本轮验证重点 |
|---|---:|---:|---:|---:|---|
| `PER-IF-001` | PASS | 67s | 5 | 0 | 外设页、接口管控、USB 接口行可见 |
| `PER-IF-002` | PASS | 118s | 7 | 0 | USB 接口策略交互与恢复后的页面回显 |
| `PER-REC-001` | PASS | 62s | 4 | 0 | 设备连接记录页与导出入口 |
| `PER-POL-001` | PASS | 63s | 4 | 0 | 黑白名单页与还原策略入口 |
| `PER-POLICY-002` | PASS | 79s | 7 | 0 | 存储设备策略交互、恢复和页面回显 |
| `PER-USB-001` | PASS | 72s | 5 | 0 | USB 默认策略入口和选项 |
| `PER-WL-USB-001` | PASS | 62s | 4 | 0 | USB 名单页和导出入口 |
| `PER-BL-USB-001` | PASS | 61s | 4 | 0 | 黑名单页与还原策略入口 |

每条用例的 `launch_app` 都记录真实 `hdc ... aa start` 命令与 `start ability successfully.`；MCP 动作证据记录 `execution_backend=real_bridge`、bridge 命令和真实 backend。完整结果位于 [`02-final-pass`](./02-final-pass/)。

## 6. 可视证据

![外设管理最终状态](./02-final-pass/peripheral-final-state.jpeg)

截图来自 MCP `screenshot` 真机调用，大小 374,510 bytes，SHA-256 为 `5AE0B5D02497FCF35170FEF9420EE95D576D35E040CF96F8B11ACDA993480458`。截图可见：

- 当前处于“外设管理 → 黑白名单”；
- USB 默认策略为“允许接入”；
- 真机识别到 `Kingston DataTraveler 3.0`；
- 当前策略和最近连接时间已在页面回显。

## 7. 归档结构

```text
2026-09-02-peripheral-real/
├─ README.md
├─ cases/                 # 本次执行的 8 份 case JSON 快照
├─ 01-initial-fail/       # 8 份失败结果、suite summary、console log
└─ 02-final-pass/         # 8 份通过结果、suite summary、console log、真机截图
```

## 8. 证据边界与待补素材

本轮能够证明：当前 HAP 可构建、签名、安装和启动；外设模块可以通过真实 UI 自动化访问；8 条页面/交互/回显断言全部通过；测试基础设施问题已被定位、修复并复验。

本轮仍不能单独证明：USB 物理插拔后内核访问是否被阻断、全局白名单与设备级名单的真实系统优先级、重启后的策略持久化、MDM 系统接口 readback 与 UI 完全一致。

需要用户补充的培训素材：

- `[需用户补充｜USB 实物视频]` 插拔前后设备是否可用；
- `[需用户补充｜系统 readback]` 全局策略、设备名单和最终生效策略的命令/接口读取结果；
- `[需用户补充｜策略优先级]` 全局白名单、USB 黑白名单、设备单独设置三层冲突矩阵的实测记录。
