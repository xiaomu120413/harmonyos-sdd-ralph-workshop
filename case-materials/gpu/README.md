# GPU 远控案例工程工作包

本目录只保存能约束开发、运行或验收的工程文档。重复稿和非工程控制材料已移除；唯一任务真源是 `13-GPU-Ralph-Story拆分与伪代码验收.md`。

## 1. 执行顺序

| 顺序 | 必须回答的问题 | 文档 | 进入下一步的门 |
|---:|---|---|---|
| 0 | 哪些是事实，哪些仍缺证据 | `00-证据状态总表.md` | 所有关键断言都有 `REPO/RUN/SESSION/UNKNOWN` 状态 |
| 1 | 问题如何稳定复现 | `01-问题与基线.md` | 场景身份与 before 采集契约冻结 |
| 2 | 在大仓库中只读哪些入口 | `02-大代码库认知地图.md` | 能定位一条 command 到 present 的调用链 |
| 3 | 真实调用和所有权边界是什么 | `09-源码调用链与任务拆解.md` | hook、consumed、fallback、owner、EndFrame 均有源码锚点 |
| 4 | 哪些跨平台契约可复用 | `03-跨平台实现调研.md` | 通用契约、平台差异和未知项分离 |
| 5 | 最终选择哪条方案 | `04-ADR-GPU-001-HarmonyOS硬解方案.md` | Decision、Alternatives、Deferred、Evidence Gate 冻结 |
| 6 | 最大未知能否用最短链证伪 | `05-最小能力穿刺计划.md` | SP-01～05 各自有 `PASS/FAIL/UNKNOWN` 与原始证据 |
| 7 | 如何按可观察结果实施 | `13-GPU-Ralph-Story拆分与伪代码验收.md` | 每个 Story 的 Read/Scope/RED/伪代码/Verify/AC/Stop 完整 |
| 8 | 失败后从哪里重新收敛 | `07-开发排障复盘.md` | first-abnormal、被证伪假设、最小修正与重放结果入账 |
| 9 | 什么状态才允许交付 | `06-工程验收计划.md` | 必选 AC 可追溯且独立 Reviewer 给出 verdict |

辅助文档：

- `11-AI文档生成与审阅链.md`：定义文档先行、审阅和变更顺序。
- `15-用户补充素材清单.md`：定义缺失运行证据的采集格式，不代表已经通过。
- `16-历史Session可复用证据.md`：保存历史运行事实和反例，只能证明对应历史快照。
- `17-复杂三方项目GPU适配源码分析与探针方案.md`：补充 FreeRDP/xrdp 背景、Linux 一帧探针与 HarmonyOS 最小探针。
- `18-GPU适配实施任务包与验收点.md`：提供 G0～G8 的另一种实施视图；若与 `13` 冲突，以 `13` 为唯一任务真源并回写差异。

## 2. 源码基线

文档中的源码锚点以本地 `demo` 仓库快照 `aa31146` 为基线，命令默认从该仓库根目录执行：

```text
harmony/third_party/FreeRDP
harmony/app/common/src/main/cpp
```

开始任务前必须先执行：

```powershell
git rev-parse --short HEAD
git status --short
rg -n "gdi_SurfaceCommand|freerdp_ohos_rdpgfx_bridge_attach|ohos_rdpgfx_surface_command" harmony/third_party/FreeRDP
rg -n "InstallRdpgfxDiagnosticsHooks|OnSurfaceCommand|ProcessCommand|PresentEndFrame" harmony/app/common/src/main/cpp
```

如果 commit、目录或符号不一致，先更新 `02`、`09` 和对应 Story 的源码锚点，不允许直接沿用旧行号实施。

## 3. 事实状态

- `REPO FACT`：当前源码能直接定位的结构和保护条件。
- `RUN FACT`：当前 commit、设备和场景产生的运行证据。
- `MEDIA FACT`：媒体中可观察到的画面；不能单独证明 decoder、性能或回退路径。
- `CASE FACT`：已经冻结的问题背景，但仍需当前运行证据量化。
- `SESSION FACT`：历史 Session 中发生过的判断、失败或纠偏；不能替代当前版本验收。
- `TARGET`：期望达到但尚未证明的状态。
- `PENDING/UNKNOWN`：证据缺失或无法判定；不得改写成成功。

## 4. 不可跳过的约束

1. 先修改 ADR、Story、AC 或运行账，再修改代码；提交时文档和实现必须同 commit 或互相引用。
2. 每轮只领取一个 Story，只处理一个 first-abnormal；禁止跨 Story 顺手重构。
3. build success 只证明可构建，画面可见只证明媒体结果，API success 只证明一次调用；三者都不能单独证明工程交付。
4. 接管前失败必须保留 original GDI；同一帧只能有一个 render owner；没有匹配 EndFrame 不得 present。
5. before/after 必须共享设备、网络、分辨率、片段和采样口径，并以同一 evidence index 关联。

## 5. 当前状态

源码调用链、方案、Story 和历史 Session 证据已整理；当前版本的 hardware decoder identity、同帧 trace、故障注入、严格 A/B 和长稳仍为 `PENDING/UNKNOWN`。准确状态只从 `00-证据状态总表.md` 读取。
