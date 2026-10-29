# dsh-novel-forge × serial-novel-session-memory 集成方案

> 把 skill 的三个核心设计吸收进锻炉工具链，让它们从「人工协议」变成「工具原生行为」。

---

## 一、伏笔日历预警 → `novel_briefing` 增强

### 现状

`novel_briefing` 的上下文包已包含「未回收伏笔」列表，每条伏笔有 `plan`（预计回收章号）。但列表是平铺的——不区分「还很远」「马上到期」「已经逾期」。

### 改动

`novel_briefing` 返回的未回收伏笔列表中，对每条伏笔计算 `plan - currentChapter`：

| 距离 | 标记 | 视觉效果 |
|---|---|---|
| > 3 章 | 无标记 | 正常列表项 |
| ≤ 3 章 | `⚠️ 临近回收` | 加粗/高亮 |
| ≤ 0 章（已逾期） | `🚨 已逾期，本章必须回收` | 红色警告 |

### 示例输出

```markdown
## 未回收伏笔
- V-001: 玉佩遇血泛光（埋3, 预计回收18章）← 距当前3章 ⚠️ 临近回收
- V-002: 老仆失言提及十岁（埋7, 预计回收25章）
- V-004: 钱三烧账本（埋8, 预计回收15章）← 已逾期 🚨 本章必须回收
- V-005: 枯井刻字（埋12, 预计回收30章）
```

### 代码位置

`novel_briefing` 组装上下文包时，在伏笔列表段加入距离计算与标记逻辑。`plan` 字段已在 `novel_ledger foreshadow_setup` 时落库——纯读取+计算，零新增存储。

---

## 二、轻量快照 → `novel_write_chapter` 回执增强

### 现状

`novel_write_chapter` 落盘后返回章节元信息（字数/版本/审计结果）。不包含面向下一章写前的状态摘要。

### 改动

`novel_write_chapter` 成功回执中新增 `snapshot_light` 字段：

```json
{
  "snapshot_light": {
    "version": "light-v1.12",
    "chapter": 12,
    "state_changes": [
      {"entity": "苏晚", "key": "位置", "value": "萧府后院"},
      {"entity": "苏晚", "key": "状态", "value": "伤愈,玉佩入袖"}
    ],
    "foreshadow_changes": {
      "new": [{"id": "V-005", "summary": "后院枯井刻字"}],
      "paid_off": ["V-003"],
      "advanced": [{"id": "V-001", "chapter": 12}]
    }
  }
}
```

### 用途

- 模型在下一章写前可引用 `snapshot_light` 快速对齐——不需要重新跑完整 briefing
- 与 skill 中「轻量快照覆盖式保留，只存最新 1 份」一致：工具链侧每次回执覆盖上一章
- **不替代 briefing**——briefing 仍负责注入人物卡/世界书/大纲等完整上下文

---

## 三、标准快照 / 卷末快照 → `novel_export snapshot` 格式

### 现状

`novel_export` 支持 `md` 和 `txt` 两种格式，输出全书正文拼装。

### 改动

新增 `novel_export format: "snapshot"` ，子参数 `level: "standard" | "volume"`：

**标准快照**：从以下数据源自动拼装，零模型调用：

| 快照字段 | 数据源 |
|---|---|
| 圣经·压缩 | `novel_worldbook list` (always=true 条目) + `novel_character list` (取核心角色) + `novel_project promise` |
| 状态 | `novel_ledger query` 遍历所有实体当前值 + `novel_ledger status_at` 核对 |
| 摘要链 | `novel_project status` 的章节索引 (summary 字段) → 近 N 章逐章 + 更早合并 |
| 伏笔日历+台账 | `novel_ledger list` (带 plan/status 字段) |

**卷末快照**：标准快照 + 全量伏笔对账（悬置检测逻辑见下文）。

### 输出

落盘到 `导出/快照/standard-v2.0-截至第15章.md`，格式与 skill 的标准快照模板一致——可被 skill 的换会话流程直接粘贴使用。

---

## 四、伏笔悬置检测 → `novel_project check` 增强

### 现状

`novel_project check` 做全书一致性校验：死人复活/伏笔倒挂/索引缺失。

### 改动

在 `check` 中新增伏笔健康度报告：

```
【伏笔悬置检测】
⚠️ V-002: 最近呼应仍在第7章（当前第20章），已13章未推进 — 可能被遗忘
⚠️ V-006: 埋设于第12章，从未被呼应，已过8章 — 静默伏笔
⚠️ V-001: 预计回收第18章，当前第20章，已逾期2章 — 逾期未回收

【跨卷伏笔】
V-005: 已标记"移交卷2"，台账状态已更新，期限待确认（当前显示"移交卷2"但无具体章号）
```

### 检测规则（纯本地计算，零 token）

| 检测项 | 条件 | 标签 |
|---|---|---|
| 长期静默 | 最近呼应章 = 埋设章，且 `current - 埋设章 > 8` | ⚠️ 静默 |
| 长期无推进 | `current - 最近呼应章 > 10` | ⚠️ 可能被遗忘 |
| 逾期 | `plan < current` 且状态 ≠ 已回收 | 🚨 逾期 |
| 跨卷悬空 | 状态含"移交"但无具体新章号 | ⚠️ 期限悬空 |

---

## 五、会话分段纪律 → 系统提示词补丁

### 拟新增纪律条目

在 dsh-novel-forge 工作流纪律中增加一条：

> **会话分段建议**：长篇写到 30 章以上后，对话历史本身会成为负担。建议：
> - 每写完一卷（或每 15–20 章）开新会话
> - 新会话首条用 `novel_briefing book:"书名" chapter:N+1` 冷启动——工具链从磁盘注入上下文，不依赖旧对话
> - 需要回溯前文细节时用 `novel_search` 检索 + `novel_ledger status_at` 时点查询，**不要翻旧会话的聊天记录**
> - 这等价于 skill 的「禁止回翻更早会话的旧章全文」——但在这里是工具原生保障而非人工纪律

### 与 skill 的对应关系

| skill 协议 | dsh-novel-forge 等价物 |
|---|---|
| 轻量快照（段内衔接） | `novel_write_chapter` 回执的 `snapshot_light` 字段 |
| 标准快照（换会话） | `novel_export snapshot level:standard` |
| 卷末快照（跨卷存档） | `novel_export snapshot level:volume` |
| 章前核查 | `novel_briefing` 未回收伏笔列表（含 ⚠️ 标记） |
| 章后对账 | `novel_write_chapter` 的 `facts_updates` + `novel_ledger foreshadow_payoff` |
| 段末悬置预警 | `novel_project check` 伏笔健康度报告 |
| 禁止回翻旧会话 | `novel_search` + `novel_ledger status_at`（工具替代记忆） |
| 双轨落盘 | 工具链原生——所有数据在磁盘，不依赖会话 |

---

## 六、实施优先级

| 优先级 | 改动项 | 工作量 | 影响面 |
|---|---|---|---|
| **P0 · 立即可做** | `novel_briefing` 伏笔距离标记（⚠️/🚨） | 小（纯读取+计算） | 每章写前自动受益 |
| **P0 · 立即可做** | `novel_project check` 伏笔悬置检测 | 小（纯本地计算） | 段末/卷末体检一键跑 |
| **P1 · 下一个迭代** | `novel_write_chapter` 回执加 `snapshot_light` | 中（回执结构变更） | 替代轻量快照的模型手写 |
| **P1 · 下一个迭代** | `novel_export snapshot` 格式 | 中（新组装逻辑） | 替代标准/卷末快照的模型手写 |
| **P2 · 提示词更新** | 会话分段纪律写入工作流规则 | 小（文本变更） | 长期写作习惯引导 |

---

## 七、不做的边界

以下 skill 设计在 dsh-novel-forge 环境中有工具原生替代，**不需要额外集成**：

| skill 设计 | dsh-novel-forge 已有替代 | 原因 |
|---|---|---|
| 人工快照搬运 | 文件系统持久化 | 工具链天然磁盘落盘，不存在"搬运丢失"问题 |
| 快照 token 预算控制 | 无需——工具注入按需裁剪 | `novel_scene` 控制出场人物 + `novel_briefing` 按预算组装，上下文永远是裁剪过的 |
| 纯会话环境的双轨保存 | 不适用 | DSH 天然有文件系统能力 |
| 伏笔编号 V-00X | `novel_ledger` 自动分配 id | 工具侧 id 全局唯一，不需要人工编号 |
| "快照是唯一跨段记忆源" | 磁盘数据是唯一真相 | `novel.json` + 账本 + 世界书 + 人物卡 → 比快照更可靠 |