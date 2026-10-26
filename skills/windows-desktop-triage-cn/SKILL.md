---
name: windows-desktop-triage-cn
description: Windows 10/11 桌面故障排查——蓝屏、开机异常、磁盘占用、驱动与系统文件修复。电脑卡、蓝屏、开不了机、功能失效时使用。
tags: ["自习"]
---

# Windows 桌面故障排查与运维

先取证再动手，修复按影响从小到大：设置 → 服务 → 命令修复 → 卸载更新 → 系统还原 → 重装。

**不做什么**：不推荐"一键优化大师"类第三方工具，不建议关闭 Defender/防火墙作为解决方案，不把重装系统当第一方案。

## When to Use

**触发场景**：
- 蓝屏（BSOD）、死机、自动重启
- 开机进不去系统、无限修复循环
- 磁盘占用高、电脑变慢
- 设备管理器黄色感叹号、驱动异常
- Windows 功能失效（搜索、输入法、网络图标、设置打不开）
- 需要 CMD / PowerShell 命令的确切写法

**不要用**：纯软件使用问题（Office 怎么用）、硬件采购选型、Linux/信创系统问题。

## Procedure

### 1. 蓝屏取证

```powershell
Get-ChildItem C:\Windows\Minidump -ErrorAction SilentlyContinue
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2; StartTime=(Get-Date).AddDays(-7)} |
  Select-Object TimeCreated, Id, ProviderName, Message -First 30
```

| 错误码 | 常见原因 |
|---|---|
| `IRQL_NOT_LESS_OR_EQUAL` | 驱动（显卡/网卡/杀软） |
| `MEMORY_MANAGEMENT` | 内存条故障或超频不稳 |
| `PAGE_FAULT_IN_NONPAGED_AREA` | 内存或驱动 |
| `WHEA_UNCORRECTABLE_ERROR` | 硬件级错误（CPU/内存/供电） |
| `CRITICAL_PROCESS_DIED` | 系统文件损坏 |
| `DPC_WATCHDOG_VIOLATION` | 存储驱动/SSD 固件 |

### 2. 系统文件修复（顺序不可颠倒）

```cmd
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow
```

先 `DISM` 修组件存储，再 `sfc` 修系统文件。反过来先 `sfc` 常因源损坏而失败。

### 3. 进入恢复环境

```powershell
shutdown /r /o /t 0
```

恢复环境内顺序：启动修复 → 系统还原 → 卸载最近更新 → 安全模式判断第三方软件 → 命令行修复 → 最后才考虑重装（**先备份数据**）。

```cmd
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
```

### 4. 磁盘占用高

```powershell
Get-Process | Sort-Object WS -Descending | Select-Object -First 15 Name, WS, CPU
Get-PSDrive -PSProvider FileSystem | Select-Object Name,
  @{n='UsedGB';e={[math]::Round($_.Used/1GB,1)}},
  @{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}
```

常见元凶：`C:\Windows\SoftwareDistribution\Download`、`hiberfil.sys`、还原点、`Windows.old`、浏览器缓存。

### 5. 服务与启动项

```powershell
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location, User
Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' } | Select-Object Name, DisplayName, Status
Get-ScheduledTask | Where-Object { $_.State -eq 'Ready' } | Select-Object TaskName, TaskPath -First 30
```

### 常用命令

```cmd
ipconfig /all
netstat -ano | findstr :80
tasklist /svc
chkdsk C: /f /r
powercfg /batteryreport
```

```powershell
Get-ComputerInfo
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, HealthStatus, Size
Get-AppxPackage | Where-Object Status -ne 'Ok'
```

## Pitfalls

- **CMD 与 PowerShell 语法混用**：`findstr` vs `Select-String`、`%VAR%` vs `$env:VAR`，必须标注运行环境
- **先 `sfc` 后 `DISM`**：顺序错了会因组件损坏而失败，看似"修不好"
- **把重装当第一方案**：先卸载更新/系统还原往往就能解决
- **`chkdsk /r` 耗时极长**：大容量磁盘可能数小时，要提前告知
- **`diskpart clean` 会清空分区表**：执行前必须确认盘符和目标磁盘

## Verification

修复完成后验证：

```powershell
sfc /scannow
```

判定：输出包含 `Windows 资源保护未找到任何完整性冲突` 即确认系统文件完好；蓝屏问题则需持续观察 `Get-WinEvent` 中 7 天内无新增 `Level=1` 事件。
