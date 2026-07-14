# iOS Render Loop

## 主链路

```text
Event
  App 接收事件并修改界面状态
  ↓
Commit Transaction
  Layout → Display → Prepare → Commit
  ↓
Render Prepare
  Render Server 使用 CPU 准备渲染命令
  ↓
Render Execute
  GPU 合成图层并生成最终画面
  ↓
Display
  显示系统在目标刷新时刻显示画面
```

这是一条并行流水线。App 可以准备新一帧，同时 Render Server 渲染上一帧，显示系统显示更早的一帧。

不要把固定的 16.67 ms 当作所有设备和所有时刻的唯一预算。刷新率可能变化，应以实际设备、刷新模式和工具显示的截止时间为准。

## 阶段职责

| 阶段 | 执行位置 | 主要工作 | 典型问题 |
| --- | --- | --- | --- |
| Event | App | 处理事件、业务逻辑和 UI 更新 | 同步 I/O、JSON、锁、对象抖动、高频刷新 |
| Layout | App | 计算 View 的位置和尺寸 | 约束复杂、重复布局、文本重复测量 |
| Display | App | 生成发生变化的图层内容 | 大面积重绘、复杂文本、自定义绘制 |
| Prepare | App | 准备图片等提交资源 | 图片首次解码、颜色转换、大图准备 |
| Commit | App | 整理并提交图层树 | 图层过深、节点过多、结构频繁变化 |
| Render Prepare | Render Server | 将图层树转换成 GPU 命令 | 复杂遮罩、阴影、裁剪和动画状态 |
| Render Execute | GPU | 合成图层和像素 | 混合、过度绘制、离屏渲染、大纹理 |
| Display | 显示系统 | 在目标刷新时刻读取缓冲区 | 前面阶段迟到导致旧帧继续显示 |

布局、绘制和图片准备是按需工作。没有失效的内容可以复用，不应假设每个 View 每一帧都重新执行全部步骤。

## Hitch 分类

### Commit Hitch

App 没有在截止时间前完成 Event 或 Commit Transaction。

优先检查：

- 主线程耗时。
- 高频回调和 UI 提交。
- 布局、文本排版和绘制。
- 图片首次准备。
- 图层树整理成本。

### Render Hitch

App 已按时提交，但 Render Server 或 GPU 没有按时完成。

优先检查：

- 图层树复杂度。
- 遮罩、阴影、模糊和裁剪。
- 透明混合和过度绘制。
- 离屏渲染。
- 纹理尺寸和数量。

## 直接来源

- [Explore UI animation hitches and the render loop](https://developer.apple.com/videos/play/tech-talks/10855/)
- [Find and fix hitches in the commit phase](https://developer.apple.com/videos/play/tech-talks/10856/)
- [Demystify and eliminate hitches in the render phase](https://developer.apple.com/videos/play/tech-talks/10857/)
- [Understanding hitches in your app](https://developer.apple.com/documentation/xcode/understanding-hitches-in-your-app)
