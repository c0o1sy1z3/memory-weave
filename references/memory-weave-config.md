# Memory Weave - 配置文件

> 运行 `memory-weave init` 时会自动生成。

## 基本信息

dream_dir: <梦境输出完整路径，例如 ~/Documents/Vault/Athena/Dream/>
memory_weave_dir: <Review 文件输出完整路径，例如 ~/Documents/Temp/sessions/>
workspace_root: <USER.md / MEMORY.md / TODO.md 所在目录，例如 ~/.openclaw/workspace-athena/>
sessions_dir: <session 文件目录，例如 ~/.openclaw/agents/athena/sessions/>

## Cron 配置

cron_schedule: "0 5 * * *"
cron_timezone: Asia/Shanghai
cron_enabled: false

## 分批处理（仅初始化阶段使用，日常 Cron 无需分批）

batch_size: <每批处理数量，默认 3>
batch_interval_hours: <批次间隔小时数，默认 2>

## 路径说明

- dream_dir: 梦境归档输出目录，每次生成 YYYY-MM-DD-梦境.md
- memory_weave_dir: L1/L4 review 文件临时存放目录，每次生成 YYYY-MM-DD.md
- workspace_root: USER.md / MEMORY.md / TODO.md 所在目录（绝对路径）
- sessions_dir: OpenClaw session 文件存储目录（绝对路径）
