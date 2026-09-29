# iOS Universal Target 设计

> 目标：iPhone + iPad 共用一份 SwiftUI 代码，全部复用 macOS App 已实现的业务逻辑
> 平台：iOS 17.0+
> 文件：`Sources/ArchivicCoreUI/`（跨平台视图）+ `Sources/ArchivicAppIOS/`（iOS 入口）
> 状态：SwiftPM build 通过（macOS host），iOS runtime 需要 Xcode project 跑（见 §「Xcode 集成」）

## 架构

```
                    ┌─────────────────────────────────┐
                    │  ArchivicCore                    │  ← 业务逻辑 / 模型 / 存储 / 解析
                    │  (纯 Swift library)               │
                    └────────────────┬────────────────┘
                                     │ depends on
                    ┌────────────────▼────────────────┐
                    │  ArchivicCoreUI                 │  ← 跨平台 SwiftUI 视图
                    │  (ContentView / ListView /      │
                    │   SearchBar / Detail / DropZone │
                    │   / ImportPanel)                │
                    └────────┬─────────────────┬──────┘
                             │                 │
              depends on     │                 │  depends on
                             │                 │
                  ┌──────────▼───────┐ ┌──────▼────────────┐
                  │  ArchivicApp     │ │  ArchivicAppIOS   │
                  │  (macOS @main)   │ │  (iOS @main)      │
                  │  + NSOpenPanel   │ │  + DocumentPicker │
                  │  + HSplitView    │ │  + NavigationStack│
                  └──────────────────┘ └───────────────────┘
                  macOS only          iOS only
```

### ArchivicCoreUI 跨平台视图清单

| 文件 | 说明 |
|---|---|
| `Models/AppPaths.swift` | 应用数据路径（macOS = `~/Library/Application Support/Archivic`，iOS = `<App>/Library/Application Support/Archivic` + `isExcludedFromBackup`） |
| `Models/AppStore.swift` | `@MainActor` ObservableObject（共享） |
| `Views/ContentView.swift` | 主视图：`#if os(iOS)` 选 `NavigationSplitView`，否则 `HSplitView` |
| `Views/SearchBarView.swift` | 顶栏搜索框（共享） |
| `Views/ConversationListView.swift` | 左侧列表（共享） |
| `Views/SearchResultsListView.swift` | 搜索结果列表（共享） |
| `Views/HighlightedSnippet.swift` | snippet 高亮渲染（共享） |
| `Views/ConversationDetailView.swift` | 右侧详情 + MessageBubble（共享） |
| `Views/DropZoneView.swift` | **macOS**: NSOpenPanel + 拖入；**iOS**: UIDocumentPickerViewController + 拖入（iOS Files app） |
| `Views/ImportPanel.swift` | **macOS**: NSOpenPanel Form；**iOS**: NavigationStack + Form + UIDocumentPickerViewController |

## 平台差异点

### 1. 路径

| | macOS | iOS |
|---|---|---|
| App Support | `~/Library/Application Support/Archivic/` | `<App>/Library/Application Support/Archivic/` |
| iCloud 备份 | 否（默认） | **是**（沙盒外），需要 `isExcludedFromBackup = true` 防数据库被上传 |
| Documents | 不建议（用户可见） | 不建议 |

`AppPaths.appSupportDir()` 用 `#if os(iOS)` 在 iOS 上设置 `isExcludedFromBackup = true`。

### 2. 文件选择器

| | macOS | iOS |
|---|---|---|
| 点击选择 | `NSOpenPanel.allowedContentTypes = [.zip]` | `UIDocumentPickerViewController(forOpeningContentTypes: [.zip])` |
| 拖入 | `.onDrop(of: [.fileURL])` | `.onDrop(of: [.fileURL, .zip])` + 仅 Files app 拖入到 SwiftUI view（iOS 桌面自由拖入不可用） |

iOS 文件选择器走 `UIViewControllerRepresentable` 包装 `UIDocumentPickerViewController`。

### 3. 窗口/导航

| | macOS | iOS |
|---|---|---|
| 多窗口 | `WindowGroup` + `commands` (⌘O) | `WindowGroup`（iPad 可 Stage Manager 多窗口，iPhone 单窗口） |
| 主布局 | `HSplitView`（macOS 专属） | `NavigationSplitView`（iOS 16+ / macOS 13+，自适应 iPhone/iPad） |
| 菜单栏 | `.commands { CommandGroup(replacing: .newItem) { Button("导入 zip…") } }` | `.toolbar { ToolbarItem(.primaryAction) { Button("导入") } }` |
| 空态 | `ContentUnavailableView` | `ContentUnavailableView`（同样 macOS 14+ / iOS 17+） |

### 4. 颜色

| | macOS | iOS |
|---|---|---|
| 文本背景 | `Color(nsColor: .textBackgroundColor)` | `Color(uiColor: .systemBackground)` |

`SearchBarView` 和 `ConversationDetailView` 用 `#if os(macOS)` 选 nsColor / uiColor。

### 5. 启动 / 生命周期

| | macOS | iOS |
|---|---|---|
| ⌘O 触发 import | 通过 `.commands` + `@State showImportPanel` | 通过 toolbar Button + `@State showImportPanel` |
| 后台 | 用户 ⌘Q 退出 | 系统 home 键退后台；再次回到前台 AppStore 状态保留（@StateObject） |
| 安全作用域 zip URL | 不需要 | 需要 `startAccessingSecurityScopedResource()`（AppStore.importZip 当前没调，将来加） |

## Xcode 集成（iOS 实际 build/run 必须用 Xcode）

SwiftPM 编译 iOS 代码**没有问题**（`swift build` 在 macOS host 上能编过 `#if os(iOS)` 隔离的代码），但 SwiftPM **不能** 跑 iOS executable target（缺 iOS simulator runtime + codesign + info.plist）。

iOS App 实际 build 必须走 Xcode。三种方法：

### 方法 A：手工创建 iOS Xcode project（推荐，零依赖）

```bash
# 1. Xcode → File → New → Project → iOS → App
#    - Product Name: Archivic
#    - Interface: SwiftUI
#    - Language: Swift
#    - 勾选 iPhone + iPad
#    - Bundle ID: com.yourname.archivic

# 2. 把 Sources/ArchivicAppIOS/ArchivicApp.swift 内容粘到自动生成的 ArchivicApp.swift

# 3. File → Add Package Dependencies → Add Local...
#    - 选 /Users/sun/Documents/AICode/Archivic
#    - 勾选 ArchivicCore library
#    - 勾选 ArchivicCoreUI library

# 4. 把 Sources/ArchivicAppIOS/DropZoneView.swift 和 ImportPanel.swift 拖进项目
#    （或者直接把这两个文件内容粘到 Xcode 项目的 .swift 文件）

# 5. 选 iOS Simulator → Cmd+R
```

### 方法 B：用 xcodegen 自动生成 Xcode project

```bash
brew install xcodegen

cat > project.yml << 'EOF'
name: Archivic
options:
  bundleIdPrefix: com.yourname
  deploymentTarget:
    iOS: "17.0"
packages:
  Archivic:
    path: .
targets:
  ArchivicIOS:
    type: application
    platform: iOS
    sources: [Sources/ArchivicAppIOS]
    info:
      path: Info.plist
      properties:
        UILaunchScreen: {}
    dependencies:
      - package: Archivic
        product: ArchivicCore
      - package: Archivic
        product: ArchivicCoreUI
EOF

xcodegen generate
open Archivic.xcodeproj
```

### 方法 C：用 tuist（与 xcodegen 类似，更重量级）

不推荐引入 tuist 这种工具链，除非已经用它。

## 验证清单

| 验证 | 状态 |
|---|---|
| `swift build` 在 macOS host 上编过（含 `#if os(iOS)` 隔离的代码） | ✅ |
| `swift test` 30 tests, 0 failures | ✅ |
| `swift run ArchivicApp` macOS GUI 启动正常 | ✅ |
| `swift build --target ArchivicAppIOS` | ⚠️  不会尝试 build（target 是 iOS-only，但 macOS toolchain 跑 `#if os(iOS)` 隔离） |
| Xcode build ArchivicIOS target 到 iOS Simulator | 🚧 需手工验证 |

## iOS 特有 UX 调整（不在 v0.3 范围，未来优化）

- [ ] Files app 拖入支持（iOS 16+ iPhone Stage Manager）
- [ ] Widget extension（最近 5 条对话）
- [ ] Live Activity（导入进度）
- [ ] Share Extension（从 ChatGPT app 直接分享导出 zip 到 Archivic）
- [ ] iPad 多窗口（Stage Manager）
- [ ] iCloud 同步开关（同步 conversations.sqlite 到 iCloud Drive）
- [ ] 「按日期分组」section 列表
- [ ] 「附件预览」原生 view（图片/PDF 走 QuickLook）

## 代码与原 macOS App 的差异

### 移走的部分（不需要在 iOS）

| macOS 专属 | 在 ArchivicCoreUI 是否还保留 |
|---|---|
| `HSplitView` | ❌ 移到 `macOSBody`（`#if os(macOS)` 隔离） |
| `NSOpenPanel` | ❌ 移到 `macOSBody` |
| `.commands { CommandGroup }` | ❌ 移到 `Sources/ArchivicApp/ArchivicApp.swift` |
| `Color(nsColor:)` | ⚠️ 用 `#if os(macOS)` 在 SearchBarView / ConversationDetailView 选 |

### 增加的部分（iOS 专属）

| iOS 专属 | 在哪 |
|---|---|
| `NavigationSplitView` | `ArchivicCoreUI/Views/ContentView.swift` 的 `iOSBody` |
| `UIDocumentPickerViewController` | `ArchivicCoreUI/Views/DropZoneView.swift` 的 iosBody + `DocumentPickerIOS` struct |
| `NavigationStack` + `.toolbar` | `ArchivicCoreUI/Views/ImportPanel.swift` 的 iosBody |
| `Color(uiColor:)` | SearchBarView / ConversationDetailView 的 `#if os(iOS)` 选 |
| `isExcludedFromBackup = true` | AppPaths.appSupportDir() |

## 总结

- **共享 80%**：所有视图、AppStore、ConversationStore、解析器、入库、搜索
- **平台专属 20%**：文件选择器、窗口 layout、菜单/toolbar
- **零代码重复**：跨平台视图用 `#if os(...)` 选不同实现，逻辑层完全共享
- **实际跑 iOS**：需要 Xcode（SwiftPM 限制），但代码 100% 跨平台已写好