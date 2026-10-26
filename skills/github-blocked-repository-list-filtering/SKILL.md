---
name: github-blocked-repository-list-filtering
description: Use when deleted or ToS-blocked GitHub repositories remain in repository listings or generated README content.
tags: ["自习"]
---

# GitHub Blocked Repository List Filtering

Diagnose repository entries that remain after deletion and remove their names from generated profile content without claiming the underlying GitHub records were purged. This workflow does not remove a GitHub ToS block, bypass GitHub restrictions, or alter repository permissions.

## When to Use

Use when phrases such as “删除了仓库但还有记录,” “deleted repos still appear,” “README still lists old repositories,” or “ToS-blocked repositories keep returning” identify the problem. Do not use it for normal pagination, rename, visibility, or permission problems unless fresh per-repository requests specifically return the ToS-block response.

## Procedure

1. Enumerate repositories visible to the current credential and preserve only safe metadata such as name, permissions, and timestamps. Confirm that a list endpoint is not authoritative about whether a repository remains directly accessible.

2. Probe each suspected repository with a fresh, non-cached request. Classify the results:
   - `HTTP 200`: repository is directly accessible.
   - `HTTP 404`: repository is not available at the endpoint.
   - `HTTP 403` with `{"message":"Repository access blocked"}` and `block.reason` equal to `tos`: the repository remains visible but is restricted by GitHub’s Terms of Service.

3. Treat the string `Repository access blocked` and `https://github.com/tos` as the decision boundary. If observed, report that API deletion and content changes are unavailable for that repository; do not misclassify the block as missing write permission merely because the token reports `admin=true` elsewhere.

4. Locate every generated representation of repository records. Inspect the generator with PI `read` or `grep` before editing, especially repository tables, activity feeds, event-derived links, and committed output. A generated README can retain names even after the user believes they were deleted.

5. Add a case-insensitive exclusion set to the generator. Include confirmed blocked names in a persistent `DEFAULT_EXCLUDED` list and support additional comma-separated names through `EXCLUDED_REPOS`. Filter repository rows before `slice(0, MAX_REPOS)` so excluded entries do not consume the display limit.

6. Filter activity and event data too. When GitHub reports an event repository as `owner/name`, compare the repository-name component, not the full qualified name. Apply exclusion before `slice(0, MAX_EVENTS)`.

7. Regenerate the managed blocks and remove stale rows from the committed file. Do not treat removal of the name from generated content as proof that GitHub removed the underlying repository record.

8. Validate syntax and project-specific checks. The observed commands were:

   `node --check scripts/update-readme.mjs`

   `& "C:\Program Files\nodejs\npm.cmd" run validate`

   The expected validation summary was `All checks passed.`

9. Check idempotency by hashing `README.md`, running `node scripts/update-readme.mjs`, hashing it again, and requiring identical hashes. If the generator depends on changing public API data, distinguish legitimate generated changes from exclusions failing to apply.

10. Commit both the generator change and regenerated output. Before pushing, verify synchronization with:

    `git ls-remote origin refs/heads/main`

    `git rev-parse HEAD`

11. If the push is rejected because the remote contains newer work, inspect it before integrating it:

    `git fetch origin`

    `git log --oneline HEAD..origin/main`

    `git log -1 --pretty='%h | %an | %s' origin/main`

    Use `git pull --rebase origin main` to retain both a remote rendered-output correction and a local prevention mechanism. Resolve generated-block conflicts in favor of fresh generator output when one side merely edits stale rendered rows.

12. Complete a conflicted rebase with:

    `git add README.md`

    `git -c core.editor=true rebase --continue`

    Do not include unrelated untracked directories merely to make the rebase or commit appear clean.

13. Push only after validation and rebase are complete:

    `git push origin main`

    A warning about LF being replaced by CRLF is not a push failure; require the remote ref to match `git rev-parse HEAD`.

## Pitfalls

- A repository appearing in `/user/repos` does not prove it is directly accessible. Always classify each repository with a fresh request.
- `admin=true` from a listing response does not override a per-repository `HTTP 403 Repository access blocked` response.
- Filtering only the repository table is insufficient. Public activity events can reintroduce excluded names unless event-derived names are filtered too.
- Filtering after `slice(0, MAX_REPOS)` or `slice(0, MAX_EVENTS)` can hide valid entries behind excluded ones; filter first.
- Editing generated README rows once does not prevent recurrence. Fix the generator and regenerate the output.
- A successful commit or push can still leave dynamic output modified by a later API refresh. Re-run validation and synchronization checks on the final state.
- A rebase conflict is not proof that the local change must be dropped. Distinguish a remote rendered-only cleanup from a local generator-level prevention fix.
- Removing names from the README is not deletion of GitHub records. ToS-blocked repositories still require GitHub resolution through `https://github.com/tos`.

## Verification

Require a successful `git push origin main`, identical hashes from `git ls-remote origin refs/heads/main` and `git rev-parse HEAD`, no confirmed blocked names in generated README repository or activity blocks, and `All checks passed.` from the project validation command.
