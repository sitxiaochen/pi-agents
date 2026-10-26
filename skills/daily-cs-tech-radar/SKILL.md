---
name: daily-cs-tech-radar
description: Use when producing a current, personalized daily computer-science and open-source technology radar.
tags: ["自习"]
---

# Daily CS Technology Radar

Build a dated technology-intelligence report by researching recent releases and projects, verifying important claims against primary sources, scoring candidates against the user's background, and turning the strongest item into an executable experiment. This skill does not fabricate dates, versions, licenses, benchmarks, or deployment requirements, and it does not recommend every trending project merely because it is popular.

## When to Use

Use when the user asks for a “daily CS frontier report,” “technology radar,” recent AI/open-source projects, or a personalized engineering learning digest. Do not use it for a one-off answer about a named project, a generic news summary without verification, or a static knowledge-base update where current web research is explicitly unnecessary.

## Procedure

1. Establish the report date and time window. Use a 7-day primary window for new projects and major updates, extend to 30 days for active mature projects, and label older foundational projects as mature or foundational rather than presenting them as new.

2. Define the personalization profile from the user's stated interests and experience. For an infrastructure-oriented learner, prioritize Linux, networking, Python, automation, Docker, databases, CI/CD, monitoring, security, virtualization, and AI infrastructure. Do not assume expertise not stated by the user.

3. Discover candidates across official repositories, releases, project documentation, technical announcements, papers, and community discussions. Treat rankings, stars, and social discussion as discovery signals only. Check current date filters and preserve URLs for every important claim.

4. Verify each leading candidate against a primary source. Distinguish repository creation date, release publication date, latest commit, announcement date, and discussion date. Confirm that the linked repository, release, documentation, and described feature actually exist.

5. Reject or downgrade candidates with unresolved license, maintenance, security, compatibility, or availability concerns. If a project requires unavailable specialist hardware or unsupported hardware, keep it as a reading candidate only when clearly labeled non-deployable for the user.

6. Score candidates from 1–5 using: background and interest fit 25%, novelty and recent technical value 20%, deployment and experiment value 20%, CS and engineering learning value 15%, maintenance/documentation/community quality 10%, and deployment cost or hardware barrier 10%. Use the score for ranking, not as a substitute for judgment; remove severe security, license, maintenance, or verification failures.

7. Produce 3–5 ranked items. For each include basic information, official links, freshness category, maintenance and license status, technical value, user-specific relevance, deployment conditions, a 30–120 minute experiment, portfolio value, risks, and known limitations. Mark every unverified fact as “未核实” or equivalent; never fill gaps with estimates presented as official requirements.

8. For experiments, copy commands and flags from official documentation. Separate Windows, Linux, and container commands. State whether the command was actually executed; if local execution is unavailable, say so explicitly rather than implying success.

9. Add three closing sections: one highest-priority action with its first concrete step, 1–3 CS concepts connected to the recommendations, and a follow-up list of items or claims that need later verification. Include a deduplication limitation when no readable history of prior reports exists.

10. End with a verification note listing the primary sources used and any research limitations. Never claim cross-task deduplication without an accessible history.

11. For recurring reports, maintain a persistent deduplication ledger with PI `read` and `write` at `tech-radar/README.md`. Before selecting candidates, read the rolling 30-day records and the permanent exclusions. After producing the dated report at `tech-radar/YYYY-MM-DD.md`, append each recommended project with its date, category, and one-sentence reason, then remove entries older than 30 days. Treat an inaccessible or empty ledger as a disclosed deduplication limitation rather than claiming cross-task deduplication. Preserve every existing ledger row during updates: prefer append-style insertion over line-number replacement, then read the file back and confirm that prior recommendations remain. If an edit removes a row accidentally, restore it before finishing.

## Pitfalls

- A successful HTTP response or a high star count does not prove technical quality, current maintenance, or safe deployment; verify the repository, release, documentation, and maintenance state directly.
- Trending pages may contain stale, manipulated, or irrelevant entries. Do not recommend a project solely because it appears there.
- A release announcement, first commit, latest commit, and news article have different meanings. Label dates according to their event type.
- A project's code license may differ from its model-weight or data license. Record both and mark commercial use as unverified when terms are unclear.
- A project may be technically impressive but unusable for the user because of hardware, firmware, platform, account, or network requirements; distinguish “worth reading” from “recommended to deploy.”
- A plausible command can still be unsafe or wrong. Use commands copied from official documentation, call out public exposure, deletion, credential, and permission risks, and disclose whether it was executed.
- If shell execution is unavailable, the report can still contain verified research, but local reproduction remains unverified.
- A recommended-project list alone is not a deduplication ledger. To suppress repeats across recurring runs, persist dated project records and explicit exclusions, read them before selection, and update them after the report is saved.
- A missing or empty `tech-radar/README.md` does not justify claiming that prior reports were checked; disclose the missing history instead.
- A successful ledger edit does not guarantee that prior rows were preserved. Line-number-based replacements can delete or displace existing records, so read the ledger back after editing and verify that every pre-existing recommendation is still present.

## Verification

Require every recommended item to have at least one accessible primary source, a correctly classified freshness date, an explicit license or “未核实” label, a concrete experiment, and a final report section stating that unexecuted commands and unavailable historical deduplication were disclosed. For recurring runs, also require that `tech-radar/YYYY-MM-DD.md` exists, `tech-radar/README.md` contains the new recommendations and applicable exclusions, and entries outside the 30-day window have been removed.
