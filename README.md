# Memory Weave（通用版）

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
| `memory_weave_dir` | L1/L4 review 文件输出目录 | `~/Documents/Temp/sessions/` |
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
sessions_dir: <session 文件目录>
dream_dir: <梦境输出目录>
memory_weave_dir: <review 文件输出目录>
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

- 以当日 review 内容为种子，通过 web_search 抓取外部随机意象
- **降级规则**：若 web_search 失败（网络错误或无结果），自动跳过此步骤，纯以内部上下文生成 Dream；Dream 仍正常产出
- 读取最近 3 篇梦境生成排除列表，避免意象重复
- 文学质量要求：L1 物理质地、L2 情绪身体化、L3 语言节奏、L4 留白、L5 情绪关系、L6 语言解构
- 禁止词汇：文件类、系统类、AI 概念类词汇

## 状态追踪

处理记录保存在 `sessions_dir/processed-sessions.json`，幂等设计确保 session 不会被重复处理。
