---
name: browser-session-data-export
description: 用 CDP 复用浏览器已登录态，从网页应用内部接口批量导出数据（会话、订单、消息等）。当需要抓取需要登录才能看的页面数据、且应用有内部 JSON 接口时使用。包含接口发现、分页、断点续跑、渲染落盘的完整流程。已在腾讯元宝上验证可行。
---

# 登录态网页数据批量导出

## 何时用

- 要导出某个网页应用里的历史数据（聊天记录、订单、消息、列表）
- 数据需要登录才能看，且**没有官方导出功能**
- 应用是 SPA，页面内容由 JSON 接口驱动
- 用户愿意提供登录态（自己扫码登录，或已登录的 profile）

## 核心思路

**不要爬 DOM，直接调它自己的接口。**

SPA 的页面数据都来自 XHR/fetch。找到这些接口后，在页面上下文里调用（自动带上 Cookie 和平台自定义请求头），比解析 DOM 稳定得多。

## 完整流程

### 第 1 步：启动带远程调试的浏览器

```powershell
$profile = "$env:USERPROFILE\cdp-profile"
Start-Process "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  -ArgumentList "--remote-debugging-port=9333","--user-data-dir=$profile","--no-first-run","https://<目标站点>"
```

**关键点**：
- `--user-data-dir` 用**独立目录**，避免污染日常浏览器
- 这个目录会保存登录态，**后续可复用**，不用反复登录
- 验证端口：`Invoke-WebRequest http://127.0.0.1:9333/json/version`

**如果进程总被杀**（某些沙箱环境会清理子进程）：把启动和操作放在**同一次命令调用**里完成，或改用 `--headless=new`。

### 第 2 步：CDP 连接（Node 24 自带 WebSocket）

```javascript
const list = await (await fetch('http://127.0.0.1:9333/json/list')).json();
const page = list.find((p) => p.type === 'page' && p.url.includes('<域名>'));
const ws = new WebSocket(page.webSocketDebuggerUrl);

// 封装 send / eval
let id = 0;
const pending = new Map();
ws.onmessage = (ev) => {
  const m = JSON.parse(ev.data);
  if (m.id && pending.has(m.id)) {
    const { res, rej } = pending.get(m.id);
    pending.delete(m.id);
    m.error ? rej(new Error(JSON.stringify(m.error))) : res(m.result);
  }
};
await new Promise((res) => { ws.onopen = res; });

const send = (method, params = {}) => new Promise((res, rej) => {
  const i = ++id;
  pending.set(i, { res, rej });
  ws.send(JSON.stringify({ id: i, method, params }));
});

const evalJs = async (expression) => {
  const r = await send('Runtime.evaluate', {
    expression, awaitPromise: true, returnByValue: true, userGesture: true,
  });
  if (r.exceptionDetails) throw new Error(JSON.stringify(r.exceptionDetails).slice(0, 400));
  return r.result.value;
};
```

### 第 3 步：发现接口（最关键的一步）

用 `Network.enable` 监听，然后**触发一次页面重载**，抓取所有 API 请求：

```javascript
await send('Network.enable');
const calls = [];
ws.addEventListener('message', (ev) => {
  const m = JSON.parse(ev.data);
  if (m.method === 'Network.requestWillBeSent') {
    const r = m.params.request;
    if (/\/api\//.test(r.url)) {
      calls.push({ url: r.url, method: r.method, postData: r.postData });
    }
  }
});
await send('Page.reload');
await new Promise((r) => setTimeout(r, 9000));
```

**必须记录 `postData`**——很多接口是 POST，参数结构藏在请求体里。

补充手段：如果接口路径被混淆，下载 JS bundle 后搜 `/api/`：

```javascript
const scripts = await evalJs(`Array.from(document.querySelectorAll('script[src]')).map(s => s.src)`);
// 下载后用正则提取路径
```

### 第 4 步：在页面上下文调用接口

**不要用 Node 直接请求**——会缺 Cookie、缺平台自定义头（如 `X-AgentID`、`X-Device-ID`、签名头）。在页面里 fetch：

```javascript
const data = await evalJs(`(async () => {
  const r = await fetch('/api/xxx/list', {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'accept': 'application/json',
      'X-Requested-With': 'XMLHttpRequest',
    },
    credentials: 'include',
    body: JSON.stringify({ page: 1, size: 40 }),
  });
  return { status: r.status, body: await r.text() };
})()`);
```

**务必先打印 `status` 和原始 body**，确认结构后再写解析逻辑。

### 第 5 步：分页策略

常见两种分页：

| 类型 | 参数 | 特点 |
|---|---|---|
| offset/limit | `offset += batch.length` | 简单，但数据变动时会错位 |
| 游标 | `last_replied_at = last.lastRepliedAt` | 更稳，适合按时间倒序的列表 |

```javascript
let cursor = 0, all = [];
while (true) {
  const j = await post({ last_replied_at: cursor, page_size: 40 });
  const list = j.data.list || [];
  if (!list.length) break;
  all.push(...list);
  const next = list[list.length - 1].lastRepliedAt;
  if (next === cursor) break;      // 防死循环
  cursor = next;
  if (all.length >= j.data.total_count) break;
}
```

**注意**：列表接口常返回 `total_count`，用它做终止判据最可靠。

### 第 6 步：详情接口的方向问题

**默认返回可能是倒序**（最新在前）。取回来后要按 `index` 排序：

```javascript
msgs.sort((a, b) => (a.index ?? 0) - (b.index ?? 0));
```

**分页加载历史消息**时，`offset` 的含义要看接口：是"从头数第 N 条"还是"从最新往前第 N 条"。用一条已知长度的会话验证。

### 第 7 步：断点续跑

大批量任务必须缓存到磁盘，否则超时全部白跑：

```javascript
const cacheFile = path.join(CACHE, id + '.json');
if (fs.existsSync(cacheFile)) continue;       // 已抓过，跳过
// ...抓取...
fs.writeFileSync(cacheFile, JSON.stringify(msgs), 'utf8');
```

列表也缓存（`_list.json`），后续批次直接读。

**分批执行**：每次跑 40-60 个，循环调用直到 `REMAINING=0`。这比一次跑完可靠，且能看到进度。

### 第 8 步：识别字段（最容易踩坑）

文本内容不一定在 `content` 字段。**实际结构要靠 dump 确认**：

```javascript
// 打印一个样本的完整结构
const shape = msgs.map((m) => ({
  keys: Object.keys(m),
  speeches: (m.speechesV2 || []).map((sp) => ({
    keys: Object.keys(sp),
    blockTypes: Array.isArray(sp.content) ? sp.content.map(c => c.type) : typeof sp.content,
    sample: JSON.stringify(sp).slice(0, 400),
  })),
}));
```

实战中遇到过：
- 正文在 `block.msg`（不是 `content`）
- 用户消息直接放在 `m.speech`（不是数组）
- 引用来源在 `searchGuid` 块的 `docs[]` 里，不影响正文

**不要猜字段名，dump 出来看。**

### 第 9 步：渲染与校验

- 文件名清洗：`replace(/[\\/:*?"<>|\r\n\t]/g, '_')`，重名加序号
- 保留原始时间戳、ID，便于回溯
- 生成索引文件（标题 → 文件名 → 条目数）
- **抽样校验**：重新抓 2-3 条，与已导出内容比对条目数

## 常见问题

| 问题 | 原因 | 处理 |
|---|---|---|
| `ECONNREFUSED 127.0.0.1:PORT` | 浏览器已退出 | 检查进程；同一次调用内完成启动+操作 |
| `fetch failed`（Node 侧） | 环境变量代理拦截 localhost | 设 `NO_PROXY=127.0.0.1,localhost` |
| 接口返回 401 | 登录态失效 | 重新登录；确认用同一 `--user-data-dir` |
| 接口返回 405 | 方法错了（该用 POST） | 看抓到的 `requestWillBeSent.method` |
| 接口返回 400 且 body 为空 | 缺必需请求头 | 复制页面原始请求的完整 headers |
| 全量运行超时 | 一次做太多 | 改成断点续跑 + 分批 |

## 合规与安全边界

- 只导出**用户自己的数据**，不碰他人数据
- 不绕过验证码、风控、付费墙
- 不索取账号密码；登录由用户本人完成
- Cookie/profile 只留本地，不进仓库、不发外部
- 控制请求频率（每次间隔 150-350ms），遇限流退避
- 导出结果仅用于个人备份/分析

## 输出物建议

```
导出目录/
├── _index.md          索引（标题、文件、条目数、时间）
├── _report.md         报告（接口、总数、成功/失败、抽样校验）
├── 条目1.md
└── ...
```
