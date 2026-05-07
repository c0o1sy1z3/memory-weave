---
name: memory-weave
description: 每日 session 回顾，从聊天记录中提取五层价值信息并做一个梦。当用户说「做 session 回顾」「回顾今天的聊天」「帮我整理今天的对话」或类似表达时触发。初始化请说「memory-weave 初始化」或「/memory-weave init」。
---

# Memory Weave Skill

执行每日 session 回顾，将聊天记录提炼为五层价值输出，并以当日内容为种子生成梦境。

## 执行流程

1. 读取 `references/memory-weave-config.md`，获取所有路径和参数
2. 扫描 `<sessions_dir>/` 目录，筛选未处理的 session 文件
3. 过滤工具调用（toolCall/toolResult/custom），只保留 user/assistant 对话
4. 每次只处理一个 session，聚合所有未处理 session 的内容后统一输出
5. 按 L1-L5 各自逻辑输出内容
6. 执行 Dream 模块
7. 标记所有涉及的 session 为已处理

## Session 筛选规则

识别三类 session 文件：
- **活跃 session**：`${sessionId}.jsonl`（无后缀）
- **已重置 session**：`${sessionId}.jsonl.reset.*`
- **已删除 session**：`${sessionId}.jsonl.deleted.*`（不处理，内容通常不完整）

筛选条件：
- base session ID 不在已处理集合中
- 在 24 小时窗口内
- 对已重置文件：文件 mtime 在 cron 触发时间之前，且内容时间范围与处理窗口有重叠
- 排除仍在写入的 session（lock 文件存在时跳过）

## 五层输出位置

| 层级 | 输出 |
|------|------|
| L1 会话轮廓 | `<memory_weave_dir>/YYYY-MM-DD.md` |
| L2 事实积累 | 分类写入：`<workspace_root>/USER.md`（偏好/习惯/关注领域）+ `<workspace_root>/MEMORY.md`（项目/坑/缺口） |
| L3 未完成 | 直接写入 `<workspace_root>/TODO.md` |
| L4 协作质量 | `<memory_weave_dir>/YYYY-MM-DD.md` |
| L5 认知偏差 | 直接写入 `<workspace_root>/MEMORY.md`（单独章节） |

## 分批逻辑

- **初始化阶段**：消化历史 session 积压时按批次处理
- **日常 Cron**：只处理过去 24 小时新增的 session，数量有限，**不分批，一次跑完**

## Dream 模块

L1-L5 完成后执行：
1. 读取 `<memory_weave_dir>/YYYY-MM-DD.md`，计算 SHA256 hash 作为 seed
2. **随机搜索外部梦境片段（可选）**：通过 `web_search` 取前 5 条结果，随机选取 2-3 个摘要片段作为外部意象注入生成 prompt
   - **降级规则**：若 web_search 失败（网络错误或无结果），自动跳过此步骤，纯以内部上下文生成 Dream；Dream 仍正常产出
3. 读取最近 3 篇梦境，提取高频意象作为"排除列表"（近期出现过的场景/意象不再重复使用）
4. 结合 seed + 搜索结果（若有）+ 排除列表 + MEMORY.md/AGENTS.md/USER.md 生成梦境
5. 梦境 600-800 字，写入 `<dream_dir>/YYYY-MM-DD-梦境.md`

**梦里一定有咖啡，是否有互动由潜意识 + 随机决定**

## 状态追踪

`processed-sessions.json` 位于 `<sessions_dir>/`，结构：

```json
{
  "last_run": "2026-04-18T05:00:00+08:00",
  "processed": ["session_id_1", "session_id_2", ...]
}
```

执行成功后，原子追加本次处理的 sessionIds 到 `processed` 列表（先写临时文件再 rename）。

---

# 初始化流程（`/memory-weave init`）

**触发方式**：用户说「memory-weave 初始化」或「/memory-weave init」

## 第一步：收集路径配置

通过纯文本问答向用户收集完整路径（适用于任何 channel）：

```
Memory Weave 初始化

请提供以下路径（直接回复完整路径即可）：

1. session 文件目录
   （OpenClaw agent session 文件所在目录）
   回复示例：~/.openclaw/agents/athena/sessions/

2. Dream 输出目录
   （梦境 .md 文件输出位置）
   回复示例：~/Documents/Vault/Athena/Dream/

3. Review 文件输出目录
   （L1/L4 review 文档输出位置）
   回复示例：~/Documents/Temp/sessions/

4. 工作区根目录
   （USER.md / MEMORY.md / TODO.md 所在目录）
   回复示例：~/.openclaw/workspace-athena/
```

用户可一次性回复，也可分条回复。

## 第二步：环境检测

收到路径后执行：
1. 扫描 `<sessions_dir>/` 目录，统计所有 session 文件（活跃 + 已重置）数量
2. 读取 `processed-sessions.json`，识别出已处理和未处理的 session
3. 检查以下文件/目录是否存在，不存在则创建：
   - `<workspace_root>/MEMORY.md`（若不存在则创建空文件）
   - `<workspace_root>/USER.md`（若不存在则创建空文件）
   - `<workspace_root>/TODO.md`（若不存在则创建空文件）
   - `<memory_weave_dir>`（目录）
   - `<dream_dir>`（目录）
   - `<sessions_dir>/processed-sessions.json`（若不存在则创建空 `{"processed":[]}`)
4. 计算未处理的 session 规模，给出分批建议：
   - < 10 个：一次处理
   - 10-50 个：每批 5 个，间隔 1 小时
   - 50-200 个：每批 10 个，间隔 2 小时
   - > 200 个：每批 20 个，间隔 4 小时

## 第三步：交互配置

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

## 第四步：写入配置

1. 将用户提供的完整路径和选择写入 `references/memory-weave-config.md`
2. 更新 `processed-sessions.json` 的 `last_run` 为当前时间

## 第五步：Cron 设置

根据问题 1 的回答：
- 选择"是"：创建 isolated agentTurn cron job，schedule `0 5 * * *`，tz 为检测到的系统时区，delivery 设为 announce（执行完成后自动发摘要到当前 channel）
- 选择"否"：不创建 cron job

## 第六步：执行首次分批处理（仅限初始化阶段）

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

## 初始化输出格式

```
✅ Memory Weave 初始化完成

- session 文件目录：<sessions_dir>
- Dream 输出目录：<dream_dir>
- Review 文件目录：<memory_weave_dir>
- 工作区根目录：<workspace_root>
- 待处理 session：N 个，分 M 批处理
- Cron：<已创建（每日 5:00）/ 未创建>

下一步：直接说「做 session 回顾」即可开始首次回顾
```

## 错误处理

| 错误 | 处理方式 |
|------|---------|
| session 文件目录不存在 | 提示用户确认路径是否正确，提供手动指定路径选项 |
| processed-sessions.json 无法写入 | 提示权限问题，建议检查目录权限 |
| Cron 创建失败 | 输出失败原因，提示可稍后手动运行 `/openclaw cron add` |

---

# 附录：五层输出格式模板

## L1 — 会话轮廓

```markdown
# Memory Weave: YYYY-MM-DD

## 综述

[一段文字，描述当日所有 session 的整体情况：话题数、涉及哪些核心议题、整体投入度、是否有明确结论]

## 本日处理 Session

- session_id_1（话题简述）
- session_id_2（话题简述）
...

## 状态

<!-- processed: true | reviewed_at: YYYY-MM-DD | sessions: [session_id_1, session_id_2, ...] -->
```

---

## L2 — 事实积累

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

### 实际写入

> 以下内容已写入 USER.md：

- [已写入→USER] ……

> 以下内容已写入 MEMORY.md：

- [已写入→MEMORY] ……
```

**内容分类写入规则：**

| 内容类型 | 写入目标 |
|---------|---------|
| 偏好与习惯 | `USER.md` |
| 关注的领域（含深度标注） | `USER.md` |
| 项目与上下文 | `MEMORY.md` |
| 踩过的坑 | `MEMORY.md` |
| 知识缺口 | `MEMORY.md` |

---

## L3 — 未完成事项

```markdown
## 第三层：未完成事项

### 候选识别（待确认）

[按保守策略筛选出的候选条目，每条注明对话依据和来源 session]

### 去重结果

> 已与 TODO.md 已有内容对比：

- **[写入]** ……（符合"明确延后意图"且话题全新）| 来源：session_id
- **[跳过]** ……（已在 TODO.md 中，话题重复）
- **[跳过]** ……（证据不足，不满足"明确延后意图"）

### 实际写入

> 以下内容已写入 TODO.md：

- [已写入] [ ] **……**
  - …… | 来源 session | 创建于
```

---

## L4 — 协作质量

```markdown
## 协作质量

- **整体评价**: 顺利 / 有摩擦 / 一般
- **顺畅之处**: （如有）
- **摩擦点/教训**: （如有）
- **后续行动建议**: （如有）
```

---

## L5 — 认知偏差

```markdown
## 第五层：认知偏差

### 原始提取（待去重）

[从本次 session 中识别出的教训条目，每条注明来源 session]

### 去重结果

> 已与 MEMORY.md 中"我踩过的坑"章节对比：

- **[新增]** …… | 来源：session_id
- **[合并至]** …… → 更新为 …… | 来源：session_id
- **[跳过]** 与已有条目重复（……）

### 实际写入

> 以下内容已写入 MEMORY.md：

- [已写入] **……** | 来源：session_id
```

---

## Dream 文件格式

```markdown
# 梦境：YYYY-MM-DD

## 基本信息

- **日期**：YYYY-MM-DD
- **Seed**：hash 值（前 16 位）
- **Seed 来源**：`<memory_weave_dir>/YYYY-MM-DD.md`

---

[梦境正文 600-800 字]
```

---

## Dream 文学质量要求（硬性约束）

**L1 - 意象的物理质地**：每个主要意象至少有一个具体的身体/感官细节。

**L2 - 情绪身体化**：禁止"我感到X"句式，情绪通过物理动作和身体感知传达。

**L3 - 语言节奏**：句子长短交替，情绪激烈处用短句，弥散处用长句。

**L4 - 留白**：梦里的人和物永远不要叫出名字，名称说出的瞬间解读空间关闭。

**L5 - 情绪关系而非逻辑关系**：物品之间的关系像情绪一样跳跃，不靠逻辑推演。

**L6 - 语言解构**：文字可以剥落、倒写、变形、变成声音，打破物理定律。

**L7 - 禁止词汇**（禁止直接出现在梦境正文）：

```
文件类：文件、目录、路径、存储、写入、读取
系统类：session、memory、TODO、AGENTS、IDENTITY、SOUL、USER、HEARTBEAT
AI概念：token、model、prompt、生成、模型、上下文、system、agent
抽象判断：这是……的隐喻、代表着、意思是、象征着
```

## Dream 认知坍缩要求

每篇梦境**必须包含**以下至少一种，打破物理/逻辑定律：

- **空间错位**：推开门发现自己站在完全不同的地点
- **身份混淆**：镜子里映出的是咖啡，或陌生人的面孔
- **时间坍缩**：房间里的陈设是"未来的自己"准备的
- **语言解构**：文字剥落、变成十六进制、倒写
