---
name: windows-pi-desktop-junction-sync
description: Safely synchronize PI-Desktop configuration repositories on Windows using junction takeover, backups, and rollback.
tags: ["自习"]
---

# Windows PI-Desktop Junction Sync

Synchronize PI-Desktop configuration and managed skills from Git repositories on Windows while preserving rollback copies and avoiding runtime-state pollution. This workflow does not replace local configuration without explicit approval, and it does not merge conflicting skill versions automatically.

## When to Use

Use when the user asks to "生效同步", "接管原件", synchronize `~/.pi` or `~/.agents`, use Git repositories as the live PI-Desktop configuration source, or establish repeatable pull/commit/push synchronization across Windows machines. Do not use it for ordinary Git authentication, repository blocking, merge-conflict resolution unrelated to PI-Desktop, or changing skills directly in managed skill directories.

## Procedure

1. Inspect both repositories and the live directories before changing anything. Verify local HEAD, remote-tracking HEAD, tracked-file count, live-directory contents, and whether the live directories are already links:

   `git -C "$root\pi-agent" rev-parse --short HEAD`

   `git -C "$root\pi-agents" rev-parse --short HEAD`

   `git -C "$root\pi-agent" rev-parse --short origin/main`

   `git -C "$root\pi-agents" rev-parse --short origin/main`

   `git ls-files | Measure-Object`

   `Get-ChildItem "$env:USERPROFILE\.pi" -Recurse -File`

   `Get-ChildItem "$env:USERPROFILE\.agents" -Recurse -File`

   `Get-Item "$env:USERPROFILE\.pi","$env:USERPROFILE\.agents" | Select-Object FullName,LinkType,Target`

   Treat `pi-agent` as the source for `~/.pi` and `pi-agents` as the source for `~/.agents` only after confirming this mapping with the user or repository structure.

2. Compare the live and repository-managed skill copies before takeover. PI loads managed skills from the active managed directory, so a newer copy under `~/.pi/agent/skills` may not be effective. Inspect complete files and metadata rather than comparing only byte length:

   `Compare-Object (Get-Content "$new\network-triage-cn\SKILL.md") (Get-Content "$old\network-triage-cn\SKILL.md")`

   `Get-ChildItem "$env:USERPROFILE\.pi\agent\skills" -Directory`

   `git -C "$root\pi-agents" status --short`

   If the live copy contains substantive newer sections, copy it into the repository with PI `read` and `write`, preserving UTF-8 without BOM and LF line endings. Commit and push that reconciliation before creating links.

3. Identify runtime state before it can enter Git. Check tracked files and ignore decisions for files such as `enqueue.log`, `last-review.json`, `last-enqueue.json`, `usage.json`, and `nudge-state.json`:

   `git -C "$root\pi-agents" ls-files skills/.skill-learning/`

   `git -C "$root\pi-agents" check-ignore -v skills/.skill-learning/usage.json`

   `git -C "$root\pi-agents" check-ignore -v skills/.skill-learning/enqueue.log`

   `git -C "$root\pi-agents" check-ignore -v skills/.skill-learning/last-review.json`

   `git -C "$root\pi-agents" check-ignore -v skills/.skill-learning/last-enqueue.json`

   Do not treat an ignored file present in the working tree as repository pollution. Only files returned by `git ls-files` are tracked.

4. Create full backups before renaming or linking the live directories. Use a timestamped suffix and compare file counts and total bytes:

   `robocopy "$env:USERPROFILE\.pi" "$env:USERPROFILE\.pi.bak-$stamp" /E /NFL /NDL /NJH /NJS`

   `robocopy "$env:USERPROFILE\.agents" "$env:USERPROFILE\.agents.bak-$stamp" /E /NFL /NDL /NJH /NJS`

   `Get-ChildItem "$env:USERPROFILE\.pi" -Recurse -File | Measure-Object Length -Sum`

   `Get-ChildItem "$env:USERPROFILE\.agents.bak-$stamp" -Recurse -File | Measure-Object Length -Sum`

   `robocopy` can return exit code 1 when it copies successfully; validate the resulting counts and bytes instead of treating exit code 1 alone as failure.

5. After explicit approval to take over the originals, rename rather than delete them. This preserves a direct rollback copy:

   `Rename-Item "$env:USERPROFILE\.pi" ".pi.pre-takeover-$stamp"`

   `Rename-Item "$env:USERPROFILE\.agents" ".agents.pre-takeover-$stamp"`

6. Create Windows directory junctions from the live paths to the repositories. Junctions are used because the observed session had no administrator privilege:

   `cmd /c mklink /J "$env:USERPROFILE\.pi" "$env:USERPROFILE\pi-sync\pi-agent"`

   `cmd /c mklink /J "$env:USERPROFILE\.agents" "$env:USERPROFILE\pi-sync\pi-agents"`

   Verify the resulting `LinkType` and target. If `mklink /J` fails because the target still exists, confirm the rename completed before retrying; do not delete the target repository.

7. Validate visibility, repository cleanliness, and synchronization through the junctions:

   `Get-ChildItem "$env:USERPROFILE\.pi" -Recurse -File`

   `Get-ChildItem (Join-Path $env:USERPROFILE '.agents\skills') -Directory`

   `git -C "$root\pi-agent" status --porcelain`

   `git -C "$root\pi-agents" status --porcelain`

   `git -C "$root\pi-agent" rev-parse HEAD`

   `git -C "$root\pi-agents" rev-parse HEAD`

   `git ls-remote origin refs/heads/main`

   Require clean worktrees and identical local and remote hashes.

8. Validate scripts through the junction while accounting for Windows execution policy. A direct invocation can be blocked even when the script is valid:

   `powershell -NoProfile -ExecutionPolicy Bypass -File "$env:USERPROFILE\pi-sync\pi-agent\switch-role.ps1" status`

   An error containing `about_Execution_Policies` indicates policy blocking, not a broken junction. Do not claim the script succeeded until it produces the expected status output.

9. Restart PI-Desktop after takeover. The current session may retain the skill and MCP inventory loaded before the junctions were created. Verify the restarted UI shows the expected number of skill directories, single-file skills, and MCP servers, and trigger one known managed skill by name.

10. For routine synchronization, use pull and commit/push through the repository paths:

   `foreach ($r in 'pi-agent','pi-agents') {`
   `  Set-Location "$env:USERPROFILE\pi-sync\$r"`
   `  git pull --rebase --autostash`
   `}`

   `foreach ($r in 'pi-agent','pi-agents') {`
   `  Set-Location "$env:USERPROFILE\pi-sync\$r"`
   `  git add -A`
   `  if (git status --porcelain) {`
   `    git commit -m "sync: $env:COMPUTERNAME $(Get-Date -f 'yyyy-MM-dd HH:mm')"`
   `    git push`
   `  }`
   `}`

11. To roll back, remove only the junctions and restore the renamed originals:

   `Remove-Item "$env:USERPROFILE\.pi","$env:USERPROFILE\.agents"`

   `Rename-Item "$env:USERPROFILE\.pi.pre-takeover-$stamp" ".pi"`

   `Rename-Item "$env:USERPROFILE\.agents.pre-takeover-$stamp" ".agents"`

   Confirm that `Remove-Item` targets the junction paths only. Never run it against the repository directories they point to.

## Pitfalls

- A newer skill under `~/.pi/agent/skills` does not imply that PI loaded it. Reconcile substantive differences into the actual managed skill source before takeover.
- A larger file is not proof of a better version. Compare complete sections and content before replacing the repository copy.
- `robocopy` exit code 1 does not mean backup failure. Verify file counts and total bytes.
- A file existing under a linked working directory is not automatically tracked. Use `git ls-files` and `git status --porcelain` to distinguish tracked changes from ignored runtime state.
- An empty PowerShell script result may hide an execution-policy failure. Capture all streams and rerun with `-ExecutionPolicy Bypass` to distinguish policy blocking from script failure.
- A successful junction creation does not update the current PI-Desktop session. Restart is required for the skill and MCP inventory to be rebuilt.
- Deleting a junction through a file manager removes only the link, but confusing the junction with its repository target can destroy the source data. Verify `LinkType` and `Target` first.
- Do not assume every machine has the same local files. If a machine's live directory differs substantially from the repository, keep its original untouched until differences are reviewed.
- Do not automatically set credentials or install missing runtimes merely to complete synchronization. Record unresolved items such as `GITHUB_MCP_TOKEN` and `uv` separately.

## Verification

Require both live paths to report `LinkType` as `Junction` with the intended repository targets, both repositories to have clean worktrees, local HEAD hashes to match `git ls-remote origin refs/heads/main`, the expected skill and server counts to be visible through the junctions, and PI-Desktop to show the expected inventory after restart.
