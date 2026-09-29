# macOS App 设计

> 目标：SwiftUI shell + 拖入 zip + 搜索 UI
> 平台：macOS 14.0+ / iOS 17.0+（统一 baseline）
> 文件：`Sources/ArchivicApp/`

## 启动

```
swift run ArchivicApp
```

或 build `.app` bundle（见文末 §"打包成 .app"）。

## 整体布局

```
┌─────────────────────────────────────────────────────────────┐
│ SearchBarView                                       [⌘O 导入]│
├─────────────────────────────────────────────────────────────┤
│ HSplitView                                                   │
│ ┌──────────────────────┬────────────────────────────────────┐│
│ │ DropZoneView          │ ContentUnavailableView            ││
│ │ (拖入 zip)            │ 或 ConversationDetailView         ││
│ ├──────────────────────┤                                    ││
│ │ ConversationListView  │                                    ││
│ │ 或 SearchResultsList  │                                    ││
│ │ (搜索时切换)           │                                    ││
│ │                       │                                    ││
│ │                       │                                    ││
│ └──────────────────────┴────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## 文件清单

```
Sources/ArchivicApp/
├── ArchivicApp.swift           @main entry（WindowGroup + 菜单）
├── Models/
│   ├── AppPaths.swift          ~/Library/Application Support/Archivic
│   └── AppStore.swift          @MainActor ObservableObject（单例）
└── Views/
    ├── ContentView.swift       主视图 + HSplitView
    ├── SearchBarView.swift     顶栏搜索框（debounced）
    ├── DropZoneView.swift      拖入 / 点击选 zip + 选平台
    ├── ConversationListView.swift  左侧对话列表
    ├── SearchResultsListView.swift 搜索结果列表（覆盖）
    ├── ConversationDetailView.swift  右侧详情（标题 + 消息气泡）
    └── ImportPanel.swift       ⌘O 弹窗（手动选 zip）
```

## 数据流

```
[ DropZoneView / ImportPanel ]
        │ 选 zip + platform
        ▼
[ AppStore.importZip(at:source:) ] ─── async ──▶ [ ImportExecutor ]
                                                     │
                                                     ▼
                                           [ store.insert([…]) ]
                                                     │
                                                     ▼
                                            [ FTS5 + 索引重建 ]
                                                     │
                                                     ▼
                                         [ refreshList() + UI 刷新 ]

[ SearchBarView ] ─── debounced ──▶ [ store.performSearch(query:) ]
                                            │
                                            ▼
                                  [ store.search(.init(text:)) ]
                                            │
                                            ▼
                                    [ FTS5 MATCH + snippet() ]
                                            │
                                            ▼
                               [ SearchResultsListView 显示 ]
```

## 关键设计决策

### 1. `AppStore` 是 `@MainActor` 的 `ObservableObject`

所有 `@Published` 必须在主线程改，UI 立即响应。`ConversationStore` 是 actor
（线程安全），从 MainActor 调 actor 方法必须 `await`。**所有 AppStore 方法
都是 `async`**，UI 调用方用 `Task { await store.xxx() }`。

### 2. 三处可能 import 的入口

| 入口 | 触发 | 选平台 |
|---|---|---|
| 拖入 `DropZoneView` | 拖文件到左侧 | 弹菜单 |
| 点击 `DropZoneView` | `NSOpenPanel` | 弹菜单 |
| ⌘O / File 菜单 → `ImportPanel` | 弹窗 + `Picker` | UI 内选 |

所有路径最终都走 `AppStore.importZip(at:source:)`。

### 3. 启动失败降级

`AppStore.init()` 不阻塞 UI。如果 `Application Support/Archivic` 路径下建库失败
（权限、磁盘满），fallback 到临时 db，并通过 `initError` 给用户 alert
提示「数据不会保存」。这样 UI 永远不会黑屏。

### 4. 实时搜索（debounced）

`SearchBarView.onChange(of: localQuery)` 直接调 `store.performSearch(query:)`
—— FTS5 trigram 查询在百万行级别仍然是 < 10ms，所以**不需要 debounce**。
如果未来数据大到需要 debounce，加 250ms 帧合并即可。

### 5. SearchResult 高亮渲染

`SearchHit.snippet` 用 `\u{02}..\u{03}` 包住匹配词（来自 FTS5 snippet() 调用）。
`HighlightedSnippet` view 把这些控制字符替换成带颜色的 `Text` 段。
**SwiftUI 不支持 NSAttributedString**，所以必须手写 parser。

### 6. 选中对话懒加载

`selectedConversation` 是懒加载的——点列表时才调 `store.get(id:)` 拿完整
messages+attachments。列表本身只查 title/source/updated_at/message_count
（来自 `conversations` 主表），不在初始 list 时 join messages。

## 后续优化（不在 v0.2 范围）

- [ ] 搜索结果点击 → 直接选中 + 滚到匹配位置
- [ ] 拖拽重新排序对话（按重要性）
- [ ] 对话分组（按来源 / 按日期）
- [ ] 暗色模式优化（已经自动跟随系统）
- [ ] iOS Universal target（步骤 3）
- [ ] iCloud 同步（步骤 4.5）
- [ ] 「导出选中对话为 Markdown」按钮
- [ ] 「隐私模式：blur 内容直到 hover」

## 打包成 .app

SwiftPM `.executableTarget` 不能直接产 `.app` bundle，需要：

### 方法 A：手写 .app wrapper（推荐用于本地开发）

```bash
APP_NAME=Archivic
APP_DIR=$HOME/Applications/$APP_NAME.app

mkdir -p "$APP_DIR/Contents/MacOS" "$APP_DIR/Contents/Resources"

# 编译 release
swift build -c release --arch arm64 --arch x86_64

# 拷贝二进制
cp .build/apple/Products/Release/$APP_NAME "$APP_DIR/Contents/MacOS/"

# 写 Info.plist
cat > "$APP_DIR/Contents/Info.plist" << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>CFBundleExecutable</key><string>ArchivicApp</string>
  <key>CFBundleIdentifier</key><string>com.yourname.archivic</string>
  <key>CFBundleName</key><string>Archivic</string>
  <key>CFBundlePackageType</key><string>APPL</string>
  <key>CFBundleShortVersionString</key><string>0.2</string>
  <key>CFBundleVersion</key><string>1</string>
  <key>LSMinimumSystemVersion</key><string>14.0</string>
  <key>NSHighResolutionCapable</key><true/>
  <key>NSPrincipalClass</key><string>NSApplication</string>
</dict>
</plist>
EOF

# 签名（ad-hoc，给本机用）
codesign --force --deep --sign - "$APP_DIR"
```

### 方法 B：Xcode project（推荐用于 TestFlight / Mac App Store）

1. Xcode → File → New → Project → macOS → App
2. 添加本地 SwiftPM 依赖：File → Add Package Dependencies → Add Local → 选 Archivic 目录
3. 把 `ArchivicApp` executable target 改成「App」scheme
4. 后续可在 Xcode 里签名、打 release archive

### 方法 C：swift-bundler（第三方）

`brew install swift-bundler`，写 `Bundle.toml`，`swift bundler make`。

## 验证

```bash
swift build                  # ✅ 通过
swift test                   # ✅ 30 tests, 0 failures
swift run ArchivicApp        # ✅ 启动 SwiftUI app（GUI 环境）
```