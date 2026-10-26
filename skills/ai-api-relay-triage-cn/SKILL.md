---
name: ai-api-relay-triage-cn
description: 排查大模型 API 与中转平台故障——503渠道分组、401/403/429、模型名映射、Ollama本地模型。调用失败或报错时使用。
tags: ["自习"]
---

# 大模型 API / 中转平台故障排查

先定位错误来自哪一层（客户端 / 中转平台 / 上游 / 链路），再看状态码，最后改配置。

**不做什么**：不提供规避平台风控的方法，不帮助滥用免费额度，不索取或展示用户的完整 API Key。

## When to Use

**触发场景**：
- `No available channel for model XXX under group YYY`
- `401 Unauthorized` / `403 Forbidden` / `429 Too Many Requests` / `503`
- 模型名"不存在"或不支持
- 中转平台调用成功但内容异常
- Ollama 本地模型下载失败、加载慢、显存不足
- 免费 API Key 申请后不可用

**不要用**：纯业务代码逻辑错误、与模型服务无关的普通 HTTP 问题。

## Procedure

### 1. 定位层级

| 错误特征 | 层级 |
|---|---|
| 返回体含 `channel`、`group`、`distributor`、`request id` | 中转平台自己拦的 |
| OpenAI 标准格式 `{"error":{"message":...}}` | 上游返回 |
| 连接超时、TLS 错误、DNS 失败 | 链路层 |
| HTTP 200 但内容为空/乱码 | 客户端解析（查 stream 模式） |

### 2. 中转平台 503 解码

典型报错：
```
No available channel for model gemini-3.1-flash-image
under group 国产模型-余额 (distributor)
```

逐句含义：
- `No available channel for model X` → 平台「模型名 → 渠道」映射表里，**没有渠道声明支持 X**
- `under group Y` → 该请求被路由到 Y 分组
- `(distributor)` → 分发器模式下游全部不支持
- `503` → **平台拦截**，不是上游返回

**根因**：模型名与渠道分组不匹配。例如把国外模型挂在"国产模型"分组下。

**修复步骤**：
1. 打开平台后台「渠道 / 分组」配置
2. 确认该模型属于哪个分组，把请求指向正确分组
3. 或在该分组下添加真正支持该模型的上游
4. 用「模型列表」接口核对准确模型名（大小写、`-preview`、`-latest` 后缀须完全一致）

### 3. 状态码速查

| 码 | 含义 | 排查方向 |
|---|---|---|
| 400 | 参数错误 | 字段名、模型名拼写、messages 格式 |
| 401 | 认证失败 | Key 是否正确、`Bearer ` 前缀、是否过期 |
| 403 | 无权限 | Key 权限范围、IP/域名绑定、分组未授权 |
| 404 | 路径或模型不存在 | endpoint 路径、模型名 |
| 429 | 限流/额度耗尽 | 额度、并发数、风控 |
| 502/504 | 网关超时 | 上游不可达或响应过慢 |
| 503 | 服务不可用 | **优先看是否中转平台在拦** |

### 4. 最小验证脚本

```javascript
const res = await fetch('https://<中转域名>/v1/chat/completions', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', Authorization: 'Bearer <KEY>' },
  body: JSON.stringify({
    model: '<精确模型名>',
    messages: [{ role: 'user', content: 'ping' }],
    max_tokens: 8,
  }),
});
console.log(res.status);
console.log(await res.text());
```

**必须打印原始响应体**——答案通常就在里面，含 request id 可拿去找平台客服。

### 5. 先查模型列表

```javascript
const r = await fetch('https://<域名>/v1/models', { headers: { Authorization: 'Bearer <KEY>' } });
const j = await r.json();
console.log(j.data.map((m) => m.id).join('\n'));
```

用返回的 **id 原样**调用，不要自己拼模型名。

### 6. Ollama 本地模型

```powershell
ollama list
ollama ps
ollama pull qwen2.5:7b
```

| 现象 | 原因 | 处理 |
|---|---|---|
| 下载卡住 | 到 registry.ollama.ai 不通 | 配置代理或镜像 |
| 加载后立即退出 | 显存/内存不足 | 换更小量化版本（q4_K_M） |
| 响应极慢 | 跑在 CPU 上 | 确认 GPU 驱动/CUDA 支持 |
| `model not found` | 标签写错 | `ollama list` 看准确名称 |

**显存估算**：参数量 × 量化位宽 ÷ 8 ≈ GB，再留 20% 余量。例：7B q4 ≈ 7×4/8 = 3.5GB，实际留 5GB。

## Pitfalls

- **看到 503 就说是服务器崩了**：中转平台的 503 多半是路由配置问题
- **反复换 Key 却不看响应体**：浪费额度且没解决根因
- **自己拼模型名**：`gpt-4` / `gpt-4-turbo` / `gpt-4-0613` 是完全不同的模型
- **让用户把 Key 贴出来**：只能显示前 4 位 + 后 4 位
- **http 200 就以为成功**：需检查响应体是否为错误 JSON 或空内容

## Verification

用同一请求复测，判定：`res.status === 200` 且响应体含 `choices[0].message.content` 非空字符串，即确认该模型可用。
