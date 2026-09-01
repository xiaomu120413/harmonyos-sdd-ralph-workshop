# 案例二证据 03｜任务拆解、验收、AI 正误判断与排障

## 1. 从穿刺方案生成任务树

| Task | 一个可观察结果 | Allowed | 验收重点 |
|---|---|---|---|
| T-GPU-00 | 保存 CPU 卡顿基线 | diagnostics / media only | 同一场景 CPU、FPS、codec path、视频 |
| T-GPU-01 | 运行时选择 OHOS hardware decoder | codec backend + build flags | decoder name、fallback reason |
| T-GPU-02 | AVC420 一帧合法输出 | OHOS codec buffer adapter | size、format、stride、plane、frameId |
| T-GPU-03 | 一帧进入正确显示 owner | bridge/output boundary | owner、target、present，不被旧 GDI 覆盖 |
| T-GPU-04 | 连续播放有界 | queue/lifecycle | backlog、drop、latency、无无限等待 |
| T-GPU-05 | resize/后台/重连恢复 | lifecycle only | generation、release/recreate、fallback |
| T-GPU-06 | AVC444 语义闭合 | dedicated 444 path | LC、单 decoder、dirty rect、EndFrame |
| T-GPU-07 | 完整 A/B 验收 | tests/scripts/evidence | 性能、画质、交互、稳定性、回退 |

每张任务卡必须写 `Forbidden`。例如 T-GPU-02 禁止顺手修改 AVC444 compositor、UI 连接页和输入系统。

## 2. AI 开发怎么协同

```text
Human / Planner
  冻结问题、任务边界、验收点和 stop 条件

Explorer
  只读输出 path:line 调用链与当前 GAP

Implementer
  一次只完成一张 Task Card，提交最小 diff 和自测证据

Tool / MCP
  构建、安装、启动、采日志、截图、录屏、性能采样

Reviewer / Evaluator
  不复述实现者结论；独立检查 trace、outcome、范围和 fallback
```

角色可以由一个人分时承担，但证据与判断必须分离。实现者说“完成”只是 review 的输入。

## 3. 怎么判断 AI 是对的

| 判定 | 必问问题 | 证据 |
|---|---|---|
| 代码路径对 | 真实运行是否选择了新 backend | subsystem/decoder/path log |
| 协议语义对 | 是否保持 FreeRDP AVC420/444、dirty rect、EndFrame 语义 | 原生路径对照 + frame trace |
| 平台用法对 | codec 生命周期、buffer、format 是否符合 HM 能力 | API return、output description、fault injection |
| 修改范围对 | 是否只改 Task Card 授权对象 | diff + forbidden scan |
| 用户结果对 | 同一视频是否更流畅、CPU 是否下降 | A/B 数据 + 同场景录屏 |
| 工程结果对 | resize、后台、断线、fallback 是否仍可用 | matrix + soak + recovery evidence |

最终结论只允许：

- `PASS`：关键断言有独立证据；
- `FAIL`：证据已经否定某项断言；
- `UNKNOWN`：没有采集到足够证据，不能用解释补齐。

## 4. AI 不对时怎么办

统一执行六步：

1. **Stop**：停止继续扩大补丁。
2. **Preserve**：保存 commit、build、设备、配置、视频、日志和失败帧。
3. **Locate**：找第一处与预期不一致的边界，不从最终黑屏倒推。
4. **Falsify**：只保留一个可证伪假设，增加一条最小观测。
5. **Repair**：回到最近可信 checkpoint，只改被证据支持的边界。
6. **Re-evaluate**：重跑目标、回归、故障注入和设备 A/B。

## 5. 典型开发问题与最小证据

| 现象 | 不要立刻做 | 第一条证据 | 可能边界 |
|---|---|---|---|
| 仍然卡顿 | 继续加线程 | decoder selected + CPU/FPS | 实际仍 fallback / queue 堵塞 |
| 黑屏 | 改 shader 颜色 | same frame input/output/owner/present | codec 无输出 / owner 过早 claim |
| 白帧 | 改 UI 背景 | upload 前像素 + remote paint readiness | 未初始化 GDI frame 被主动 repaint |
| 粉/绿块 | 猜色彩矩阵 | output format、stride、planes、rect | NV12/NV21/I420 或 plane math |
| 闪烁/旧帧覆盖 | 增加刷新 | command consumed、EndFrame、owner | GDI/GPU 双写或 present 时机错误 |
| resize 后失效 | 重建整个 session | surface/target generation | 旧 decoder target 或 stale queue |

## 6. 最终验收矩阵

| 维度 | 最低验收 | 当前材料状态 |
|---|---|---|
| Build | FreeRDP OHOS arm64 + HAP 成功 | 可从本地工程复核 |
| Path | 真机明确选择 hardware decoder，fallback 可解释 | 需要本次演示日志 |
| Correctness | AVC420/444 画面、颜色、脏区、frame boundary 正确 | 已有部分图片/视频，需绑定 runId |
| Performance | 同设备同视频 CPU 下降、FPS/卡顿改善 | `gpu-cpu-stutter-before.mp4` 与采样待补 |
| Stability | 连续播放、resize、后台、重连无死锁/无限队列 | 需要 soak evidence |
| Interaction | 画面、鼠标、键盘、滚轮使用同一 viewport | 已有动态播放/交互素材，完整矩阵待补 |
| Scope | diff 未越过 Task Card | 由 Reviewer 独立检查 |

当前仓库里的黑屏与播放视频能证明“存在失败现象”和“某次运行可见播放/交互”，不能单独证明 CPU 性能目标已经达成。课程必须保留这个 `UNKNOWN`，直到 before/after 性能证据补齐。

## 7. 最终证据包

```text
evidence/gpu/<runId>/
├── identity.json
├── codebase-map.md
├── platform-research.md
├── ADR-GPU-001.md
├── task-cards/
├── build.log
├── runtime-path.log
├── frame-trace.log
├── cpu-fps-before.csv
├── cpu-fps-after.csv
├── video-before.mp4
├── video-after.mp4
├── fault-injection.md
├── regression-matrix.md
└── reviewer-verdict.md
```
