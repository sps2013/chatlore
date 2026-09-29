# ChatGPT Web 抓取层调研

> 调研日期：2026-09-22
> 目标：确认 Safari Web Extension 自动捕获 ChatGPT 对话的**监听点**
> 状态：调研完成，技术方案已选型，**代码等 App 跑通后再写**

## ChatGPT 后端 API（2026 现状）

| 用途 | Method | Endpoint | Payload / Response |
|---|---|---|---|
| 对话列表 | GET | `/backend-api/conversations?offset=0&limit=N` | JSON 数组（每项含 `id`、`title`、`update_time`、`default_model_slug`） |
| 单条对话 | GET | `/backend-api/conversation/{id}` | **单条 dict**（含完整 `mapping`，格式与导出的 `conversations.json` 中一项一致） |
| 新建/续接 | POST | `/backend-api/f/conversation` | 请求体 `{action, messages, conversation_id, parent_message_id, model, …}`；响应是 **SSE 流**（`data: …\n\n`，最后 `data: [DONE]`） |
| 删除 | PATCH | `/backend-api/conversation/{id}` | `{is_visible: false}` |
| 项目内对话 | GET | `/backend-api/gizmos/{id}/conversations` | 团队空间项目内对话列表 |

### 关键发现

1. **新接口路径**：旧版 `/backend-api/conversation`（POST）已被 `/backend-api/f/conversation` 取代。
   监听扩展写脚本时**必须用新路径**，否则新版 chatgpt.com 完全不发请求。
2. **响应是 SSE（Server-Sent Events），不是 WebSocket**。
   SSE 通过 `fetch` + `ReadableStream` 推送，每个 chunk 是 `data: <json>\n\n`。
   这意味着 **hook `fetch` 就能拦到流**，不需要 hook WebSocket。
3. **GET `/backend-api/conversation/{id}` 返回单条 dict**（不是数组），
   跟导出 zip 里 `conversations.json` 中一项的 schema 完全兼容。
   → 可以**直接复用 `ChatGPTParser.parseOne`**，无需重新实现解析逻辑。

## Safari Web Extension 能力

| 能力 | macOS | iOS | 备注 |
|---|---|---|---|
| Content script 注入（isolated world） | 14.0+ | 15.0+ | 默认 |
| **MAIN world 注入**（`world: "MAIN"`） | **14.0+** | **17.0+** | **必需**——否则 hook `fetch` 拿不到引用 |
| `host_permissions` | ✅ | ✅ | manifest V3 必填 |
| `declarativeNetRequest` | ✅ | ✅ | 不能改 body，只能改 header/重定向 |
| `webRequest` (blocking) | ⚠️ Safari 14-15 支持 | ❌ 不支持 | 不能读 body，不能 cancel |
| 自定义 `fetch` 包装器 | ✅ | ✅ | **通过 MAIN world 脚本实现** |

### 限制

- **iOS 17 之前 MAIN world 不稳**（脚本偶尔未注入）。
  Archivic 计划最低 iOS 17.4，正好对齐 WebKit 主线支持。
- **修改 fetch body 不现实**：没有 `webRequest.onBeforeRequest` 拦截 body 的能力。
- **`browser.runtime.sendMessage` 跨上下文**：isolated ↔ MAIN 通过
  `window.postMessage({type: "…", payload}, "*")` 中转。

## 三种抓取方案对比

### A. Hook POST `/backend-api/f/conversation` 响应（流式）

```js
// MAIN world
const origFetch = window.fetch;
window.fetch = async (...args) => {
  const res = await origFetch(...args);
  const url = args[0].toString();
  if (url.includes("/backend-api/f/conversation")) {
    // patch ReadableStream reader，把每个 SSE chunk 转发到 isolated world
    const reader = res.body.getReader();
    const decoder = new TextDecoder();
    let buffer = "";
    const newStream = new ReadableStream({
      async pull(ctrl) {
        const {done, value} = await reader.read();
        if (done) { ctrl.close(); return; }
        const chunk = decoder.decode(value, {stream: true});
        buffer += chunk;
        window.postMessage({type: "CHATGPT_SSE_CHUNK", chunk}, "*");
        ctrl.enqueue(value);
      }
    });
    return new Response(newStream, res);
  }
  return res;
};
```

- ✅ **实时捕获**：用户在网页上发一条消息，扩展立刻抓
- ✅ **覆盖对话续接**：同一个 conversation_id 多次发消息都能聚合
- ✅ **资源最省**：只在用户主动使用时抓
- ❌ **历史对话不抓**：用户必须打开网页才会触发

### B. Hook GET `/backend-api/conversation/{id}` 响应（按需拉取）

```js
if (url.includes("/backend-api/conversation/") && args[0].method === "GET") {
  const clone = res.clone();
  clone.json().then(conv => {
    window.postMessage({type: "CHATGPT_SINGLE", payload: conv}, "*");
  });
  return res;
}
```

- ✅ **API 直接给完整 mapping**，喂给 `ChatGPTParser.parseOne` 即用
- ✅ **用户回看历史对话时也顺手抓了**
- ❌ **依赖用户主动打开**（同样需要 A 来覆盖新建）

### C. Hook GET `/backend-api/conversations` 列表 + 遍历单条

- ✅ **一次性补齐所有历史对话**
- ❌ **调用次数 = 历史对话数**，大账号会被 rate limit
- ❌ **触发频率不可控**：用户每次打开主页都会触发
- ❌ 适用于「补抓」场景，不适合实时

## 推荐方案（已选定）

**A + B 组合，按需触发 + 用户手动「补抓」按钮：**

1. **A**（hook POST 流）作为默认路径：用户在网页上发消息就自动抓
2. **B**（hook GET 单条）作为辅助：用户翻看历史时也顺手抓
3. **C**（拉列表 + 遍历）作为**手动触发**的「补抓历史」按钮（不在 main world 自动跑）
4. 抓到的所有 raw 数据通过 `window.postMessage` 发到 isolated world，
   转发到 background script，background script 调 Native Messaging 把 JSON
   推到 macOS App 的 `ImportExecutor`

## 复用解析层的可行性

**结论：直接复用 `ChatGPTParser`，零代码重复。**

证据：
- 导出的 `conversations.json` 是 `[{id, title, mapping, …}]` 数组
- API 返回的 `GET /backend-api/conversation/{id}` 是 `{id, title, mapping, …}` 单条 dict
- **两者 schema 完全相同**——`ChatGPTParser.parseOne` 已经能处理

唯一需要扩展的 API（在 App 跑通后做）：
```swift
// 当前签名
public func parse(exportRoot: URL) throws -> [Conversation]

// 新增（未来）
public func parseSingle(_ raw: [String: Any]) throws -> Conversation?
// 或
public func parseData(_ data: Data) throws -> [Conversation]  // 自动判 数组/单条
```

**这一步技术风险低，等 App 跑通后再加即可。**

## 验证测试

技术方案验证见 `Tests/ArchivicCoreTests/ChatGPTAPIShapeTests.swift`：
- mock 一段与 API 响应 schema 相同的 JSON
- 验证 `ChatGPTParser` 能正确解析

## TODO（不在本次范围）

- [ ] Safari Extension 容器（Xcode target、manifest.json）
- [ ] Native Messaging bridge
- [ ] MAIN world hook 脚本（pre-build）
- [ ] `ChatGPTParser.parseData` 重载
- [ ] 「补抓历史」按钮 UI
- [ ] OAuth / cookie 注入（如果需要的话）

## 结论

✅ 监听点确认：`fetch('https://chatgpt.com/backend-api/f/conversation', …)` + SSE
✅ Safari Web Extension MAIN world hook 在 macOS 14 / iOS 17+ 可行
✅ 解析层 100% 复用，扩展代码量很小
🚧 代码等 macOS App 跑通后再写，避免在没看到 UI 的情况下盲写