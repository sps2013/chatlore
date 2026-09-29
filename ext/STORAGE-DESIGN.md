# Storage 设计

> 位置：`Sources/ArchivicCore/Storage/`
> 文件：`StorageSchema.swift` / `SQLiteWrapper.swift` / `ConversationStore.swift` / `SearchModels.swift` / `ImportExecutor.swift`

## 目标

- **本地**存储所有导入的对话（SQLite，单文件）
- **全文检索**支持中英文子串（trigram tokenizer）
- **可重复导入**：同一份 zip 导入两次不会产生重复行
- **导入成本 O(1) 内存**：streaming 一个大仓库 + delta 写入 FTS

## 表结构

```
conversations                          ← 主表
├─ rowid          INTEGER PRIMARY KEY AUTOINCREMENT   ← 内部稳定 ID，用于 FTS JOIN
├─ app_uuid       TEXT UNIQUE                         ← 应用层 UUID（导出到 UI 的）
├─ stable_id      TEXT UNIQUE NOT NULL                 ← 跨重导出的稳定 ID
│                   (source + ":" + sourceID)
├─ source         TEXT NOT NULL                       ← SourcePlatform.rawValue
├─ source_id      TEXT NOT NULL                       ← 平台内 ID
├─ title          TEXT NOT NULL
├─ model          TEXT                                ← 默认模型，可空
├─ created_at     REAL NOT NULL                       ← Unix 秒
├─ updated_at     REAL NOT NULL
├─ message_count  INTEGER NOT NULL
└─ is_pinned      INTEGER NOT NULL DEFAULT 0          ← 用户固定

UNIQUE (source, source_id)                            ← 同一平台同 ID 不重复
UNIQUE (stable_id)                                    ← 跨重导出去重

attachments
├─ id          TEXT PRIMARY KEY    (UUID)
├─ message_id  TEXT NOT NULL       → messages.id
├─ kind        TEXT NOT NULL       (image/audio/file/…)
├─ mime_type   TEXT
├─ file_name   TEXT
├─ file_size   INTEGER
├─ local_path  TEXT                (相对归档根的相对路径，可空)
└─ ordinal     INTEGER NOT NULL

messages
├─ id              TEXT PRIMARY KEY    (UUID)
├─ conversation_id TEXT NOT NULL       → conversations.app_uuid
├─ role            TEXT NOT NULL       (user/assistant/system/…)
├─ content         TEXT NOT NULL
├─ model           TEXT
├─ created_at      REAL NOT NULL
├─ is_edited       INTEGER NOT NULL DEFAULT 0
├─ is_regenerated  INTEGER NOT NULL DEFAULT 0
└─ ordinal         INTEGER NOT NULL   (0-based 在对话内的顺序)

conversations_fts                  ← FTS5 虚表（trigram tokenizer）
├─ rowid        ← 对应 conversations.rowid
├─ title        ← conversation.title
└─ body         ← 拼接所有 message.content（按 ordinal 升序）

conversations_fts_map              ← FTS 与主表的桥接表
├─ fts_rowid   INTEGER PRIMARY KEY  ← conversations_fts.rowid
├─ conv_rowid  INTEGER UNIQUE       ← conversations.rowid
├─ title       TEXT                 ← 冗余存储（snippet 可读）
└─ body        TEXT                 ← 冗余存储（snippet 可读）

messages_attachments
├─ messages.id  TEXT    → messages.id
└─ attachments.id TEXT → attachments.id
```

> **关键设计**：每个对话产生一个 FTS 行（不是每个 message 一个），
> 因为 snippet/BM25 算的是整个对话级别的相关度。
> 桥接表 (`conversations_fts_map`) 隔开 FTS 的内部 rowid 和主表 rowid，
> 这样 rebuild / upsert 时 FTS rowid 可以独立分配。

## Upsert 流程

每次 `ConversationStore.insert([…])` 对每条对话执行：

1. 算 `stable_id = "\(source.rawValue):\(sourceID ?? uuid)"`
2. 查主表：`(source, source_id)` 是否已存在？
   - **不存在** → `INSERT` 主表（拿到新 rowid）+ 插入 messages + 插入 attachments
   - **存在** → 拿到旧 `stable_id` 和 `app_uuid`，**DELETE 旧 messages + 旧 attachments**（用 rowid），然后 `UPDATE` 主表（更新 title/updated_at/message_count 等）
3. 重建该对话的 FTS 行：
   - 删 `conversations_fts_map.conv_rowid = ?`
   - 删 `conversations_fts WHERE rowid IN (SELECT fts_rowid FROM map WHERE conv_rowid = ?)`
   - 新 `conversations_fts_map` 行（拿新 fts_rowid = max+1）
   - 新 `conversations_fts` 行（rowid = 新 fts_rowid，body = 拼接 messages）
4. 整个流程包在 `BEGIN IMMEDIATE` / `COMMIT` / `ROLLBACK` 事务里

> **为什么用 stable_id 而不是 (source, sourceID) 直接覆盖？**
> stable_id 跟平台内的 sourceID 完全解耦 —— 如果将来 sourceID 格式变了，stable_id
> 仍然能让旧数据被识别为「同一条对话」。

## 搜索

```
SELECT c.app_uuid, c.source, c.title, c.updated_at,
       snippet(conversations_fts, -1, char(2), char(3), '…', 16)
FROM conversations_fts
JOIN conversations_fts_map m ON m.fts_rowid = conversations_fts.rowid
JOIN conversations c ON c.rowid = m.conv_rowid
WHERE conversations_fts MATCH ?
[ AND c.source = ? ]
ORDER BY rank
LIMIT ? OFFSET ?
```

- `snippet` 第二个参数 `-1` = 自动选包含匹配词的最佳列
  （**注意**：FTS5 trigram 的 column index 是 0-based 的；`-1` 是「自动」）
- 用 `\u{02}` / `\u{03}` 作为 snippet 的首尾标记（ASCII 控制字符，几乎不可能出现在正文里）
- 客户端再把 `\u{02}…\u{03}` 替换成 `<mark>…</mark>` 用于高亮

## FTS 表达式构造

`SearchQuery.ftsExpression`：
- 把用户输入 escape 双引号后，用 `"…"` 包成 phrase query
- trigram 不需要 `*` 前缀（天然子串匹配）
- query 太短（< 3 字符）时 trigram 不命中（已知限制）

## ImportExecutor

把「zip → 解析 → 入库」串成一个原子操作：

```
importZip(at: zipURL, source: .chatgpt)
  │
  ├─ 解压到临时目录
  ├─ 调对应解析器（ChatGPTParser / ClaudeParser / …）→ [Conversation]
  ├─ store.insert([Conversation])           ← upsert 自动去重
  └─ 返回 ImportResult { imported, skipped, failed }
```

未来扩展：
- 进度回调（long import）
- 增量导入（只入时间戳之后的对话）
- 解析器失败时的「per-conversation」错误粒度

## 测试覆盖

| 文件 | 测试数 |
|---|---|
| `ConversationStoreTests.swift` | 21 |
| `ImportExecutorTests.swift` | 5 |
| **小计** | **26** |

覆盖：CRUD、upsert 去重、级联删除、跨平台过滤、中文检索、trigram 短查询限制、
snippet 高亮、覆盖后 FTS 重建、import 幂等、import 不支持平台、import 缺文件。