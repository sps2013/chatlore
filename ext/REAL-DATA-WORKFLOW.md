# 真实数据回归工作流

> 把 **你导出的真实 AI 对话** 一键脱敏 + 转成 Archivic 的测试 fixture，让解析器在任何代码改动后都能用「真数据」回归。

---

## 0. 一次性准备（已完成）

```bash
cd /Users/sun/Documents/AICode/Archivic
swift build            # 第一次要 5-10 秒
```

工具路径：`.build/debug/archivic-anonymize`

---

## 1. 导出 AI 平台数据（每平台）

### ChatGPT
1. 网页/手机 ChatGPT → 头像 → **Settings**
3. 直接进入 **Data Controls** → **Export Data**
4. 点确认。OpenAI 几分钟到几小时后给你邮箱发邮件。
5. 邮件点链接下载 `chatgpt-export-2026-09-22.zip`

### Claude
1. 网页 claude.ai → 头像 → **Settings**
3. **Account** → **Download your data**
4. 几分钟后邮件收到 zip

---

## 2. 跑脱敏 + 转 fixture（核心步骤）

### 一键模式（推荐）

```bash
# 在 Archivic 项目任意子目录都可以（工具会自己找项目根）
.build/debug/archivic-anonymize fixture ~/Downloads/chatgpt-export-2026-09-22.zip
```

输出示例：
```
✓ Fixture generated
  platform: ChatGPT (chatgpt)
  file:     chatgpt-20260922-1037.json
  path:     /Users/sun/Documents/AICode/Archivic/Tests/ArchivicCoreTests/Fixtures/Resources/chatgpt-20260922-1037.json
  records:  47
  pii hits: 12 (email=3, ipv4=2, phone-cn=1, bearer=4, jwt=2)

Next steps:
  1. 检查文件内容是否符合预期（jq . <path> | head）
  2. 在 Tests/ArchivicCoreTests/ChatGPTParserTests.swift 加引用
  3. git add Tests/ArchivicCoreTests/Fixtures/Resources/chatgpt-20260922-1037.json
```

### 先统计再决定（谨慎模式）

```bash
.build/debug/archivic-anonymize stats ~/Downloads/chatgpt-export-2026-09-22.zip
```

只扫描不修改。打印每个文件命中了哪些 PII。如果 PII 命中过多（比如 1000+），说明数据里可能有大量 token / URL 里的密钥，先回看是不是数据敏感度确实高。

---

## 3. 检查 fixture（必须）

```bash
# 在 Archivic 项目根目录
FIX=Tests/ArchivicCoreTests/Fixtures/Resources/chatgpt-20260922-1037.json

# 看一下顶层结构
jq 'if type == "array" then length else keys end' "$FIX"

# 抽查一条对话结构
jq '.[0] | keys' "$FIX"

# 确认所有 PII 都已被替换（应该返回空）
grep -E "@example\.com|1[3-9][0-9]{9}|192\.168\." "$FIX" || echo "✓ 没有残留 PII"
```

如果 grep 还能匹配到 PII → 工具的脱敏规则覆盖不到你的数据类型**，立刻报告（你需要加新规则）**。

---

## 4. 加进测试

在 `Tests/ArchivicCoreTests/ChatGPTParserTests.swift` 加一个新测试：

```swift
func testRealExport_20260922_1037() throws {
    let url = fixtureURL("chatgpt-20260922-1037")
    let data = try Data(contentsOf: url)
    let conversations = try ChatGPTParser.parse(data)
    XCTAssertGreaterThan(conversations.count, 30, "真实导出应该至少有几十条对话")
    XCTAssertTrue(conversations.allSatisfy { !$0.title.isEmpty }, "所有对话都有标题")
}
```

> 把「20260922-1037」换成你的实际文件名。

跑测试：
```bash
swift test --filter ChatGPTParserTests.testRealExport_20260922_1037
```

---

## 5. 提交 fixture 到 git

```bash
git add Tests/ArchivicCoreTests/Fixtures/Resources/chatgpt-20260922-1037.json
git add Tests/ArchivicCoreTests/ChatGPTParserTests.swift
git commit -m "test: add real ChatGPT export fixture from 2026-09-22"
```

---

## 注意事项

| 项 | 说明 |
|---|---|
| **文件大小** | 真实导出可能 50MB-2GB；脱敏后 JSON 会小一些，但依然可能上百 MB |
| **隐私二次确认** | 提交前请人工 `grep` 一遍自己的姓名、邮箱、电话、身份证后几位、住址、公司名 |
| **共享设置** | `shared_conversations.json` 里可能有公开分享链接（不含敏感内容）但建议人工看 |
| **多平台** | ChatGPT + Claude 各跑一次就能覆盖 v1 的全部真实场景 |
| **DALL-E / 生成图** | ChatGPT 导出的 `dalle-generations/` 是图片本身，**不要提交**。我们的工具不处理二进制图片（仅处理文本类），但你要是担心可以删掉这个目录再跑 |

---

## 下次代码改动后

```bash
swift test
```

如果 `testRealExport_*` 失败 → 解析器回归。修复后这个 fixture 就成为了长期保护伞。