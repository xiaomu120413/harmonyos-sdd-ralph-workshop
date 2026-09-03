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

## 5. 共用执行基线

所有命令从 `demo` 仓库根目录执行。当前文档基线是 commit `aa31146`；行号变化时以文件路径和符号为准。

### 5.1 开工前检查

```powershell
git rev-parse --short HEAD
git status --short
rg -n "gdi_SurfaceCommand|freerdp_ohos_rdpgfx_bridge_attach|ohos_rdpgfx_surface_command|ohos_rdpgfx_record_avc420_gpu_candidate" harmony/third_party/FreeRDP
rg -n "InstallRdpgfxDiagnosticsHooks|OnSurfaceCommand|ProcessCommand|PresentEndFrame|OH_AVBuffer_GetNativeBuffer" harmony/app/common/src/main/cpp
```

若工作区已有修改，不得覆盖或回滚不属于当前 Story 的内容。若基线 commit 或符号不一致，先更新 `02`、`09` 和本文件，再提交 docs checkpoint。

### 5.2 构建、安装与启动

修改 FreeRDP/OHOS adapter 后先交叉构建并同步 runtime：

```powershell
wsl.exe bash -lc "cd /mnt/c/Users/mu/Desktop/code/demo && export OHOS_NDK_HOME=/opt/ohos/sdk-6.1.0.830/command-line-tools/sdk/default/openharmony/native && ./harmony/scripts/wsl/build-freerdp-ohos.sh"
powershell -NoProfile -ExecutionPolicy Bypass -File .\harmony\scripts\windows\sync-freerdp-runtime.ps1
```

构建并安装 2in1 产物：

```powershell
.\harmony\app\build_hap.bat 2in1
$GpuDevice = "<hdc-serial>"
hdc -t $GpuDevice install -r .\harmony\app\common\build\default\outputs\default\common-default-signed.hsp
hdc -t $GpuDevice install -r .\harmony\app\entry\build\default\outputs\default\entry-default-signed.hap
hdc -t $GpuDevice shell aa start -a EntryAbility -b com.muhub.desktop
```

`<hdc-serial>` 必须来自 `hdc list targets`，不得把个人设备序列号提交到仓库。tablet 使用 `build_hap.bat tablet`、`entry_tablet-default-signed.hap` 和 `TabletEntryAbility`。

### 5.3 运行证据目录

```powershell
$GpuRunId = "gpu-<yyyyMMdd-HHmmss>"
New-Item -ItemType Directory -Path ".\evidence\gpu\runs\$GpuRunId" -Force
git rev-parse HEAD | Tee-Object -FilePath ".\evidence\gpu\runs\$GpuRunId\commit.txt"
Get-FileHash -Algorithm SHA256 ".\harmony\app\entry\build\default\outputs\default\entry-default-signed.hap" | Format-List | Out-File -Encoding utf8 ".\evidence\gpu\runs\$GpuRunId\package-hash.txt"
hdc -t $GpuDevice hilog -r
```

执行目标场景后保存原始日志：

```powershell
hdc -t $GpuDevice hilog -x | Tee-Object -FilePath ".\evidence\gpu\runs\$GpuRunId\hilog.txt"
```

日志、媒体和指标必须使用同一 `$GpuRunId`；任何一项来自别次运行时标记 `UNBOUND`。

---

## 6. Story 明细

## S0｜保存 CPU 路径与卡顿基线

**As a** Reviewer，**I want** 在不修改生产路径的前提下保存 before，**so that** 后续硬解结果有可比较的事实基线。

**Read first：** `00-证据状态总表.md`、`01-问题与基线.md`、`15-用户补充素材清单.md`，以及源码仓库 `docs/freerdp-ohos-feature-matrix.md` 中 `CPU-RECORD-001`。

**Output：** `evidence/gpu/runs/<run_id>/U-GPU-01-before/`，至少包含 identity、视频、path log、metrics CSV 和 verdict。

### 输入与结果契约

| 字段 | 输入约束 | 输出要求 |
|---|---|---|
| build variant | `RDP_BRIDGE_CPU_ONLY=ON`；不得和正常 GPU 包混用 | CMake 配置行、HAP SHA256、安装时间 |
| scene | device/server/network/resolution/clip/duration 全部冻结 | `scene.json`；任一变化生成新 runId |
| path | 强制 `gdi`，关闭 RDPGFX/H.264、OH_AVCodec 和 AVC compositor | 原始 hilog 能证明 CPU-only 开关和旧路径，且无 GPU/GLES 初始化 |
| metrics | 固定采样周期与 warm-up 区间 | 原始时间序列，不只保留平均值 |
| media | 录制与 metrics 使用同一时间范围 | 视频起止时间写入 identity |
| verdict | 只判断“基线是否可复查” | 不判断 GPU 是否改善 |

### 代码落点

| 文件/符号 | 当前事实 | 本 Story 允许动作 |
|---|---|---|
| `harmony/app/common/build-profile.json5` | 当前参数可设置 `-DRDP_BRIDGE_CPU_ONLY=ON` | 记录构建变体；不得永久改变商用默认值 |
| `harmony/app/common/src/main/cpp/CMakeLists.txt` | `RDP_BRIDGE_CPU_ONLY` 生成编译定义 | 只补可观测性或构建断言 |
| `napi/napi_exports.cpp` | CPU-only 分支限制连接图形能力 | 只核对实际分支，不改 GPU 代码 |
| `surface/surface_bridge.cpp` | CPU-only 使用 NativeWindow CPU painter，绕过 EGL/GLES | 只补一次性模式日志和结果状态 |
| 指标/录屏脚本 | 当前 evidence 仍不完整 | 增加 runId、固定采样和归档，不改生产路径 |

### 实施任务

1. 回读 `CPU-RECORD-001`，记录开关影响的协议能力、decoder 和 renderer，禁止只改最终绘制端。
2. 创建 `identity.txt` 与 `scene.json`，在启动应用前写入 commit、HAP hash、设备和场景。
3. 构建 CPU-only 包，运行 Native/ArkTS 测试与 2in1 HAP 构建；把退出码和产物写入 build log。
4. 清空 hilog、安装、启动、连接固定服务端，播放同一片段 30 秒并同步采集视频和指标。
5. 反向检查日志：CPU-only 模式存在；AVC420/444 compositor、OH_AVCodec 和 GLES renderer 初始化不存在。
6. 输出原始文件索引和 `PASS/PENDING/UNKNOWN`；不对性能优劣下结论。

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

**Verify：** 使用源码中的 CPU-only 编译开关生成独立 before 包；运行固定片段 30 秒后，从原始日志检查不得出现 AVC420/AVC444 GPU compositor 或 GLES renderer 初始化，并把编译开关值、HAP hash、路径日志和性能 CSV 写入同一 evidence index。CPU-only 开关字段若与 `docs/freerdp-ohos-feature-matrix.md` 不一致，停止并先更新 S0，不猜测字段名。

### Done

1. `scene.json`、identity、path log、metrics、视频和 build log 同 runId。
2. 能通过正向模式日志与反向 forbidden scan 同时证明 CPU-only 路径。
3. 原始时间序列可用于 S7 同口径 A/B。
4. 生产 GPU/decoder/compositor diff 为零；否则 S0 判 Scope FAIL。

---

## S1｜真机选择 HarmonyOS 硬件 Decoder

**As a** 远控视频管线，**I want** 在目标设备选择硬件 H.264 decoder，**so that** 后续输出确实来自新路径，同时不破坏旧路径回退。

**Read first：** `04-ADR-GPU-001-HarmonyOS硬解方案.md`、`05-最小能力穿刺计划.md`，以及 `harmony/app/common/src/main/cpp/channels/rdpgfx_pipeline.cpp::InstallRdpgfxDiagnosticsHooks`、`harmony/third_party/FreeRDP/client/OHOS/ohos_rdpgfx_bridge.c::freerdp_ohos_rdpgfx_bridge_attach`、`harmony/app/common/src/main/cpp/surface/avc420_gpu_compositor_internal.cpp` 的 decoder init 区域。

**Output：** 当前 run 的 build log、symbol scan、decoder identity、各初始化阶段返回码、fallback reason 和 S1 verdict。

### 初始化状态机

| 当前状态 | 事件 | 下一状态 | 对外行为 | 证据 |
|---|---|---|---|---|
| `UNATTACHED` | bridge attach 成功且 original callbacks 已保存 | `BRIDGE_READY` | 允许评估 GPU candidate | attach 与 hook 地址/状态 |
| `BRIDGE_READY` | capability 为空或无 hardware H.264 | `UNSUPPORTED` | `outputActive=false`，保留 original GDI | capability/category/reason |
| `BRIDGE_READY` | 找到 hardware capability | `CREATED` | 按 name 创建 decoder；不能静默退回 MIME 软件实现并标 PASS | capability name + create result |
| `CREATED` | `IsValid` 失败 | `FAILED` | Close，保留 original GDI | rc + valid |
| `CREATED` | Surface/RGBA 两种 direct-sampleable format 均配置失败 | `FAILED` | Close，不允许 mapped-plane 冒充主链 | 每种 pixelFormat + rc |
| `CONFIGURED` | Prepare 成功 | `PREPARED` | 不接管 command | rc |
| `PREPARED` | Start 成功 | `READY` | 只表示 decoder 可接受输入，尚未表示 command 已消费 | decoder name、尺寸、format、deadline |
| 任意初始化态 | 失败 | `FAILED/UNSUPPORTED` | `consumed=false`，original 路径仍可达 | stage + reason + fallback |

### 代码落点与接口契约

| 文件/符号 | 必须保证的行为 | 禁止 |
|---|---|---|
| `rdpgfx_pipeline.cpp::InstallRdpgfxDiagnosticsHooks` | 注册 AVC420 callback、EndFrame callback、output policy 和 surface target；配置失败时复位 | 在 UI 层创建 decoder |
| `ohos_rdpgfx_bridge.c::freerdp_ohos_rdpgfx_bridge_attach` | attach 前保存原 `StartFrame/SurfaceCommand/EndFrame`，再安装 wrapper | 覆盖 original 后不保存 |
| `Avc420HardwareDecoder::Ensure` | `GetCapabilityByCategory(HARDWARE)` → Create → IsValid → Configure → Prepare → Start | 只凭 CMake 开关判断硬解已启用 |
| `ConfigureWithPixelFormat` | 记录 requested format 与返回码；只接受直接可采样输出 | 不记录实际 format；接受未知格式继续 |
| `Avc420HardwareDecoder::Close` | Stop（started 时）→ Destroy → 清空全部状态 | 失败后保留半初始化 decoder |

### 实施任务

1. 对比 attach 前后 callback，确认 original hooks 在所有失败路径可达。
2. 为 capability、name、category、create、valid、configure、prepare、start 建立一次性结构化日志；禁止逐帧刷日志。
3. 明确 `CreateByName` 与 `CreateByMime` 的判定：只有 capability 能证明 hardware category 时才允许 S1 PASS。
4. 对 Surface Format 和 RGBA 的两次 configure 分别记录 rc；两者都失败立即 Close。
5. 对 capability missing、create、valid、configure、prepare、start 各保留至少一条失败 oracle。
6. 构建并安装后，正常设备路径和 unsupported/failure 路径各执行一次，确认 original GDI 未被删除。

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

**Verify：** 先执行 5.2 的 FreeRDP 构建、runtime 同步、2in1 构建、安装和启动；再执行 `rg -n "AVC420 GPU decoder (create failed|invalid|prepare failed|start failed|ready)" harmony/app/common/src/main/cpp/surface/avc420_gpu_compositor_internal.cpp` 固化期望日志。真机日志必须给出 `ready` 与 decoder identity；不支持或任一步失败时必须出现 reason，并观察 original 路径仍可用。

### Done

1. build/runtime/HAP identity 能证明目标代码进入设备。
2. 正常路径能列出完整初始化状态转移，decoder category 有独立 hardware 证据。
3. 每个失败阶段都 Close 且保持 `outputActive=false`；至少一个故障路径证明 original GDI 可见。
4. S1 只交付 `READY`，不宣称已有有效 output、画面或性能提升。

---

## S2｜一个短片段产生合法解码结果

**As a** compositor，**I want** 把一份压缩输入关联到一份可解释的解码输出，**so that** codec 问题和显示问题可以分开验收。

**Read first：** S1 verdict、`avc420_gpu_compositor_internal.cpp` 中 `MakeDecoderPts`、input queue、output query、`OH_AVBuffer_GetNativeBuffer` 和 output description 相关代码。

**Output：** `input-output-frame.log`、`output-description.json`、native buffer identity 和 S2 verdict；不得只保存过滤后的单行日志。

### 解码结果契约

| 结果 | 触发条件 | 状态提交 | 资源处理 | Story verdict |
|---|---|---|---|---|
| `Decoded` | output PTS 与本次 PTS 相等，native buffer 和 config 合法 | 只返回 `NativeDecodedFrame`，不 claim render owner | frame 持有 output index，消费完成后统一 Release | 候选 PASS |
| `NoOutput` | 有界次数/总 deadline 内只有 TRY_AGAIN_LATER | 不提交 decoded/pending 状态 | 不持有 output buffer；保留 timeout 统计 | `FAIL` 或 warm-up 条件下的受控重试 |
| `Failed` | input buffer、attr、push、query、description 或 native buffer 失败 | 不提交任何显示状态 | 已取得的 output 必须 Free；必要时 decoder Close/Reset | `FAIL` |
| stale output | output PTS 与 expected PTS 不同 | 丢弃该 output，继续在剩余 budget 内查询 | 立即 FreeOutputBuffer | 不是当前帧成功证据 |
| unsupported format | 两种请求格式均不可直接采样，或实际 config 不可导入 | 不进入 mapped-plane 主链 | 保存完整 description 并关闭 decoder | `NEEDS_REPLAN` |

### 输入输出字段

```text
InputKey  = frameId + streamSequence + generatedPts
Input     = bytes + capacity + attr(offset/size/flags/pts) + pushRc
Output    = outputIndex + outputPts + logicalSize + nativeSize + nativeStride + nativeFormat
Lifetime  = acquiredAt + releasedAt + releaseReason
Result    = Decoded | NoOutput | Failed | Stale | Unsupported
```

同一 `frameId` 可能产生多个内部 sequence；不得只用 frameId 推断 input/output 一一对应。实际关联以 `MakeDecoderPts(frameId, sequence)` 生成的 PTS 为主。

### 代码落点与实施任务

| 文件/符号 | 本 Story 动作 | 验证重点 |
|---|---|---|
| `MakeDecoderPts` | 冻结 frameId/sequence 到 PTS 的关联规则 | 重复 frameId 时仍可区分输出 |
| `PrepareH264Packet` | 保存 SPS/PPS bootstrap 和 packet verdict | warm-up 与真实失败不混淆 |
| `Avc420HardwareDecoder::Decode` | 对 input/query/output 每个返回分支记录原因 | 所有 wait 有 deadline，无无限阻塞 |
| `UpdateOutputDescription` | 保存 stream-changed/start 后实际 pixel format | requested 与 actual 不混写 |
| `AttachNativeOutput` | 读取 `OH_NativeBuffer_Config` | width/height/stride/format 非零且可解释 |
| `NativeDecodedFrame::Release` | 统一释放 output index | 成功、失败、stale 均无泄漏/双释放 |

实施顺序：

1. 先为一个已知 AVC420 packet 建立 input key，不接显示层。
2. 输入 buffer 容量不足时 PushEmptyInput/释放并返回 Failed，保存 capacity 与 size。
3. 设置 attr 并 push；任何 rc 非 OK 立即停止本次 sample。
4. 按最大次数和总 deadline 查询 output；遇到 stream changed 先更新 description。
5. stale PTS 立即释放并继续；匹配 PTS 后才构造 `NativeDecodedFrame`。
6. 读取 native buffer config，保存 logical/native 尺寸、stride 和 format。
7. 在 success、timeout、stale、invalid format 四个分支核对 output index 的 acquire/release 次数。

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

**Verify：** 执行 `rg -n "MakeDecoderPts|QueryInputBuffer|PushInputBuffer|QueryOutputBuffer|GetOutputDescription|OH_AVBuffer_GetNativeBuffer" harmony/app/common/src/main/cpp/surface/avc420_gpu_compositor_internal.cpp`；运行已知短片段并从完整 hilog 提取同一 frameId 的 input PTS、output PTS、format、stride/planes 和 native buffer。任一字段无法关联时 S2 保持 `UNKNOWN/FAIL`，不得进入显示改动。

### Done

1. 至少一条 input→matching output→native config→release 的完整记录。
2. timeout、stale PTS、invalid/missing native buffer 三类失败均有确定结果和资源释放证据。
3. 等待次数与总 deadline 有上限，线程不会因 decoder 无输出永久阻塞。
4. diff 不包含 owner、retained composite、EndFrame、AVC444 或 queue policy 修改。

---

## S3｜在正确边界显示，并能失败回退

**As a** 用户，**I want** 硬解画面在正确时机显示且失败时仍可使用旧方案，**so that** 新路径不会带来黑屏、撕裂或不可恢复。

**Read first：** S2 verdict、`09-源码调用链与任务拆解.md` 第 2～4 节、`ohos_rdpgfx_avc444_policy.c::ohos_rdpgfx_record_avc420_gpu_candidate`、`Avc420GpuCompositor::OnSurfaceCommand`、`ProcessCommand`、`PresentQueuedUpdate`、`PresentEndFrame`。

**Output：** 同一 frameId 的 ordered trace、可见结果、接管前故障注入记录、original GDI 恢复记录和 S3 verdict。

### 接管决策表

| 阶段 | 条件 | wrapper 返回 | GDI 行为 | Owner/Present 行为 |
|---|---|---|---|---|
| 非 AVC420 / compositor disabled | 不属于本路径或未开启 | `consumed=false` | 调 original SurfaceCommand | owner 维持 GDI |
| 协议校验失败 | surface/command/dirty rect 非法 | `consumed=true`，返回 validation error status | **不再调用 original**，保持 FreeRDP 原生校验顺序 | 不 claim owner，不 present |
| 接管前 target 不可用 | callback 无可用 window/size | `callbackReady=false` | 调 original | owner 维持 GDI |
| 接管前 partial update 无可信 background | full surface=false 且 GDI seed 缺失/过期/尺寸不符 | `callbackReady=false` | 调 original | 不 claim owner |
| 接管前 decode/composite 失败 | `outputActive=false` 且 ProcessCommand false | `callbackReady=false` | 调 original | 清理临时资源，不建 pending |
| 首次完整处理成功 | ProcessCommand true，随后 ClaimOutput | `callbackReady=true/consumed=true` | suppress 当前 command | stop old renderer，owner→AVC420，等待 EndFrame |
| 接管后 command 入队成功 | `outputActive=true` | consumed | suppress GDI | worker 负责顺序处理 |
| 接管后 callback miss | output 已 active，但当前 command 失败 | active-miss policy 返回 consumed | **禁止逐 command 重新进入 GDI**，避免 H.264 状态落后后重入 | preserve owner，记录 suppressed failure；由显式 detach/reset 决定恢复 |
| partial background 失去可信度 | active 且 background 不满足 | compositor 显式 detach 后 callback false | original GDI 恢复 | destroy GPU resources，owner→GDI，清队列 |
| EndFrame 不匹配 | `matched=false` 或 id != pending | 不 present | 不触发 GDI 双写 | 丢弃/保留 pending 按代码策略并记录 mismatch |

这个表区分两类容易混淆的“失败”：协议输入非法要返回原生错误；接管前执行失败可回 original；接管后单帧失败不能直接把已落后的 GDI H.264 context 拉回来。

### 代码落点

| 文件/符号 | 责任 | 本 Story 验证 |
|---|---|---|
| `ohos_rdpgfx_surface.c::ohos_rdpgfx_surface_command` | 先尝试 AVC420/444 candidate；未 consumed 才调保存的 original | consumed/status/original 三者顺序 |
| `ohos_rdpgfx_validate_avc420_gpu_surface_update` | 复用 FreeRDP surface/rect 校验顺序 | invalid command 返回 status，不进入 GPU callback |
| `ohos_rdpgfx_record_avc420_gpu_candidate` | 组装 info、调用 callback、切换 outputActive、应用 active-miss policy | 接管前/后分支不混淆 |
| `Avc420GpuCompositor::SeedBackgroundBeforeTakeover` | full surface 或可信 GDI snapshot 建立 retained 初态 | partial update 无 seed 时不接管 |
| `ClaimOutputAfterTakeover` | stop/release 旧 renderer，owner→AVC420 | 只有 ProcessCommand 成功后调用 |
| `ProcessCommand` | decode、native import、retained composite、pending | 失败不留下半提交状态 |
| `PresentQueuedUpdate/PresentEndFrame` | matched frame 时唯一 present | mismatch 不 swap |

### 实施任务

1. 为 wrapper 画出并核对 `validation error / pre-takeover fallback / active miss / explicit detach` 四条返回路径。
2. 把 callbackReady、outputActive(before/after)、consumedStatus、owner、pendingFrameId 写入同一 ordered trace。
3. 对 partial update 强制检查 background seed 的完整性、尺寸和最大年龄；失败时保持 GDI。
4. ProcessCommand 成功后再 claim owner；claim 时停止旧 renderer、释放旧 target，再设置 output policy active。
5. EndFrame 只消费匹配的 pending frame；对 mismatch 保存 active/pending/end 三个 id。
6. 设计三个故障注入：invalid rect（期待 error status）、接管前 decoder/output 失败（期待 original）、接管后 miss（期待 preserve owner，不重入 GDI）。
7. 清除注入后重放正常路径，确认 owner 和回退状态没有残留。

### Ralph 边界

- Source Scope：AVC420 candidate policy、`OnSurfaceCommand`、`ProcessCommand`、`PresentQueuedUpdate`、`PresentEndFrame`。
- Allowed：bridge、最小合成接线、显示 Owner、present 与 fallback diagnostics。
- Forbidden：性能优化、AVC444、输入系统、生命周期大重构。
- RED / Probe：现有截图只能证明画面可见，不能证明硬解、正确 present 或回退。

### 伪代码

```text
surfaceCommandWrapper(command):
  result = tryAvc420Candidate(command)
  if result.consumed:
    return result.status
  return originalSurfaceCommand(command)

tryAvc420Candidate(command):
  if not AVC420 or compositor disabled:
    return { consumed: false }

  validationStatus = validateLikeNativeGdi(command)
  if validationStatus != OK:
    return { consumed: true, status: validationStatus }

  outputWasActive = outputActive
  callbackReady = compositor.onSurfaceCommand(command, outputWasActive)

  if callbackReady:
    if not outputWasActive:
      setOutputActive(true)
    return { consumed: true, status: OK }

  if outputWasActive:
    recordActiveMissAndPreserveOwner()
    return { consumed: true, status: OK }

  return { consumed: false }

compositor.onSurfaceCommand(command, outputActive):
  if targetUnavailable:
    if outputActive: pauseTargetButPreserveState()
    else: return false

  if partialUpdate and backgroundNotTrusted:
    if outputActive: detachAndReleaseToGdi()
    return false

  decoded = decodeAndValidate(command)
  if not decoded: return false
  if not compositeRetained(decoded, command.rects): return false
  markPending(command.frameId)
  if not outputActive: claimUniqueOwner()
  return true

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
| S3-AC6 | 非法 dirty rect | 返回 validation error，不调用 GPU callback 和 original | status + callback/original counters | `FAIL` |
| S3-AC7 | 接管后 callback miss | 保持 GPU owner 并 suppress GDI re-entry，随后由显式策略恢复 | active-miss + owner + release trace | `FAIL` |
| S3-AC8 | partial update 无可信背景 | 接管前留在 GDI；已 active 时先 detach/清队列/owner→GDI | seed age/size + detach trace | `FAIL` |

**Stop：** GDI/GPU 双写、提前 present 或回退不可达时立即停止；不得用“屏幕亮了”替代完整穿刺证据。

**Verify：** 执行 `rg -n "pendingFrameId|PresentQueuedUpdate|PresentEndFrame|fallback before takeover|consumed" harmony/app/common/src/main/cpp/surface/avc420_gpu_compositor_internal.cpp harmony/third_party/FreeRDP/client/OHOS`；正常运行时证明 `decode→owner→pending→matched EndFrame→present` 有序且同帧。然后分别运行三个注入用例：① invalid rect 必须得到 validation error，GPU callback 与 original counter 都不增加；② 接管前 decoder/output 失败必须得到 `consumed=false` 且 original counter 增加；③ 接管后 callback miss 必须保持 GPU owner、original counter 不增加，直到显式 detach/reset 才恢复 GDI。测试结束清除全部注入并重放正常路径，确认 owner、pending 与 queue 无残留。

### Done

1. 正常、validation error、接管前 fallback、接管后 active miss、explicit detach 五条路径都有独立 trace。
2. 首次 owner claim 发生在 retained composite 和 pending 成功之后；GDI 与 GPU 不会写同一 frame。
3. matched EndFrame 才 present，mismatch 能从日志直接指出三个 frame id。
4. fault injection 清理后正常路径可重放，owner、queue、decoder 无残留状态。

---

## S4｜连续播放时队列与延迟保持有界

**As a** 用户，**I want** 视频连续播放不会越播越慢，**so that** 硬解接入不会把一帧成功变成持续积压。

**Read first：** S3 verdict、`Avc420GpuCompositor::EnqueueSurfaceCommand`、worker loop、compaction/backpressure、`QueueStats` 或等价诊断字段。

**Output：** 固定时长的 queue timeline、command age、drop/compaction reason、frame order、CPU/FPS 和 S4 verdict。

### 队列对象与所有权

| 对象 | 入队时所有权 | 允许丢弃 | 顺序约束 |
|---|---|---|---|
| `Prewarm` | 只保存目标尺寸 | 队列压缩时可丢弃 | 不得阻塞真实 command |
| `SurfaceCommand` | 必须深拷贝 stream bytes、rects 和 info；不得保存 FreeRDP 回调期裸指针 | 当前策略不因压缩随意丢 command；超过硬上限时明确拒绝/失败 | command 顺序保持 |
| `EndFrame` | 拷贝 frameId、activeFrameId、matchedFrame 和 callbacks 快照 | 压缩时可丢旧 EndFrame，但保留最新 matched EndFrame；没有 matched 时保留最新 EndFrame | 被保留 EndFrame 不能越过其相关 command |
| callback/target snapshot | 入队时复制 callable；执行时仍须检查当前 target/output state | stale/unavailable 时不执行 present | 不允许旧 target 被异步任务重新写入 |

### 背压状态与判定

```text
queuedCommands - processedCommands = commandBacklog
queuedEndFrames - processedEndFrames = presentBacklog
queueDepth <= kMaxWorkerTasks

Healthy:
  depth/age 围绕稳定区间波动，处理计数持续追上入队计数
Pressure:
  到达 coalesce depth，压缩 prewarm/旧 EndFrame，并记录原因和保留帧
OverLimit:
  到达硬上限，拒绝继续无界增长并增加 queueOverLimit
Broken:
  command 顺序破坏、matched EndFrame 丢失、age 持续单调上升或 worker 无法停止
```

平均 FPS 正常不能抵消 `maxCommandGap/maxEndFrameGap/maxPresentGap` 的长尾异常；必须同时检查时间序列和 first-abnormal。

### 代码落点与实施任务

| 文件/符号 | 当前责任 | 本 Story 要核对/补齐 |
|---|---|---|
| `OwnedAvc420Command` | 持有 info、bytes、rects | 深拷贝后所有内部指针指向 owned storage |
| `EnqueueSurfaceCommand` | 拷贝并入队 command | allocation error、worker stopped、compact、over-limit 均有 verdict |
| `EnqueueEndFrame` | 入队 frame boundary | 保留 latest matched frame 的规则可由测试覆盖 |
| `CompactWorkerBacklogLocked` | 丢 prewarm/旧 EndFrame，保留 commands 和一个 EndFrame | 输出 depth before/after、drop counts、preserved frame |
| `WorkerLoop/ProcessWorkerTask` | FIFO 取任务并调用 Process/Present | stop 时可退出；异常时 fail-open/clear 的证据明确 |
| `WorkerBacklogText/AppendPeriodicStats` | 输出 depth/backlog/processed/drop/limit | 增加 age 与 gap 时保持低频、可关联 runId |

实施顺序：

1. 写纯策略测试构造 `Prewarm/Cmd1/End1/Cmd2/End2`，确认压缩只丢允许对象且保留正确 EndFrame。
2. 写 owned-copy 测试：入队后修改/释放原 bytes 与 rects，worker 仍读取原始内容。
3. 写边界测试覆盖 `kEndFrameCoalesceDepth-1 / == / +1` 和 `kMaxWorkerTasks`。
4. 增加 enqueueAt/processAt，生成 depth、oldestAge、commandAge、presentGap 的低频时间序列。
5. 连续场景运行至少覆盖稳定段、压力段和恢复段，保留原始统计而不是只留平均值。
6. 若发现积压，先按 command arrival → queue → decode → EndFrame → present 找第一处增长，不先加线程。

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
| S4-AC6 | 入队后原 buffer 释放 | worker 读取 owned copy，内容和 rect 不变 | deterministic test + checksum | `FAIL` |
| S4-AC7 | 达到压缩阈值 | 只丢 prewarm/旧 EndFrame，保留全部 command 和最新 matched EndFrame | queue before/after trace | `FAIL` |
| S4-AC8 | 达到硬上限 | 不创建无界线程，不继续增长；明确 over-limit 与恢复 | limit counter + depth timeline | `FAIL` |
| S4-AC9 | worker stop/reset | 唤醒、清队列、join 完成，depth 归零 | stop test + final counters | `FAIL` |

**Stop：** backlog 持续增长或顺序被破坏时，回到第一处异常；不能先增加线程数。

**Verify：** 执行 `rg -n "queueDepth|commandBacklog|presentBacklog|queueOverLimit|compaction|drop" harmony/app/common/src/main/cpp/surface/avc420_gpu_compositor.cpp` 固化指标来源；按 S0 的同一场景运行目标时长，绘制 queue depth/age 时间序列。验收阈值未在项目文档冻结时只能输出实测分布与 `TARGET TBD`，不得自行判 PASS。

### Done

1. owned-copy、压缩、硬上限、停止/清理四类确定性测试通过。
2. 连续运行能同时给出 depth、age、drop、order、gap、CPU/FPS 原始时间序列。
3. 队列策略不会丢 SurfaceCommand 或丢失最后一个匹配 EndFrame，也不会创建无界线程。
4. 阈值未冻结时只交付分布和 `TARGET TBD`；不得用主观流畅度判 PASS。

---

## S5｜生命周期变化后拒绝旧目标并恢复

**As a** 用户，**I want** resize、切后台、Surface 重建和重连后继续看到正确画面，**so that** 已排队任务不会向失效 NativeWindow 写入，暂停与彻底释放也不会混成同一种处理。

**Read first：** S4 verdict、`bridge_types.h::DecoderSurfaceTarget`、`surface_bridge.cpp::DecoderSurface`、`rdpgfx_pipeline.cpp::ResetAvcSurfaceOutput/RestoreRdpgfxDiagnosticsHooks`、`avc420_gpu_compositor.h::Avc420OutputState/WorkerTask`、`OnSurfaceTargetChanged/PauseOutputForTargetUnavailable/DetachOutputActive`。

**Output：** target identity 设计与 diff、状态转移测试，以及 resize、后台恢复、Surface destroy/recreate、重连四个子场景的 target/owner/stale trace、视频和 S5 verdict。

### 当前缺口与目标模型

`REPO FACT`：当前 `DecoderSurfaceTarget` 只有 `window/width/height`，`WorkerTask` 保存 callbacks 快照和 outputActive，但没有显式 target generation。`Avc420OutputState` 已有 `Detached/Active/TargetPaused/Failed`，并区分 pause 与 detach。

因此本 Story 不是假设“generation 已存在”，而是补齐可验证的 target identity：

```text
DecoderSurfaceTarget = window + width + height + generation
WorkerTask            = payload + targetGenerationAtEnqueue

generation increments when:
  native window created/replaced/destroyed
  target dimensions become a new render target identity
  session/bridge reset invalidates queued work
```

是否把纯 resize 视为新 generation，必须由 SurfaceBridge 和远端 resize 设计共同冻结；不得在 compositor 内凭 reason 字符串自行发明第二套规则。

### 生命周期决策表

| 事件 | 当前状态 | 目标状态 | decoder/retained | queue | owner | 恢复条件 |
|---|---|---|---|---|---|---|
| target 暂时不可用 | Active | TargetPaused | 保留 | 保留，但执行前校验 generation | 保持 AVC420 | 同 generation target 恢复且 ready |
| target ready | TargetPaused | Active | 复用并重新 attach/re-present | 只执行当前 generation | 保持 AVC420 | 首次成功 present |
| surface destroyed | Active/Paused | Detached | Destroy | 清空并统计 dropped | →GDI | 新 target + 新 command 重新接管 |
| hook restore / bridge detached / output reset | 任意 | Detached | Destroy | 清空，worker 可停止 | →GDI | 新 session 重新 attach |
| fatal compositor failure | Active | Failed | Destroy | 清空 | →GDI | 显式 reset 后重新评估 |
| stale worker task | 任意 | 状态不变 | 不触碰 | 丢该任务并记 reason | 不改变 | 等当前 generation 的任务 |

### 代码落点与实施任务

| 文件/符号 | 计划修改 | 不变量 |
|---|---|---|
| `common/bridge_types.h::DecoderSurfaceTarget` | 增加不可回退的 target identity/generation 字段 | 默认空 target 的 generation 语义明确 |
| `surface/surface_bridge.cpp` | 在 window create/change/destroy 的唯一所有者处递增 generation，并由 `DecoderSurface()` 返回 | 同一事件只递增一次；线程安全 |
| `native_bridge_context.cpp::SnapshotDecoderSurfaceTarget` | 传递完整 snapshot | 不缓存旧 window |
| `avc420_gpu_compositor.h::WorkerTask` | 保存 enqueue 时 target generation | task 不保存无所有权的可变引用 |
| `EnqueueSurfaceCommand/EnqueueEndFrame/ProcessWorkerTask` | 入队记录、执行前比对；stale 立即 drop | stale task 不 decode/composite/present |
| `OnSurfaceTargetChanged` | 依据显式事件选择 pause 或 detach | 不只靠任意字符串模糊匹配；若仍用 reason，枚举来源必须冻结 |
| `DetachOutputActive/Reset/Restore...` | Destroy、clear queue、owner→GDI、统计一致 | 完全释放后不存在旧 task 写入 |

实施顺序：

1. 先新增 target identity 的纯数据契约与 SurfaceBridge 单测，证明 create/change/destroy 的递增序列。
2. 再让 `DecoderSurface()` 和 callbacks 传递 generation，不改变 compositor 行为。
3. 给 WorkerTask 保存 enqueue generation，写 stale/current 两组队列测试。
4. 在 ProcessWorkerTask 的第一条业务动作前比较当前 target；stale 时只计数和释放 owned payload。
5. 把 pause、resume、detach、fatal 的资源和 owner 行为写成状态测试。
6. 最后在真机分别执行四个生命周期子场景；每个场景使用独立日志窗口和证据索引。

### Ralph 边界

- Source Scope：decoder/surface generation、pause/detach/recreate、stale worker rejection、Owner release/reclaim。
- Allowed：generation、局部 release/recreate、恢复与 fallback。
- Forbidden：用重建整个业务 Session 掩盖局部状态错误。
- RED / Probe：恢复后能看到画面，但旧 worker 是否仍能写入未知。

### 伪代码

```text
surfaceBridge.onTargetEvent(event, newTarget):
  if event changes target identity:
    targetGeneration += 1
  target = snapshot(newTarget, targetGeneration)
  notifyCompositor(event, target)

compositor.onTargetEvent(event, target):
  if event in [surfaceDestroyed, bridgeDetached, outputReset]:
    detachOutputActive(clearQueue=true, owner=GDI)
    return
  if outputActive and not target.ready:
    state = TargetPaused
    preserve(decoder, retainedComposite, owner)
    return
  if state == TargetPaused and target.ready:
    reattach(target)
    state = Active

enqueue(task):
  task.targetGeneration = snapshotTarget().generation
  queue.push(owned(task))

processWorkerTask(task):
  current = snapshotTarget()
  if not current.ready or task.targetGeneration != current.generation:
    drop(task, "stale_generation")
    return
  processAgainst(current)
```

### 验收点

| AC | 场景 | 通过标准 | 必须证据 | 不通过时 |
|---|---|---|---|---|
| S5-AC1 | create/change/destroy target | generation 按冻结规则单调递增且事件只递增一次 | SurfaceBridge unit test | `FAIL` |
| S5-AC2 | 旧 generation task 已排队后替换 Surface | stale task 在 decode/composite/present 前被拒绝 | deterministic worker test + counter | `FAIL` |
| S5-AC3 | target 暂时不可用后恢复 | Active→TargetPaused→Active；decoder/retained/owner 按表保留 | state trace + video | `FAIL` |
| S5-AC4 | Surface destroy/recreate | Active/Paused→Detached，队列清空、owner→GDI；新 target 不被旧 task 写入 | owner/queue/generation trace | `FAIL` |
| S5-AC5 | resize | 使用冻结的 identity 规则；画面恢复且 viewport/target 与新尺寸一致 | resize trace + video | `FAIL` |
| S5-AC6 | bridge detach/reset/reconnect | old decoder/target/worker 状态不可达，新 session 重新 attach | reconnect trace | `FAIL` |
| S5-AC7 | fatal failure | Failed→显式 reset→Detached/可重新接管，fallback reason 明确 | fault trace + recovery | `FAIL` |
| S5-AC8 | 四类场景回归 | mouse/keyboard/scroll viewport 与实际画面一致 | interaction matrix | `FAIL/PENDING` |

**Stop：** 旧 target 仍能 present 即判 FAIL，即使肉眼看到恢复后的新画面。

**Verify：** 先执行 `rg -n "struct DecoderSurfaceTarget|Avc420OutputState|WorkerTask|TargetPaused|OnSurfaceTargetChanged|DetachOutputActive" harmony/app/common/src/main/cpp` 固化变更前事实；实现后运行 target generation 和 worker stale 的确定性测试，再执行四个真机子场景。每个场景逐个清空日志，证明旧 generation 在任何 decode/composite/present 前被拒绝，pause 与 detach 的 owner/资源行为符合决策表。

### Done

1. 文档明确记录“旧结构无 generation”以及新增字段/事件的评审决定。
2. SurfaceBridge 是 generation 唯一写入者，compositor 只消费 snapshot。
3. stale task、TargetPaused 恢复、surface destroy、reset/reconnect、fatal recovery 均有测试和 trace。
4. 四类真机场景逐项有 verdict；缺设备或凭据时保持 `PENDING`，不因单元测试通过而外推。

---

## S6｜AVC444 按 LC 与 retained state 闭合

**As a** AVC444 视频管线，**I want** 正确处理 LC、双 bitstream 与 retained state，**so that** AVC444 不会被当成两段普通 AVC420 视频错误合成。

**Read first：** S5 verdict、FreeRDP `gdi_SurfaceCommand_AVC444`、`ohos_rdpgfx_record_avc444_gpu_candidate`、`avc444_gpu_compositor.cpp` 与 `avc444_gpu_compositor_internal.cpp` 的 LC、retained、EndFrame 逻辑。

**Output：** LC/stream/single-decoder/retained readiness/consumed/EndFrame trace、AVC420 回归和 S6 verdict。

### LC 与 stream 决策表

| LC | stream1 语义 | stream2 语义 | 本 command 更新 | present 前 retained 前提 |
|---:|---|---|---|---|
| 0 | luma | chroma | 先 luma，后 chroma | 本轮结束后 luma/chroma 都 ready |
| 1 | luma | unused | 只更新 luma | 必须已有可用 chroma retained state |
| 2 | chroma | unused | 只更新 chroma | 必须已有可用 luma retained state |
| 其它 | invalid | invalid | 不进入 GPU callback 或返回明确 error | 不接管 |

单一 decoder 按 stream sequence 依次处理 luma/chroma；两条 bitstream 不是两个互不相关的视频，也不得创建两个无法共享 H.264 状态的 decoder。每次 decode 都要有独立 PTS，并在应用 luma/chroma 后释放对应 output。

### Retained 状态契约

```text
hasLuma / hasChroma = 已验证并成功应用到 retained surface 的状态
ready               = hasLuma && hasChroma
pendingFrameId      = 本 command 已完成所需更新并等待的帧边界

pre-takeover + not ready:
  callbackReady=false → original GDI 继续拥有输出

active + transient not ready/decode timeout:
  不允许逐 command 重入落后的 GDI H264 state
  记录 suppressed/ignored update，由 reset/detach 策略恢复

ready:
  更新 pendingFrameId；只在 matched EndFrame present retained surface
```

### 代码落点与实施任务

| 文件/符号 | 责任 | 本 Story 检查 |
|---|---|---|
| `libfreerdp/gdi/gfx.c::gdi_SurfaceCommand_AVC444` | 原生 AVC444/LC 与 H.264 状态基线 | GPU 语义不得比原生路径少校验 |
| `ohos_rdpgfx_validate_avc444_gpu_surface_update` | surface、LC、stream、rect 的原生顺序校验 | invalid status 不进入 App callback |
| `ohos_rdpgfx_record_avc444_gpu_candidate` | 组装 LC/stream info，处理 callbackReady/outputActive | 接管前 fallback 与 active miss 分离 |
| `Avc444GpuCompositor::OnSurfaceCommand` | target/background/queue/owner 状态 | 不复用 AVC420 单流假设 |
| `Avc444GpuCompositorImpl::ProcessCommand` | needsLuma/needsChroma、单 decoder、retained readiness | LC0/1/2 三路可追踪 |
| `ApplyLuma/ApplyChromaV1/ApplyChromaV2` | 更新离屏 retained planes | rect、format、shader 失败不提交 ready |
| `PresentEndFrame` | matched frame 显示 | mismatch 不 present |

实施顺序：

1. 从 FreeRDP 原生 `gdi_SurfaceCommand_AVC444` 提取 LC0/1/2 输入和更新不变量，写入测试 fixture。
2. 为 invalid LC、空 stream、越界 rect 写 validation 测试，确认 App callback 不执行。
3. 对 LC0 记录 `stream1→luma→release→stream2→chroma→release` 的单 decoder 顺序。
4. 对 LC1/LC2 分别构造 retained 已就绪和未就绪场景，核对 callbackReady/consumed/fallback。
5. 对 outputActive 后的 timeout/format/shader failure 核对 active miss/reset 策略，禁止直接重入 GDI。
6. 对每个成功 command 记录 frameId、LC、两路 PTS、hasLuma/hasChroma、pending 和 EndFrame。
7. 重跑 AVC420 S3/S4，确认共享 owner、surface target 与队列修改未回归。

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

  if command.lc == 0:
    decodeAndApply(stream1, LUMA,  nextPts())
    releaseOutput(stream1)
    decodeAndApply(stream2, CHROMA, nextPts())
    releaseOutput(stream2)
  else if command.lc == 1:
    decodeAndApply(stream1, LUMA, nextPts())
    releaseOutput(stream1)
  else if command.lc == 2:
    decodeAndApply(stream1, CHROMA, nextPts())
    releaseOutput(stream1)

  if retained prerequisites incomplete:
    record("avc444_not_ready")
    if not outputActive:
      return NOT_CONSUMED_FOR_ORIGINAL_GDI
    recordActiveMissAndPreserveOwner()
    return CONSUMED_WITHOUT_PRESENT

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
| S6-AC6 | invalid LC/stream/rect | 返回明确 validation status，不进入 compositor 和 original 双处理 | callback/original counters + status | `FAIL` |
| S6-AC7 | LC0 | 单 decoder 顺序处理 luma(stream1) 与 chroma(stream2)，两次 output 均释放 | ordered PTS/lifetime trace | `FAIL` |
| S6-AC8 | LC1 retained missing/ready | missing 时不首次接管；ready 时只更新 luma 并保留 chroma | readiness + consumed trace | `FAIL` |
| S6-AC9 | LC2 retained missing/ready | missing 时不首次接管；ready 时只更新 chroma 并保留 luma | readiness + consumed trace | `FAIL` |
| S6-AC10 | active 后单路失败 | 不逐 command 重入 GDI；由显式 reset/detach 产生可解释恢复 | active-miss + owner trace | `FAIL` |
| S6-AC11 | EndFrame mismatch | retained state 不被错误帧提前 present | pending/active/end ids | `FAIL` |

**Stop：** 任一 retained 前提未知时，不允许 suppress 原生 GDI，也不允许用 AVC420 结果替代 AVC444 验收。

**Verify：** 执行 `rg -n "LC|luma|chroma|retained|PresentEndFrame|record_avc444_gpu_candidate" harmony/third_party/FreeRDP/libfreerdp/gdi/gfx.c harmony/third_party/FreeRDP/client/OHOS harmony/app/common/src/main/cpp/surface/avc444_gpu_compositor.cpp harmony/app/common/src/main/cpp/surface/avc444_gpu_compositor_internal.cpp`；用 AVC444 场景保存两个 bitstream 的处理顺序、单 decoder identity、readiness 和 matched EndFrame，再完整重跑 AVC420 S3 门。

### Done

1. LC0/1/2、invalid LC、retained missing、active miss、EndFrame mismatch 均有独立用例。
2. 单 decoder、两路 PTS、output release、retained readiness 和 owner 可由同一 ordered trace 复查。
3. 接管前与接管后失败语义不混淆；不存在 GDI/GPU 双写。
4. AVC420 S3/S4 回归通过；缺真机 AVC444 服务端时保持 `PENDING`。

---

## S7｜A/B 与工程交付证据闭环

**As a** Reviewer，**I want** 独立复查正确性、性能、稳定、交互和回退，**so that** “代码完成”不会被误报成“工程交付”。

**Read first：** S0～S6 全部 verdict、`00-证据状态总表.md`、`06-工程验收计划.md`、`15-用户补充素材清单.md` 和所有原始 evidence index。

**Output：** `delivery/gpu/<run_id>/` 完整证据包、逐 AC verdict、未通过项和新缺陷 Story；验收阶段不得修改生产代码。

### Evidence binding 契约

```text
EvidenceIdentity =
  acceptedCommit + packageSha256 + runId + device + osBuild +
  server + networkProfile + resolution + clip + timeRange

Assertion -> AC -> Raw evidence -> Location -> Verdict -> Reviewer
```

`Location` 必须精确到日志行/时间范围、CSV 窗口、视频时间或 diff 文件；“见附件”“日志正常”“画面不错”均不构成 oracle。

### 进入验收的依赖门

| 上游 | 必须状态 | 缺失时处理 |
|---|---|---|
| S0 | before identity、path、metrics、video 可关联 | AC-PERF/现象基线为 `PENDING` |
| S1 | hardware decoder identity 与 fallback | AC-PATH 为 `UNKNOWN` |
| S2 | input/output/native buffer/lifetime trace | AC-420 不得开始 |
| S3 | owner/EndFrame/fallback/active-miss | AC-420、AC-FALLBACK 为 `NOT READY` |
| S4 | queue/age/gap/soak 数据 | AC-PERF/STABLE 为 `PENDING` |
| S5 | target identity、stale rejection、生命周期矩阵 | AC-STABLE/INPUT 为 `PENDING` |
| S6 | AVC444 LC0/1/2 与 retained trace | AC-444 为 `PENDING`；不得用 AVC420 代替 |
| threshold | 性能与 soak 目标已批准 | 只输出测量结果，不判 PASS |

### 验收执行分离

| 角色 | 允许动作 | 禁止动作 | 输出 |
|---|---|---|---|
| Implementer | 提供 accepted commit、Story verdict、原始 evidence index | 在验收阶段解释性修改生产代码 | handoff |
| Runner | 从冻结 commit 构建、安装、执行固定场景和采集 | 改验收口径、删失败样本 | raw bundle |
| Reviewer | 逐 AC 对照 oracle，复核 Scope 与身份 | 采信实现者口头结论、把 UNKNOWN 改 PASS | reviewer verdict |
| Planner | 对 FAIL/UNKNOWN 建新 Story 或 REPLAN | 在验收提交里混入修复 | next workpack |

同一人可以分时承担角色，但 accepted commit、运行原始文件和 Reviewer verdict 必须形成分离的提交或清晰 checkpoint。

### 实施任务

1. 冻结 `$GpuAcceptedCommit`，确认工作区无生产代码修改；记录 HAP 和 runtime hash。
2. 根据 `06` 生成空验收矩阵，逐行绑定上游 Story AC，不能先填总体结论。
3. 运行 BUILD/PATH/420/FALLBACK/444，逐个场景清日志并产生独立 run 子目录。
4. 使用 S0 相同 identity 运行 before/after；比较原始时间序列、p50/p95/max 和 first-abnormal，不只比平均 FPS。
5. 执行 resize、后台、Surface destroy/recreate、重连、soak 和 input matrix。
6. 对每条 AC 填 Expected/Actual/Raw evidence/Location/Verdict；无法定位即 UNKNOWN。
7. 运行 Scope diff，确认代码改动与 Story Source Scope、文档 checkpoint 和 accepted commit 一致。
8. 对每个 FAIL/UNKNOWN 生成独立 defect Story，写清最小复现、first-abnormal 和禁止扩散范围。

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
| S7-AC8 | before/after identity 不同 | 不生成性能提升结论，标 `UNBOUND/UNKNOWN` | identity diff | 误判 PASS 即验收失败 |
| S7-AC9 | 阈值未冻结 | 只报告分布，不用主观判断补阈值 | threshold source | `PENDING` |
| S7-AC10 | 验收中发现缺陷 | accepted commit 不变，生成新 Story | git status/diff + defect packet | 修改生产代码即 Scope FAIL |
| S7-AC11 | 原始日志与摘要冲突 | 以原始运行事实为准并更新状态表 | raw log location | `FAIL/REPLAN` |
| S7-AC12 | 敏感信息检查 | evidence 中无账号、密码、序列号、私网地址 | scan report | 不允许提交 evidence |

**Stop：** 任一证据不能关联同一 commit/runId 时标记 `UNKNOWN`；发现实现缺陷时生成新 Story，不在验收脚本中修代码。

**Verify：** 先设置 `$GpuAcceptedCommit = "<accepted-code-commit>"`，再执行 `git diff --name-only "$GpuAcceptedCommit..HEAD"` 确认生产代码冻结；随后按 `06` 的 BUILD/PATH/420/444/PERF/STABLE/INPUT/FALLBACK/SCOPE 逐项核对原始文件。before/after identity 不一致、阈值未冻结或缺 fault injection/soak 时，整体 verdict 必须为 `NOT READY`。

### Done

1. `06` 中每条 AC 都有明确 verdict；没有“基本通过”或“看起来正常”。
2. PASS 项能从断言跳转到原始 evidence 的精确位置，FAIL/UNKNOWN 项有新 Story 或明确 Stop。
3. accepted commit、package、run、device、scene 和阈值来源可独立核对。
4. 验收阶段生产代码 diff 为零，evidence 已脱敏。
5. 只有必选项全部 PASS 才输出 `RELEASE READY`；否则输出 `NOT READY/UNSUPPORTED`。

---

## 7. Story 评审门

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

## 8. 提交门

每个 Story 至少形成两个 checkpoint：先提交文档和 RED，再提交实现与验证证据。提交前执行：

```powershell
git diff --check
git status --short
git diff --name-only
```

Reviewer 必须确认：当前 Story 文档先于代码、Source Scope 没有越界、AC 未被放宽、原始证据路径存在、状态表和 progress 已回写。缺任一项时不得进入下一 Story。
