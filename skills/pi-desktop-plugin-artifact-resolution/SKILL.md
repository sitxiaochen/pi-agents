---
name: pi-desktop-plugin-artifact-resolution
description: Use when diagnosing or resolving PI-Desktop plugin downloads, catalog URLs, or corrupted plugin artifacts.
tags: ["自习"]
---

# PI-Desktop Plugin Artifact Resolution

Diagnose PI-Desktop plugin download failures by locating the authoritative catalog, distinguishing real JSON and binary responses from SPA fallbacks, and resolving artifact URLs before installation. This skill identifies and verifies the correct package; it does not install a plugin into PI-Desktop's application data without approval.

## When to Use

Use this skill when the user reports a PI-Desktop plugin timeout, a package that downloads as HTML, a missing `artifactBaseUrl`, an invalid `.piplug` download URL, or a plugin that is missing from `plugins/installed`.

Concrete triggers include:
- “插件中心下载一直超时，提示 `curl (28)`.”
- “下载的 `.piplug` 实际是 HTML.”
- “`/catalog.json` 有插件，但 `artifactBaseUrl` 是空的.”
- “插件包明明存在，PI-Desktop 却无法安装.”

Do not use it for:
- Writing or modifying plugin source code.
- Installing a package into `plugins/installed` without explicit approval.
- Treating a successful HTTP status as proof that the response is a valid plugin package.
- Searching unrelated application data when the authoritative catalog URL has not been established.

## Procedure

1. Probe likely catalog endpoints and validate their media type and body.
   - Fetch the known official catalog with:
     `Invoke-WebRequest -Uri 'https://plugins.aiuo.net/catalog.json' -UseBasicParsing -TimeoutSec 40 -OutFile $s -ErrorAction Stop`
   - When comparing multiple endpoints, inspect status, `Content-Type`, length, and final URL. A `200` response containing `text/html` is the Plugin Center SPA fallback, not a catalog or package.
   - The authoritative endpoint was confirmed as `https://plugins.aiuo.net/catalog.json`, returning `200 application/json; charset=utf-8`.

2. Parse the catalog without dumping it.
   - Read the saved file as UTF-8 and inspect its top-level fields:
     `Get-Content $c -Raw -Encoding UTF8 | ConvertFrom-Json`
   - Confirm these fields exist:
     `schemaVersion`, `providerId`, `catalogId`, `name`, `homepage`, `updatedAt`, `generatedAt`, `policyVersion`, `artifactBaseUrl`, `plugins`.
   - In the observed catalog, `plugins` was an array of 77 entries and `artifactBaseUrl` was `https://raw.githubusercontent.com/AIUO-Net/pi-desktop-plugins/main`.

3. Resolve the target plugin and version.
   - Filter by plugin ID, for example:
     `$j.plugins | Where-Object { $_.id -eq 'pi.session-orchestrator' -or $_.id -like '*session-orchestrator*' }`
   - For each version, record `version`, `yanked`, `sizeBytes`, `shasum`, `url`, and `permissions`.
   - Prefer the newest version with `yanked=False`; for `pi.session-orchestrator`, the observed newest non-yanked version was `0.7.1`.

4. Construct the real artifact URL using the catalog's base URL.
   - If `url` is relative, join it to `artifactBaseUrl`:
     `$real = "$($j.artifactBaseUrl.TrimEnd('/'))/$($v.url)"`
   - Do not prepend `artifactBaseUrl` to an already absolute URL.
   - For the observed target, the relative URL `packages/pi.session-orchestrator-0.7.1.piplug` resolved to:
     `https://raw.githubusercontent.com/AIUO-Net/pi-desktop-plugins/main/packages/pi.session-orchestrator-0.7.1.piplug`
   - If the server catalog lacks `artifactBaseUrl`, or the local catalog has an empty value, treat artifact resolution as unverified rather than guessing a host.

5. Validate the response before downloading the body.
   - Check status, content type, and byte count. A package must not be `text/html`.
   - The observed wrong URL `https://plugins.aiuo.net/packages/pi.session-orchestrator-0.7.1.piplug` returned `200` but only 1626 bytes of SPA HTML on five consecutive requests; it was not a valid package.
   - The correct raw GitHub URL returned `200`, `54915` bytes, and matched the catalog's declared `sizeBytes`.

6. Verify cryptographic integrity.
   - Compute SHA256 over the downloaded `.piplug` and compare it with the selected catalog version's `shasum`.
   - The observed `0.7.1` artifact matched:
     `4b7c380692614761f0be9b8408665e722e77bc656e36d2efff18ac023fcd5e36`
   - Continue only when the local hash equals the catalog hash and the local byte count equals `sizeBytes`.

7. Compare the authoritative and cached catalogs.
   - The local Plugin Center catalog is at:
     `$env:USERPROFILE\.pi-desktop\plugins\market\catalog.json`
   - The cache metadata is at:
     `$env:USERPROFILE\.pi-desktop\plugins\market\cache-meta.json`
   - The observed cache metadata identified:
     `"sourceUrl": "https://plugins.aiuo.net/catalog.json"`
   - Compare `artifactBaseUrl`, each version's `url`, `updatedAt`, and `generatedAt`. If the live catalog is correct but the cached one is not, refresh the Plugin Center catalog cache or restart PI-Desktop before considering a local cache repair.

8. Check installation state without modifying application data.
   - The PI-Desktop plugin root is:
     `$env:USERPROFILE\.pi-desktop\plugins`
   - Relevant subdirectories are `cache`, `data`, `disabled`, `installed`, `logs`, and `market`; registry state is in `registry.json`.
   - Enumerate `installed` and confirm whether the target plugin directory exists. In the observed case, `pi.session-orchestrator` was absent, so the failure was not mistaken for a broken existing installation.

9. Choose a remediation boundary.
   - First prefer refreshing the Plugin Center catalog or restarting PI-Desktop when the live catalog is correct and the local cache is stale.
   - If the live catalog is missing `artifactBaseUrl`, report that as a server-side catalog defect with the exact endpoint and affected relative URLs.
   - Treat direct cache editing or copying a package into `plugins/installed` as application-data modification. Do not do it without explicit approval, and back up the affected file first.
   - After any approved repair, reacquire the artifact from the authoritative resolved URL, verify SHA256 and size, restart PI-Desktop, and confirm the plugin appears in the UI and in `plugins/installed`.

## Pitfalls

- HTTP `200` does not identify the resource type. SPA fallback routes can return `200 text/html` for nonexistent package paths.
- A `curl (28)` message can describe the client's later processing timeout rather than the actual cause; a stable HTML fallback may be returned in under two seconds.
- `plugins.aiuo.net/packages/...` is not automatically the package's canonical URL when the catalog provides `artifactBaseUrl`.
- An empty `artifactBaseUrl` in a cached catalog can silently degrade relative artifact URLs to the wrong host.
- The live catalog and `plugins/market/catalog.json` may differ even though both came from the same source URL.
- A matching filename is not integrity verification. Compare the actual downloaded bytes' SHA256 with the selected version's `shasum` and compare length with `sizeBytes`.
- A plugin absent from `plugins/installed` has not been successfully installed, regardless of whether a `.piplug` file downloaded successfully.
- A `200` package response can still be unusable if its content type is HTML or its size and hash do not match.
- Do not manually populate `plugins/installed` without explicit approval and a backup; the directory may require application-managed metadata and registration.

## Verification

Confirm that the selected version is non-yanked, its resolved URL returns a non-HTML package of exactly `sizeBytes`, the locally computed SHA256 equals the catalog `shasum`, and the target plugin is present under `$env:USERPROFILE\.pi-desktop\plugins\installed` after refresh or approved installation. If installation is not authorized, verification stops at a verified artifact plus a confirmed absence from `plugins/installed`.
