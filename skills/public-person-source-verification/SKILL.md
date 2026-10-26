---
name: public-person-source-verification
description: Use when researching a named person's public role, employer, appointment, or biography.
tags: ["自习"]
---

# Public Person Source Verification

Research a public figure or employee by matching the exact name, locating the most authoritative accessible source, and separating confirmed facts from weak search snippets. This workflow verifies public-role claims; it does not expose private information or establish identity beyond the cited records.

## When to Use

Use this skill when the user asks to look up a Chinese or English personal name and wants public information such as employer, organization, job title, appointment, biography, or public-service record.

Concrete triggers include:
- “邵信潇这个姓名你查一下.”
- “查一下这个人任什么职务.”
- “帮我查某人的公开任职或简历.”
- “这个人是不是在某个单位工作.”

Do not use it for:
- Finding private addresses, private phone numbers, family members, or other sensitive personal data.
- Identifying an anonymous or private individual without an explicit legitimate public-role context.
- Treating a similar name, search-result count, or AI-generated snippet as confirmation.
- Claiming a current position from an old document without checking for newer records.
- Presenting government search snippets as primary evidence when the original document is accessible.

## Procedure

1. Define the exact identity tokens.
   - Preserve the full written name exactly, including all Chinese characters. Search with double quotes first, then pair it with likely role terms such as `任免`, `公示`, `医保`, the organization, or region.
   - Keep similar names such as `邵潇`, `邵明潇`, and `邵信潇` separate. Do not merge them merely because search engines broaden the query.
   - Treat the requested name as unverified until at least one source contains the complete exact string.

2. Use the preferred search tool first, then select a bounded fallback.
   - If the search tool fails with `net::ERR_PROXY_CONNECTION_FAILED`, confirm that the host still has direct network access before changing strategy:
     `curl.exe -s -o NUL -w "%{http_code}" --max-time 8 https://www.gov.cn 2>&1; Write-Output ""`
   - A result of `200` confirms only that this request reached the site; it does not prove that the person was found.
   - If direct requests fail but a local HTTP proxy is available, probe it explicitly before changing search strategy:
     `Invoke-WebRequest -Uri $u -Headers $h -TimeoutSec 20 -Proxy "http://127.0.0.1:7890" -UseBasicParsing`
     Use the same proxy for the exact-name search and require a matching result passage in the cleaned body. A proxy-connected `200` proves only that the proxy path reached the site; it does not prove the person was found.
   - If the shell is available, query accessible search engines with `Invoke-WebRequest`, strip scripts/styles/tags, and inspect the returned text for the complete name. Use a realistic browser `User-Agent` and a short timeout.
   - Treat Google pages containing `/httpservice/retry/enablejs`, CAPTCHA pages, and Baidu security-verification responses as degraded search coverage, not negative results.
   - When historical records are requested, query the Wayback CDX endpoint with bounded date and domain filters, such as:
     `https://web.archive.org/cdx/search/cdx?url=hangzhou.gov.cn&matchType=domain&from=2015&to=2019&output=text&limit=40&collapse=urlkey`
     Inspect returned timestamps, original URLs, MIME types, and record counts. A domain query containing only the homepage and assets does not establish that no relevant historical page existed.
   - Stop escalating when search engines begin returning CAPTCHA or security-verification pages; report the reduced coverage rather than treating the page as evidence.

3. Confirm that the exact name occurs in the result body.
   - Search-engine headlines can refer to the query text, recommendations, or a different person. Require a result passage containing the exact full name.
   - A result such as `邵信潇 市医疗保障事务受理中心` is a lead, not final verification.
   - When broad engines split or relax rare names, use a second engine or direct site-restricted query such as `邵信潇 site:hangzhou.gov.cn`.

4. Locate and fetch the authoritative original.
   - Prefer the issuing organization’s own government or institutional page over an aggregator, encyclopedia, media snippet, or AI summary.
   - For the observed Hangzhou case, the authoritative page was:
     `https://www.hangzhou.gov.cn/art/2021/1/7/art_1229063389_1716644.html`
   - Fetch it directly with `Invoke-WebRequest`, then remove scripts, styles, and tags while preserving the surrounding text. Search the cleaned body for the exact name and nearby headings.
   - The source line in the original page was:
     `邵信潇 市医疗保障事务受理中心`
     under `3.信息支持组`.
   - For Hangzhou public-role checks, inspect these parent columns directly when the bureau has no resolvable standalone domain:
     - 人事信息: `https://www.hangzhou.gov.cn/col/col1229444340/index.html`
     - 公告公示: `https://www.hangzhou.gov.cn/col/col1229445413/index.html`
     - 市政府人事任免: `https://www.hangzhou.gov.cn/col/col1229063387/index.html`
   - In the observed case, `https://ybj.hangzhou.gov.cn/` failed DNS resolution, while the Hangzhou government portal remained accessible with status `200`. Treat that as evidence that the standalone hostname was unavailable, not that the bureau or its records did not exist.
   - The 人事信息 parent page exposed current 市政府人事任免 article links under `/col/col1229063387/art/2026/...`, but the dedicated column list was JavaScript-driven and did not yield a complete historical set from raw HTML. Do not describe an incompletely loaded list as an exhaustive search.
   - For the 公告公示 check, inspect the cleaned text for the exact name and separately inspect the 医疗卫生/医保 section; a general `HIT=False` result does not prove that every dynamically loaded item was searched.
   - For Hangzhou historical recruitment checks, inspect the 招考录用 column `https://www.hangzhou.gov.cn/col/col1229057405/index.html` and its 事业单位招考 child column `https://www.hangzhou.gov.cn/col/col1229063421/index.html`.
   - When a column page returns `200` but its article list is JavaScript-rendered, treat the raw HTML as incomplete. Guessed pagination URLs such as `https://www.hangzhou.gov.cn/col/col1229063421/index_1.html` returned `404 未找到`; a column-level `HIT=False` therefore does not establish that no historical article exists.
   - If the visible list contains only recent years, record the visible coverage rather than claiming an archive search reached the requested years. Search the issuing authority's historical domain, archived snapshots, or the recruiting system's own archive when available.
   - Search engines can reflect the quoted query in their interface without returning a matching result. Require an actual result passage containing the complete exact name; query reflection alone is not a hit.

5. Determine the strongest defensible claim.
   - Record what the source explicitly states: organization, group, role, date, issuing body, and document title.
   - Do not infer a specific job title or rank when the document lists only a unit or committee membership.
   - In the observed source, the defensible finding was membership on the `信息支持组` as a member affiliated with `市医疗保障事务受理中心`; it did not establish a director, manager, rank, or appointment.
   - A later article mentioning the person’s work can corroborate ongoing involvement in a timeframe, but it remains secondary evidence.

6. Search for newer or contradictory records within the same official domain.
   - Search the exact name together with `任免`, `公示`, and the confirmed organization or region. Check the original official domain whenever possible.
   - Distinguish document date, publication date, and the date of the event described.
   - If no newer record is found, report “未检索到” rather than “没有”, and disclose CAPTCHA, proxy, indexing, or coverage limitations.
   - Do not rely on future-dated or anomalous search snippets as proof; verify against the underlying page.

7. Report source tiers and confidence.
   - High confidence: exact name in an original government or institutional document.
   - Medium confidence: exact name in a reputable media report or institutional article that identifies the person and role.
   - Low confidence: search snippets, aggregators, similarly named people, or generated summaries without an accessible source.
   - Clearly label each fact by its source and date, and separate the confirmed historical record from the current-status question.

## Pitfalls

- A `200` response is only transport success. It does not establish that a page contains the person or is the authoritative source.
- Search engines may split a rare Chinese name or silently broaden it to similar names. Inspect the body for the complete exact string.
- A large result count is not evidence of relevance or identity.
- Search snippets can be stale, truncated, generated, or extracted from the wrong page. Fetch the original when possible.
- A search engine may broaden a rare Chinese name into a single-character or surname-only result. Treat that output as search degradation, not evidence that the full name has no records.
- A government portal's column page may return `200` while its article list is loaded dynamically. A raw-HTML scan of that page is not an exhaustive archive search.
- A working-group member list proves participation, not a formal job title, rank, or leadership appointment.
- An old document establishes the role at that date, not necessarily the person’s current position.
- A CAPTCHA or proxy error reduces search coverage. It must be disclosed, not converted into a definitive negative finding.
- Similar names must not be merged without an explicit authoritative linkage.
- Do not fill missing job-title information from biography clues or organizational hierarchy.

## Verification

Confirm that at least one accessible original source contains the exact full name, that the extracted text preserves its surrounding organizational or role context, and that the final answer distinguishes confirmed facts, historical dates, unresolved current status, and search-coverage limitations. Before reporting a negative claim, verify that the wording is limited to the searched records and coverage, not the person's actual status or rank.
