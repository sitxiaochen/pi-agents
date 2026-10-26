---
name: pi-desktop-browser-cdp-session
description: Use when PI-Desktop must inspect or automate an authenticated browser page across tool calls.
tags: ["自习"]
---

# PI-Desktop Browser CDP Session

Use Chrome DevTools Protocol to inspect or automate a browser page in PI-Desktop. This workflow handles browser lifetime and CDP inspection; it does not bypass login, CAPTCHA, account security, or website access controls, and it does not export credentials or cookies as a substitute for user authentication.

## When to Use

Use when a user asks to inspect a logged-in browser page, export browser-session data, automate a web UI, or continue browser work after opening Yuanbao or another site. Do not use it for API-only retrieval, ordinary HTTP requests, or actions that require bypassing authentication.

## Procedure

1. Define the target URL and the exact data or action needed. Ask the user to complete login, CAPTCHA, or any account authorization directly; do not request passwords, tokens, or raw cookies unless the user explicitly chooses a secure credential-handling path.

2. Prefer a single PI `bash` call for browser launch, CDP discovery, WebSocket attachment, JavaScript evaluation, and cleanup. The observed cleanup behavior was that Chrome/Edge processes launched in one tool call could disappear before the next call, while a process kept alive inside the same call remained usable.

3. Launch Chrome with a disposable profile and remote debugging port. The observed successful pattern used:

   `--headless=new`

   `--remote-debugging-port=9337`

   `--user-data-dir=<profile>`

   `--no-first-run`

   `--no-default-browser-check`

   `--disable-gpu`

   `--window-size=1440,1200`

   `--lang=zh-CN`

   `https://yuanbao.tencent.com/chat/`

4. In the same call, poll `http://127.0.0.1:<port>/json/list` until a target page with `type: "page"` and a URL matching the site is returned. Then attach to its `webSocketDebuggerUrl`.

5. Send `Runtime.enable`, evaluate only the minimum page-state expression needed, and use `returnByValue: true`. For a UI-state check, inspect `location.href`, `document.title`, a bounded `document.body.innerText` sample, and a login-state indicator.

6. If visual confirmation is needed, call `Page.captureScreenshot` with `{ format: 'png' }`, decode the returned base64 data, and write it with PI `write` or Node `fs` inside the same tool call. Read the image from the workspace if multimodal inspection is required.

7. Keep authentication interaction bounded. If the page is logged out, report that state and ask the user to log in; do not synthesize successful session data from local-storage keys, anonymous identifiers, or page metadata. Do not rely only on the absence or presence of UI login labels. When available, verify authentication with a protected same-origin endpoint from inside the page; for Yuanbao, `/api/convs?page=1&pageSize=100` returned `status: 401` with error code `20000` while logged out, so a `401` response overrides a misleading UI-state inference.

8. For bulk Yuanbao conversation export, reuse the authenticated page context rather than copying cookies from the browser profile. From the Yuanbao origin, fetch `/api/user/agent/conversation/v3/list` with cursor pagination and `/api/user/agent/conversation/v1/detail` with `{ conversationId, offset, limit: 50, agentId }`. Continue while `hasMore` is true, each batch is nonempty, and a guard remains; persist each response before rendering.

9. Normalize conversation output before writing Markdown. Yuanbao detail responses may be newest-first, so sort messages by `index` ascending. User message text may be under `m.speech`, while assistant text may be under `block.msg`; accept both structures and preserve message roles and separators. Sanitize Windows-illegal filename characters and disambiguate duplicate titles with a numeric suffix. Create `_index.md` and `_report.md` alongside one Markdown file per conversation.

10. Verify the export in two layers. Require report counts for server total, enumerated conversations, written files, failures, empty conversations, and accumulated messages. Then re-fetch at least the first, a middle, and the largest conversation through the live authenticated API and compare message counts with the corresponding files; inspect the largest file for expected role counts and nonempty message bodies.

11. Close the WebSocket and terminate the browser after the requested inspection. Treat the export as successful only after the requested output exists, its aggregate counts agree, and the live sample comparison passes.

## Pitfalls

- A browser process may be alive during one tool call and gone before the next; an initial `HTTP 200` from `/json/version` is not persistent-session evidence. Keep the whole operation in one call.
- A headless browser can load a site while still being logged out. Text such as `未登录` or `登录` is evidence of unauthenticated state, not proof that session data is available. Conversely, the absence of `未登录` can be a false positive while a login overlay is loading; require a protected API response before declaring authentication successful.
- A visible browser launched from a long-running PI `bash` call may disappear when the tool times out. Before asking the user to scan or log in, confirm that the window is actually visible and remains alive; do not claim the user had a usable login window.
- A page containing local-storage keys or anonymous identifiers does not prove authentication or a valid user session. Do not export those as cookies or account credentials.
- A screenshot may be written outside the workspace, but a multimodal reader may reject it with `No workspace is open`; keep the file inside the workspace or use a workspace-relative path.
- Do not retry a browser launch indefinitely or encode process-killing behavior as a site failure. First confirm the target page and CDP response within one call.
- Do not claim that a session was exported when only a page screenshot or local-storage-key list was produced.
- Aggregate success counts alone do not prove body fidelity. Verify pagination completion, preserve chronological order, and compare live API message counts for representative conversations including the largest one. A detail API returning newest-first will produce reverse chronology unless messages are sorted by `index`.
- Reading cookies from a browser profile is unnecessary and creates a credential-handling risk. Prefer same-origin `fetch(..., { credentials: 'include' })` inside the authenticated page so cookies remain in the browser session and are not exported.

## Verification

Require a target page to appear in `/json/list`, a CDP `Runtime.evaluate` result with the expected URL/title/login-state fields, and the requested export artifact to exist with nonempty contents. For authenticated data, require the page to show an authenticated state after the user completes login; otherwise report the task as unauthenticated or incomplete.
