# GPU 适配实施任务包与验收点

> 目标：把 `17-复杂三方项目GPU适配源码分析与探针方案.md` 转换成可以逐轮交给 Ralph 执行的 Story。每轮只关闭一个主要未知，禁止跨轮顺手扩展。

## 0. 固定输入与全局约束

- 源码仓库：`C:/Users/mu/Desktop/code/demo`
- 分析基线：`aa31146`
- 主场景：HarmonyOS FreeRDP Client 控制 Windows，服务端发送 RDPGFX AVC420 surface command。
- 保留路径：FreeRDP original GDI callback 必须始终可恢复。
- 本轮不承诺：最终 CPU/FPS 改善、AVC444 完整交付、所有设备兼容性。

全局禁止项：

1. 不修改 `rdpgfx_main.c` 与 `rdpgfx_codec.c` 的协议解析语义。
2. 不删除 original GDI 路径。
3. 没有有效 target/native output/EGL import 时不设置 `consumed=true`。
4. 不在 `SurfaceCommand` 内直接把最终画面 swap 到窗口。
5. 不用“看起来流畅”替代同 run 的路径、画面和性能证据。
6. 不把不同构建、设备、场景的证据拼在一起。

## 1. G0｜冻结一帧调用链和日志主键

**Goal**：建立贯穿协议、decoder、buffer、compositor、EndFrame 和 target 的一帧 trace；本轮不改变显示行为。

**Read First**：

- `rdpgfx_main.c:1190-1247,1322-1472`
- `rdpgfx_codec.c:185-214`
- `libfreerdp/gdi/gfx.c:316-351,1246-1301`
- `client/X11/xf_gfx.c:39-150,507-530`

**Allowed**：OHOS bridge/app pipeline 的诊断字段、测试和文档。

```cpp
TraceKey key { runId, sessionId, generation, surfaceId, frameId, pts };
onStartFrame(frameId):     log(key, "start")
onSurfaceCommand(command): log(key, codec, rects, streamBytes)
onDecoderOutput(pts):      log(key, decoderName, nativeFormat)
onEndFrame(frameId):       log(key, pendingFrameId, owner)
onPresent(frameId):        log(key, swapResult, fallbackReason)
```

**AC**：

- AC-G0-1：任取一帧，可从 StartFrame 追到 SurfaceCommand 与 EndFrame。
- AC-G0-2：日志明确 owner 是 GDI 还是 GPU。
- AC-G0-3：同一记录包含 build、设备、场景组成的 runId。
- AC-G0-4：字段缺失则 `FAIL`，不进入 G1。

**Evidence**：trace schema、一条完整 frameId 日志、执行命令与退出码。

**Stop**：连续两轮仍不能关联 SurfaceCommand 与 EndFrame，`REPLAN`，先修观测。

## 2. G1｜Linux 参考实现探针

**Goal**：证明平台显示责任位于回调表，而不是 PDU 解析器。

**Allowed**：只读源码；只更新 codebase map、时序图和 ADR 输入。

```text
WireToSurface1 → rdpgfx_decode → context->SurfaceCommand
EndFrame → context->EndFrame → context->UpdateSurfaces
→ xf_OutputUpdate → XPutImage / XSync
```

**AC**：

- AC-G1-1：每条箭头都有 `path::symbol:line`。
- AC-G1-2：能区分协议层、通用层和平台层。
- AC-G1-3：明确 SurfaceCommand 与 EndFrame 的不同职责。
- AC-G1-4：产出 CONTINUE/REPLAN/STOP Verdict。

**Stop**：只有类名清单、没有输入输出和失败语义，探针不通过。

## 3. G2｜XComponent Surface 生命周期

**Goal**：让 GPU pipeline 获得带 generation 的可信显示目标；本轮不创建 decoder。

**Read First**：

- `surface/xcomponent_native_host.cpp:22-76`
- `napi/native_bridge_context.cpp:284-323,419-438`

**Allowed**：XComponent host、native bridge、SurfaceBridge/target 测试。

```cpp
onCreated(window, size):
    target = { window, size, generation++, ready=true }
    publishTarget(target)

onChanged(window, size):
    target = { window, size, generation++, ready=true }
    rejectOlderGeneration(); publishTarget(target)

onDestroyed():
    target = { generation++, ready=false }
    publishTarget(target); transitionOwner(GPU, GDI)
```

**AC**：created/changed/destroyed 均有日志；changed 后拒绝旧 generation；destroy 后 target 为空且 owner 回 GDI；重复 Attach/Detach 不泄漏节点或 callback。

**Fault**：零尺寸 target、changed 紧接 destroy、恢复时注入旧 generation frame。

## 4. G3｜可逆 RDPGFX Bridge

**Goal**：插入 OHOS candidate，并确保未消费 command 继续走 original GDI。

**Read First**：

- `ohos_rdpgfx_bridge.c:532-666`
- `ohos_rdpgfx_surface.c:112-138`
- `rdpgfx_pipeline.cpp:319-366`

```cpp
attach:
    save(original StartFrame, EndFrame, SurfaceCommand)
    install(OHOS callbacks)

surfaceCommand(command):
    candidate = tryGpu(command)
    if candidate.consumed: return candidate.status
    return original.SurfaceCommand(command)

detach: restore(original callbacks)
```

**AC**：disabled 时全部走 original；not-ready/not-consumed 时 original 调用一次；consumed 时 original 不调用；detach 后三个 callback 与 attach 前一致；目标 HAP/so 能确认当前 bridge 符号。

**Stop**：original 不可达立即回退，不进入硬解。

## 5. G4｜单帧硬解与 NativeBuffer

**Goal**：用最小 AVC420 输入获得硬件 decoder output 和非空 `OH_NativeBuffer`；本轮不做最终窗口 present。

**Read First**：`avc420_gpu_compositor_internal.cpp:545-712,780-803`。

```cpp
capability = GetCapability(video/avc, HARDWARE)
decoder = CreateByName(capability.name)
configure(SURFACE_FORMAT or RGBA); prepare(); start()

input = QueryInputBuffer(timeout)
write(sample, pts); PushInputBuffer(input)
output = QueryOutputBuffer(deadline)
assert output.pts == expectedPts
native = GetNativeBuffer(output)
read format/width/height/stride
```

**AC**：记录实际 decoder name 与 HARDWARE category；Configure/Prepare/Start 成功；input/output PTS 对应；NativeBuffer 与尺寸/stride 合法；任何失败保持 `consumed=false`。

**Fault**：缺 SPS/PPS、首帧非关键帧、output 超时、NativeBuffer 为空。

## 6. G5｜NativeBuffer 导入与离屏合成

**Goal**：把硬解输出导入 OES texture，只把 dirty rect 合成到 retained FBO；本轮不接管连续 RDP 输出。

**Read First**：`avc420_gpu_compositor_internal.cpp:889-1217,1569-1632`。

```cpp
windowBuffer = CreateNativeWindowBufferFromNativeBuffer(nativeBuffer)
eglImage = eglCreateImageKHR(EGL_NATIVE_BUFFER_OHOS, windowBuffer)
bind(GL_TEXTURE_EXTERNAL_OES, eglImage)
ensureRetainedFbo(surfaceSize)
for rect in regionRects:
    composite(texture, sourceRect(rect), destinationRect(rect), retainedFbo)
```

**AC**：NativeWindowBuffer/EGLImage 成功；目标为 external OES；未知 format/stride 明确拒绝；两个局部更新后其他区域仍保留；EGLImage/texture/FBO/buffer 可按相反顺序释放。

**Fault**：不支持 format、eglCreateImageKHR 失败、dirty rect 越界或为空。

## 7. G6｜EndFrame、Owner 与 Fallback

**Goal**：把离屏结果接回 FreeRDP 帧边界，证明每帧只有一个输出 owner。

**Read First**：

- `ohos_rdpgfx_bridge.c:243-347`
- `avc420_gpu_compositor_internal.cpp:2248-2393`

```cpp
onSurfaceCommand(command):
    if allGatesPass(command):
        composite(command); pendingFrameId = command.frameId
        pending = true; return consumed
    return notConsumed

onEndFrame(frame):
    if !pending: return notHandled
    if frame.frameId != pendingFrameId:
        clearPending(); recordMismatch(); return ownerIsGpu
    if !target.ready: handlePausedTarget(); return ownerIsGpu
    presentComposite(); clearPending(); return handled
```

**AC**：SurfaceCommand 后只有 pending；matched EndFrame 才 present；mismatch 不显示错误帧并记录 active/pending/endFrameId；takeover 前故障回 original；GPU active 后同帧不双写；destroy/detach/reset 后 owner 回 GDI。

## 8. G7｜连续播放与有界节奏

**Goal**：从“一帧可见”扩展到连续播放，用队列和 gap 指标定位卡顿。

必须记录：

- processed/decoded/queued/presented
- pending overwrite/endFrame mismatch/ignored/failure
- queue depth/oldest age
- command/endFrame/present gap
- decode/import/composite/present 耗时

**AC**：decoded/presented 持续前进；queue depth 与 oldest age 有阈值；没有持续增长的 overwrite/mismatch/failure；卡顿时能指出第一段异常 gap；没有同 run before A/B 时不宣称性能 PASS。

## 9. G8｜生命周期、故障与最终验收

必须覆盖：resize、background/foreground、reconnect、decoder failure、EGL import failure、EndFrame mismatch 和 soak。

| 维度 | Oracle | PASS 证据 |
|---|---|---|
| 产物身份 | commit/buildId/HAP hash | 当前安装包与源码一致 |
| 路径 | decoder/native/import/owner/present 日志 | 目标场景走 GPU 主路径 |
| 画面 | 同 run 截图/视频 | 无黑屏、绿屏、残影和双写 |
| 回退 | 故障注入 | original GDI 可达且原因可解释 |
| 性能 | before/after 同场景 | CPU、FPS、帧时满足预设阈值 |
| 生命周期 | resize/background/reconnect | owner/generation 正确恢复 |
| 长稳 | soak + 资源/队列指标 | 无持续增长和不可恢复冻结 |

**Exit Gate**：所有必需证据属于同一 runId；任一关键维度仍 UNKNOWN，最终 Verdict 只能是 `PARTIAL`；文档、代码、测试和 Evidence Ledger 的 commit/buildId 必须可追踪。

## 10. Ralph 单轮输出格式

```text
Story: Gx
Status: PASS | FAIL | UNKNOWN | BLOCKED
Read: 实际读取的 path::symbol
Changed: 实际修改的文件与原因
Commands: 命令 + exit code
Evidence: 日志/截图/视频/测试路径
Acceptance: AC-Gx-n 逐条结果
Unexpected: 与预期不同的第一条事实
Rollback: 是否执行、回到哪个 commit/开关
Next: CONTINUE | REPLAN | STOP + 下一张 Story
```

这份输出必须短于输入上下文。它让下一轮从已验证状态继续，而不是重新总结整个 30W+ 项目。
