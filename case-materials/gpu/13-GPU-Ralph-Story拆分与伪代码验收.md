# GPU 案例｜Ralph Story 拆分、伪代码与验收点

> 本文参考 MDM 案例的 Feature / Story 结构，把 GPU 方案拆成 Ralph 可以逐卡执行的 Story。Story 不是功能标题，也不是文件清单；它必须冻结一个可观察结果、伪代码级行为、验收点、证据和停止条件。

## 1. 三层不要混在一起

```text
Plan 阶段：安排先后顺序，回答“先做什么、后做什么”
  ↓
Story：冻结一个可独立证明的行为，回答“这一段完成后用户/系统得到什么”
  ↓
Ralph Worker Packet：限定一次 Session 的上下文、源码范围、验证命令和 Stop
```

同一个 Plan 阶段可以包含多个 Story；一个 Story 可以经过多轮 Ralph 循环，但每轮只能收敛当前 Story 的一个未知。进入门没有通过时，不允许 AI 顺便修改后续 Story。

## 2. Feature 定义

**Feature：HarmonyOS 远控视频硬解与 GPU 显示路径**

在保留 FreeRDP 协议语义和原生软件路径的前提下，让 HarmonyOS 远控视频能够选择硬件解码、获得可验证输出、在正确帧边界显示，并在连续播放、生命周期变化、AVC444 和失败场景中保持可回退、可观测、可验收。

Feature 级不变量：

```text
接管前失败 → 必须回原路径
接管后成功 → 同一帧只有一个显示 Owner
没有匹配显示边界 → 不允许提前 present
证据不足 → 只能是 UNKNOWN / PENDING
```

## 3. Story 地图

| Story | 对应 Task | 可独立交付的结果 | 主要修改面 | 进入下一 Story 的验收门 |
|---|---|---|---|---|
| S0 基线与观测 | T-GPU-00 | 同一 runId 保存 CPU 路径、卡顿视频和基线指标 | diagnostics / scripts | 当前路径和问题可复查 |
| S1 真机硬解选择 | T-GPU-01 | 真机选择硬件 decoder，失败仍可回旧路径 | hook / bridge / decoder init | decoder identity 为硬件类别 |
| S2 一帧合法输出 | T-GPU-02 | 一个短片段产生可关联、可解释的解码结果 | codec input/output adapter | 输入与输出属于同一帧 |
| S3 正确显示与回退 | T-GPU-03 | 结果只在正确边界显示，接管前失败可回退 | bridge / compositor / present | 画面、日志、回退三证据齐全 |
| S4 连续播放有界 | T-GPU-04 | 连续播放时队列和延迟不持续增长 | queue / backpressure | depth / age / order 达标 |
| S5 生命周期恢复 | T-GPU-05 | resize、后台、重连后拒绝旧目标并恢复 | generation / lifecycle | 四类场景均无旧任务写入 |
| S6 AVC444 语义 | T-GPU-06 | LC 与双 bitstream 按 retained 语义闭合 | AVC444 policy / compositor | 不能复用 AVC420 假设冒充通过 |
| S7 A/B 工程验收 | T-GPU-07 | 汇总正确性、性能、稳定、交互和回退证据 | test / scripts / report | Reviewer 可独立复查 |

```text
S0 → S1 → S2 → S3 → S4 → S5 → S6 → S7
```

## 4. Ralph Worker Packet 固定格式

每个 Story 进入 Ralph 前必须生成以下 Worker Packet：

```text
Story ID / Outcome
Input：上游文档、上一 Story verdict、源码锚点、设备与场景
Source Scope：允许读取和修改的文件/符号
Allowed：本卡允许的修改
Forbidden：明确禁止顺便处理的内容
RED / Probe：先看到的失败或最小探针
Pseudocode：输入、分支、状态提交和失败语义
Acceptance Criteria：场景、预期、证据、失败结论
Verify：可重复执行的命令或设备动作
Stop：何时停止扩大范围
Output：diff、日志、截图/视频、progress verdict
Next：进入下一 Story 所需的 Exit Gate
```

Ralph 循环固定为：

```text
读取 Worker Packet
  → 只做一个最小修改
  → 运行本卡 Verify
  → 对照 AC 找第一处不一致
  → 只修当前证据边界
  → 同场景重放
  → 写回 PASS / FAIL / UNKNOWN / NEEDS_REPLAN
```

---

## 5. Story 明细

## S0｜保存 CPU 路径与卡顿基线

**As a** Reviewer，**I want** 在不修改生产路径的前提下保存 before，**so that** 后续硬解结果有可比较的事实基线。

### Ralph 边界

- Source Scope：diagnostics、日志、性能采样、录屏脚本。
- Allowed：增加 runId、codec path、CPU/FPS、queue 指标采集。
- Forbidden：修改 decoder、compositor、queue policy、renderer。
- RED / Probe：当前截图能看到视频，但无法把卡顿、CPU 路径和指标绑定到同一次运行。

### 伪代码

```text
captureBefore(scene):
  require fixed(device, server, resolution, clip, network)
  runId = newRunId(commit, device, scene)
  startVideoRecording(runId)
  startMetricSampling(runId, CPU, FPS, queueDepth, frameTime)
  path = observeDecoderAndRenderPath(runId)
  play(scene.clip, 30s)
  stopAndArchive(runId)

  if path unknown:
    return UNKNOWN with observability gap
  if video or metrics missing:
    return PENDING
  return PASS with beforeEvidenceIndex
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S0-AC1 | 固定场景播放 30 秒 | 设备、服务端、分辨率、片段一致 | scene manifest | `FAIL`，重录 |
| S0-AC2 | 判断当前路径 | 日志明确 software / CPU path | path log + runId | `UNKNOWN`，先补观测 |
| S0-AC3 | 采集基线 | CPU、FPS、queue、frame time 同 runId | CSV | `PENDING` |
| S0-AC4 | 保存现象 | 视频能看出实际播放与交互 | before video | `PENDING` |
| S0-AC5 | 建立索引 | commit、runId、设备、视频、日志互相可跳转 | evidence index | 不得进入 S1 |

**Stop：** 无法证明当前 decoder/path 时，不允许开始改硬解；先生成观测性子任务。

---

## S1｜真机选择 HarmonyOS 硬件 Decoder

**As a** 远控视频管线，**I want** 在目标设备选择硬件 H.264 decoder，**so that** 后续输出确实来自新路径，同时不破坏旧路径回退。

### Ralph 边界

- Source Scope：`InstallRdpgfxDiagnosticsHooks`、`freerdp_ohos_rdpgfx_bridge_attach`、OH_AVCodec capability/create/configure/prepare/start。
- Allowed：build flags、subsystem registration、capability 与 selection logs。
- Forbidden：修改合成、显示 Owner、AVC444、性能队列。
- RED / Probe：日志只能看到 API 调用，不能证明实际 decoder 类别。

### 伪代码

```text
selectDecoder(stream, device):
  attachResult = attachRdpgfxBridge(originalCallbacks)
  if attachResult failed:
    record("bridge_attach_failed")
    return USE_ORIGINAL_PATH

  capability = queryHardwareH264(device, stream.profile)
  if capability unsupported:
    record("hardware_decoder_unsupported")
    return USE_ORIGINAL_PATH

  decoder = createAndStart(capability)
  if decoder failed:
    record(stage, reason, decoderName)
    return USE_ORIGINAL_PATH

  record(hardware=true, decoderName, configure, start)
  return GPU_PATH_READY(decoder)
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S1-AC1 | 构建目标 HAP | bridge / compositor 符号进入 arm64 产物 | symbol + package log | `FAIL` |
| S1-AC2 | 启动真机 | bridge attach 成功且保存原回调 | attach log | `FAIL` |
| S1-AC3 | 选择 decoder | decoder identity 明确为硬件类别 | capability + decoder name | `UNKNOWN/UNSUPPORTED` |
| S1-AC4 | 初始化 | configure / prepare / start 全部成功 | stage log | `FAIL`，停在失败阶段 |
| S1-AC5 | 模拟不支持 | 不 suppress 原路径，应用仍可播放 | fallback reason + video | `FAIL`，不得进入 S2 |

**Stop：** 设备不提供目标硬件能力时返回 `UNSUPPORTED`，不能用“调用了硬解 API”冒充 PASS。

---

## S2｜一个短片段产生合法解码结果

**As a** compositor，**I want** 把一份压缩输入关联到一份可解释的解码输出，**so that** codec 问题和显示问题可以分开验收。

### Ralph 边界

- Source Scope：`avc420_gpu_compositor_internal.cpp` 的 input/output buffer、PTS、output description、`OH_AVBuffer_GetNativeBuffer`。
- Allowed：codec buffer adapter、format / stride / plane diagnostics、bounded wait。
- Forbidden：AVC444、显示 Owner、EndFrame、zero-copy 优化、队列重构。
- RED / Probe：output callback 返回，但无法证明它属于哪份输入或能否显示。

### 伪代码

```text
decodeOneSample(sample, storyFrameId):
  inputPts = allocatePts(storyFrameId)
  queueInput(sample.bytes, inputPts)
  output = waitOutput(timeout = bounded)

  if timeout: return FAIL("decoder_output_timeout")
  if output.pts != inputPts: return FAIL("input_output_unrelated")

  description = readOutputDescription(output)
  nativeBuffer = getNativeBuffer(output)
  if description.format not supported:
    preserve(description); return NEEDS_REPLAN
  if nativeBuffer missing:
    return FAIL("native_buffer_missing")

  return PASS { storyFrameId, inputPts, outputPts,
                size, format, stride, planes, nativeBufferIdentity }
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S2-AC1 | 输入已知 sample | bytes、PTS、frame 标识已记录 | input trace | `FAIL` |
| S2-AC2 | 等待输出 | bounded wait 内出现 output | callback + wait time | `FAIL` |
| S2-AC3 | 关联输入输出 | input PTS 与 output PTS 可映射到同一帧 | trace row | `UNKNOWN` |
| S2-AC4 | 校验格式 | size / format / stride / planes 完整且受支持 | output description | `NEEDS_REPLAN` |
| S2-AC5 | 获取 native buffer | buffer identity 可记录且生命周期可追踪 | buffer log | `FAIL`，不得进入 S3 |

**Stop：** 只看到 output callback 不算 PASS；输出格式未知时保留原始描述并停止扩大显示改动。

---

## S3｜在正确边界显示，并能失败回退

**As a** 用户，**I want** 硬解画面在正确时机显示且失败时仍可使用旧方案，**so that** 新路径不会带来黑屏、撕裂或不可恢复。

### Ralph 边界

- Source Scope：AVC420 candidate policy、`OnSurfaceCommand`、`ProcessCommand`、`PresentQueuedUpdate`、`PresentEndFrame`。
- Allowed：bridge、最小合成接线、显示 Owner、present 与 fallback diagnostics。
- Forbidden：性能优化、AVC444、输入系统、生命周期大重构。
- RED / Probe：现有截图只能证明画面可见，不能证明硬解、正确 present 或回退。

### 伪代码

```text
handleSurfaceCommand(command):
  if gpuPath not ready:
    return originalSurfaceCommand(command)

  candidate = validateCandidate(command)
  if candidate failed:
    record(fallbackReason)
    return originalSurfaceCommand(command)

  decoded = decode(command)
  if decoded failed before takeover:
    record(fallbackReason)
    return originalSurfaceCommand(command)

  claimDisplayOwner(command.frame)
  compose(decoded, command.rects)
  markPending(command.frame)
  return CONSUMED

onEndFrame(frame):
  if frame != pendingFrame: return NO_PRESENT
  present(frame)
  releasePending(frame)
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S3-AC1 | 正常短片段 | 硬解结果进入显示模块且画面可见 | screenshot/video + run log | `FAIL` |
| S3-AC2 | 显示边界 | 只有匹配帧结束时才 present | ordered trace | `FAIL` |
| S3-AC3 | Owner 唯一 | 同一帧不存在旧路径与 GPU 双写 | owner transition | `FAIL` |
| S3-AC4 | 接管前故障注入 | 记录原因并回原路径，应用仍可播放 | failure log + fallback video | `FAIL` |
| S3-AC5 | 穿刺结论 | 画面、硬解日志、回退三类证据同 commit/runId | evidence bundle | 缺项则 `PENDING` |

**Stop：** GDI/GPU 双写、提前 present 或回退不可达时立即停止；不得用“屏幕亮了”替代完整穿刺证据。

---

## S4｜连续播放时队列与延迟保持有界

**As a** 用户，**I want** 视频连续播放不会越播越慢，**so that** 硬解接入不会把一帧成功变成持续积压。

### Ralph 边界

- Source Scope：`EnqueueSurfaceCommand`、worker backlog/compaction、EndFrame coalescing、queue metrics。
- Allowed：bounded queue、backpressure、drop/compaction reason、指标。
- Forbidden：无界线程、无条件清队列、通过降低画质掩盖顺序问题。
- RED / Probe：短片段通过，但长片段 queue depth / age 是否增长未知。

### 伪代码

```text
enqueue(command):
  if target paused or generation stale:
    drop(command, "stale_target")
    return

  if queue at capacity:
    compactOnlyWhenFrameSemanticsPreserved()
    if cannot preserve order:
      applyBackpressureOrDropWithReason(command)
  queue.push(command)
  record(depth, oldestAge, dropReason)

workerLoop():
  command = queue.popInOrder()
  process(command)
  record(decodeUs, composeUs, presentGap, frameOrder)
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S4-AC1 | 连续播放目标时长 | queue depth 不持续增长 | depth timeline | `FAIL` |
| S4-AC2 | 高负载 | command age p95 保持在验收阈值内 | age CSV | `FAIL` |
| S4-AC3 | 触发 compact/drop | 每次都有原因，且不破坏帧顺序 | reason + order trace | `FAIL` |
| S4-AC4 | 前后对照 | CPU/FPS 与 S0 同场景可比较 | A/B CSV | `PENDING` |
| S4-AC5 | 长时运行 | 无持续内存/延迟爬升 | soak report | `FAIL/UNKNOWN` |

**Stop：** backlog 持续增长或顺序被破坏时，回到第一处异常；不能先增加线程数。

---

## S5｜生命周期变化后拒绝旧目标并恢复

**As a** 用户，**I want** resize、切后台、Surface 重建和重连后继续看到正确画面，**so that** 旧任务不会写入已经失效的目标。

### Ralph 边界

- Source Scope：decoder/surface generation、pause/detach/recreate、stale worker rejection、Owner release/reclaim。
- Allowed：generation、局部 release/recreate、恢复与 fallback。
- Forbidden：用重建整个业务 Session 掩盖局部状态错误。
- RED / Probe：恢复后能看到画面，但旧 worker 是否仍能写入未知。

### 伪代码

```text
onTargetChanged(newTarget):
  oldGeneration = activeGeneration
  activeGeneration += 1
  pauseInput(oldGeneration)
  releaseOwner(oldGeneration)
  rejectQueuedTasks(where generation == oldGeneration)
  bind(newTarget, activeGeneration)
  resumeInput(activeGeneration)

process(task):
  if task.generation != activeGeneration:
    drop(task, "stale_generation")
    return
  decodeComposePresent(task)
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S5-AC1 | resize | generation 更新，旧任务被拒绝，画面恢复 | generation trace + video | `FAIL` |
| S5-AC2 | 前后台 | pause/resume 顺序明确，无旧目标 present | lifecycle trace | `FAIL` |
| S5-AC3 | Surface 重建 | Owner 释放后在新目标重新获取 | owner transition | `FAIL` |
| S5-AC4 | 重连 | 旧 decoder/target 不再写入，新链恢复 | reconnect trace | `FAIL` |
| S5-AC5 | 故障路径 | 局部恢复失败时进入可解释 fallback | reason + visible recovery | `FAIL` |

**Stop：** 旧 target 仍能 present 即判 FAIL，即使肉眼看到恢复后的新画面。

---

## S6｜AVC444 按 LC 与 retained state 闭合

**As a** AVC444 视频管线，**I want** 正确处理 LC、双 bitstream 与 retained state，**so that** AVC444 不会被当成两段普通 AVC420 视频错误合成。

### Ralph 边界

- Source Scope：`ohos_rdpgfx_record_avc444_gpu_candidate`、AVC444 compositor/decoder policy、LC / stream retained state、EndFrame。
- Allowed：AVC444 专用 policy、compositor、diagnostics。
- Forbidden：直接复用 AVC420 dirty-rect 假设、创建两个互不关联的 decoder 状态。
- RED / Probe：AVC420 已通过，但 AVC444 的 LC 和 retained readiness 仍是未知。

### 伪代码

```text
handleAvc444(command):
  require validLC(command.lc)
  state = retainedState(command.surface)

  decodeAndApply(stream1, state, command.lc)
  decodeAndApply(stream2, state, command.lc)
  if retained prerequisites incomplete:
    record("avc444_not_ready")
    return originalPathWithoutSuppressingGdi()

  markPending(command.frame, state)
  return CONSUMED

onMatchingEndFrame(frame):
  presentRetainedState(frame)
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S6-AC1 | AVC444 sample | LC 值和两个 stream 顺序可追踪 | frame trace | `UNKNOWN` |
| S6-AC2 | decoder 状态 | 使用一个一致的 decoder / retained 状态 | identity log | `FAIL` |
| S6-AC3 | retained readiness | 前置状态未满足时不接管 | readiness + fallback | `FAIL` |
| S6-AC4 | 正常显示 | 只在匹配帧边界 present | EndFrame trace + video | `FAIL` |
| S6-AC5 | 回归 AVC420 | AVC444 修改不破坏已通过的 420 | regression result | `FAIL` |

**Stop：** 任一 retained 前提未知时，不允许 suppress 原生 GDI，也不允许用 AVC420 结果替代 AVC444 验收。

---

## S7｜A/B 与工程交付证据闭环

**As a** Reviewer，**I want** 独立复查正确性、性能、稳定、交互和回退，**so that** “代码完成”不会被误报成“工程交付”。

### Ralph 边界

- Source Scope：测试、脚本、日志、性能 CSV、录屏、验收报告；生产代码冻结。
- Allowed：运行、采集、整理、判定、生成缺陷 Story。
- Forbidden：为了让验收变绿而顺手修改生产代码。
- RED / Probe：现有画面截图为 READY，但硬解、回退、A/B 和长稳仍有 PENDING。

### 伪代码

```text
acceptRelease(commit, scene):
  before = loadEvidence(S0, sameScene=true)
  after = collect(path, decoder, video, CPU, FPS,
                  queue, lifecycle, fallback, input, soak)

  for each acceptancePoint:
    if oracle missing: mark PENDING or UNKNOWN
    else if oracle failed: mark FAIL
    else mark PASS

  if productionCodeChangedDuringAcceptance:
    createDefectStory(); return NEEDS_REPLAN
  if any required point != PASS:
    return NOT_YET
  return RELEASE_READY
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S7-AC1 | BUILD/PATH | 目标 HAP 与真机路径可证明 | build + decoder log | `FAIL/UNKNOWN` |
| S7-AC2 | 画面与交互 | 目标片段可见，输入与窗口交互不回退 | video + interaction matrix | `FAIL` |
| S7-AC3 | 性能 | 同设备同片段 CPU/FPS 优于 before，口径一致 | A/B CSV + video | `PENDING` |
| S7-AC4 | 稳定 | 目标 soak 时长、resize/后台/重连通过 | soak + lifecycle matrix | `PENDING/FAIL` |
| S7-AC5 | fallback | 故障注入后旧路径可见且原因明确 | log + fallback video | `FAIL` |
| S7-AC6 | Scope | 验收阶段生产代码未变化 | commit diff | 生成新 Story |
| S7-AC7 | 可复查 | 每项 AC 可回到 commit/runId/原始文件 | evidence index | `NOT_YET` |

**Stop：** 任一证据不能关联同一 commit/runId 时标记 `UNKNOWN`；发现实现缺陷时生成新 Story，不在验收脚本中修代码。

---

## 6. Story 评审门

一张 Story 只有同时满足以下条件，才能进入 Ralph：

| 检查项 | 通过标准 |
|---|---|
| Outcome | 只有一个可观察结果 |
| Source Scope | 可以列出允许修改的文件或符号 |
| Forbidden | 明确写出本卡不能顺便做什么 |
| Pseudocode | 写清输入、分支、成功提交和失败不提交 |
| Acceptance | 每个 AC 都有场景、预期、证据和失败结论 |
| Stop | 触发后能停止扩大修改范围 |
| Handoff | 新 Session 只读 Worker Packet 就能开始 |
| Exit Gate | 能明确判断是否允许进入下一 Story |

**不能过门的例子：** “完成 GPU 优化”同时包含 decoder、buffer、显示、队列、生命周期和性能；任何失败都无法定位，也无法决定 Ralph 下一轮应修改什么。

## 7. 课堂怎么展示

第 35 页先展示 Story 地图和一张真实 Worker Packet，说明“不是按文件拆，而是按可验收结果拆”。第 36 页截取一个 Story 的伪代码和 AC 表，强调验收点必须在开发前写好。源码级 `frameId / owner / EndFrame` 仅在 Story 的伪代码、证据字段和讲师备注中出现，不回到主线标题。
