---
name: optimize-ios-ui-performance
description: Diagnose, review, and optimize UIKit, SwiftUI, and Core Animation UI smoothness in iOS apps. Use when Codex needs to investigate scrolling or animation hitches; review Objective-C or Swift page code for performance risks; analyze SwiftUI body updates, dependencies, identity, List or ForEach behavior, platform representables, or Instruments evidence; design a smoother page; or implement and verify targeted fixes across the app update and render pipeline. Do not use for general launch-time, networking, memory-only, or battery optimization unless it directly contributes to a UI hitch.
---

# iOS 页面流畅度优化

## 目标

沿 UIKit 与 SwiftUI 共用的 iOS Render Loop 定位页面卡顿，找到有证据支持的瓶颈，实施最小且安全的优化，并在相同条件下重新测量。

不要把性能优化变成固定技巧清单。始终遵循：

> 复现 → 测量 → 定位阶段 → 最小修改 → 验证

## 读取参考资料

先识别页面是 UIKit、SwiftUI，还是两者混合，再根据任务类型读取对应文件：

- 主动检查 UIKit 页面或评审 UIKit 代码时，完整读取 `references/optimization-patterns.md`。
- 主动检查 SwiftUI 页面，或涉及 `body`、状态依赖、视图身份、`List`、`ForEach`、`Layout`、`UIViewRepresentable` 时，完整读取 `references/swiftui-performance.md`；涉及图片、绘制、图层或共同渲染阶段时，再读取 `references/optimization-patterns.md`。
- 排查已经发生的卡顿，或用户提供了 FPS、CPU、GPU、日志、Trace、Instruments 截图时，完整读取 `references/render-loop.md` 和 `references/diagnostics.md`，并根据页面框架读取上述对应参考。
- UIKit 与 SwiftUI 混合页面同时读取 `references/swiftui-performance.md` 和 `references/optimization-patterns.md`。
- 确认卡顿阶段并准备给出方案或修改代码前，确保已经读取与页面框架对应的优化参考。
- API 可用性、系统版本或框架行为会影响结论时，查询 Apple 官方文档，不依赖记忆猜测。

## 确定任务类型

先判断用户需要什么：

- **检查**：评审现有代码，指出性能风险，不修改文件。
- **诊断**：确定卡顿原因和所属阶段，不自动实施修复。
- **优化**：在定位问题后修改代码，并进行适当验证。
- **设计**：在开发新页面时，提前设计数据准备、布局、绘制和图片加载方案。
- **解释**：解释某个性能现象、指标或 Instruments 结果。

用户只要求检查、诊断或解释时，不修改代码。

## 1. 明确场景

先确定：

- 哪个页面、控件或交互发生卡顿。
- 是首次进入、滚动、动画、数据刷新还是图片出现时发生。
- 使用的设备、系统版本和目标刷新率。
- 数据量、图片尺寸、消息频率等复现条件。
- 问题能否稳定复现。
- 用户提供了哪些性能指标或诊断材料。

缺少非关键条件时继续检查代码并明确假设，不要用笼统问题阻断工作。

## 2. 建立基线

记录优化前能够获得的证据：

- 卡顿复现路径。
- FPS、CPU、内存和 GPU 的变化。
- Animation Hitches 中的 Commits、Renders、GPU 和 Frame Lifetimes。
- Time Profiler 中的主线程调用栈。
- SwiftUI Instrument 中的 Update Groups、Long View Body Updates、Long Platform View Updates、Other Long Updates 和 Cause & Effect Graph；只记录当前 Xcode 与系统版本实际提供的通道。
- 相关方法的调用频率和单次耗时。
- 图片像素尺寸、显示尺寸和内存峰值。
- 页面 View、Layer 和约束结构。

FPS、CPU、内存和 GPU 指标只作为线索，不能单独证明卡顿原因。

无法实际运行或测量时，明确区分：

- 已观察到的事实。
- 根据代码作出的推断。
- 仍需通过工具确认的假设。

## 3. 映射到 Render Loop

UIKit 与 SwiftUI 最终共用下面的渲染链路。SwiftUI 在进入共同链路前，还包含状态变化、依赖失效以及 `body`、布局或平台视图更新：

```text
Event / data change
        ↓
App update
  ├─ UIKit: event / data / UI work
  └─ SwiftUI: dependency invalidation → body / platform view update
        ↓
Layout → Display → Prepare → Commit
  ↓
Render Prepare
  ↓
Render Execute
  ↓
Display
```

先区分：

- `Commit Hitch`：App 没有按时完成事件处理或提交图层树。
- `Render Hitch`：App 已经提交，但 Render Server 或 GPU 没有按时完成渲染。

不要把 `body` 执行直接等同于重新绘制，也不要把问题简单归类为“CPU 卡”或“GPU 卡”。Render Server 的 Render Prepare 同样会使用 CPU。

不是每一帧都会执行所有昂贵工作。只有布局、内容或资源确实失效时，对应阶段才需要重新计算。

## 4. 检查实际代码路径

从触发点追踪到最终 UI 更新。UIKit 重点检查：

1. 找到事件、网络回调、定时器或滚动回调。
2. 找到数据解析、模型转换和展示数据构建。
3. 找到主线程数据源更新和 UI 提交。
4. 找到布局、约束、文本测量和强制布局调用。
5. 找到绘制、图片准备和图层内容更新。
6. 找到 View、Layer、遮罩、阴影和透明效果。
7. 找到调用频率、缓存失效条件和复用逻辑。

SwiftUI 重点检查：

1. 找到状态变化的来源，以及在哪个执行上下文发布 UI 状态。
2. 找到视图实际读取的状态、环境值和可观察对象属性。
3. 找到 `body`、计算属性和 View 初始化过程中的同步工作。
4. 找到 `List`、`ForEach` 的身份来源、排序过滤和每项生成的行数。
5. 找到布局、几何信息、Preference 和状态写回之间的反馈循环。
6. 找到 `UIViewRepresentable` 或 `UIViewControllerRepresentable` 的创建与更新频率。
7. 用 Cause & Effect Graph 或调用次数确认更新原因，不根据代码外观猜测。

静态代码中的高风险 API 只能作为调查入口，不能直接作为性能结论。

## 5. 选择最小优化

按照以下优先级处理：

1. 删除没有必要执行的工作。
2. 复用已经计算好的结果。
3. 缩小布局、绘制和提交范围。
4. 合并过于频繁的 UI 更新。
5. 把纯计算和 I/O 移出主线程。
6. 提前准备即将使用的资源。
7. 简化图层结构和图层效果。
8. 只有在确认成为瓶颈后，才引入异步绘制、后台释放或复杂缓存。

每个建议都要说明：

- 它针对哪个阶段。
- 现有证据是什么。
- 为什么能够缩短关键路径。
- 有哪些正确性、内存或维护成本。
- 如何验证是否有效。

不要承诺未经测量的具体提升比例。

## 6. 实施修改

用户明确要求优化时：

- 保留现有架构和无关修改。
- 使用项目当前语言；Objective-C 项目使用 Objective-C 示例，Swift 项目使用 Swift 示例。
- SwiftUI 示例和实现使用 Swift，并遵循项目已有的数据流与并发模型。
- 只修改已经定位到的热点。
- 纯数据和线程安全的计算可以放到后台。
- UIKit 对象、数据源变更和最终 UI 提交仍然放在主线程。
- SwiftUI 可观察的 UI 状态在正确的 actor 上发布，通常由 `MainActor` 维护；不要把 UI 状态更新机械迁移到后台。
- 不在 `body` 中执行同步 I/O、重复排序过滤、图片解码或其他已确认昂贵的工作；把结果提前准备、缓存到模型，或使用异步任务生成。
- 为 `List` 和 `ForEach` 使用稳定且符合业务语义的身份，不在每次更新时临时生成新 ID。
- 异步任务捕获不可变快照，并处理取消、版本和复用对象身份校验。
- 高频数据接收与 UI 提交分开设计，不因降低 UI 刷新频率而默认丢弃业务数据。
- 修改列表数据源和刷新方式时，确保数据源状态与 Table View 或 Collection View 的更新完全一致。
- 对版本相关 API 标明最低系统版本和回退路径。

代码只保留关键步骤，不为展示完整性堆积无关实现。

## 7. 验证结果

在与基线相同的条件下重新验证：

- 使用相同设备、系统版本、数据量和操作路径。
- 比较 Commit、Render、GPU 和帧生命周期。
- 检查主线程耗时是否下降。
- 检查 FPS 波动和卡顿次数是否改善。
- 检查内存峰值、CPU、GPU、功耗和输入延迟是否恶化。
- 检查 Cell 复用、异步回调、列表更新和图片显示是否正确。
- SwiftUI 页面检查更新次数与长更新耗时是否下降，并确认状态归属、视图身份、滚动位置和动画语义没有改变。
- 检查优化是否引入闪烁、错位、旧数据上屏或崩溃。

无法运行验证时，说明已完成的静态检查和仍需执行的测量步骤，不声称问题已经解决。

## 禁止机械套用

- 不看到耗时操作就直接放到后台。
- 不把所有 `reloadData` 都替换为 `insertRows` 或批量更新。
- 不把一次 `body` 调用直接当成一次重绘，也不以 `body` 调用次数单独判定性能好坏。
- 不机械添加 `.equatable()`，不把 `AnyView` 一概判定为慢。
- 不假设 `LazyVStack`、`LazyHStack` 或拆分子 View 在所有场景下都更快。
- 不认为 `frame` 一定比 Auto Layout 快。
- 不认为圆角、阴影或 `masksToBounds` 一定触发离屏渲染。
- 不认为把 View 换成相同数量的 Layer 就是压平图层树。
- 不把含透明像素的内容错误声明为不透明。
- 不在后台访问或销毁线程约束不明确的 UIKit 资源。
- 不在没有证据时使用异步绘制、栅格化或大规模缓存。
- 不为了减少层级而破坏交互、动画、无障碍和可维护性。
- 不使用单次 FPS 下降作为优化完成或失败的唯一依据。

## 输出格式

检查或诊断时按以下顺序输出：

1. **结论**：最可能的阶段和问题。
2. **证据**：代码、Trace 或指标如何支持结论。
3. **建议**：按收益、风险和实施成本排序。
4. **边界**：哪些结论仍然是推断。
5. **验证**：修改前后应该比较什么。

修改代码后说明：

- 修改了哪些文件。
- 每处修改对应 Render Loop 的哪个阶段。
- 完成了哪些验证。
- 还有哪些需要在真机或 Instruments 中确认。

## 自检

交付前确认：

- 是否先定位阶段，再提出优化。
- 是否识别 UIKit、SwiftUI 或混合页面，并读取对应参考。
- 是否区分 App CPU、Render Server CPU、GPU 和显示系统。
- 是否把性能线索误写成确定原因。
- 是否说明缓存和异步任务的失效条件。
- 是否保证 UI 和数据源更新的线程正确性。
- 是否考虑列表复用和过期异步结果。
- SwiftUI 页面是否检查了依赖范围、更新原因、视图身份和平台视图桥接。
- 是否提供了可执行的验证方法。
- 是否避免为了优化而增加更大的复杂度。
