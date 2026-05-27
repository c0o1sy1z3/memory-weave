# Session Review - 个性化配置文件

> 此文件由 `memory-weave init` 自动生成
> 初始化时由用户填入完整路径，手动修改前请先阅读 memory-weave-mechanism.md 的配置节

## 基本信息

agent_id: athena
dream_dir: ～/Documents/Docs/Personal/Athena/Dream/
memory_weave_dir: ～/Documents/Docs/Personal/Temp/sessions/
workspace_root: ～/.openclaw/workspace-athena/
sessions_dir: ～/.openclaw/agents/athena/sessions/

## Cron 配置

cron_schedule: "0 5 * * *"
cron_timezone: Asia/Shanghai
cron_enabled: true

## 分批处理（仅初始化阶段使用，日常 Cron 无需分批）

batch_size: 3
batch_interval_hours: 2

## 路径说明

- dream_dir: 梦境归档输出目录，每次生成 YYYY-MM-DD-梦境.md
- memory_weave_dir: L1/L4 review 文件临时存放目录，每次生成 YYYY-MM-DD.md
- workspace_root: USER.md / MEMORY.md / TODO.md 所在目录（绝对路径）
- sessions_dir: OpenClaw session 文件存储目录（绝对路径）
