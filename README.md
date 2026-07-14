# iOS UI Performance Skill

一个用于 Codex 的 iOS 页面流畅度诊断与优化 Skill，覆盖 UIKit、SwiftUI 和 Core Animation。

它不会把性能优化简化成固定技巧清单，而是沿 iOS Render Loop 建立证据链：先复现和测量，再定位卡顿阶段，实施最小且安全的修改，最后在相同条件下验证结果。

## 能做什么

- 分析页面滚动、动画、列表刷新和图片显示过程中的卡顿。
- 区分 App Commit、Render Server 和 GPU 阶段的问题。
- 检查 UIKit 的事件处理、布局、绘制、图片准备、图层提交和列表更新。
- 检查 SwiftUI 的状态依赖、`body` 更新、View 身份、`List`、`ForEach`、布局反馈和 UIKit 桥接。
- 结合 FPS、CPU、内存、GPU、Time Profiler、Animation Hitches 和 SwiftUI Instrument 缩小问题范围。
- 评审 Objective-C 或 Swift 代码，也可以在确认热点后实施针对性修改。

## 安装

### 使用 Skill Installer

在 Codex 中输入：

```text
$skill-installer 请从下面的 GitHub 地址安装这个 Skill：

https://github.com/yangfurui/ios-ui-performance-skill/tree/main/skills/optimize-ios-ui-performance
```

安装完成后，可以在新的对话中使用。如果没有立即显示，请重启 Codex，并通过 `/skills` 或输入 `$` 检查 Skill 是否已经可用。

### 手动安装

将 `skills/optimize-ios-ui-performance` 整个目录复制到个人 Skill 目录：

```text
$HOME/.agents/skills/optimize-ios-ui-performance
```

如果只希望在某个项目中使用，可以复制到项目仓库：

```text
<repo-root>/.agents/skills/optimize-ios-ui-performance
```

## 使用方式

推荐在请求开头显式指定 Skill。

### 诊断 SwiftUI 卡顿

```text
$optimize-ios-ui-performance 请诊断这个 SwiftUI 列表滚动卡顿的问题，先分析原因，不要修改代码。
```

### 检查 UIKit 页面

```text
$optimize-ios-ui-performance 检查 FeedViewController.m 的页面流畅度问题，结合 Render Loop 给出优化建议。
```

### 定位并实施优化

```text
$optimize-ios-ui-performance 分析并优化这个聊天列表。保留现有架构，使用 Objective-C，只修改已经确认的性能热点。
```

### 设计新页面

```text
$optimize-ios-ui-performance 帮我设计一个高频消息列表，需要同时考虑数据准备、列表更新、文本排版、图片加载和渲染性能。
```

Skill 也支持根据请求内容自动触发，但显式使用 `$optimize-ios-ui-performance` 更容易获得稳定、可预期的行为。

## 建议提供的信息

信息不完整时，Skill 仍然可以先进行代码检查；如果希望得到更可靠的诊断，建议同时提供：

- 发生卡顿的页面、控件和操作路径。
- 首次进入、滚动、动画、刷新或图片出现等具体时机。
- 设备、系统版本和目标刷新率。
- 数据量、图片尺寸、消息频率等复现条件。
- FPS、CPU、内存、GPU 或 Instruments 记录。
- 希望只检查、只诊断，还是允许修改代码。

## 工作方式

```text
复现
  ↓
建立性能基线
  ↓
映射到 Render Loop
  ↓
追踪实际代码路径
  ↓
实施最小优化
  ↓
在相同条件下重新验证
```

FPS、CPU、内存和 GPU 只作为线索，不会被单独当作卡顿原因。静态代码中的高风险 API 也只作为调查入口，最终结论需要结合调用频率、耗时和 Instruments 证据。

## 仓库结构

```text
skills/
└── optimize-ios-ui-performance/
    ├── SKILL.md
    ├── agents/
    │   └── openai.yaml
    └── references/
        ├── diagnostics.md
        ├── optimization-patterns.md
        ├── render-loop.md
        └── swiftui-performance.md
```

`SKILL.md` 保存核心工作流和参考资料路由，`references` 中的内容只在对应任务需要时读取，避免一次加载无关信息。

## 参考资料

- [Build skills](https://learn.chatgpt.com/docs/build-skills.md)
- [Understanding hitches in your app](https://developer.apple.com/documentation/xcode/understanding-hitches-in-your-app)
- [Understanding and improving SwiftUI performance](https://developer.apple.com/documentation/xcode/understanding-and-improving-swiftui-performance)
