---
name: browser-session-data-export
description: 用CDP复用浏览器登录态，调内部接口批量导出网页应用数据。需要登录才能看且无官方导出功能时使用。
tags: ["自习"]
---

# 登录态网页数据批量导出

通过 Chrome DevTools Protocol 复用已登录的浏览器 profile，找到网页应用的内部 JSON 接口并批量拉取数据，最终渲染落地。

**不做什么**：不绕过验证码/风控/付费墙，不导出他人数据，不索取账号密码，不把 Cookie 写进产物或仓库。

## When to Use

**触发场景**：
- 需要导出某网页应用的历史数据（会话、订单、消息、列表）
- 数据需登录可见，且**没有官方导出功能**
- 应用是 SPA，内容由 JSON 接口驱动
- 用户愿意自己完成登录

**不要用**：静态页面（直接抓 HTML 即可）、有官方 API 或导出功能的应用、需要逆向加密签名的接口。

## Procedure

### 1. 启动带调试端口的浏览器

```powershell
$profile = "$env:USERPROFILE\cdp-profile"
Start-Process "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  -ArgumentList "--remote-debugging-port=9333","--user-data-dir=$profile","--no-first-run","https://<目标站点>"
```

用**独立 user-data-dir**，登录态会持久保存，后续无需重复登录。

验证端口：
```powershell
Invoke-WebRequest http://127.0.0.1:9333/json/version
```

### 2. CDP 连接（Node 24 自带 WebSocket）

```javascript
const list = await (await fetch('http://127.0.0.1:9333/json/list')).json();
const page = list.find((p) => p.type === 'page' && p.url.includes('<域名>'));
const ws = new WebSocket(page.webSocketDebuggerUrl);

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
  const r = await send('Runtime.evaluate', { expression, awaitPromise: true, returnByValue: true, userGesture: true });
  if (r.exceptionDetails) throw new Error(JSON.stringify(r.exceptionDetails).slice(0, 400));
  return r.result.value;
};
```

### 3. 发现接口

```javascript
await send('Network.enable');
const calls = [];
ws.addEventListener('message', (ev) => {
  const m = JSON.parse(ev.data);
  if (m.method === 'Network.requestWillBeSent') {
    const r = m.params.request;
    if (/\/api\//.test(r.url)) calls.push({ url: r.url, method: r.method, postData: r.postData });
  }
});
await send('Page.reload');
await new Promise((r) => setTimeout(r, 9000));
```

**必须记录 `postData`**——多数接口是 POST，参数结构在请求体里。

### 4. 在页面上下文调用

**不要用 Node 直接请求**——会缺 Cookie 和平台自定义头。在页面里 fetch：

```javascript
const data = await evalJs(`(async () => {
  const r = await fetch('/api/xxx/list', {
    method: 'POST',
    headers: { 'content-type': 'application/json', accept: 'application/json', 'X-Requested-With': 'XMLHttpRequest' },
    credentials: 'include',
    body: JSON.stringify({ page: 1, size: 40 }),
  });
  return { status: r.status, body: await r.text() };
})()`);
```

先打印 `status` 和原始 body 确认结构，再写解析。

### 5. 分页

| 类型 | 参数 | 特点 |
|---|---|---|
| offset/limit | `offset += batch.length` | 简单，数据变动会错位 |
| 游标 | `cursor = last.lastRepliedAt` | 更稳，适合时间倒序 |

```javascript
let cursor = 0, all = [];
while (true) {
  const j = await post({ last_replied_at: cursor, page_size: 40 });
  const list = j.data.list || [];
  if (!list.length) break;
  all.push(...list);
  const next = list[list.length - 1].lastRepliedAt;
  if (next === cursor) break;
  cursor = next;
  if (all.length >= j.data.total_count) break;
}
```

### 6. 排序与字段识别

详情接口常**倒序返回**，需重排：

```javascript
msgs.sort((a, b) => (a.index ?? 0) - (b.index ?? 0));
```

**字段名必须 dump 确认，不能猜**：

```javascript
const shape = msgs.map((m) => ({
  keys: Object.keys(m),
  speeches: (m.speechesV2 || []).map((sp) => ({
    keys: Object.keys(sp),
    types: Array.isArray(sp.content) ? sp.content.map(c => c.type) : typeof sp.content,
    sample: JSON.stringify(sp).slice(0, 400),
  })),
}));
```

### 7. 断点续跑

```javascript
const cacheFile = path.join(CACHE, id + '.json');
if (fs.existsSync(cacheFile)) continue;
fs.writeFileSync(cacheFile, JSON.stringify(msgs), 'utf8');
```

列表也缓存（`_list.json`）。**每次跑 40-60 条，循环调用直到 `REMAINING=0`**。

### 8. 渲染与产物

```
导出目录/
├── _index.md     索引（标题、文件、条目数、时间）
├── _report.md    报告（接口、总数、成功/失败、抽样校验）
└── <条目>.md
```

文件名清洗：`replace(/[\\/:*?"<>|\r\n\t]/g, '_')`，重名加序号。

## Pitfalls

- **`ECONNREFUSED 127.0.0.1:PORT`**：浏览器已退出。须把启动和操作放在**同一次调用**内，或加 `--headless=new`
- **Node 侧 `fetch failed` 而浏览器正常**：环境变量代理拦截了 localhost，设 `NO_PROXY=127.0.0.1,localhost`
- **接口 405**：方法用错（该用 POST），回看抓到的 `method`
- **接口 400 且 body 为空**：缺必需请求头，复制页面原始请求的完整 headers
- **一次性全量跑超时**：必须断点续跑 + 分批，单批 40-60 条
- **用 DOM 文本判断登录态**：页面有"未登录"字样但实际已登录的情况存在，**以接口状态码为准**
- **字段名想当然**：正文可能在 `block.msg` 而非 `content`，用户消息可能在 `m.speech` 而非数组

## Verification

1. 重新抓 2-3 条会话，与已导出文件的条目数比对，须完全一致
2. 校验产物：`Get-ChildItem <导出目录> -Filter *.md | Measure-Object` 的文件数应等于 `_report.md` 中记录的成功数
3. `_report.md` 中失败数应为 0，否则逐条复查失败原因
