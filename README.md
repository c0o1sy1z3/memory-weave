# Memory Weave

本 Skill 基于 OpenClaw 设计，主要是为了解决 Agent 失忆的问题。

目标：让 Agent 更懂你！

## 设计思路

每日定时对过去 24 小时完成的 session 记录文件进行回顾，从聊天记录中提取五层价值信息，去重后合并写入对应的设定文件，来确保记忆不被遗漏

完成cron之后会向用户推送几个关键信息：

1. L1-L5写入特定文件的内容

2. TODO当前待办内容，并提醒最近7天会到期的内容

被写入的目标文件列表如下：

• USER.md 对应 偏好/习惯/关注领域
• MEMORY.md 对应 项目/坑/缺口
• TODO.md 对应 待办事项 （此文件为本 Skill 新增）

最后以当日内容为种子生成梦境。

## 注意事项

• 本 Skill 不对已有文件做删除，但会对 MEMORY.md 等设定文件进行去重合并等操作，在使用之前请记得备份
• 本 Skill 在首次初始化时会尝试对指定 agent 的已有 session 记录文件进行全量处理，如果你的记录文件较多，记得先预估一下规模后分批处理，避免撑爆上下文或是处理时间过长导致失败

每日 session 回顾，从聊天记录中提取五层价值信息，并以当日内容为种子生成梦境。

## 功能概述

- **L1 会话轮廓**：文字综述当日所有 session 的整体情况
- **L2 事实积累**：分类写入 USER.md（偏好/习惯/关注领域）+ MEMORY.md（项目/坑/缺口）
- **L3 未完成事项**：提取并写入 TODO.md
- **L4 协作质量**：记录当日合作顺畅度
- **L5 认知偏差**：记录从当日 session 学到的教训，写入 MEMORY.md
- **Dream 模块**：以当日内容为种子生成 600-800 字梦境

## 安装

### 方式一：手动安装

```bash
# 将 skill 目录放入 OpenClaw skills 目录
cp -r memory-weave ~/.openclaw/skills/
```

### 方式二：AI Agent 自动安装

Agent 读取 SKILL.md 后，根据初始化流程（见下文）引导用户完成配置。

## 初始化

```bash
# 在对话中触发
memory-weave 初始化
# 或
/memory-weave init
```

初始化时需要提供以下路径（完整绝对路径）：

| 路径 | 说明 | 示例 |
|------|------|------|
| `sessions_dir` | OpenClaw session 文件目录 | `~/.openclaw/agents/athena/sessions/` |
| `dream_dir` | 梦境输出目录 | `~/Documents/Vault/Athena/Dream/` |
| `Memory Weave_dir` | L1/L4 review 文件输出目录 | `~/Documents/Temp/sessions/` |
| `workspace_root` | USER.md / MEMORY.md / TODO.md 所在目录 | `~/.openclaw/workspace-athena/` |

初始化流程会自动：
1. 扫描 session 文件，检测积压量并给出分批建议
2. 创建缺失的目录和文件
3. 写入 `references/memory-weave-config.md`
4. 可选：创建每日 5:00 的 Cron job

## 使用

### 手动触发

```
做 session 回顾
```

### Cron 自动执行

初始化后创建 Cron job，每日 5:00 AM 自动执行，完成后将摘要发回当前 channel。

## 配置文件

路径：`references/memory-weave-config.md`

```markdown
agent_id: <你的 agent id>
sessions_dir: <session 文件目录>
dream_dir: <梦境输出目录>
Memory Weave_dir: <review 文件输出目录>
workspace_root: <USER.md / MEMORY.md / TODO.md 所在目录>

cron_schedule: "0 5 * * *"
cron_timezone: Asia/Shanghai
cron_enabled: false

batch_size: 3
batch_interval_hours: 2
```

## 分批处理说明

- **仅初始化阶段**：消化历史积压时按批次处理，日常执行不看此参数
- **日常 Cron**：只看过去 24 小时新增 session，数量有限，一次跑完

## 梦境模块

- 以当日 review 内容为种子，通过 web search 抓取外部随机意象
- 读取最近 3 篇梦境生成排除列表，避免意象重复
- 文学质量要求：L1 物理质地、L2 情绪身体化、L3 语言节奏、L4 留白、L5 情绪关系、L6 语言解构
- 禁止词汇：文件类、系统类、AI 概念类词汇

## 状态追踪

处理记录保存在 `sessions_dir/processed-sessions.json`，幂等设计确保 session 不会被重复处理。
