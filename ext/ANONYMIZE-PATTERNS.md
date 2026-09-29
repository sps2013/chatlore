# 脱敏模式清单

`archivic-anonymize` 当前支持 14 类 PII 识别。每条规则都有：命中样本、正则、误报率说明。

---

## 1. 联系方式

| 类别 | 正则 | 命中样本 | 备注 |
|---|---|---|---|
| **email** | `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}` | `alice@example.com` | 标准邮箱，最稳 |
| **phone-cn** | `\b1[3-9]\d{9}\b` | `13800138000` | 仅大陆 11 位手机号 |
| **phone-intl** | `(?:\+\|00)\s*\d{1,3}[\s\-]?\d{3,4}...` | `+1 555-123-4567` | 国际号码，能匹配 80%+ 真实情况 |

---

## 2. 网络

| 类别 | 正则 | 命中样本 | 备注 |
|---|---|---|---|
| **ipv4** | `\b(?:\d{1,3}\.){3}\d{1,3}\b` | `192.168.1.1` | 包括内网、回环 |
| **ipv6** | `\b(?:[0-9a-fA-F]{1,4}:){2,7}[0-9a-fA-F]{1,4}\b` | `2001:0db8::1` | 包括完整和压缩形式 |

---

## 3. 金融

| 类别 | 正则 | 命中样本 | 备注 |
|---|---|---|---|
| **iban** | `\b[A-Z]{2}\d{2}[A-Z0-9]{11,30}\b` | `DE89370400440532013000` | 欧元区银行账号 |
| **credit-card** | `\b(?:\d[ \-]?){13,19}\b` | `4111 1111 1111 1111` | 13-19 位数字组合。**未做 Luhn 校验** |
| **ssn** | `\b\d{3}-\d{2}-\d{4}\b` | `123-45-6789` | 美国 SSN |

---

## 4. Token / 密钥

| 类别 | 正则 | 命中样本 | 备注 |
|---|---|---|---|
| **jwt** | `\beyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\b` | `eyJhbGciOi...` | 标准 JWT 三段式 |
| **bearer** | `(?i)bearer\s+[A-Za-z0-9_\-.=]{20,}` | `Bearer abc123def456...` | HTTP Authorization 头 |
| **api-key-hex** | `\b(?:sk\|pk\|key)[-_][A-Za-z0-9]{20,}\b` | `sk-abc123def456...` | OpenAI/Anthropic 风格 |
| **github-pat** | `\bghp_[A-Za-z0-9]{36}\b` | `ghp_abc123...` | GitHub Personal Access Token |
| **aws-access-key** | `\bAKIA[0-9A-Z]{16}\b` | `AKIA<your-aws-key>` | AWS Access Key ID |
| **private-key** | `-----BEGIN (?:RSA \|EC \|DSA \|OPENSSH \|PGP )?PRIVATE KEY-----` | `-----BEGIN RSA PRIVATE KEY-----` | PEM 私钥头标记 |

---

## 误报案例（你可能需要）

### 信用卡号 `credit-card` 规则

13-19 位数字+可选分隔符的规则可能误报：

- **长数字 ID**（如 ChatGPT 的 `id: 173456789012345`）→ **会被替换为 `[credit-card]`**。
- 解决方案：要么让用户确认这条不是信用卡，要么加 Luhn 校验。

### Email 在 URL 里

URL 里的邮箱（如 `mailto:foo@bar.com`）会被识别为 email。如果在 URL 路径里出现 `?from=user@email.com&to=other@email.com`，也会全部替换。

### IPv4 可能误报版本号

`1.0.0.0` 这种完全有可能是 IP 也可能是版本号。目前规则不区分。

---

## 添加新规则的步骤

1. 在 `Sources/AnonymizeTool/main.swift` 的 `allRules` 数组里加一行：
   ```swift
   RegexRule(name: "stripe-key", pattern: #"\bsk_live_[A-Za-z0-9]{24,}\b"#)
   ```
2. 在 `Tests/ArchivicCoreTests/AnonymizerTests.swift` 加测试样本：
   ```swift
   let text = "My key is sk_live_<your-stripe-key>"
   let (out, hits) = anonymize(text)
   XCTAssertEqual(hits["stripe-key"], 1)
   ```
3. `swift test`，更新本文件加新行。

---

## 不覆盖（已知漏洞，按需加）

- ❌ **中国身份证**（18 位，含地址码）— 加正则 `\b\d{17}[\dXx]\b` 即可
- ❌ **AWS Secret Access Key**（40 字符 base64）— 难精准，依赖上下文
- ❌ **Stripe / Square / PayPal 各种 Key** — 各家格式不同，按需加
- ❌ **数据库连接串** — `postgres://user:pass@host` 之类，依赖工具链泄露
- ❌ **地理位置坐标** — `40.7128, -74.0060` 这种，需要上下文
- ❌ **生物特征（指纹 hash）** — 极少出现在 AI 对话里