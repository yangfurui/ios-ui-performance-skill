# 卡顿诊断

## 目录

- [指标的作用](#指标的作用)
- [建立复现条件](#建立复现条件)
- [Animation Hitches](#animation-hitches)
- [SwiftUI Instrument](#swiftui-instrument)
- [代码检查入口](#代码检查入口)
- [证据记录格式](#证据记录格式)
- [直接来源](#直接来源)

## 指标的作用

开发阶段可以持续观察：

- FPS：发现画面更新是否出现波动。
- CPU：发现主线程或计算任务压力。
- 内存：发现大图、位图和缓存峰值。
- GPU：发现渲染压力变化。

这些指标只负责发现异常和缩小范围，不能直接确定 Render Loop 中的具体阶段。

例如：

- FPS 下降、CPU 上升，不等于一定是布局问题。
- GPU 上升，不等于一定发生了离屏渲染。
- 内存上升，不等于一定是图片解码导致卡顿。
- 平均 FPS 正常，不等于没有短暂 Hitch。

## 建立复现条件

记录：

- 页面入口和操作步骤。
- 首次进入还是重复进入。
- 滚动速度和持续时间。
- 数据数量和内容长度。
- 图片数量、来源和像素尺寸。
- 动画类型和并发数量。
- 设备型号、系统版本和刷新率。
- 是否连接调试器及是否使用 Release 配置。

优化前后保持这些条件一致。

## Animation Hitches

优先观察：

- `Commits`
- `Renders`
- `GPU`
- `Frame Lifetimes`
- `User Events`

### Commits 延迟

继续使用 Time Profiler 检查主线程：

- 数据解析和模型转换。
- 同步文件或数据库读取。
- 锁等待和 `dispatch_sync`。
- 文本测量和富文本排版。
- Auto Layout 约束求解。
- `layoutIfNeeded`。
- `drawRect:` 和 Core Graphics 绘制。
- 图片解码和缩略图生成。
- View、Layer 和对象批量创建销毁。
- `reloadData` 和高频数据源刷新。

### Renders 或 GPU 延迟

继续检查：

- View 和 Layer 的层级。
- 每帧发生变化的图层数量。
- `mask`、`masksToBounds` 和复杂裁剪。
- 动态阴影和缺少 `shadowPath`。
- 模糊、Effect View 和复杂透明效果。
- 大面积透明图层。
- 重复覆盖同一区域的背景和蒙层。
- 大尺寸纹理和一帧内的纹理数量。
- `shouldRasterize` 的命中和失效情况。

这些 API 或属性只是调查入口。必须结合时间线、图层内容和实际 GPU 耗时判断。

## SwiftUI Instrument

在支持专用 SwiftUI Instrument 的 Xcode 与系统组合上，优先检查：

- `Update Groups`：把同一轮 SwiftUI 更新组织到一起，并与 Hitch 时间线对齐。
- `Long View Body Updates`：定位耗时较长的 `body` 更新。
- `Long Platform View Updates`：定位 `UIViewRepresentable`、`UIViewControllerRepresentable` 等平台视图桥接更新。
- `Other Long Updates`：定位不属于前两类的 SwiftUI 长更新。
- `Cause & Effect Graph`：追踪状态变化与 View 更新之间的因果关系。

然后使用 Time Profiler 查看长更新内部的实际调用栈，并用 Animation Hitches 判断它是否延长了帧生命周期。如果 CPU 峰值发生时 SwiftUI 更新通道没有对应活动，应继续调查 SwiftUI 更新之外的 App 工作，而不是强行归因于 `body`。

SwiftUI 页面还应完整读取 `swiftui-performance.md`。Xcode 26 引入了专用 SwiftUI Instrument；较旧工具链缺少相同通道时，使用 Time Profiler、Animation Hitches、signpost 和有针对性的调用计数补充证据。

## 代码检查入口

### Event

搜索并追踪：

- 网络回调、Socket 回调和通知。
- 定时器、Display Link 和滚动回调。
- JSON、正则、字符串拼接和富文本构建。
- 同步 I/O、数据库和锁。
- 高频 `reloadData`、`reloadSections` 和布局请求。
- 大量对象创建和释放。

### Layout

搜索并追踪：

- `layoutSubviews`
- `layoutIfNeeded`
- `setNeedsLayout`
- 约束的重复创建、删除和激活。
- `sizeThatFits:`、`boundingRect` 和文本排版。
- 递归触发父视图布局的代码。

### Display 和 Prepare

搜索并追踪：

- `drawRect:`
- `setNeedsDisplay`
- `setNeedsDisplayInRect:`
- `UIGraphicsBeginImageContext`
- `UIGraphicsImageRenderer`
- `imageWithData:`
- `prepareForDisplay`
- `prepareThumbnail`
- ImageIO 和图片加载库的缩略图配置。

### Commit 和 Render

搜索并追踪：

- 高频 `addSubview:`、`removeFromSuperview`。
- 高频 `addSublayer:`、`removeFromSuperlayer`。
- `mask`、`masksToBounds`、`shadowPath`。
- `alpha`、透明背景和全尺寸蒙层。
- `shouldRasterize`。
- 原图像素尺寸与最终显示尺寸。

## 证据记录格式

使用下面的结构记录每个发现：

| 观察 | 证据 | 对应阶段 | 可信度 | 还需验证 |
| --- | --- | --- | --- | --- |
| 示例：主线程重复排版 | Time Profiler 中排版函数占用明显 | Layout | 高 | 缓存后重新录制 |
| 示例：头像原图过大 | 3000 px 原图显示为 40 pt | Prepare | 中 | 开启缩略图后比较内存和 Hitch |
| 示例：透明图层较多 | 图层检查发现多层透明覆盖 | Render Execute | 中 | 结合 GPU 时间确认 |

## 直接来源

- [Improving app responsiveness](https://developer.apple.com/documentation/xcode/improving-app-responsiveness)
- [Understanding hitches in your app](https://developer.apple.com/documentation/xcode/understanding-hitches-in-your-app)
- [Performance and metrics](https://developer.apple.com/documentation/xcode/performance-and-metrics)
- [Understanding and improving SwiftUI performance](https://developer.apple.com/documentation/xcode/understanding-and-improving-swiftui-performance)
- [Optimize SwiftUI performance with Instruments](https://developer.apple.com/videos/play/wwdc2025/306/)
