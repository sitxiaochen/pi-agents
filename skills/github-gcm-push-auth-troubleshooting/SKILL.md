---
name: github-gcm-push-auth-troubleshooting
description: Diagnose and recover GitHub push authentication failures caused by stale or limited Git Credential Manager credentials.
tags: ["自习"]
---

# GitHub GCM Push Authentication Troubleshooting

This skill diagnoses Git Credential Manager authentication and permission failures on Windows when pushing to GitHub. It does not create or modify tokens, change repository permissions, or replace GitHub's web-based credential management.

## When to Use

Use this when phrases such as “403 when pushing,” “push authentication failed,” “Git Credential Manager uses the wrong account,” or “the token can read but cannot push” identify the problem. Do not use it for ordinary file, branch, merge, or commit problems; verify that the failure is specifically authentication or authorization first.

## Procedure

1. Inspect the active Git credential without printing the secret. Use `git credential fill` with the protocol and host, then classify the returned token shape rather than exposing it.

2. Inspect Git Credential Manager configuration and available credential storage. The observed configuration was:

   `file:C:/Program Files/Git/etc/gitconfig\tcredential.helper=manager`

   Also check whether `.gcm`, `%LOCALAPPDATA%\GitCredentialManager`, or `%APPDATA%\GitCredentialManager` exists; absence of a visible credential does not prove that the credential is usable.

3. Re-authenticate through GCM when the stored credential may be stale or belong to the wrong GitHub account. The observed command was:

   `git-credential-manager.exe github login`

   The transcript showed that this command can produce no visible output even when it changes the active credential.

4. Re-check the credential after login. Compare the username and token class, not just the command exit status. A token beginning `github_pat_` is a fine-grained PAT; do not assume its permissions from its name alone.

5. Before concluding that the remote lacks write access, run the non-destructive check:

   `git push --dry-run`

   If it succeeds, treat the authentication path as usable and proceed. A single earlier `push:false` permission snapshot may be stale while GCM is replacing a token.

6. Push only after the dry run succeeds:

   `git push origin main`

7. Verify the remote branch rather than relying on the push command alone:

   `git ls-remote origin refs/heads/main`
   `git rev-parse HEAD`

   Compare the two hashes and confirm they are identical.

## Pitfalls

- Empty output from `git-credential-manager.exe github login` is not proof that login failed. Re-check the active credential and run `git push --dry-run`.
- A token prefix does not prove write permission. In particular, `github_pat_` identifies a fine-grained PAT, but repository and branch permissions still require verification.
- A temporary API response reporting `push:false` can contradict a successful dry run. Prefer the fresh non-destructive Git check before declaring a persistent permission block.
- Never print the password/token returned by `git credential fill`; expose only safe metadata such as length and prefix class.

## Verification

A successful check is `git push --dry-run` returning exit code 0, followed by `git ls-remote origin refs/heads/main` and `git rev-parse HEAD` producing the same hash.
