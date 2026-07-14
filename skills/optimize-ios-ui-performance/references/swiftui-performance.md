# SwiftUI 页面流畅度

## 目录

- [与 Render Loop 的关系](#与-render-loop-的关系)
- [诊断顺序](#诊断顺序)
- [减少单次更新的工作量](#减少单次更新的工作量)
- [减少不必要的更新](#减少不必要的更新)
- [保持稳定的视图身份](#保持稳定的视图身份)
- [优化 List 和 ForEach](#优化-list-和-foreach)
- [避免布局反馈循环](#避免布局反馈循环)
- [控制绘制和合成成本](#控制绘制和合成成本)
- [处理 UIKit 桥接](#处理-uikit-桥接)
- [并发和异步任务](#并发和异步任务)
- [不要机械套用](#不要机械套用)
- [验证清单](#验证清单)
- [直接来源](#直接来源)

## 与 Render Loop 的关系

SwiftUI 没有绕开 UIKit、Core Animation、Render Server 和 GPU。它在 App 侧增加了声明式更新过程：

```text
State / Environment / Observable change
                  ↓
        dependency invalidation
                  ↓
 body / view update / layout / platform view update
                  ↓
            layer-tree commit
                  ↓
       Render Server → GPU → Display
```

因此，共同的图片准备、文本排版、图层提交和 GPU 优化规则仍然适用；SwiftUI 还需要额外检查：

- 哪个状态变化触发了更新。
- 哪些 View 读取了该状态并形成依赖。
- 单次 `body` 或布局更新做了多少工作。
- View 身份是否稳定。
- `List`、`ForEach` 和平台视图桥接是否扩大了更新范围。

`body` 被调用只表示 SwiftUI 正在重新计算 View 描述，不代表一定发生绘制，也不代表最终每个像素都会变化。

## 诊断顺序

先稳定复现，再按下面的顺序取证：

1. 使用 Animation Hitches 确认是 App Commit 延迟，还是后续 Render 或 GPU 延迟。
2. 在支持 SwiftUI Instrument 的 Xcode 与系统组合上，查看 Update Groups 和 Hitches 时间线。
3. 检查 Long View Body Updates、Long Platform View Updates 和 Other Long Updates。
4. 使用 Cause & Effect Graph 找到“哪个状态变化导致哪些 View 更新”。
5. 使用 Time Profiler 查看长更新内部实际执行的函数。
6. 修改一个主要变量后，在相同数据量和操作路径下重新录制。

Xcode 26 引入了专用 SwiftUI Instrument。较旧工具链没有相同通道时，使用 Time Profiler、Animation Hitches、signpost 和有针对性的计数补足证据，不根据 `body` 代码长度猜测耗时。

## 减少单次更新的工作量

`body` 应快速返回 View 描述。不要在其中反复执行：

- 同步文件、数据库或网络读取。
- 大数组排序、过滤和分组。
- 日期、数字或富文本的重复格式化。
- 图片解码、缩略图生成和其他重型资源准备。
- 每次更新都不变的对象创建与 Bundle 查找。

把昂贵结果放到模型、缓存或异步任务中，`body` 只读取已经准备好的值。例如：

跨 actor 传递的输入和结果类型需要满足 `Sendable`。一种可控的边界是让独立 actor 完成纯数据转换，再由主 actor 发布界面状态：

```swift
actor MessageRowBuilder {
    func makeRows(from messages: [Message]) -> [MessageRowModel] {
        messages
            .filter(\.isVisible)
            .sorted { $0.timestamp < $1.timestamp }
            .map(MessageRowModel.init)
    }
}

@MainActor
@Observable
final class MessageListModel {
    private let builder = MessageRowBuilder()
    private(set) var visibleMessages: [MessageRowModel] = []

    func replaceMessages(_ messages: [Message]) async {
        let rows = await builder.makeRows(from: messages)
        guard !Task.isCancelled else { return }

        visibleMessages = rows
    }
}
```

示例只表达边界：纯数据转换在后台完成，供界面观察的状态由主 actor 提交。真实项目还需要处理取消、版本覆盖、`Sendable` 和任务生命周期。

## 减少不必要的更新

SwiftUI 根据 View 读取的数据建立依赖。依赖范围过大时，一个无关字段变化也可能让更多 View 重新计算。

检查：

- 子 View 是否只读取自己需要的属性。
- 是否把一个变化频繁、字段很多的模型作为所有子树的共同依赖。
- 是否把高频定时器、滚动位置或几何数据放入大范围 Environment。
- 是否在每次更新中写入值相同的状态。
- 是否存在“读取几何信息 → 写状态 → 再布局”的循环。

从 iOS 17、iPadOS 17、macOS 14、tvOS 17 和 watchOS 10 开始，SwiftUI 支持 Observation。使用 Observation 时，让叶子 View 直接读取它需要的属性，可以利用属性级依赖跟踪：

```swift
@Observable
final class ProfileModel {
    var nickname = ""
    var unreadCount = 0
}

struct NicknameView: View {
    let model: ProfileModel

    var body: some View {
        Text(model.nickname)
    }
}
```

目标系统更早时需要使用 `ObservableObject` 等现有数据流，并通过拆分职责或缩小观察对象范围控制更新。无论使用哪套机制，都不是要求把所有 View 拆得很碎。先用 Cause & Effect Graph 或更新计数确认依赖确实过宽，再选择能够隔离变化的自然组件边界。

## 保持稳定的视图身份

身份决定 SwiftUI 能否把更新前后的 View 视为同一个实体。身份不稳定会导致状态丢失、动画异常、滚动位置变化和不必要的创建销毁。

优先使用业务数据中稳定、唯一的 ID：

```swift
struct Message: Identifiable {
    let id: String
    let text: String
}

ForEach(messages) { message in
    MessageRow(message: message)
}
```

避免：

- 在 `body` 或计算属性中用 `UUID()` 临时生成 ID。
- 对会插入、删除或重排的数据使用数组下标作为身份。
- 在没有语义需要时频繁改变 `.id(...)`，迫使整棵子树重建。
- 让同一业务实体在不同刷新中使用不同 ID。

## 优化 List 和 ForEach

大型列表重点检查数据准备、身份和每个元素生成的行数：

- 在进入 `List` 前完成过滤、排序和分组，避免每次 `body` 更新都重复计算。
- 使用稳定的业务 ID，避免为了显示列表重新包装并生成 ID。
- 让 `ForEach` 的一个输入元素尽量对应稳定数量的行。
- 只有可见区域需要延迟创建时，考虑 `List` 或 Lazy 容器，并通过真实滚动场景验证。
- 图片加载、异步任务和播放器等资源需要与行身份绑定并支持取消。

Apple 对 `List` 和 `Table` 的性能说明指出，框架可能需要先收集标识符。在这一特定上下文中，行内条件分支或类型擦除可能迫使框架创建更多内容来确定身份。更稳妥的做法通常是提前过滤数据：

```swift
struct MessageList: View {
    let visibleMessages: [Message]

    var body: some View {
        List(visibleMessages) { message in
            MessageRow(message: message)
        }
    }
}
```

不要把这条结论扩大成“所有 `AnyView` 都慢”或“所有条件分支都必须移出 `body`”。必须结合容器、身份和 Instruments 证据判断。

## 避免布局反馈循环

重点检查：

- `GeometryReader` 或 `onGeometryChange` 是否持续写回状态。
- Preference 是否从子树向上频繁传播，再触发整个子树布局。
- 自定义 `Layout` 是否在测量阶段执行昂贵计算或产生不稳定结果。
- 尺寸变化是否由动画、滚动和状态写回相互放大。

只有值真正变化时才提交状态，并尽量把几何依赖限制在最小子树：

```swift
.onGeometryChange(for: CGFloat.self) { proxy in
    proxy.size.width.rounded(.toNearestOrAwayFromZero)
} action: { width in
    guard width != measuredWidth else { return }
    measuredWidth = width
}
```

先确认目标系统版本支持所用 API，并为较旧版本保留合适实现。去重只能阻断相同值的重复写入，不能替代对布局结构的分析。

## 控制绘制和合成成本

SwiftUI 最终仍要生成可提交和可合成的内容，因此需要检查：

- 大面积 `blur`、`shadow`、`mask`、半透明叠加和持续变化的复杂 Shape。
- 动画是否每帧改变布局；视觉等价时优先考虑 `transform`、`offset` 或 `opacity`，并验证交互和无障碍语义。
- 大图是否按显示尺寸准备，是否在关键帧同步解码。
- 同一区域是否存在多层没有必要的背景、材质或透明覆盖。

`.drawingGroup()` 会先把子树合成为离屏图像。它可能帮助复杂矢量内容，也会增加离屏纹理、内存和重绘成本，不能作为通用加速开关。`Canvas` 适合一次绘制大量动态图形，但会改变原有的 View 粒度、交互和无障碍实现方式。只有 Instruments 证明绘制或合成是瓶颈时才考虑它们，并读取 `optimization-patterns.md` 中对应的 Display、Prepare 和 Render 规则。

## 处理 UIKit 桥接

`UIViewRepresentable` 和 `UIViewControllerRepresentable` 会把声明式更新映射为 UIKit 对象更新：

- `makeUIView` 或 `makeUIViewController` 只负责创建稳定实例。
- `updateUIView` 或 `updateUIViewController` 保持幂等，只更新真正变化的属性。
- 不在每次 update 中重建播放器、Web View、手势、约束或 delegate。
- 使用 Coordinator 保存需要稳定身份的 delegate、target 或订阅。
- 避免 UIKit 回调无条件写回同一 SwiftUI 状态并形成循环。
- 使用 Long Platform View Updates 和 Time Profiler 确认桥接更新是否是热点。

示意：

```swift
struct PlayerView: UIViewRepresentable {
    let player: AVPlayer

    func makeUIView(context: Context) -> PlayerContainerView {
        PlayerContainerView()
    }

    func updateUIView(_ view: PlayerContainerView, context: Context) {
        guard view.playerLayer.player !== player else { return }
        view.playerLayer.player = player
    }
}
```

## 并发和异步任务

- 把线程安全的解析、排序、格式化、图片准备等纯工作移出主 actor。
- 在正确的 actor 上发布界面观察的最终状态。
- 给任务建立取消、版本和 View 身份校验，避免旧结果覆盖新状态。
- 不在 `body` 中创建无法稳定管理生命周期的非结构化任务。
- `.task(id:)` 适合把任务生命周期与 View 身份或输入关联，但仍需控制重复请求和缓存。
- 不通过启动大量并发任务把主线程问题转成 CPU、内存和调度压力。

## 不要机械套用

- 不认为每次 `body` 调用都会绘制像素。
- 不为了减少更新而给所有 View 添加 `.equatable()` 或自定义相等判断。
- 不把 `AnyView` 一概判定为性能问题。
- 不认为提取子 View 一定能阻止父 View 的计算。
- 不认为 Lazy 容器一定比普通 Stack 更快。
- 不用不稳定 ID 换取看似即时的刷新。
- 不把可观察 UI 状态的写入随意迁移到后台线程。
- 不先测量就重写数据流或引入复杂缓存。

## 验证清单

- 相同操作下，Update Group 的频率是否下降。
- Long View Body、Platform View 或 Other Long Update 的持续时间是否下降。
- Cause & Effect Graph 中触发更新的状态是否符合预期。
- Time Profiler 中目标函数的调用次数和总耗时是否下降。
- Animation Hitches 中 Commit、Render 和 GPU 延迟是否改善。
- 列表身份、滚动位置、焦点、动画和局部状态是否保持正确。
- 异步结果是否可能过期、重复请求或覆盖新状态。
- CPU、内存、GPU 和功耗是否出现新的回退。

## 直接来源

- [Understanding and improving SwiftUI performance](https://developer.apple.com/documentation/xcode/understanding-and-improving-swiftui-performance)
- [Performance analysis](https://developer.apple.com/documentation/swiftui/performance-analysis)
- [Optimize SwiftUI performance with Instruments](https://developer.apple.com/videos/play/wwdc2025/306/)
- [Demystify SwiftUI performance](https://developer.apple.com/videos/play/wwdc2023/10160/)
- [Migrating from the ObservableObject protocol to the Observable macro](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)
- [onGeometryChange(for:of:action:)](https://developer.apple.com/documentation/swiftui/view/ongeometrychange%28for%3Aof%3Aaction%3A%29)
- [Drawing and graphics](https://developer.apple.com/documentation/swiftui/drawing-and-graphics)
