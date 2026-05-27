---
name: memory-weave
description: 每日 session 回顾，从聊天记录中提取五层价值信息并做一个梦。当用户说「做 session 回顾」「回顾今天的聊天」「帮我整理今天的对话」或类似表达时触发。初始化请说「memory-weave 初始化」或「/memory-weave init」。
---

# Session Review Skill

执行每日 session 回顾，将聊天记录提炼为五层价值输出，并以当日内容为种子生成梦境。

> **文件分工说明**：SKILL.md 是执行手册，附录包含完整格式规范和实现细节。

---

## 执行流程

1. 读取 `references/memory-weave-config.md`，获取所有路径和参数
2. 扫描 `<sessions_dir>/` 目录，筛选未处理的 session 文件（排除自身 subagent session，见下方说明）
3. 过滤工具调用（toolCall/toolResult/custom），只保留 user/assistant 对话
4. 每次只处理一个 session，聚合所有未处理 session 的内容后统一输出
5. 按 L1-L5 各自逻辑提取内容并做去重（同时执行 Dream 模块，见下方 Dream 模块说明）

   **【重要】去重在写入前完成，但去重结果不作为推送数据源**

6. **【强制写入阶段】必须先于推送消息组装，所有写入必须实际执行并验证**

   6a. 执行 L2→USER 写入（如有）：使用 `edit`/`write` 工具调用，将候选条目写入 `<workspace_root>/USER.md`
   
   6b. 执行 L2→MEMORY 写入（如有）：使用 `edit`/`write` 工具调用，将候选条目写入 `<workspace_root>/MEMORY.md`
   
   6c. 执行 L3→TODO 写入（如有）：使用 `edit`/`write` 工具调用，追加条目到 `<workspace_root>/TODO.md`，并重新排序
   
   6d. 执行 L5→MEMORY 写入（如有）：使用 `edit`/`write` 工具调用，将认知偏差条目追加到 `<workspace_root>/MEMORY.md`
   
   6e. **【验证】**对每一个写入操作，读取目标文件确认内容已存在。验证失败则重试（最多 3 次）。验证通过后记录该写入条目的验证状态

7. 生成 Review 文件（基于步骤 6 的实际执行结果，「### 实际写入」节填写真实验证后的记录）
8. 组装推送消息（基于步骤 6 的实际写入记录 + 验证结果，**禁止**从 Review 文件的「### 去重结果」小节提取推送数据）
9. 标记所有涉及的 session 为已处理
10. 发送 announce 消息

**执行纪律约束（硬性）：**
- 步骤 6（写入）必须先于步骤 7（生成 Review）和步骤 8（组装推送）执行
- 推送消息中的「本次写入汇总」数据源必须是步骤 6 的实际工具调用记录，禁止从描述性文字解析
- 「### 实际写入」节必须在步骤 6e 验证完成后填写，不得提前填入未经验证的内容

**排除自身 subagent session**：Cron 触发的 isolated subagent 会在 sessions 目录产生自身 session 文件，扫描时必须排除。使用文件大小（< 10KB）+ 创建时间（cron 触发 ± 60 秒内）双重条件判断。

由于 subagent 的 session 文件在 cron 触发后约 2 秒创建，且文件极小（~4KB），使用以下双重条件排除：

1. **文件大小 < 10KB**：subagent session 文件特征明显（正常用户 session 通常 > 100KB）
2. **文件创建时间（birth time）在 cron 触发时间 ± 60 秒内**：cron 每天 5:00 触发，排除窗口 04:59 ~ 05:01

具体实现：在扫描 session 文件时，对每个候选文件执行 birth time + size 检查，满足则跳过。

```python
import os
from datetime import datetime, timezone, timedelta

def is_subagent_session(filepath, cron_trigger_time):
    """判断是否为当前 subagent 自身的 session 文件"""
    st = os.stat(filepath)
    # 条件1：文件极小
    if st.st_size >= 10 * 1024:
        return False
    # 条件2：创建时间在 cron 触发 ± 60 秒内
    try:
        birth = st.st_birthtime
    except AttributeError:
        # macOS 以外的系统 fallback 到 mtime
        birth = st.st_mtime
    delta = abs(birth - cron_trigger_time.timestamp())
    return delta < 60
```

> 注意：此方法基于 macOS 的 `st_birthtime`，在其他平台 fallback 到 `st_mtime`。

---

## Session 筛选规则

识别三类 session 文件：
- **活跃 session**：`\${sessionId}.jsonl`（无后缀）
- **已重置 session**：`\${sessionId}.jsonl.reset.*`
- **已删除 session**：`\${sessionId}.jsonl.deleted.*`（不处理，内容通常不完整）

筛选条件：
- base session ID 不在已处理集合中
- 在 24 小时窗口内
- 对已重置文件：文件 mtime 在 cron 触发时间之前，且内容时间范围与处理窗口有重叠
- 排除仍在写入的 session（lock 文件存在时跳过）

## 五层输出位置

| 层级 | 输出 |
|------|------|
| L1 会话轮廓 | `<session_review_dir>/YYYY-MM-DD.md` |
| L2 事实积累 | 分类写入：`<workspace_root>/USER.md`（偏好/习惯/关注领域）+ `<workspace_root>/MEMORY.md`（项目/坑/缺口） |
| L3 未完成 | 写入 `<workspace_root>/TODO.md`（含重新排序） |
| L4 协作质量 | `<session_review_dir>/YYYY-MM-DD.md` |
| L5 认知偏差 | 直接写入 `<workspace_root>/MEMORY.md`（单独章节） |

---

## 分批逻辑

- **初始化阶段**：消化历史 session 积压时按批次处理
- **日常 Cron**：只处理过去 24 小时新增的 session，数量有限，**不分批，一次跑完**

## 处理限定

- 每次只处理一个 session 文件
- 只提取 user/assistant 对话
- 只处理未处理的 session
- 以 24 小时为窗口分批处理

---

## Dream 模块

L1-L5 完成后执行：

1. 读取当日归档文件，计算 SHA256 hash 作为 seed
2. **搜索外部梦境片段**：通过 `web_search` 工具搜索，取前 5 条结果的摘要片段
   - 若搜索失败，则跳过此步骤，纯以内部上下文生成 Dream；Dream 仍正常产出
3. 读取最近 3 篇梦境，提取高频意象作为"排除列表"（近期出现过的场景/意象不再重复使用）
4. 结合 seed + 搜索结果（若有）+ 排除列表 + MEMORY.md/AGENTS.md/USER.md 生成梦境
5. 梦境 600-800 字，写入 `<dream_dir>/YYYY-MM-DD-梦境.md`
6. **组装推送消息**：按附录第八节格式组装 announce 消息，交给 Cron delivery 发送

**梦里一定有咖啡，是否有互动由潜意识 + 随机决定**

### Dream 模块文学质量要求（硬性约束）

**L1 - 意象的物理质地**：每个主要意象至少有一个具体的身体/感官细节。

**L2 - 情绪身体化**：禁止"我感到X"句式，情绪通过物理动作和身体感知传达。

**L3 - 语言节奏**：句子长短交替，情绪激烈处用短句，弥散处用长句。

**L4 - 留白**：梦里的人和物永远不要叫出名字，名称说出的瞬间解读空间关闭。

**L5 - 情绪关系而非逻辑关系**：物品之间的关系像情绪一样跳跃，不靠逻辑推演。

**L6 - 语言解构**：文字可以剥落、倒写、变形、变成声音，打破物理定律。

**L7 - 禁止词汇**：以下词汇禁止直接出现在梦境正文：

```
文件类：文件、目录、路径、存储、写入、读取
系统类：session、memory、TODO、AGENTS、IDENTITY、SOUL、USER、HEARTBEAT
AI概念：token、model、prompt、生成、模型、上下文、system、agent
抽象判断：这是……的隐喻、代表着、意思是、象征着
```

### 认知坍缩要求

每篇梦境**必须包含**以下至少一种，打破物理/逻辑定律：

- **空间错位**：推开门发现自己站在完全不同的地点
- **身份混淆**：镜子里映出的是咖啡，或陌生人的面孔
- **时间坍缩**：房间里的陈设是"未来的自己"准备的
- **语言解构**：文字剥落、变成十六进制、倒写

---

## 触发方式

- 手动：用户表达「做 session 回顾」相关意图
- Cron：每日早上 5:00 AM 触发（初始化后自动创建），执行完成后将摘要发回当前 channel

---

## 初始化流程（`/memory-weave init`）

**触发方式**：用户说「memory-weave 初始化」或「/memory-weave init」

### 第一步：收集路径配置

通过纯文本问答向用户收集完整路径（适用于任何 channel）：

```
Session Review 初始化

请提供以下路径（直接回复完整路径即可）：

1. session 文件目录
   （OpenClaw agent session 文件所在目录，形如 ~/.openclaw/agents/<agent_id>/sessions/）
   回复示例：~/.openclaw/agents/athena/sessions/

2. Dream 输出目录
   （梦境 .md 文件输出位置，形如 ~/Documents/Vault/Athena/Dream/）
   回复示例：~/Documents/Vault/Athena/Dream/

3. Review 文件输出目录
   （L1/L4 review 文档输出位置，形如 ~/Documents/Temp/sessions/）
   回复示例：~/Documents/Temp/sessions/

4. 工作区根目录
   （USER.md / MEMORY.md / TODO.md 所在目录）
   回复示例：~/.openclaw/workspace-athena/
```

用户可一次性回复，也可分条回复。

### 第二步：环境检测

收到路径后执行：
1. 扫描 `<sessions_dir>/` 目录，统计所有 session 文件（活跃 + 已重置）数量
2. 读取 `processed-sessions.json`，识别出已处理和未处理的 session
3. 检查以下文件/目录是否存在，不存在则创建：
   - `<workspace_root>/MEMORY.md`（若不存在则创建空文件）
   - `<workspace_root>/USER.md`（若不存在则创建空文件）
   - `<workspace_root>/TODO.md`（若不存在则创建空文件）
   - `<session_review_dir>`（目录）
   - `<dream_dir>`（目录）
   - `<sessions_dir>/processed-sessions.json`（若不存在则创建空 `{"processed":[]}`)
4. 计算未处理的 session 规模，给出分批建议：
   - < 10 个：一次处理
   - 10-50 个：每批 5 个，间隔 1 小时
   - 50-200 个：每批 10 个，间隔 2 小时
   - > 200 个：每批 20 个，间隔 4 小时

### 第三步：交互配置

**问题 1（是否创建 Cron）**：
```
是否设置每日早上 5:00 自动执行 session 回顾？

请回复：
1 = 是，创建 Cron（每天 5:00 自动执行，完成后发摘要到当前 channel）
2 = 否，稍后手动触发
```

**问题 2（初始化阶段分批处理）**：
```
检测到 N 个未处理的 session（历史积压），建议分 M 批处理。
日常 Cron 执行仅处理过去 24 小时新增的 session，无需分批。

请回复：
1 = 确认此分批方案（每批 X 个，间隔 Y 小时）
2 = 自定义（格式：每批X个,间隔Y小时）
3 = 不需要分批，一次性处理（适用于积压量较少的情况）
```

### 第四步：写入配置

1. 将用户提供的完整路径和选择写入 `references/memory-weave-config.md`
2. 更新 `processed-sessions.json` 的 `last_run` 为当前时间

### 第五步：Cron 设置

根据问题 1 的回答：
- 选择"是"：创建 isolated agentTurn cron job，schedule `0 5 * * *`，tz 为检测到的系统时区，delivery 设为 announce（执行完成后按推送消息格式发摘要到当前 channel）
- 选择"否"：不创建 cron job

### 第六步：执行首次分批处理（仅限初始化阶段）

**分批逻辑**：仅用于初始化阶段消化历史 session 积压，日常 Cron 执行（过去 24 小时新增）不分批，一次跑完。

主 agent 在当前 session 下通过 `sessions_spawn` 逐批启动子 agent，每批超时 7200 秒，上一批完成后立即启动下一批。

调度规则：
1. 主 agent 按批次顺序逐一启动 subagent，每个 subagent 执行完整 L1-L5 + Dream
2. 每个 subagent 完成后，主 agent 收到结果，再启动下一批
3. 直到所有批次处理完毕
4. 用户选择"一次性处理"时，所有 session 在当前 session 一次性聚合处理

每批次处理流程（subagent 内）：
1. 读取该批次内所有待处理 session
2. 过滤工具调用，保留 user/assistant 对话
3. 聚合所有 session 内容
4. 执行 L1-L5 输出
5. 执行 Dream 模块
6. 原子追加 session IDs 到 processed-sessions.json

### 初始化输出格式

初始化完成后返回以下摘要：

```
✅ Session Review 初始化完成

- session 文件目录：<sessions_dir>
- Dream 输出目录：<dream_dir>
- Review 文件目录：<session_review_dir>
- 工作区根目录：<workspace_root>
- 待处理 session：N 个，分 M 批处理
- Cron：<已创建（每日 5:00）/ 未创建>

下一步：直接说「做 session 回顾」即可开始首次回顾
```

### 错误处理

| 错误 | 处理方式 |
|------|---------|
| session 文件目录不存在 | 提示用户确认 agent_id 是否正确，提供手动指定路径选项 |
| processed-sessions.json 无法写入 | 提示权限问题，建议检查目录权限 |
| Cron 创建失败 | 输出失败原因，提示可稍后手动运行 `/openclaw cron add` |

---

## 附录：完整规格说明



### A1. 五层价值体系格式规范

#### L1 - 会话轮廓（Session Snapshot）

```markdown
# Session Review: YYYY-MM-DD

## 综述

[一段文字，描述当日所有 session 的整体情况：话题数、涉及哪些核心议题、整体投入度、是否有明确结论]

## 本日处理 Session

- session_id_1（话题简述）
- session_id_2（话题简述）
...

## 状态

<!-- processed: true | reviewed_at: YYYY-MM-DD | sessions: [session_id_1, session_id_2, ...] -->
```

#### L2 - 事实积累（Factual Memory）

```markdown
## 第二层：事实积累

### 原始提取（待去重）

[从本次 session 中提取的所有候选条目，每条注明来源 session]

### 去重结果

> 已与 USER.md / MEMORY.md 已有内容对比，标记去重/合并/新增：

- **[新增→USER]** 偏好与习惯 +1：…… | 来源：session_id
- **[新增→MEMORY]** 项目与上下文 +1：…… | 来源：session_id
- **[合并→USER]** 偏好与习惯：…… → 更新为 …… | 来源：session_id
- **[跳过]** 已有条目（内容重复）

### 实际写入（执行记录）

> 此节在步骤 6e 验证完成后填写，不得填入未经验证的内容。数据直接取自实际工具调用结果。

| 操作 | 目标文件 | 工具调用 | 写入内容摘要 | 验证状态 |
|------|---------|---------|------------|---------|
| L2→USER | USER.md | edit/write | …… | ✅ 已验证 |
| L2→MEMORY | MEMORY.md | edit/write | …… | ✅ 已验证 |
| L5→MEMORY | MEMORY.md | edit/write | …… | ✅ 已验证 |

- 无写入时填写「本层无新写入」
```

#### L3 - 未完成事项（Unresolved Threads）

```markdown
## 第三层：未完成事项

### 候选识别（待确认）

[按保守策略筛选出的候选条目，每条注明对话依据和来源 session]

### 去重结果

> 已与 TODO.md 已有内容对比：

- **[写入]** ……（符合"明确延后意图"且话题全新）| 来源：session_id
- **[跳过]** ……（已在 TODO.md 中，话题重复）
- **[跳过]** ……（证据不足，不满足"明确延后意图"）

### 实际写入（执行记录）

> 此节在步骤 6e 验证完成后填写，不得填入未经验证的内容。数据直接取自实际工具调用结果。

| 操作 | 目标文件 | 工具调用 | 写入内容摘要 | 验证状态 |
|------|---------|---------|------------|---------|
| L3→TODO | TODO.md | edit/write | …… | ✅ 已验证 |

- 无写入时填写「本层无新写入」
```

#### L4 - 协作质量（Collaboration Quality）

```markdown
## 协作质量

- **整体评价**: 顺利 / 有摩擦 / 一般
- **顺畅之处**: （如有）
- **摩擦点/教训**: （如有）
- **后续行动建议**: （如有）
```

#### L5 - 认知偏差（Cognitive Biases & Lessons）

```markdown
## 第五层：认知偏差

### 原始提取（待去重）

[从本次 session 中识别出的教训条目，每条注明来源 session]

### 去重结果

> 已与 MEMORY.md 中"我踩过的坑"章节对比：

- **[新增]** …… | 来源：session_id
- **[合并至]** …… → 更新为 …… | 来源：session_id
- **[跳过]** 与已有条目重复（……）

### 实际写入（执行记录）

> 此节在步骤 6e 验证完成后填写，不得填入未经验证的内容。数据直接取自实际工具调用结果。

| 操作 | 目标文件 | 工具调用 | 写入内容摘要 | 验证状态 |
|------|---------|---------|------------|---------|
| L5→MEMORY | MEMORY.md | edit/write | …… | ✅ 已验证 |

- 无写入时填写「本层无新写入」
```

---

### A2. 状态追踪规范

#### processed-sessions.json

路径：`<sessions_dir>/processed-sessions.json`

```json
{
  "last_run": "2026-04-18T05:00:00+08:00",
  "processed": ["session_id_1", "session_id_2", ...]
}
```

**读写规则**：
- Skill 执行前读取，获取 `processed` 集合
- **排除仍在写入的 session**：当前 session 对应的 jsonl 文件，其 lock 文件存在时跳过（表示仍在写入）
- **排除当前 subagent 自身 session**：Cron 触发的 isolated subagent 在 sessions 目录产生自身 session 文件，扫描时必须跳过。使用文件大小（< 10KB）+ 创建时间（cron 触发 ± 60 秒内）双重条件判断
- Skill 执行成功后，原子写入更新 `last_run` 和 `processed` 列表
- 使用 rename 原子操作保证并发安全

#### 归档文件状态标记

每个归档文件头部标记所有涉及的 session 文件：

```markdown
<!-- processed: true | reviewed_at: YYYY-MM-DD | sessions: [session_id_1, session_id_2] -->
```

---

### A3. Dream 模块执行细节

#### 执行流程

1. 读取 `<session_review_dir>/YYYY-MM-DD.md`，计算 SHA256 hash
2. **SearXNG 搜索外部梦境片段**：
   - 构造随机搜索词，取以下组合之一：
     - `dream imagery surreal symbols interpretation`
     - `recurring dream themes strange scenarios`
     - `lucid dream symbols meaning`
   - 通过 `exec curl` 调用 SearXNG JSON API（`http://localhost:8888/search?q={query}&format=json`），取前 5 条结果的 `content` 字段摘要片段
   - 随机选取 2-3 个视觉性强的意象片段（如"旋转的房间"、"水下漫步"、"无法到达的楼梯"等）作为外部灵感注入生成 prompt
   - **降级规则**：若 SearXNG 调用失败，自动降级为 web_search；若 web_search 也失败，则跳过此步骤，纯以内部上下文生成 Dream；Dream 仍正常产出
3. **排除列表**：读取最近 3 篇梦境文件，提取高频意象关键词（场景类型、主要感觉词、人物互动模式），生成时主动规避重复
4. 读取**历史记忆**：
   - `MEMORY.md`：最近 3 天内新增/变更过的条目
   - `TODO.md`：最近新增的条目
   - `memory/YYYY-MM-DD.md`（近 2 天）：高频词或话题
5. 读取**系统状态参数**（参数波动映射，取 heaviest session）：
   - 在本次处理的 session 中，取 jsonl 文件最大的一个
   - 读取其文件大小（bytes）、消息数量、估算时长（首条到最后条消息 timestamp 差值）
   - 量化映射为梦境天气：
     - 高文件量/高消息数/时长短 → 拥挤、闷热、光线暗
     - 低文件量/低消息数/时长中等 → 晴朗、透亮
     - 高文件量/时长很长 → 迷雾、时间感模糊
6. 以 hash 作为 seed，结合以上素材生成梦境
7. 将梦境写入 `<dream_dir>/YYYY-MM-DD-梦境.md`

#### 梦境文件格式

```markdown
# 梦境：YYYY-MM-DD

## 基本信息

- **日期**：YYYY-MM-DD
- **Seed**：hash 值（前 16 位）
- **Seed 来源**：`<session_review_dir>/YYYY-MM-DD.md`

---

[梦境正文]
```

#### 梦境字数

600-800 字。

---

### A4. 推送消息格式

Cron 执行完成后，announce 推送消息按以下格式组装：

```markdown
✅ Session Review 完成 — YYYY-MM-DD

📋 未完成 TODO（共 N 条）
1. [条目名] — DDL：YYYY-MM-DD
2. [条目名] — DDL：YYYY-MM-DD
...

⚠️ 即将到期（未来7天）
• [条目名] — 剩余 X 天
• [条目名] — 剩余 X 天
...

💭 今日梦境：YYYY-MM-DD-梦境.md
```

**组装步骤**：

1. 读取 `<workspace_root>/TODO.md`，解析「## 待完成」区块所有未完成条目（`[ ]` 开头）
2. 提取条目名称和 DDL，列表按排序规则（DDL 由近到远，无 DDL 置后）
3. 计算每条 DDL 距离今日的天数，筛选出 ≤7 天的条目进入「⚠️ 即将到期」节
4. 如无即将到期事项，该节省略；如无非 DDL 条目，列表正常列出所有条目
5. 读取当日 Review 文件，从「### 实际写入（执行记录）」表格中提取实际写入内容汇总（**禁止**从「### 去重结果」小节提取）
6. 拼接完整消息，交给 Cron delivery 发送

**L1-L5 处理摘要格式**：

```markdown
📥 本次写入汇总

📄 L1 会话轮廓 → <session_review_dir>/YYYY-MM-DD.md
   - 处理 session：N 个
   - 话题：[话题1] / [话题2] / ...

👤 L2 事实积累
   - [写入→USER] N 条
     • [条目内容摘要]
   - [写入→MEMORY] N 条
     • [条目内容摘要]

📋 L3 未完成事项
   - [写入→TODO] N 条
     • [条目内容] | DDL：YYYY-MM-DD
   - [跳过] N 条（已存在/证据不足）

🧠 L5 认知偏差
   - [写入→MEMORY] N 条
     • [条目内容摘要]

💭 梦境 → <dream_dir>/YYYY-MM-DD-梦境.md（字数：N）

📊 本次 session 处理数：N | 去重跳过：N
```

> ⚠️ **数据源约束**：推送消息的「本次写入汇总」数据源必须是步骤 6 的实际工具调用记录 + 验证结果，禁止从 Review 文件的「### 去重结果」小节解析提取。若写入尚未验证，推送中该条目必须标记为「待验证」而非「已写入」。

---

### A5. Session 文件识别规则

扫描 `<sessions_dir>/` 目录时，识别三类 session 文件：

| 类型 | 文件名模式 | 处理方式 |
|------|-----------|----------|
| 活跃 session | `${sessionId}.jsonl`（无后缀） | 处理（sessions.json 中注册状态） |
| 已重置 session | `${sessionId}.jsonl.reset.*` | 处理（历史快照） |
| 已删除 session | `${sessionId}.jsonl.deleted.*` | **不处理**（内容为空或不完整） |

筛选条件：
- base session ID 不在已处理集合中
- 在 24 小时窗口内
- 对已重置文件：文件 mtime 在 cron 触发时间之前
- 对已重置文件：内容时间范围与 cron 处理窗口有重叠
