---
name: pi-desktop-mcp-dependency-triage
description: Diagnose PI-Desktop MCP startup failures caused by missing uv/uvx, absent tokens, or unreadable server configs.
tags: ["自习"]
---

# PI-Desktop MCP Dependency Triage

Diagnose PI-Desktop MCP servers that fail to start because their executable dependencies or environment variables are missing. This workflow also distinguishes an authentication success from an MCP protocol failure; it does not replace or expose user credentials.

## When to Use

Use when the user says one or more MCP servers are unavailable, asks to “处理 MCP 环境”, reports that `fetch` or `time` cannot start because `uvx` is missing, or reports that the GitHub MCP cannot start because `GITHUB_MCP_TOKEN` is unset.

Also use this skill when PI-Desktop lists the expected server count but some entries fail, when Windows PowerShell displays incorrect server names or commands, or when a newly installed command exists but PI-Desktop still cannot see it.

Do not use this skill for:

- Ordinary Git push authentication failures unrelated to an MCP dependency.
- Editing server definitions without first identifying the missing runtime or credential.
- Asking the user to paste a token into chat.
- Replacing PI-Desktop configuration or taking over live junctions; use `windows-pi-desktop-junction-sync` for that work.

## Procedure

1. Inventory the configured server files under `$env:USERPROFILE\.agents\servers` and identify each server's `command` or `transport`. Treat `uvx` as requiring `uv`, `npx` as requiring Node/npm, and `http` as requiring its configured URL, headers, and environment variables.

2. Read JSON configuration as UTF-8 bytes when Windows PowerShell appears to decode Chinese or Unicode content incorrectly. Do not infer duplicate server IDs or missing dependencies from a mis-decoded `Get-Content` result. Re-read the same files and compare the file count with the number of reported servers.

3. Check `GITHUB_MCP_TOKEN` at both user and machine scope without printing its value:

   `$envVal = [Environment]::GetEnvironmentVariable('GITHUB_MCP_TOKEN','User')`

   `$sysVal = [Environment]::GetEnvironmentVariable('GITHUB_MCP_TOKEN','Machine')`

   If neither scope contains a value and the GitHub server requires it, continue to credential discovery.

4. Check whether Windows Package Manager and the `uv` command are available. If `uv` is absent, install it with the observed package identifier and flags:

   `cmd /c "winget install --id astral-sh.uv --accept-source-agreements --accept-package-agreements --silent 2>&1"`

   Installation output may say `Path environment variable modified; restart your shell to use the new value.` Refresh the current process PATH before verifying:

   `$machinePath = [Environment]::GetEnvironmentVariable('Path','Machine')`

   `$userPath = [Environment]::GetEnvironmentVariable('Path','User')`

   `$env:Path = "$machinePath;$userPath"`

   `$env:Path = "$machinePath;$userPath"`

   Then verify both commands:

   `Get-Command uv -ErrorAction SilentlyContinue`

   `Get-Command uvx -ErrorAction SilentlyContinue`

   `uv --version`

   `uvx --version`

5. Before requesting a new token, inspect Git Credential Manager for an existing GitHub identity:

   `cmd /c "git credential-manager github list 2>&1"`

   If a credential exists, retrieve it without displaying it:

   `$input = "protocol=https`nhost=github.com`n`n"`

   `$res = $input | cmd /c "git credential fill 2>&1"`

   `$token = ($res | Select-String -Pattern '^password=').ForEach({ ($_.Line -replace '^password=','').Trim() }) | Select-Object -First 1`

   Confirm that `$token` exists and report only its length and a non-secret prefix. A `github_pat_` prefix identifies the credential as a fine-grained PAT; it does not prove that the token has the permissions required by a particular MCP endpoint.

6. Validate the credential independently against the GitHub API to determine repository capabilities. Record whether repository read, push, and administrative access are available. Do not infer these permissions from the token prefix or from successful Git credential retrieval alone.

7. Probe the GitHub MCP endpoint with the MCP-required `Accept` headers. Interpret results as follows:

   - `HTTP 200` plus an `event: message` response and `serverInfo: github-mcp-server` means the endpoint accepted the credential and completed an MCP handshake.
   - `HTTP 200` plus `Accept must contain both 'application/json' and 'text/event-stream'` means authentication was accepted but the probe omitted a required MCP media type. Correct the request headers rather than replacing the token.
   - A 401 or 403 response means authentication or authorization failed; inspect token validity and endpoint permissions.

8. Persist `GITHUB_MCP_TOKEN` at user scope without printing the token, and refresh the current process environment if the MCP must be tested in the same session. A user-level variable survives new processes but PI-Desktop must still be restarted because MCP configuration is read during startup.

9. Restart PI-Desktop and verify every server against its declared dependency: `npx` servers need Node/npm, `uvx` servers need `uv`, and HTTP servers need their environment variables and reachable endpoints. A successful package installation or credential probe alone is not final acceptance.

10. After environment changes, verify that the PI configuration repositories were not polluted with generated runtime state. Use `git status --short` in each repository and inspect any untracked path before committing. Only commit intentional managed-skill changes; do not sweep ignored runtime files into Git.

## Pitfalls

- `Get-Content` can decode a UTF-8 server JSON file incorrectly under Windows PowerShell, producing duplicate or wrong-looking server IDs. Re-read as UTF-8 bytes before diagnosing configuration corruption.
- `Successfully installed` does not mean the current shell can resolve `uv` or `uvx`. Refresh the machine and user PATH and run `Get-Command uv` and `Get-Command uvx`.
- A Git credential existing in Git Credential Manager does not guarantee that the same token works with every GitHub service or MCP endpoint. Validate its permissions and perform a real MCP handshake.
- `HTTP 200` is not by itself success. The response body must satisfy the MCP protocol, including the required `Accept` behavior and a `serverInfo: github-mcp-server` result.
- `Accept must contain both 'application/json' and 'text/event-stream'` is not an authentication failure when it accompanies `HTTP 200`; it identifies a malformed MCP probe.
- Setting a user-level environment variable does not update an already-running PI-Desktop process. Restart PI-Desktop before judging the server inventory.
- Do not print the PAT, place it in a server JSON file, commit it, or paste it into chat. Reuse a locally stored credential only after verifying that it is intended for the target service.
- Do not commit untracked files merely because a server dependency check created them; first determine whether they are intentional managed content or runtime state.

## Verification

After restarting PI-Desktop, require the Settings → MCP view to show every configured server as usable, with `fetch` and `time` resolving `uvx`, the GitHub server completing an MCP handshake using `GITHUB_MCP_TOKEN`, and no secret appearing in command output, logs, or Git changes.
