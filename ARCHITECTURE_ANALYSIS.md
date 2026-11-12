# ArcCreate iPad/iOS 架构审计与构建可行性

审计基线：`GooPigeon/ArcCreate` 的 `trunk`，提交 `a00f94ca602b990bc15d7ca530434da9f875d152`。本报告基于源码和项目设置静态检查；Unity 编辑器、iPad 真机及 GitHub Actions 构建尚未验证。

## 结论

- 项目使用 Unity `2022.3.62f3`。仓库已有 iOS 导出和发布流程，说明 iOS 并非从零适配；新的手动工作流应独立于现有发布流程，先导出未签名 Xcode 工程并执行 `xcodebuild`。
- 现有 Practice Mode 已实现 A/B 端点、区间循环、时间轴拖动及实时变速。开发应在现有实现上补齐参数、边界、循环次数和成绩统计，不能重复造一套播放系统。
- 当前变速通过 `AudioSource.pitch` 实现，改变音高。保音调需要另行验证 iOS 可用的音频处理方案及延迟、音画同步，不能直接宣称已支持。
- iPad 横屏允许，触控输入路径存在。场景内 Canvas 使用多种参考分辨率，仍需在目标 iPad 的横屏比例、刘海/安全区域和多指操作下实测。

## 项目与平台

| 项目 | 已核实内容 | 依据 |
| --- | --- | --- |
| Unity | `2022.3.62f3` | `ProjectSettings/ProjectVersion.txt` |
| 场景 | Boot、Compose、Gameplay、Remote、Select、Storage、Result、Greeting 均列入构建 | `ProjectSettings/EditorBuildSettings.asset` |
| iOS 目标 | `targetDevice: 2`（iPhone 与 iPad）；允许左右横屏，关闭正反竖屏 | `ProjectSettings/ProjectSettings.asset` |
| iOS 构建模块 | 仓库配置不能证明 Runner 已装 iOS Build Support；必须由首次 CI 运行确认 | GitHub Actions 运行日志待验证 |
| iPad UI | Gameplay 场景内 Canvas 参考分辨率包括 `2048×1536`、`1280×720`、`800×600` | `Assets/Scenes/Gameplay.unity` |
| 输入 | 移动端使用 `InputMode.Touch`，`TouchInputHandler` 读取 Unity 触点 | `Assets/Scripts/Gameplay/GameplayManager.cs`、`Assets/Scripts/Gameplay/Judgement/Input/TouchInputHandler.cs` |
| 依赖 | Unity UGUI、TextMeshPro、UniTask 等；部分包来自 Git 仓库 | `Packages/manifest.json` |
| 测试 | 有 Unity Test Framework 和 `Assets/Tests` | `Packages/manifest.json`、`Assets/Tests` |

## Gameplay 数据流

`GameplayManager.Update` 先调用 `AudioService.UpdateTime`，取得 `ChartTiming`，再依次更新谱面判定、处理触控、更新成绩、scenecontrol、相机和渲染。`AudioService` 在移动端使用 `AudioSettings.dspTime` 与实时时钟计算音频时间，并把播放速度纳入时间增量；谱面时间由音频时间减去偏移量得到。谱面读取从 `ChartReader` 进入 `ChartService`，事件按 Timing Group 分发。

暂停使用 `AudioService.Pause`，跳转/续播使用 `AudioTiming`、`PlayWithDelay` 或 `ResumeWithDelay`。这些路径调用 `ChartService.ResetJudge`，后者在目标时间重置各 Timing Group、成绩、输入请求、scenecontrol、相机与击打音历史。Tap、Hold、Arc、Arctap 分别有 `ResetJudgeTo`；Hold/Arc 的中途恢复仍需逐类验证，尤其跨越 A 点的持续音符、残留粒子和循环边界的额外 Miss。当前事件目录未见独立 Flick 类型；实施前应确认用户所指 Flick 是否由现有事件表示。

## 现有练习功能与缺口

- `PracticeMenu` 已有 A/B 点、启停循环和时间轴范围显示；`CheckRepeat` 到 B 点后暂停并延时返回 A 点。当前最短区间固定为 1000 ms，未见手动毫秒输入、循环次数及分段成绩。
- `SpeedSlider` 当前范围 `0.01×–2×`，步进 `0.05×`；任务目标 `0.5×–1.5×` 需在练习模式内调整并验证 UI。
- `AudioService.OnPlaybackSpeedChange` 同时设置音频 `pitch` 和视频播放速度。保音调与当前实现冲突，需要独立音频方案与 iPad 性能测试。
- `ScoreService.ResetScoreTo` 在回跳时清空待结算事件和计数；分段统计需要在重置前捕获本轮结果，避免把历史成绩混入下一轮。

## iOS 云端构建方案

原仓库 `release_internal.yml` 和 `release_public.yml` 用 Ubuntu 的 `game-ci/unity-builder@v4` 导出 iOS 工程，再交给 macOS/Fastlane 签名发布；流程依赖多项旧有发布密钥。新手动工作流应使用 macOS Runner、Unity `2022.3.62f3` 和 iOS Build Support，在同一作业中导出 Xcode 工程，执行无签名 `xcodebuild`，上传工程和日志。GameCI 自身的 macOS 测试矩阵包含此 Unity 版本及 iOS 目标，但 ArcCreate 的实际构建尚需运行确认。

Unity 激活凭据必须作为 GitHub Secrets 提供。现有发布流程还调用 `decrypt_secret.sh` 解密仓库中的 `secret_files.zip.gpg`；新流程初版不使用旧发布密钥。若缺失加密资源导致 Unity 构建失败，需要根据日志辨认具体依赖并单独处理，不应直接复制上游发布密钥。生成 Xcode 工程及无签名编译无需 Apple 签名材料；安装到 iPad、IPA 与 TestFlight 留待后续阶段。

## 阶段验证

1. 手动 CI：核对 Unity/iOS 模块安装、导出目录、`xcodebuild` 结果及工程产物。
2. iPad 触控：覆盖横屏比例、安全区域、时间轴拖动、多指 Tap/Hold/Arc/Arctap，记录音画延迟。
3. A/B 循环：测试 A/B 点恰在音符开始、结束和 Hold/Arc 中途；核对不多判、不漏判、不新增 Miss，并保存每轮成绩。
4. 变速与保音调：分别测 0.5×、1×、1.5× 的声音、谱面时钟和判定窗口；保音调方案通过 iPad 实测后再启用。
5. 每个功能阶段单独提交，运行可用测试并检查 diff 与构建日志。不得修改原有判定算法或进行大范围重构。
