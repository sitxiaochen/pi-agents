---
name: scheduled-agent-task-execution-diagnostics
description: Use when a PI-Desktop scheduled agent appears not to run, produced no output, or ran in a separate session.
tags: ["自习"]
---

# Scheduled Agent Task Execution Diagnostics

Determine whether a PI-Desktop scheduled agent was triggered, whether its prompt reached a session, whether the run completed, and where its output went. This workflow diagnoses scheduler, session, RPC-timeout, and artifact problems; it does not repair a broken model provider, increase RPC timeouts, or guarantee that the next run succeeds.

## When to Use

Use when the user says “定时任务没有执行,” “scheduled task did not run,” “the scheduled report is missing,” “I cannot see the scheduled output,” or similar. Do not use it before checking the task list and current time, because a future `nextRunAt` can mean the task is simply not due yet.

## Procedure

1. Inspect the task record and current local time. Confirm `enabled`, `schedule`, `lastRunAt`, `nextRunAt`, `sessionMode`, `workspacePath`, and the task prompt. In the observed case, `lastRunAt` was `2026-10-22T01:00:24.448Z`, while local time was `2026-10-22 10:34:54 +08:00`; this proved the 09:00 Beijing run had already been triggered.

2. Interpret scheduler timestamps carefully. Convert UTC timestamps such as `2026-10-22T01:00:24.448Z` to the user's local timezone before comparing them with the schedule. A nonempty `lastRunAt` indicates a run attempt; it does not prove successful completion or visible output.

3. Account for `sessionMode`. With `sessionMode: perRun`, each execution may create a separate session rather than posting into the conversation where the task was configured. In the observed case, the task used `sessionMode: perRun`, so absence from the original chat was not evidence of non-execution.

4. Check for output artifacts in the configured workspace. Use PI `bash` with the task's actual `workspacePath`, and inspect both normal and hidden entries. For the observed workspace, use:

   `Test-Path $p; if(Test-Path $p){ Get-ChildItem -Path $p -Force | ForEach-Object { $_.Mode + "  " + $_.Name } }`

   An existing but empty workspace means no report file was produced; it does not mean the scheduler never fired.

5. Inspect PI-Desktop application and host logs around the scheduled timestamp. Relevant log roots observed were:

   `C:\Users\sitxiaochen\.pi-desktop\logs`

   `C:\Users\sitxiaochen\.pi-desktop\sessions`

   Use PI `read` or `grep` for `scheduled`, `ScheduledTask`, the task ID, or a distinctive prompt phrase. Also inspect `app\lifecycle.log` for both `app.boot` and `app.shutdown`. Interpret the lifecycle as intervals: a boot with no later shutdown proves the application remained running through that interval. Therefore, “no new boot since before the due time” does not mean the application was closed; it may mean it stayed open continuously. Do not treat absence of a text match alone as proof that no run occurred.

6. Distinguish prompt acceptance from completion. The app session log contained:

   `{"event":"prompt.accepted","sessionId":"0d0bb36a-5448-45e4-a23d-1cc648b0b343","turnId":"ffd1c760-620c-4e92-ba05-72ddb2b1ffb2","data":{"providerId":"39a7f877-5c15-4486-8610-02475e18e7c2","modelId":"workbuddy/space-bunny"}}`

   This proves that the scheduler created a session and submitted the prompt, but it does not prove that the agent produced a final response.

7. Inspect the per-run session file. In the observed case it was:

   `C:\Users\sitxiaochen\.pi-desktop\sessions\0d0bb36a-5448-45e4-a23d-1cc648b0b343.jsonl`

   Use PI `read` or `bash` to inspect the last messages. A session with a user prompt, assistant attempts, and tool calls but no final assistant report indicates an incomplete run. Preserve the session ID for correlation with app logs.

8. Check for RPC wall-clock exhaustion. The host runtime log contained:

   `rpc request exceeded its wall-clock budget; slot released budget_ms=270000`

   The affected Bash tool call had `durationMs: 270020` and `isError: true`. Repeated 270-second overruns at `2026-10-22T01:05:02.721Z`, `2026-10-22T01:09:36.095Z`, and `2026-10-22T01:14:09.065Z` aligned with the scheduled run and explain why the report was incomplete. Classify this as a timed-out run, not a scheduler that failed to trigger.

9. Correct the task design only after identifying the failure stage. For an unattended report, require a deterministic artifact path such as:

   `C:/Users/sitxiaochen/Documents/学习计划/tech-radar/YYYY-MM-DD.md`

   Use the actual date and create the directory if necessary. The prompt should require the full Markdown report in that file, then in the session reply, and explicitly prohibit questions to the user.

10. Make the task resilient to the observed failure mode. If any PI `bash` call returns text containing `host rpc budget`, do not retry that call immediately; switch to PI `write` for artifacts and other enabled research tools for retrieval. State limitations honestly rather than returning an empty report.

11. Prefer primary-source retrieval through enabled PI/GitHub tools. For HN discovery, the observed endpoint template was:

   `https://hn.algolia.com/api/v1/search?tags=story&numericFilters=created_at_i>...`

    Use a seven-day Unix-second filter. When using browser evaluation, treat `created_at_i>...` as a placeholder that must be replaced with the computed cutoff, and verify that the API response contains stories inside the requested window. Cross-check repository dates, releases, commits, and licenses with GitHub tools rather than anonymous `api.github.com` requests when rate limits may apply.

12. Check the task registry and the scheduler’s skipped-run behavior. The task record was stored in:

   `C:\Users\sitxiaochen\.pi-desktop\pi.sqlite`

   Compare `lastRunAt`, `nextRunAt`, `enabled`, `schedule`, `updatedAt`, and the task ID with the due time. In the observed case, `lastRunAt` remained `2026-10-22T01:00:24.448Z` while the next due occurrence on `2026-10-23` was absent and `nextRunAt` advanced to `2026-10-24T01:00:00.000Z`. This is a missed scheduler trigger despite a continuously running application, not an application-closed skip. A daily scheduler may advance to the next day without retrying the missed occurrence.

13. When the task is enabled but no trigger evidence exists, use the task-management interface to disable and then re-enable it, optionally changing the schedule, and inspect the returned task record. Treat this as a reset attempt, not a proven root-cause fix. Require the returned record to contain the intended `enabled`, schedule, `sessionMode: perRun`, workspace, and complete prompt, and verify `nextRunAt`; only a subsequent execution can verify recovery.

14. For artifact-producing recurring tasks, add a catch-up step that checks deterministic dated artifacts before producing the current artifact. For example, a daily report can list `tech-radar/YYYY-MM-DD.md` files, detect a missing prior date, and backfill at most two missing days before generating today. This limits missed-run damage but does not guarantee that the scheduler will trigger.

15. After updating the task, verify the returned task record still has the intended enabled state, schedule, `sessionMode: perRun`, workspace, and complete prompt. Do not claim the repair fixed the underlying RPC timeout or scheduler defect; the reliable correction is to avoid depending on the timing-out tool, require a persistent artifact, and verify a later execution.

## Pitfalls

- An empty current chat does not prove a `perRun` task did not execute; inspect `lastRunAt` and the separate session.
- If PI-Desktop was not running at the scheduled time, the task may have been skipped before creating a session. Conversely, a boot with no later shutdown means the application was continuously running, not closed; absence of a new boot record is not evidence of non-execution.
- A daily task can miss one occurrence and advance `nextRunAt` to the following day without creating a session. Do not claim that the schedule was reset merely because `enabled` remains true; inspect timestamps and verify the next execution.
- A nonempty `lastRunAt` proves only that an attempt was recorded. It does not prove a final response or artifact.
- A workspace that exists but contains no files is different from a missing workspace. Check both and state the distinction.
- Log messages with `rpc request exceeded its wall-clock budget` show timed-out requests, not necessarily a broken Bash installation. Correct the workflow by changing dependencies rather than encoding failed retries as best practice.
- A session log with `prompt.accepted` is not proof of successful completion. Verify the session transcript and output artifact.
- A separate session may use a scheduled model different from the model's `auto` selection in the configuration conversation. Check `modelId` in the session log before attributing behavior to the conversational model.
- Do not report success after rewriting the prompt. The next scheduled execution is required to verify that the report is generated and persisted.

## Verification

Require evidence of the prompt's acceptance in a session log, a final assistant message or artifact in the corresponding session/workspace, and a task record whose prompt names the exact expected output file and fallback behavior. When diagnosing a non-run, also verify PI-Desktop lifecycle evidence showing whether the application was running at the due time. If the task has not run again after the repair, report the configuration as updated but the next execution as unverified.
