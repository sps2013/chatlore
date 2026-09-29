# 浏览器插件（Archivic Capturer）安装与测试

> 实时捕获 chatgpt.com 对话，自动推给本机 Archivic App（127.0.0.1:8765）。
> 同一套代码支持 Chrome 和 Safari（MV3）。

## 前置条件

1. **App 必须在运行**（插件把数据 POST 到 App 内置的本地导入服务器）
   ```bash
   cd ~/Documents/AICode/Archivic && swift run ArchivicApp
   ```
2. 验证服务器已监听：
   ```bash
   curl -s http://127.0.0.1:8765/status
   # {"ok":true,"app":"Archivic"}
   ```

## Chrome 安装（开发模式）

1. 打开 `chrome://extensions`
2. 右上角开「开发者模式」
3. 「加载已解压的扩展程序」→ 选 `~/Documents/AICode/Archivic/Extension` 目录
4. 工具栏出现 Archivic 图标

## Safari 安装

### 方式一：临时扩展（Safari 26+，开发调试推荐）

Safari 26 在「设置 → 开发者」新增了「**添加临时扩展…**」，无需 Xcode 打包：

1. Safari 设置 → 高级 → 勾选「在菜单栏中显示"开发"菜单」
2. 设置 → **开发者** 标签 → 勾选「允许未签名扩展」
3. 点「**添加临时扩展…**」→ 输密码 → 选 `~/Documents/AICode/Archivic/Extension` 文件夹
4. 设置 → 扩展 → 勾选启用 Archivic Capturer，网站权限给「允许」

⚠️ 临时扩展的坑（都是实测踩过的）：
- **退出 Safari 即卸载**，下次要重新添加
- 添加时是**拷贝**，改代码后必须卸载重加才生效（「重新载入」按钮不可靠）
- 「允许未签名扩展」勾选**每次退出 Safari 后失效**，需重新勾
- 扩展设置里的网站权限（localhost 等）在重新添加后要重新确认

### 方式二：Xcode 打包（常驻，macOS 14+）

1. Xcode 打开 `Archivic.xcodeproj`（`ArchivicExtension` target 已建好，同步文件夹自动打包）
2. 改扩展代码后记得同步：`Extension/` ↔ `ArchivicExtension/Resources/`
3. ⌘B/⌘R 跑 ArchivicApp → Safari → 设置 → 扩展 → 勾选 ArchivicExtension
4. 未签名构建需在 Safari 开发者设置允许未签名扩展

### Safari 兼容性备忘（v0.1 实测）

| 事项 | 说明 |
|---|---|
| `world: "MAIN"` | Safari 忽略该 manifest 字段 → 已改用 bridge.js 动态插 `<script>` 注入 hook.js（Chrome 也兼容） |
| `content_scripts.matches` | `localhost` 和 `127.0.0.1` 是**两个 host**，都要写；漏写 127.0.0.1 会导致静默不注入 |
| `background` | 用 `scripts` + `type: module` 写法（Chrome MV3 / Safari 都认），不用 `service_worker` |
| 诊断注入 | 页面 Console 敲 `window.__archivicHooked`：`true` = hook 已在 MAIN world 生效 |

## 测试流程

| # | 步骤 | 预期 |
|---|---|---|
| 1 | 点插件图标 | 绿点「Archivic App 已连接」 |
| 2 | chatgpt.com 发一条消息 | 图标闪 ✓；App 弹 toast「浏览器捕获 · 成功 1」；列表出现新对话 |
| 3 | 打开一条历史对话 | 同样被捕获（GET hook） |
| 4 | 插件 popup → 「补抓最近 20 条」 | 结果行显示「完成：处理 N 条，新入库 M 条」 |
| 5 | 关掉 App 再发消息 | 图标变红色 !（App 未运行），数据不丢——重开 App 后再发一条即可 |
| 6 | 重复捕获同一对话 | upsert 覆盖，列表不重复 |

## 数据通路

```
chatgpt.com 页面 fetch
   │ hook.js（MAIN world 拦截，SSE 流 + 单条 GET）
   ▼ window.postMessage
bridge.js（isolated world）
   │ browser.runtime.sendMessage
   ▼
background.js（service worker）
   │ POST http://127.0.0.1:8765/import
   ▼
App ImportServer → ChatGPTParser → SQLite → toast
```

## 安全说明

- 服务器只绑 `127.0.0.1`（局域网不可访问）
- 无任何鉴权（本机回环，风险=本机其他进程可写库；介意可后续加 token）
