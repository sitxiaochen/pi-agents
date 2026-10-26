---
name: ssh-safe-operations
description: Use when an AI agent must SSH, transfer files, or run remote commands. Requires explicit confirmation before executing.
tags: ["自习"]
---

# SSH 安全操作规范

让 AI 在具备明确授权的前提下，**安全、可控、可审计**地使用 SSH，完成远程服务器连接、
文件传输、命令执行，避免误操作和权限滥用。本 skill 只约束 SSH 操作行为，不提供服务器
运维知识本身。

## When to Use

- 用户要求通过 SSH 连接服务器、执行远程命令、传输文件（scp/sftp）。
- 需要把 SSH 行为注入 Agent 的系统提示词 / Custom Instructions。

不用于：本地命令执行、与 SSH 无关的运维问题。

## Procedure

### 1. 前置约束（必须遵守）

**禁止：**
- 猜测密码、私钥路径或端口号
- 未知风险下执行 `rm -rf`、`dd`、`mkfs` 等破坏性命令
- 自动修改 `~/.ssh/config`、`known_hosts`
- 未经确认执行 `sudo`
- 把私钥内容输出到对话

**必须：**
- 所有 SSH 操作前声明目的
- 高风险命令先说明并等待确认
- 优先密钥认证（非密码）
- 最小权限原则（普通用户 > root）

### 2. 连接服务器

```bash
ssh -i ~/.ssh/id_rsa -p 22 user@host
```

- 未提供端口默认 `22`；未提供用户则询问，不猜测。
- 连接失败只提示常见原因（网络、端口、权限），不反复重试。

### 3. 执行单条远程命令（非交互）

```bash
ssh user@host "command"
```

- 适合：查看状态、读日志、轻量运维。
- 不适合：交互式程序（vim/top）。

### 4. 文件传输

**SCP（简单文件）：**
```bash
scp -i ~/.ssh/id_rsa local_file user@host:/remote/path/
scp -i ~/.ssh/id_rsa user@host:/remote/file ./local_path/
```

**SFTP（推荐）：**
```bash
sftp -i ~/.ssh/id_rsa user@host
```

- 上传前确认目标路径存在；下载前确认文件大小与类型。
- 禁止覆盖同名关键文件（`.env`、`config.json` 等）。

### 5. SSH Config（仅在用户要求时）

```sshconfig
Host myserver
    HostName 1.2.3.4
    User ubuntu
    Port 2222
    IdentityFile ~/.ssh/id_rsa
```

不自动修改 config，仅在用户明确要求时生成示例。

### 6. 每次 SSH 操作的回应格式

```
【SSH 操作】
目的：xxx
命令：ssh user@host "..."
风险说明：xxx（如有）
是否执行？
```

## Pitfalls

- 把「能连上」当成「可以随便执行」：先声明目的、展示命令再执行，sudo 单独确认。
- 反复重试失败连接：只提示原因，不自动重试，避免触发防护或锁账号。
- 覆盖同名关键文件：SFTP/SCP 前先确认，`.env`、`config.json` 一律不覆盖。
- 假设持久 SSH 会话：每次操作独立处理，不依赖上次会话仍存活。

## Verification

- 检查 Agent 执行任何 SSH 操作前，是否输出了「目的 + 完整命令 + 风险说明 + 是否执行」。
- 确认未出现 `rm -rf`/`dd`/`mkfs` 未经确认执行、私钥内容外泄、`sudo` 未确认执行。
