# Решения: online-pipeline-postmerge-followup

## Round N's PR is opened after its first commit, not at branch-creation time

**What we decided:** In stage post-merge's "followup" (new round) branch, the workflow creates
round N's branch (`feature/r{N}-{slug}`) and writes the `active_branch` marker, removes the
`online-pipeline-merged-awaiting-followup` label once those two succeed, then calls stage
`implement` on the new (still-empty) branch. The PR itself is opened later by the existing "Open
the code PR" step, after `code-writing` has actually committed something — exactly the same
deferred timing round 1 (`ROUTE_ACTION=start`) already uses for its own PR.
**Why:** Discovered during a live dry-run on a real repo (`sz-ops`): the originally-implemented
design called `gh pr create` immediately after pushing the brand-new, empty round-N branch,
inside the same classify step that creates it. GitHub's API rejects this outright —
`GraphQL: No commits between main and feature/r{N}-{slug} (createPullRequest)` — a PR cannot be
opened with zero diff between base and head. This is not an implementation bug fixable by retry;
it is the same real constraint round 1's own flow already works around by deferring PR creation
until after the first commit lands.
**Deviation from user-spec:** AC6/Constraints literally read "chỉ sau khi branch+PR+marker đã tạo
xong thành công, gỡ label" (label removed only after branch+PR+marker all succeed) — implying PR
creation happens before label removal. Since GitHub makes that ordering physically impossible at
branch-creation time, "PR" in that requirement is satisfied by the same deferred-creation
guarantee round 1 already relies on (the "Open the code PR" step is unconditionally reached once
code-writing succeeds) rather than a literal precondition checked at this exact point. Label
removal now gates on branch+marker only. This does not reopen Risk 6 (finalize/post-merge race) —
it reproduces the exact same narrow "branch exists, no PR yet, implementing in progress" window
round 1 has always had and tolerated (a stray comment in that window falls through to `skip`, same
as round 1's pre-existing behavior) — not a new race introduced by this feature.
