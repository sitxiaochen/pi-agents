---
name: network-triage-cn
description: 中文环境网络故障分级排查——DNS污染、IP跳转域名、端口不通、SSL证书异常。用户说打不开、跳转、解析不对、连不上时使用。
tags: ["自习"]
---

# 中文环境网络故障分级排查

按层排查网络故障：本机 → DNS → hosts → 端口 → TLS → 代理。每层给出可执行命令和判据，避免无效的重装网卡。

**不做什么**：不绕过任何网络管控，不提供规避地区限制的方法，不代用户执行破坏性网络重置。

## When to Use

**触发场景**：
- 网站打不开、时好时坏、只有某个站点异常
- 用 IP 访问被跳转到域名，且域名不对
- DNS 解析结果与预期不符、怀疑 DNS 污染
- 端口不通、内网服务无法访问
- 证书报错：`NET::ERR_CERT_*`、`SSL_ERROR_*`、`certificate has expired`
- `ERR_CONNECTION_TIMED_OUT` / `ERR_NAME_NOT_RESOLVED`

**不要用**：纯应用层报错（接口 401/500）、需要抓包做协议逆向、性能压测。

## Procedure

**原则：从本机 → 出口 → 目标逐层排除，每步可证伪，不跳步。**

### 1. 本机网络栈

```powershell
ipconfig /all
route print -4
```

判据：DNS 是路由器 IP 还是公共 DNS；是否有多个虚拟网卡（VMware/VirtualBox/WSL/VPN）抢路由。**VPN 虚拟网卡是"部分站点不通"的常见真凶**。

### 2. DNS 解析

```powershell
nslookup example.com
nslookup example.com 223.5.5.5
nslookup example.com 119.29.29.29
Resolve-DnsName example.com -Type A
```

| 现象 | 结论 |
|---|---|
| 系统 DNS 与公共 DNS 结果不同 | 本地 DNS 有问题或被劫持 |
| 两个公共 DNS 都不一致且可疑 | DNS 污染或 CDN 就近 |
| 解析正常但访问失败 | 问题在 TCP/TLS 层，继续下一步 |
| 解析超时 | 检查 DNS 服务器可达性、53 端口 |

### 3. hosts 与本地干预

```powershell
type C:\Windows\System32\drivers\etc\hosts
Get-Content C:\Windows\System32\drivers\etc\hosts | Select-String -NotMatch "^#"
```

常见坑：本地开发时配的 hosts 忘了删，线上域名被指向 127.0.0.1。

### 4. TCP 连通与端口

```powershell
ping example.com
Test-NetConnection -ComputerName example.com -Port 443 -InformationLevel Detailed
tracert -d example.com
```

注意：很多服务器禁 ICMP，**ping 不通不等于网络不通**，以端口测试为准。

### 5. TLS / 证书

```powershell
curl.exe -vI https://example.com
wsl openssl s_client -connect example.com:443 -servername example.com -showcerts
```

| 报错 | 原因 |
|---|---|
| `certificate has expired` | 服务端证书过期，客户端无解 |
| `unable to get local issuer certificate` | 中间证书缺失或本机根证书库问题 |
| `subject name does not match` | SNI/域名不匹配，常见于共用 IP 虚拟主机 |

### 6. 代理与系统设置

```powershell
netsh winhttp show proxy
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" | Select-Object ProxyEnable, ProxyServer, ProxyOverride
echo $env:HTTP_PROXY; echo $env:HTTPS_PROXY; echo $env:NO_PROXY
```

**关键陷阱**：`NO_PROXY` 不含 `127.0.0.1` 时，本地服务（含 CDP 调试端口）会走代理而失败。

### IP 访问跳转到错误域名

按序判断：
1. 服务端重定向：`curl -I http://<IP>` 看 `301/302` 的 `Location`
2. HSTS 缓存：清 `chrome://net-internals/#hsts`
3. CDN/反代默认站点命中
4. 恶意劫持：跳转到陌生域名且带参数，查 DNS 与 hosts
5. 路由器 DNS 被篡改：登录光猫/路由管理页确认

## Pitfalls

- **`netsh winsock reset` 不是首选**：破坏性且常无效，会重置所有网络配置
- **把 ping 不通当成网络不通**：多数公网服务器禁 ICMP，应测端口
- **不做对比解析就断言 DNS 污染**：必须先有多种 DNS 的结果对照
- **给用户大段 TCP/IP 原理**：用户要在 16 字问题后拿到可执行命令，不是教材
- **忽略虚拟网卡**：装了 VPN/虚拟机后路由表变了，是"只有某些站点不通"的高频原因

## Verification

修复后跑一次闭环验证：

```powershell
Test-NetConnection -ComputerName <目标域名> -Port 443 -InformationLevel Detailed
curl.exe -sS -o NUL -w "%{http_code}`n" https://<目标域名>
```

判定：`TcpTestSucceeded : True` 且 HTTP 状态码为 `200`/`301`/`302` 即可确认链路恢复；浏览器侧再刷一次确保证书无告警。
