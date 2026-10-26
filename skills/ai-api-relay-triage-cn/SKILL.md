---
name: ai-api-relay-triage-cn
description: 排查大模型 API 调用失败——中转/代理平台的 503、渠道分组、模型映射、API Key 权限、额度与限流，以及 Ollama 本地模型的配置问题。当用户遇到"No available channel"、401/403/429/503、模型不存在、调用超时、本地模型跑不起来时使用。
---

# 大模型 API / 中转平台故障排查

## 何时用

- `No available channel for model XXX under group YYY`
- `401 Unauthorized` / `403 Forbidden` / `429 Too Many Requests` / `503 Service Unavailable`
- 模型名"不存在"或"不支持"
- 中转平台调用成功但返回内容异常
- Ollama 本地模型下载失败、加载慢、显存不足
- 免费 API Key 申请后不可用

## 核心认知：错误来自哪一层

调用链有四层，**先定位层级，再谈修复**：

```
你的客户端 → [1] 中转/网关平台 → [2] 上游供应商 → [3] 模型服务
```

| 错误特征 | 层级 | 说明 |
|---|---|---|
| 返回体含 `channel`、`group`、`distributor`、`request id` | [1] 中转平台自己拦的 | 不是上游问题，是**平台内的路由配置**问题 |
| 返回体是 OpenAI 标准格式 `{"error":{"message":...}}` | [2] 上游返回 | 看 message 里的具体原因 |
| 连接超时、TLS 错误、DNS 失败 | 链路层 | 网络问题，不是 API 问题 |
| HTTP 200 但内容是空/乱码 | 客户端解析 | 检查 stream 模式与响应格式 |

## 中转平台 503 详解（最常见）

典型报错：
```
No available channel for model gemini-3.1-flash-image
under group 国产模型-余额 (distributor)
```

逐句解码：
- `No available channel for model X` → 平台有「模型名 → 渠道」映射表，拿 X 去渠道里找，**没找到声明支持 X 的渠道**
- `under group Y` → 你把这个模型**挂在 Y 这个渠道分组下**
- `(distributor)` → 分发器模式，下游所有渠道都不支持该模型
- `503` → **平台自己拦的**，不是上游返回

**根因**：模型名与渠道分组不匹配。比如把国外模型（gemini / grok）挂在"国产模型"分组下，该分组里的供应商自然没有这个模型。

**修复**：
1. 打开平台后台的「渠道 / 分组」配置
2. 确认该模型名**属于哪个分组**，把请求指向正确分组
3. 或在该分组下添加一个真正支持该模型的上游
4. 用平台提供的「模型列表」接口核对准确模型名（大小写、版本后缀 `-preview`、`-latest` 都要一致）

## 状态码速查

| 码 | 含义 | 排查方向 |
|---|---|---|
| 400 | 参数错误 | 请求体字段名、模型名拼写、messages 格式 |
| 401 | 认证失败 | Key 是否正确、是否带了 `Bearer ` 前缀、Key 是否过期 |
| 403 | 无权限 | Key 的权限范围、是否绑定 IP/域名、分组未授权 |
| 404 | 路径或模型不存在 | endpoint 路径、模型名 |
| 429 | 限流/额度耗尽 | 检查额度、并发数、是否被风控 |
| 500 | 上游内部错误 | 换模型或重试 |
| 502/504 | 网关超时 | 上游不可达或响应太慢 |
| 503 | 服务不可用 | **优先看是不是中转平台在拦** |

## 快速验证脚本（Node）

```javascript
const url = 'https://<你的中转域名>/v1/chat/completions';
const res = await fetch(url, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer <KEY>',
  },
  body: JSON.stringify({
    model: '<精确模型名>',
    messages: [{ role: 'user', content: 'ping' }],
    max_tokens: 8,
  }),
});
console.log(res.status);
console.log(await res.text());   // 一定要看原始返回体
```

**关键**：失败时**必须打印原始响应体**。很多"玄学错误"的答案就写在里面（含 request id，可拿去找平台客服）。

## 先查模型列表

```javascript
const r = await fetch('https://<域名>/v1/models', {
  headers: { Authorization: 'Bearer <KEY>' },
});
const j = await r.json();
console.log(j.data.map((m) => m.id).join('\n'));
```

用返回的 **id 原样**去调用，不要自己拼模型名。

## Ollama 本地模型

```powershell
# 是否在跑
ollama list
ollama ps

# 拉模型（注意标签）
ollama pull qwen2.5:7b

# 看服务日志
# Windows 下 ollama 日志通常在 %LOCALAPPDATA%\Ollama\server.log
```

常见问题：
| 现象 | 原因 | 处理 |
|---|---|---|
| 下载卡住/极慢 | 网络到 registry.ollama.ai 不通 | 配置代理或用镜像 |
| 加载后立即退出 | 显存/内存不足 | 换更小的量化版本（`q4_K_M`） |
| 响应极慢 | 模型跑在 CPU 上 | 确认 GPU 驱动与 CUDA/ROCm 支持 |
| `model not found` | 标签写错 | `ollama list` 看准确名称 |

**显存估算**：参数量 × 量化位宽 ÷ 8 ≈ 显存需求（GB），再留 20% 余量。
例：7B 模型 q4 量化 ≈ 7×4/8 = 3.5GB，实际留 5GB 左右。

## 申请免费 API Key 的注意点

- 优先选**官方渠道**（各厂商控制台的 free tier），而非第三方"免费 Key 分享"
- 第三方分享的 Key 通常：**有并发限制、随时失效、可能被滥用封禁、有隐私风险**
- 免费额度一般有**有效期**和**速率限制**，429 是常态
- 不要把 Key 提交进 Git 仓库；用环境变量或 `.env`（并加入 `.gitignore`）

## 安全提醒

- Key 只放环境变量或本地配置文件，**不要贴进对话、不要提交仓库**
- 排查时如果必须展示 Key，只显示前 4 位 + 后 4 位
- 第三方中转平台会看到你的全部请求内容，**不要传敏感数据**

## 输出格式

```markdown
**定位**：错误来自 <哪一层>，依据是 <响应体里的关键字>

**验证**：
```javascript
<最小复现代码>
```

**修复**：<具体改哪一项配置>
```

## 反模式

- ❌ 看到 503 就说是"服务器崩了"——中转平台 503 多半是配置问题
- ❌ 反复换 Key 而不看响应体
- ❌ 自己拼模型名（`gpt-4` vs `gpt-4-turbo` vs `gpt-4-0613` 完全不同）
- ❌ 让用户把 Key 贴出来"帮你看看"
