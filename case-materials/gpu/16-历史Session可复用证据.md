# GPU 案例｜历史 Session 可复用证据

> 用途：把本地真实开发 Session 中已经发生过的构建、安装、GPU 路径、帧率抖动、错误假设与回退整理成课堂证据。它们属于 `HISTORICAL SESSION FACT`，足以讲“怎么判断 AI 对不对、错了怎么办”，但不能替代当前版本同一次运行的最终验收。

## 1. 证据来源

| Session | 本地记录 | 课堂价值 |
|---|---|---|
| `019e867d-a315-7b50-ad13-5e2c9a05d8fb` | `C:/Users/mu/.codex/archived_sessions/rollout-2026-06-02T12-00-46-019e867d-a315-7b50-ad13-5e2c9a05d8fb.jsonl` | AVC420 方案、构建安装、GPU 运行统计、卡顿定位、错误修改与回退 |
| `019e8c2e-56e9-7102-936a-f0767ee11244` | `C:/Users/mu/.codex/archived_sessions/rollout-2026-06-03T14-31-52-019e8c2e-56e9-7102-936a-f0767ee11244.jsonl` | 黑屏背景、retained composite、GDI sideband 与第一异常分析的续篇 |

本文件只摘取可公开讲述的技术字段，不复制设备序列号、账号、密码、私网地址等信息。

## 2. 已有：方案不是凭空生成的

历史 Session 先对齐了 FreeRDP 原生 AVC420、OHOS bridge 和既有 AVC444 compositor，再给出候选方案：

```text
SurfaceCommand
  → OHOS bridge candidate
  → AVCodec buffer mode
  → OH_AVBuffer_GetNativeBuffer
  → EGLImage / GL_TEXTURE_EXTERNAL_OES
  → dirty-rect composite
  → matched EndFrame present
```

同时明确了三条否决条件：拿不到 native buffer 不接管；GPU cache 未 ready 不 suppress GDI；不能把 CPU mapped-plane fallback 当 GPU 主链。这部分可以直接支撑第 30～32 页的“源码定位 → 跨平台调研 → ADR”。

## 3. 已有：真实构建与安装记录

Session 中多次记录：

```text
hvigor assembleHap: PASS
hdc install -r: PASS
AVC420 独立 compositor 进入 libentry.so
```

其中一次明确说明临时包不含完整 HNP，只用于验证新的 `libentry.so`。因此课堂上可写“历史构建/安装成功”，不能写成“当前最终交付包已经验收”。

## 4. 已有：GPU 路径与同帧运行统计

2026-06-02 15:06～15:08 的历史运行日志包含：

```text
active=yes
source=native-buffer-oes
decoded=2397
presented=2396
droppedCommands=0
droppedEndFrames=0
mismatch=0
failures=0
importFallbacks=0
queueDepth=1
```

这组字段可以证明那次历史运行进入了 AVC420 GPU native-buffer 路径，并且 command、EndFrame 与 present 计数能对应。它不能证明当前仓库的 decoder identity，也不能与仓库里的 16 秒视频自动绑定成同一 run。

## 5. 已有：卡顿不是“平均 FPS 低”这么简单

同一 Session 先得到约 `24.9 fps` 的短窗口，但继续检查后发现帧节奏明显抖动：

| 时间窗口 | 估算 present FPS | 解释 |
|---|---:|---|
| 一段稳定窗口 | 约 24.9 | 平均值看起来正常 |
| 后续窗口 | 约 10 | 开始可感知卡顿 |
| 后续窗口 | 约 7.5 | 抖动加剧 |
| 极端窗口 | 约 1.3 | 明显停顿 |

同时 `droppedCommands=0`、`droppedEndFrames=0`、`mismatch=0`，decode/present 单次耗时也没有顶满。于是判断从“decoder 慢”转向“帧到达/调度 pacing 抖动”，并新增 `presentDelta/fps`、`maxCommandGapMs`、`maxEndFrameGapMs`、`maxPresentGapMs`。这正是第 37 页最有价值的真实案例：平均数不能替代第一异常。

## 6. 已有：AI 修错了，如何被证据推翻

黑屏问题中，AI 曾提出“GDI 背景合成后立即 present”的修复，并完成构建、安装。随后日志证明该路径确实触发，但画面仍黑：

```text
路径跑到了
present 也发生了
画面仍然黑
⇒ 假设被推翻：问题不是“没有刷新”，而是 retained composite 没有有效完整背景
```

Session 随后执行：

```text
Stop
→ Preserve 日志和截图
→ Locate 到 retained background seed
→ Falsify “缺一次 present”
→ 回退无效修改
→ 重新构建
→ 覆盖安装
→ 重启应用
```

这是完整的 Ralph 排障闭环，因此原清单中的 `U-GPU-06` 不再需要用户重新补造。

## 7. 已有：代码提交和 ownership 纠偏

历史 Session 记录了一笔基线提交：

```text
a78c43d Stabilize AVC420 GPU output ownership
```

后续继续针对 GDI sideband 的裸指针、dirty bbox owned copy、污染 background seed 等问题做验证。Session 当时保存过 `muhub_after_owned_sideband2.jpeg` 等截图，但当前本地扫描未找到原文件，因此只能引用 Session 文本与运行日志，不能把缺失截图放进 PPT 冒充现存资产。

## 8. 仓库当前已有媒体

| 资产 | 可证明 | 不可证明 |
|---|---|---|
| `gpu-validation-video-playback-16s.mp4` | 某次运行可见远端播放和交互 | 不证明与历史 GPU 日志同 run，也不证明性能提升 |
| `gpu-failure-black-screen-13s.mp4` | 某次运行出现黑屏 | 不单独证明 retained background 是根因 |
| `gpu-e2e-interaction-public.jpg` | 页面和交互关键帧可见 | 不证明连续性或路径 |
| 两份 contact sheet | 可在 PPT 静态展示视频内容 | 不替代原视频或日志 |

## 9. 课堂最小补充结论

现有材料已经足够讲完整方法论与真实排障，不要求讲师再采集六套资料。

| 优先级 | 是否需要讲师补 | 内容 |
|---|---|---|
| 课堂正文 | 不需要 | 方案文档、16 秒播放、13 秒黑屏、GPU 历史日志、FPS 抖动、错误修改回退 |
| 推荐但可选 | 1 项 | 原始 CPU/软件路径卡顿 before，20～30 秒；用于增强第 28 页开场冲击力 |
| 工程附录 | 按需 | 当前版本 same-run identity、故障注入、严格 A/B、lifecycle/soak；只有要宣布最终工程 PASS 时才补 |

因此 PPT 正文应写“已有历史证据 / 可选 before”，不再把 U-GPU-02～06 全部标成讲师待办。
