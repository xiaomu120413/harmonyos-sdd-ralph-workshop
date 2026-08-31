# T01 行为级 RED / GREEN 证据

> 用途：第 15–17 页课堂回放。这里保存的是可复核的命令与判定，不把编译错误、环境错误或覆盖率工具错误伪装成测试 RED。

## 身份

- 需求：`FR-ACC-001`，账号新增事件到达时，旧账号快照不得进入 handler。
- RED 基线：`4b372d0d`（只加入目标行为测试，不带生产修复）。
- GREEN 实现：`c0c1bc9f`。
- 命令：`hvigorw test --mode module -p product=default -p module=entry@default`
- 复核日期：2026-08-31。

## RED：失败点与需求同源

测试数据让系统账号读取依次返回 `[100,122]`、`[100,122,123]`，期望第二次读取后才 dispatch。基线实现第一次读取就 dispatch，真实输出为：

```text
ERROR: Error in should wait until added account appears before dispatching handlers,
expect 1 equals 2
at entry/src/test/firewall/account-change-coordinator.test.ets:77:26

BUILD SUCCESSFUL in 16 s 961 ms
HVIGOR_EXIT=0
```

解释：`expect 1 equals 2` 说明读取次数只有 1，旧快照提前进入 handler。这是目标行为 RED，不是 import、SDK、mock 或编译失败。

同时暴露了一个工具链陷阱：Hvigor 仍打印 `BUILD SUCCESSFUL` 且退出码为 0。因此课堂判定必须读取用例级 `ERROR: Error in ...` 或测试结果文件，不能只看构建横幅和进程退出码。

## GREEN：同一行为用例通过

切换到 `c0c1bc9f` 后运行同一命令，目标断言不再出现，测试结果文件记录：

```text
class=AccountChangeCoordinator
test=should wait until added account appears before dispatching handlers
result=Success

BUILD SUCCESSFUL in 10 s 911 ms
HVIGOR_EXIT=0
```

历史开发 Session 也保存了该实现后的全模块 GREEN，例如：

```text
2026-07-01 16:03:52  BUILD SUCCESSFUL in 14 s 659 ms
2026-07-01 16:09:25  BUILD SUCCESSFUL in 10 s 763 ms
```

## 独立 GAP：覆盖率报告器噪声

RED 与 GREEN 两次运行都出现：

```text
00507008 getInitCoverageData failed
```

它没有随生产修复消失，因此不能作为目标行为的 RED/GREEN。当前结论是：

| 判定项 | 结果 |
|---|---|
| 目标行为 RED | VERIFIED |
| 目标行为 GREEN | VERIFIED |
| 测试命令退出码可独立判定结果 | NO |
| 覆盖率报告链路 | GAP |

## 可迁移规则

1. RED 必须失败在目标断言，环境失败不能推动实现。
2. GREEN 必须检查同一目标用例，而不是换一条更容易通过的命令。
3. 构建结果、测试结果、覆盖率结果分别判定；任何一个横幅都不能替代用例级证据。
