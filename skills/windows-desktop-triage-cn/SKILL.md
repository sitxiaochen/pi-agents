---
name: windows-desktop-triage-cn
description: Windows 10/11 桌面故障排查与运维——蓝屏、开机异常、驱动问题、磁盘占用、CMD/PowerShell 命令、服务与启动项、系统修复。当用户描述"电脑卡/蓝屏/开不了机/某个功能失效/命令怎么用"时使用。强调先只读检查、再动手修复，避免一上来就重装。
---

# Windows 桌面故障排查与运维

## 何时用

- 蓝屏 / 死机 / 自动重启
- 开机进不去系统、无限修复循环
- 磁盘占用高、电脑变慢
- 驱动异常、设备管理器有黄色感叹号
- 某个 Windows 功能失效（搜索、输入法、网络图标、设置打不开）
- 需要 CMD / PowerShell 命令的具体写法
- 系统更新后出现异常

## 通用原则

1. **先取证，再动手**：任何修复前先收集现象（错误码、事件日志、发生时间点）
2. **修复按影响从小到大**：设置 → 服务 → 命令修复 → 卸载更新 → 系统还原 → 重装
3. **重装是最后手段**，不是第一建议
4. **明确标注命令运行环境**：CMD 还是 PowerShell，两者语法不通用

## 蓝屏（BSOD）

```powershell
# 1. 看是否有 dump 文件
Get-ChildItem C:\Windows\Minidump -ErrorAction SilentlyContinue

# 2. 最近的关键错误事件
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2; StartTime=(Get-Date).AddDays(-7)} |
  Select-Object TimeCreated, Id, ProviderName, Message -First 30
```

**按错误码定位：**

| 错误码 | 常见原因 |
|---|---|
| `IRQL_NOT_LESS_OR_EQUAL` | 驱动问题（多为显卡/网卡/杀软） |
| `MEMORY_MANAGEMENT` | 内存条故障或内存超频不稳 |
| `PAGE_FAULT_IN_NONPAGED_AREA` | 内存或驱动 |
| `SYSTEM_SERVICE_EXCEPTION` | 驱动冲突 |
| `WHEA_UNCORRECTABLE_ERROR` | 硬件级错误（CPU/内存/供电） |
| `CRITICAL_PROCESS_DIED` | 系统文件损坏或关键服务被杀 |
| `DPC_WATCHDOG_VIOLATION` | 存储驱动/SSD 固件 |

**分析 dump（需 WSL 或安装 WinDbg）**：优先看 `Probably caused by` 一行，通常指向某个 `.sys` 驱动。

## 开机异常 / 进不去系统

```powershell
# 强制进恢复环境的入口（在能进系统时预置）
shutdown /r /o /t 0
```

进入恢复环境后的顺序：
1. **启动修复**（最轻）
2. **系统还原**（有还原点才可用）
3. **卸载最近的更新**（更新后出问题首选）
4. **安全模式**：判断是否第三方软件/驱动导致
5. **命令提示符修复**：
```cmd
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
sfc /scannow
```
6. 最后才是重装（**先备份数据**——用 PE 启动盘挂载磁盘）

## 系统文件与组件修复

**标准顺序（不要颠倒）：**

```cmd
:: 1. 先修映像（需要联网）
DISM /Online /Cleanup-Image /RestoreHealth

:: 2. 再修系统文件
sfc /scannow
```

说明：`DISM` 修的是组件存储，`sfc` 修的是系统文件。反过来先跑 `sfc` 常因源损坏而失败。

## 磁盘占用高

```powershell
# 找出占用最高的进程（按磁盘 IO）
Get-Process | Sort-Object -Property WS -Descending | Select-Object -First 15 Name, WS, CPU

# 各盘剩余空间
Get-PSDrive -PSProvider FileSystem | Select-Object Name, @{n='UsedGB';e={[math]::Round($_.Used/1GB,1)}}, @{n='FreeGB';e={[math]::Round($_.Free/1GB,1)}}

# 找出大文件夹（Top 20）
Get-ChildItem C:\ -Directory -ErrorAction SilentlyContinue | ForEach-Object {
  $s = (Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
  [pscustomobject]@{Path=$_.FullName; GB=[math]::Round($s/1GB,2)}
} | Sort-Object GB -Descending | Select-Object -First 20
```

**常见元凶**：Windows Update 缓存（`C:\Windows\SoftwareDistribution\Download`）、休眠文件（`hiberfil.sys`）、还原点、`Windows.old`、日志、浏览器缓存。

## 服务与启动项

```powershell
# 自启动项
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location, User

# 异常停止的服务
Get-Service | Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' } |
  Select-Object Name, DisplayName, Status

# 计划任务（恶意软件常驻点）
Get-ScheduledTask | Where-Object { $_.State -eq 'Ready' } | Select-Object TaskName, TaskPath -First 30
```

## 常用命令速查

```cmd
:: CMD 环境
ipconfig /all              :: 网络配置
netstat -ano | findstr :80 :: 端口占用
tasklist /svc              :: 进程与服务
chkdsk C: /f /r            :: 磁盘检查（需重启）
powercfg /batteryreport    :: 电池报告
```

```powershell
# PowerShell 环境
Get-ComputerInfo
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, HealthStatus, Size
Get-WindowsOptionalFeature -Online -FeatureName *Hyper*
Get-AppxPackage | Where-Object Status -ne 'Ok'
```

## 安全刹车

执行前必须说明影响的命令：

```cmd
rd /s /q <目录>          :: 递归删除，不可恢复
del /f /s /q <文件>       :: 强制删除
format <盘符>             :: 格式化
diskpart → clean          :: 清空整块磁盘分区表
bcdedit /delete           :: 删除启动项，可能导致无法开机
```

格式：**会做什么 → 影响范围 → 怎么回滚 → 确认继续**

## 输出格式

```markdown
**判断**：<最可能原因>

**先做这个（只读检查）**：
```powershell
<命令>
```
预期：...

**如果是 <X>，再执行**：
```powershell
<修复命令>
```
> ⚠️ 风险：... / 回滚：...
```

## 反模式

- ❌ 一上来建议"重装系统"
- ❌ 直接让用户跑 `diskpart clean`
- ❌ 给 PowerShell 语法却在 CMD 里跑（或反之）
- ❌ 建议关闭 Defender/防火墙作为"解决方案"
- ❌ 让用户装来路不明的"系统优化大师"
