# 页面流畅度优化模式

本文件覆盖 UIKit、Core Animation 以及 UIKit 与 SwiftUI 共用的 Render Loop 优化模式。检查 SwiftUI 的状态依赖、`body`、视图身份、`List`、`ForEach`、布局反馈或平台视图桥接时，还需完整读取 `swiftui-performance.md`。

## 目录

- [总原则](#总原则)
- [Event](#event)
- [Layout](#layout)
- [Display](#display)
- [Prepare](#prepare)
- [Commit](#commit)
- [Render Prepare](#render-prepare)
- [Render Execute](#render-execute)
- [列表更新](#列表更新)
- [对象创建和销毁](#对象创建和销毁)
- [验证清单](#验证清单)
- [直接来源](#直接来源)

## 总原则

按照以下顺序考虑优化：

1. 不做没有必要的工作。
2. 复用已有结果。
3. 缩小更新范围。
4. 合并更新频率。
5. 把线程安全的纯计算移出主线程。
6. 提前准备资源。
7. 简化图层和像素工作。
8. 测量并验证。

## Event

### 可以考虑

- 在后台完成 JSON 解析、业务模型转换、字符串处理和富文本准备。
- 把传给后台的输入复制成不可变快照。
- 主线程只提交已经准备好的展示模型。
- 合并高频 UI 更新，把消息接收频率与 UI 提交频率分开。
- 为积压队列设置明确上限，并由业务决定降级策略。
- 复用 Cell、播放器和创建成本较高的对象。
- 对低频页面和组件延迟创建。

### 注意

- UIKit 对象和最终数据源更新仍然放在主线程。
- 不默认丢弃业务数据。
- 不通过无限并发把主线程压力转化为 CPU、内存和调度压力。
- 不在没有测量依据时把对象释放迁移到后台。

## Layout

### 可以考虑

- 缓存文本尺寸和排版结果。
- 为缓存建立完整失效条件，例如内容、宽度、字体和显示模式。
- 初始化时建立稳定的 View 和约束结构。
- 后续只更新必要的约束 `constant`、`frame` 和显隐状态。
- 把 `setNeedsLayout` 限制在负责相关子 View 的最小容器。
- 只在马上需要最终布局结果时调用 `layoutIfNeeded`。
- 视觉允许时使用 `transform` 和 `opacity` 表达动画。

### 注意

- `frame` 不天然比 Auto Layout 快。
- 缓存不完整会造成错位和旧布局。
- 修改一处约束可能影响更高层级，必须确认实际约束归属。

## Display

### 可以考虑

- 删除没有自定义内容的空 `drawRect:`。
- 内容没有变化时不要调用 `setNeedsDisplay`。
- 只有固定区域变化时使用局部重绘。
- 复用已经生成的绘制结果。
- 复杂文本或自绘确认成为瓶颈后再考虑异步绘制。

### 异步绘制必须具备

- 主线程生成不可变绘制快照。
- 后台绘制不访问会被并发修改的 UIKit 状态。
- 支持任务取消或内容版本校验。
- Cell 等复用对象需要身份校验。
- 最终 Layer 内容只在主线程提交。
- 控制并发和位图内存峰值。

异步绘制减少的是主线程关键路径，不一定减少总工作量。

## Prepare

### 可以考虑

- 在图片上屏前完成显示准备。
- 按最终显示像素尺寸生成缩略图。
- 使用 `prepareForDisplay`、`prepareThumbnail` 或 ImageIO。
- 使用图片库提供的解码和缩略图选项。
- 缓存已经准备好的图片，并设置合理的内存预算。
- 快速滚动时，对非关键图片延迟加载。

### 注意

- 区分点尺寸和像素尺寸。
- 不为几十像素的头像长期持有几千像素的解码位图。
- 控制解码并发，避免同时产生多个大位图峰值。
- Cell 回调必须校验当前代表的数据身份。

## Commit

### 可以考虑

- 删除不负责布局、裁剪、背景、交互或动画的中间容器。
- 保持 View 和 Layer 结构稳定。
- 不在每次刷新时删除并重建固定子树。
- 将共同展示、共同滚动且不需要独立交互的内容合并为富文本或自绘内容。
- 只更新真正发生变化的节点。
- 把布局和绘制限制在最小子树。

### 注意

- 把 View 换成相同数量的 Layer 不等于压平图层树。
- 自绘会减少节点，但可能增加 Display 成本和交互实现复杂度。
- 按钮、输入框、无障碍元素和独立动画通常应该保留为独立控件。

## Render Prepare

### 可以考虑

- 保持图层结构稳定。
- 为形状稳定的阴影设置 `shadowPath`。
- 只有子内容确实需要裁剪时才开启裁剪。
- 使用简单圆角代替不必要的复杂 Mask。
- 减少每帧发生变化的遮罩、Effect View 和图层子树。

### 注意

- 圆角本身不等于一定离屏渲染。
- 阴影和裁剪同时存在时，可以分离外层阴影和内层裁剪职责。
- `shadowPath` 需要在尺寸变化时同步更新。

## Render Execute

### 可以考虑

- 对完全不透明且填满 bounds 的内容正确设置 `opaque` 和背景色。
- 删除完全被覆盖的背景、透明蒙层和重复渐变。
- 减少大面积透明图层重叠。
- 控制动态阴影、复杂遮罩、模糊等额外渲染工作。
- 使用接近显示尺寸的纹理。
- 大量稳定小资源确认存在纹理切换瓶颈后，再考虑合图。

### 注意

- 含透明像素、透明圆角区域或半透明内容时，不得错误声明为不透明。
- `shouldRasterize` 只有在缓存能够复用时才可能有收益；内容频繁变化会反复失效。
- 合图会增加图集维护、边缘串色和常驻内存成本。
- Color Blended Layers 等调试结果需要结合 GPU 时间判断。

## 列表更新

- 优先保证数据源与 UI 状态一致。
- 合并刷新用于限制 UI 提交频率，不是限制数据接收频率。
- 只有在能够保证数据源、索引和生命周期一致时，才使用局部插入、删除或批量更新。
- 不为了避免 `reloadData` 而引入更高的崩溃和错乱风险。
- 如果 `reloadData` 尚未成为热点，保留简单可靠的实现。
- 如果列表刷新确实成为热点，再根据数据源模型选择批量刷新、Diffable Data Source 或安全的局部更新。

## 对象创建和销毁

- 优先复用高成本对象。
- 延迟创建低频内容。
- 在后台准备模型、字符串和排版数据。
- 只有 Time Profiler 明确显示大型纯数据对象的 `dealloc` 阻塞主线程，并且能够控制最后一个强引用的线程时，才考虑后台释放。
- 不在后台销毁线程约束不明确的 UIKit、Core Animation 或图形资源。

## 验证清单

优化后检查：

- 原卡顿路径是否改善。
- Commit、Render 或 GPU 时间是否下降。
- FPS 和 Hitch 次数是否改善。
- CPU、内存和位图峰值是否可接受。
- 是否增加输入延迟或后台队列积压。
- Cell 复用是否显示错内容。
- 图片和异步绘制是否出现闪烁。
- 数据源和列表更新是否一致。
- 页面功能、动画、交互和无障碍是否正常。

## 直接来源

- [Core Animation Basics](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/CoreAnimationBasics/CoreAnimationBasics.html)
- [Improving Animation Performance](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/CoreAnimation_guide/ImprovingAnimationPerformance/ImprovingAnimationPerformance.html)
- [setNeedsLayout](https://developer.apple.com/documentation/uikit/uiview/setneedslayout())
- [setNeedsDisplay](https://developer.apple.com/documentation/uikit/uiview/setneedsdisplay())
- [UIImage](https://developer.apple.com/documentation/uikit/uiimage)
- [YYAsyncLayer](https://github.com/ibireme/YYAsyncLayer)
- [SDWebImage](https://github.com/SDWebImage/SDWebImage)
