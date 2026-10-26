---
name: network-triage-cn
description: 中文环境下的网络故障分级排查——DNS 污染、IP/域名跳转、代理与端口不通、SSL 证书异常、内网访问异常。当用户描述"打不开/跳转/解析不对/连不上/证书报错/时好时坏"时使用。按固定顺序给出可执行验证命令，避免无效的反复重装。
---

# 中文环境网络故障分级排查

## 何时用

用户描述以下任一现象时加载本技能：

- 网站打不开、时好时坏、只有某个站点不行
- 用 IP 访问被跳转到域名，且域名不对
- DNS 解析结果和预期不符、怀疑 DNS 污染
- 代理/梯子配置后部分站点异常
- 端口不通、内网服务无法访问
- SSL/证书报错（`NET::ERR_CERT_*`、`SSL_ERROR_*`）
- 浏览器报 `ERR_CONNECTION_TIMED_OUT` / `ERR_NAME_NOT_RESOLVED`

## 排查顺序（严格遵守）

**原则：从本机 → 出口 → 目标逐层排除，每步可证伪。不要跳步。**

### 第 1 层：本机网络栈

```powershell
# 本机 IP、网关、DNS 服务器
ipconfig /all

# 路由表（看默认网关是否正确）
route print -4
```

判断：
- DNS 服务器是路由器 IP 还是公共 DNS（223.5.5.5 / 119.29.29.29）
- 是否有多个网卡/虚拟网卡（VMware、VirtualBox、WSL、VPN 虚拟适配器）抢路由
- **VPN/代理软件的虚拟网卡经常是"部分站点不通"的真凶**

### 第 2 层：DNS 解析

```powershell
# 用系统 DNS 解析
nslookup example.com

# 指定公共 DNS 对比（绕过本地 DNS）
nslookup example.com 223.5.5.5
nslookup example.com 119.29.29.29

# PowerShell 更清晰
Resolve-DnsName example.com -Type A
```

判据：
| 现象 | 结论 |
|---|---|
| 系统 DNS 与公共 DNS 结果不同 | 本地 DNS 服务器有问题，或被劫持 |
| 两个公共 DNS 结果也不同，且返回可疑 IP | 可能 DNS 污染或 CDN 就近 |
| 解析正常但访问失败 | 问题在 TCP/TLS 层，往下走 |
| 解析超时 | 检查 DNS 服务器可达性、53 端口是否被封 |

**DNS 污染特征**：返回的 IP 属于已知的"黑洞"或境外无关机房；不同 DNS 返回不一致且都不可用；某些域名固定被干扰。

### 第 3 层：hosts 与本地干预

```powershell
# 查看 hosts 是否被改
type C:\Windows\System32\drivers\etc\hosts

# 检查是否有安全软件/加速器写入的记录
Get-Content C:\Windows\System32\drivers\etc\hosts | Select-String -NotMatch "^#"
```

**常见坑**：曾经配过 hosts 做本地开发，忘了删，导致线上域名指向 127.0.0.1。

### 第 4 层：TCP 连通与端口

```powershell
# ICMP（部分服务器禁 ping，不通不代表真不通）
ping example.com

# 端口级测试（最能说明问题）
Test-NetConnection -ComputerName example.com -Port 443 -InformationLevel Detailed

# 路径追踪
tracert -d example.com
```

### 第 5 层：TLS / 证书

```powershell
# Windows 原生
curl.exe -vI https://example.com

# 需要证书细节时走 WSL
wsl openssl s_client -connect example.com:443 -servername example.com -showcerts
```

判据：
- `certificate has expired` → 服务端证书过期，客户端无解，只能等对方修
- `unable to get local issuer certificate` → 中间证书缺失或本机根证书库问题
- `certificate subject name does not match` → SNI/域名不匹配，常见于共用 IP 的虚拟主机
- 证书正常但握手失败 → 检查 TLS 版本、是否被中间设备拦截

### 第 6 层：代理与系统设置

```powershell
# 系统代理
netsh winhttp show proxy
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" | Select-Object ProxyEnable, ProxyServer, ProxyOverride

# 环境变量代理（Node/curl 等会读）
echo $env:HTTP_PROXY; echo $env:HTTPS_PROXY; echo $env:NO_PROXY
```

**关键陷阱**：`NO_PROXY` 没包含 `127.0.0.1` 时，本地服务（含 CDP 调试端口）会走代理而失败。

## IP 访问被跳转到域名

用户说"用 IP 访问却跳到域名，而且域名不对"，按此顺序判断：

1. **服务端重定向**：`curl -I http://<IP>` 看返回 `301/302` 的 `Location` 头
2. **HSTS 强制 HTTPS**：浏览器缓存了 HSTS，之后强制走 HTTPS 并按 SNI 匹配——清 `chrome://net-internals/#hsts`
3. **CDN/反代默认站点**：IP 直接访问命中默认虚拟主机，配置了跳转
4. **恶意劫持**：跳转到陌生域名且带参数，检查 DNS 与 hosts
5. **路由器 DNS 劫持**：光猫/路由器被篡改，登录管理页确认 DNS 设置

## 输出格式

```markdown
**判断**：<最可能原因，一句话>

**先验证**：
```powershell
<命令>
```
- 预期：<正常结果>
- 若看到 <异常>：<说明什么>

**下一步**：<按结果分支>
```

## 反模式（不要做）

- ❌ 一上来就让用户"重装网卡驱动"或"重置网络"（`netsh winsock reset`）——破坏性且常无效
- ❌ 不做对比解析就断言 DNS 污染
- ❌ 把"ping 不通"直接当成"网络不通"（很多服务器禁 ICMP）
- ❌ 给用户看大段 TCP/IP 原理而不给命令
