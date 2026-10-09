# Code Research: online-pipeline-postmerge-followup

Scope: decouple `finalize` from the code-PR-merge event; let the issue stay open and accept
repeated comment-driven mini feature/fix cycles (new branch + new PR each time) after the code PR
merges, until some explicit signal triggers `finalize`. Facts only, no proposed design.

All source paths below are the canonical copies under `skills/online-pipeline/assets/workflows/` and
`skills/online-pipeline/SKILL.md`. Two vendored mirrors exist and are kept byte-identical per
`SKILL.md`'s "Setup Script Maintenance" section (not separately cited below unless they differ):
`skills/project-initialization/assets/new-project/.github/workflows/online-pipeline-*.yml` and
`skills/project-initialization/assets/new-project/.claude/skills/online-pipeline/assets/workflows/online-pipeline-*.yml`.

## 1. Exact current finalize trigger

`skills/online-pipeline/assets/workflows/online-pipeline-finalize.yml:7-22`:

```yaml
on:
  pull_request:
    types: [closed]

concurrency:
  group: online-pipeline-finalize
  cancel-in-progress: false

jobs:
  finalize:
    if: github.event.pull_request.merged == true && contains(github.event.pull_request.labels.*.name, 'userspec-implement')
```

So the filter for "this is THE code PR for a feature" is exactly two conditions on the
`pull_request: closed` event payload: `merged == true` and label `userspec-implement` present.
There is no branch-name check (`feature/*`) in this `if:` — the branch pattern is only used
afterward, to derive the slug, not to decide whether to run.

Slug derivation (`online-pipeline-finalize.yml:37-48`, step "Derive slug and check idempotency"):

```bash
SLUG="${HEAD_REF#feature/}"
echo "slug=$SLUG" >> "$GITHUB_OUTPUT"
echo "issue_number=${SLUG##*-}" >> "$GITHUB_OUTPUT"
if [ -d "work/completed/$SLUG" ]; then
  echo "already_done=true" >> "$GITHUB_OUTPUT"
else
  echo "already_done=false" >> "$GITHUB_OUTPUT"
fi
```

`HEAD_REF` comes from `env.HEAD_REF: ${{ github.event.pull_request.head.ref }}` (line 28) — i.e.
the merged PR's own head branch name, read directly from the event payload, stripped of the
`feature/` prefix. The trailing segment after the last `-` is the issue number (per the Naming
Contract). No `gh pr view`/`gh pr list` call is needed for this stage at all — everything comes
straight from the webhook payload.

**This stage does not run at all on `implement`'s own trigger** — `implement`'s merge-detection
pattern (point 2 below) lives entirely inside `online-pipeline-implement.yml`'s own `route` job and
is unrelated code; `finalize` has its own separate, simpler `pull_request: closed` trigger with no
route job at all (single `finalize` job, gated by the `if:` above).

`already_done` gates every subsequent step (`if: steps.slug.outputs.already_done == 'false'` on
every step from "Select Claude auth method" onward, e.g. line 51, 63, 68, 72, 88, 140, 159). If
`work/completed/$SLUG` already exists, the whole job is a silent no-op success (no step runs, no
error) — this is the stage's only idempotency mechanism today, and it is driven purely by a
directory existing on `main`, not by any issue/PR state.

## 2. Route job mechanics in `online-pipeline-implement.yml`

Single `route` job, gated by (`online-pipeline-implement.yml:20-22`):

```yaml
if: |
  (github.event_name == 'pull_request' && github.event.pull_request.merged == true && contains(github.event.pull_request.labels.*.name, 'userspec-spec')) ||
  (github.event_name == 'issue_comment' && github.event.comment.user.type != 'Bot' && github.event.issue.pull_request == null)
```

So this workflow fires on two distinct events: the **spec** PR merging (label `userspec-spec`,
not `userspec-implement` — that is the trigger that *starts* `implement`), and any non-bot comment
on an **issue** (not a PR comment — `github.event.issue.pull_request == null` excludes PR-thread
comments).

Dispatch logic, step "Determine stage" (`online-pipeline-implement.yml:33-76`):

- `pull_request` event (spec PR merged): slug from `HEAD_REF#userspec/`, `action=start`
  unconditionally (lines 44-50). No branch/status check at all for this path — it's a one-shot
  trigger.
- `issue_comment` event: slug recomputed from `ISSUE_TITLE` + `ISSUE_NUMBER` (line 52, same
  slugify formula used everywhere). Then:
  1. `git ls-remote --exit-code --heads origin "feature/$SLUG"` (line 56) — if `feature/{slug}`
     does not exist on `origin` at all, `action=skip` immediately (lines 57-58). **This is the
     exact point where branch-deletion-after-merge matters** — see point 3 below.
  2. If it exists: `git fetch`+`checkout` that branch (lines 60-61), then read
     `work/$SLUG/logs/working/online-pipeline-status.yml`'s `status:` line via
     `grep -oP '(?<=^status: ).*'` (lines 63-64) — a plain local file read on the checked-out
     branch, **not** any `gh` API call.
  3. `status == awaiting_decision` → `action=resume` (line 66-67).
  4. `status == ready_for_pr` **and** `gh pr list --head "feature/$SLUG" --state open --json
     number -q '.[0].number'` returns a non-empty PR number (line 68) → `action=followup`. This
     *is* a `gh` API call (PR state is not derivable from the branch's local files) — the only
     place in this route job that calls `gh`.
  5. Anything else (including `ready_for_pr` but PR **not** open — i.e. merged or closed) →
     `action=skip` (lines 74-75). The code comment at lines 69-72 explicitly states this is how a
     late post-merge comment currently falls through to skip: "Once the code PR merges (finalize
     runs), `gh pr list --state open` stops matching here, so a late comment correctly falls
     through to `skip`."

So today there is **no** distinct state for "PR merged, issue still open, awaiting next comment" —
it is indistinguishable, at the route-job level, from "this feature was never implemented online at
all" or "this feature's branch was deleted" — all three collapse to `action=skip` via either the
branch-ls-remote check (step 1) or the PR-not-open fallthrough (step 5). The only data point that
would need to change to add a new state is: after step 1 confirms the branch exists (or doesn't)
and after reading `status`, also check whether `work/completed/$SLUG` exists on `main` (not on the
feature branch) — that is the one signal the `finalize`-idempotency check already uses (point 1
above) to know a feature is fully done; the route job here never reads it today.

The `implement` job itself re-checks staleness post-concurrency-gate for both `resume` and
`followup` (`online-pipeline-implement.yml:124-151`, "Prepare the feature branch" step) — this is
the same local-file-grep + `gh pr list` pattern as the route job, re-run because the route job's
snapshot can be stale by the time a queued job actually gets to run (see point 8, concurrency).

## 3. Does `feature/{slug}` get deleted on merge?

**Spec PR (`userspec/{slug}`)**: yes, explicitly —
`online-pipeline-userspec.yml:171-176`, step "Merge the spec PR":

```yaml
run: gh pr merge "${{ needs.route.outputs.pr_number }}" --merge --delete-branch
```

This is deterministic workflow bash triggered by the `/approve` comment (`action == 'approve'`),
not a human click — `--delete-branch` is hardcoded.

**Code PR (`feature/{slug}`)**: grepped the entire `skills/` tree for `delete-branch`/
`delete_branch`/`deleteBranch` — the **only** match in the whole repo is the one above, for the
*spec* PR. There is no `gh pr merge --delete-branch` call anywhere for the code PR in this
codebase. `online-pipeline-implement.yml`'s "Open the code PR" step (lines 352-364) only creates
the PR; nothing in this repo's workflows or `scripts/` ever merges it — merging the code PR is
either a human clicking "Merge" on the GitHub PR UI, or (per `SKILL.md` Slack Bridge section,
lines 217-227) the Worker in the separate `control-plane` repo calling "the merge REST endpoint
directly" — that Worker's source is not in this repository, so whether its merge call passes a
`delete_branch` option is unknown/unverifiable from this codebase.

Also checked `scripts/setup-online-pipeline.sh` in full (it is short, ends right after printing
manual browser steps) and grepped the whole repo for `"Automatically delete"`/`"delete head
branch"` (the GitHub repo-level setting that would auto-delete merged branches regardless of how
the merge happened): zero matches anywhere in `skills/`, `scripts/`, or any project scaffold. No
script in this repo sets that GitHub repository setting. So `feature/{slug}`'s survival after the
code PR merges depends entirely on: (a) GitHub's repo-level "Automatically delete head branches"
setting, which is off by default and never touched by any script here, or (b) a human manually
clicking "Delete branch" in the GitHub UI after merge, or (c) whatever the control-plane Worker's
merge call does (unverified, out-of-repo). None of these is controlled or guaranteed by
online-pipeline itself today.

Separately: the code PR's body (`online-pipeline-implement.yml:361-363`) is `"Auto-generated by
online-pipeline from work/$SLUG/user-spec.md. Review and merge to trigger finalize."` — it contains
no GitHub closing keyword (`Closes #N`, `Fixes #N`, etc.), so merging the code PR does **not**
auto-close the originating issue via GitHub's own linked-issue mechanism. The issue staying open
post-merge (requirement 2 in the task) is already true today, incidentally — nothing currently
closes it on code-PR merge.

## 4. `/approve` and PR-merge-via-Worker

**`/approve` (spec PR, this repo's own mechanism):** `online-pipeline-userspec.yml:53-71`
(route job, PR-comment branch) — a PR-thread comment (`IS_PR_COMMENT == true`, i.e.
`github.event.issue.pull_request != null`) is checked with a literal, case-sensitive string
equality: `if [ "$COMMENT_BODY" != "/approve" ]; then action=skip; fi` (line 54). If it matches,
the job additionally re-verifies `gh pr view "$PR_NUMBER" --json state` is `OPEN` and the PR
carries label `userspec-spec` (lines 59-64) before setting `action=approve`. This is 100%
deterministic bash string comparison in the route job — the agent (`claude -p`) is never invoked
for this decision at all; `approve`'s own job (lines 128-176) does the merge in pure bash, no LLM
call.

**Code PR merge:** per `SKILL.md` lines 217-227 (Slack Bridge section), `!approve` typed in the
Slack thread (not `/approve` — a different literal string, Slack-side) is resolved entirely by the
Worker in the separate `control-plane` repo: it computes `slug`, looks up whether `userspec/{slug}`
or `feature/{slug}` has an open PR, and for the code PR case "calls the merge REST endpoint
directly" — bypassing any GitHub Actions `pull_request: closed` trigger race, exactly as the task
description suspected. `SKILL.md:221-223` states explicitly this is "a new capability, no workflow
trigger needed since `pull_request: closed` doesn't distinguish merge actor/method" — i.e. the
Worker's direct REST merge and a human's manual UI merge are indistinguishable to
`online-pipeline-finalize.yml`'s trigger; both fire the same `pull_request: closed` event with
`merged: true`. None of the Worker's merge-call mechanics (including any `delete_branch` option)
are implemented in this repo — `SKILL.md:227` states this plainly ("None of this is implemented in
this skill or these workflows").

## 5. Precedent for "explicit command via comment"

Two existing literal-string comment commands, both recognized the same way — a deterministic bash
`==`/`!=` string comparison inside a route job's own shell step, never inside `claude -p`:

- `/approve` — `online-pipeline-userspec.yml:54`, shown above. Scoped to PR-thread comments only
  (`IS_PR_COMMENT == true`); a non-matching PR comment is `action=skip`.
- `/switch-online` — not a route-job match at all. Per `SKILL.md:58-63` and the `userspec-turn`
  stage prompt (`SKILL.md:339-343`) / `implement-resume` stage prompt (`SKILL.md:400-406`), this
  string is recognized **inside the stage prompt** (i.e. by the agent reading the latest comment
  text it was handed), not by any route job — `SKILL.md:62-63` states this explicitly: "read by
  the `userspec-turn` and `implement-resume` stage prompts themselves, not by any route job — route
  jobs never need to read comment content to dispatch correctly." The route jobs dispatch
  `userspec-turn`/`implement-resume` here based on branch/label/status state alone (same mechanism
  as point 2 above), and only once dispatched does the agent itself check the comment body for the
  literal `/switch-online` string to decide "this is a handoff signal, not an answer."

So there are two different patterns already in use for "a literal command string in a comment":
(a) deterministic route-job bash match that changes *dispatch* (`/approve`), and (b) the route job
dispatches on unrelated state and the *agent* then recognizes a literal string to change its own
*framing* of that same turn (`/switch-online`).

**`implement-followup`'s "real request vs. acknowledgement" decision — explicitly agent-judgment,
not a bash keyword filter.** From `work/online-pipeline-implement-followup/user-spec.md`'s Accepted
Decisions (lines 149-152): "Việc nhận định ‘comment có phải là yêu cầu thay đổi code thật hay chỉ
là lời đáp xã giao' giao cho agent tự quyết định trong stage prompt, không viết filter từ khoá cứng
trong bash — nhất quán với cách routing trong skill này luôn ‘dumb’/deterministic, mọi nhận định
nội dung nằm trong `claude -p`." (Translation: deciding "is this a real change request or just a
social acknowledgement" is left to the agent inside the stage prompt, not a hardcoded keyword
filter in bash — consistent with this skill's routing always being "dumb"/deterministic, with all
content judgment living inside `claude -p`.) This is implemented in `SKILL.md`'s Stage:
implement-followup step 1 (`SKILL.md:417-426`): the agent itself decides real-request vs.
acknowledgement ("ok cảm ơn" example given), and only a real request runs `code-writing`; a
non-request gets a short reply with no commit, no status change. The route job (point 2 above)
never attempts this judgment — it only ever checks `status`/PR-open-state to decide `followup` vs.
`resume` vs. `skip`, never comment content.

## 6. Project Knowledge / finalize side effects

`documentation-writing/SKILL.md:165-186`, "Feature Finalization Mode" (full section, 6 numbered
steps):

1. Read `user-spec.md`, `decisions.md`, implementation, git history; compare implemented result to
   spec.
2. If evidently incomplete: normally ask whether to continue; **skipped** when context includes
   literal `ONLINE_PIPELINE_AUTOMATED` (lines 170-174) — the merge itself stands in for user
   confirmation; the gap is noted in the finalization commit message instead, and finalization
   continues regardless.
3. Update only affected Project Knowledge through "Phases 2-3 and Documentation Review"; if Project
   Knowledge is missing, skip the doc update and continue archival anyway.
4. Remove active Project Knowledge/backlog links treating `work/{feature}/` as current.
5. Move `work/{feature}/` → `work/completed/{feature}/` after documentation review.
6. Commit the Project Knowledge changes + archive move in one commit; report.

Line 184: "This is the only mode that reads feature artifacts by default, archives a feature, or
creates the finalization commit."

**Idempotency / multi-PR safety, as implemented today in `online-pipeline-finalize.yml`:** the
stage's own idempotency is entirely external to `documentation-writing` itself — it is the
`already_done` check in point 1 above (`work/completed/$SLUG` existence on `main`). If finalize
were invoked more than once for the same feature today (e.g. a second `pull_request: closed`+
`merged:true`+`userspec-implement` event somehow fired again), the second run's "Derive slug and
check idempotency" step would set `already_done=true` and every subsequent step is skipped — a
silent no-op success, not an error, not a duplicate archive/commit. This mechanism is unconditional
on the *number* of merged PRs that preceded it; it only checks whether the archive move already
happened. So running `finalize` exactly once at the end of a chain of several merged
`feature/{slug}`-labeled PRs for the same issue is already safe under this exact check, **provided**
every one of those merges still carries the `userspec-implement` label and still produces a
`pull_request: closed` event with `merged: true` pointing at a `head.ref` the slug-stripping logic
(`HEAD_REF#feature/`) can parse — i.e. provided each later PR's branch name still follows the
`feature/{slug...}` naming pattern closely enough for `${HEAD_REF#feature/}` to yield something
whose trailing `-N` segment is still the right issue number. A branch name that isn't exactly
`feature/{slug}` (e.g. a mentioned `feature/{slug}-2` scheme) would still strip correctly as long as
it starts with `feature/` and still ends in `-{issue_number}`, since the slug variable itself isn't
checked against `work/{slug}/user-spec.md` existing — it's only used to build `work/completed/$SLUG`
and the `gh issue comment` issue number.

## 7. Status marker lifecycle

Full value list and every read/write site, confirmed by `SKILL.md`'s Naming Contract
(`SKILL.md:42-48`) cross-checked against the workflow YAML:

| Value | Set by (writer) | Read by (reader) |
|---|---|---|
| `in_progress` | Written as the *first* action of every attempt, before `claude -p` runs, in all three driving workflows: `online-pipeline-userspec.yml:308` (userspec-turn, inside the retry loop, each attempt), `online-pipeline-implement.yml:153` ("Prepare the feature branch", once per job before the retry loop — not per attempt) and implicitly re-read as the default fallback value (`echo in_progress`) at `online-pipeline-implement.yml:312` and `online-pipeline-userspec.yml:362`. Also the value `documentation-writing`/`code-writing` leave in place for a plain continuing turn that is neither a stop nor a completion (`SKILL.md:333`, stage userspec-turn step 1). | The retry loop's own `STATUS` variable in both workflows, compared against the terminal values below to decide success/stop/retry. |
| `awaiting_decision` | Written by the agent itself (`claude -p`), as its last action before commit+push, in stage `userspec-turn` (`SKILL.md:347-348`) and stage `implement`/`implement-resume`/`implement-followup` (`SKILL.md:375-384`, `SKILL.md:432-434`) whenever a reviewer/interview step needs a user decision. | `online-pipeline-implement.yml`'s route job (`online-pipeline-implement.yml:66-67`) — dispatches `action=resume` on this exact value; re-checked post-gate at `online-pipeline-implement.yml:132-133`. Also by `implement-resume`'s own stale-check (same lines) and the "Hybrid Local/Online Switch" case 3 local procedure (`SKILL.md:577-585`), which writes/checks this same value from a local session. |
| `ready_for_review` | Written by the agent, userspec stage only, when all three validators return clean (`SKILL.md:358-360`). | `online-pipeline-userspec.yml`'s "Open or update the spec PR" step (`online-pipeline-userspec.yml:469-473`) — only opens/updates the spec PR if this value is present; otherwise exits 0 without opening a PR. |
| `ready_for_pr` | Written by the agent, implement stage only, on successful completion — first-time implement (`SKILL.md:385-387`) or a later `implement-followup` round writing the same value again (`SKILL.md:408-411`, `434-436`). | `online-pipeline-implement.yml`'s route job (line 68, combined with the PR-open `gh pr list` check) to dispatch `action=followup`; the retry loop's own success check (`online-pipeline-implement.yml:314-317`); the "Open the code PR" step's implicit gate (`if: env.RESULT == 'success'`, line 352, which is only set from this status); the followup post-gate re-check (lines 145-150). |

All four values live in exactly one file, `work/{slug}/logs/working/online-pipeline-status.yml`,
on the feature's own branch (`userspec/{slug}` or `feature/{slug}`) — **never on `main`**, and
never in a location independent of that branch. This is the fact most relevant to a new post-merge
state: if `feature/{slug}` is deleted after the code PR merges (point 3 above — unverified either
way, but possible), this file is gone with it; nothing today reads or writes this status marker
from `main`, so a new post-merge-awaiting-next-comment state cannot reuse this file's existing
read/write mechanics unmodified if the branch may not survive. `finalize`'s own "is this feature
done" signal (point 1/6 above) deliberately does **not** use this status-marker file at all — it
uses `work/completed/$SLUG` directory existence on `main` instead, which is the one piece of this
feature's state that is guaranteed to survive regardless of what happens to any feature branch.

## 8. Concurrency groups

Exact `concurrency:` blocks, one per workflow/job:

- `online-pipeline-userspec.yml` job `approve` (line 137-139) and job `userspec` (line 182-184):
  **same group**, `online-pipeline-userspec-${{ needs.route.outputs.slug }}` (per-feature-slug),
  `cancel-in-progress: false` for both. Comment at lines 132-136 explains why both jobs share one
  group: a PR-comment's `issue.number` is the PR's own number (different from the originating
  issue's), so keying by slug (computed identically in both event shapes) is what prevents an
  `/approve` run and a same-feature "continue" run from racing on the same branch.
- `online-pipeline-implement.yml` job `implement` (lines 82-87):
  `online-pipeline-implement-${{ needs.route.outputs.slug }}` (per-feature-slug), shared across
  `start`/`resume`/`followup` — confirmed by `work/online-pipeline-implement-followup/user-spec.md`
  Accepted Decisions (lines 138-148): deliberately **one shared group**, not two, specifically so
  GitHub Actions' own single-job-per-group guarantee rules out two jobs (e.g. a `resume` and a
  `followup`) ever pushing to `feature/{slug}` concurrently. `cancel-in-progress:
  ${{ needs.route.outputs.action == 'followup' }}` — only a `followup` dispatch cancels whatever
  else is running/queued in the group; `start`/`resume` keep `cancel-in-progress: false` (queue,
  never cancel).
- `online-pipeline-finalize.yml` job `finalize` (lines 15-18): **fixed, global** group
  `online-pipeline-finalize` (not slug-scoped), `cancel-in-progress: false`. Comment at line 16
  explicitly: "Fixed, shared across every feature so two finalize runs never race on Project
  Knowledge" — i.e. this group intentionally serializes finalize runs across *all* features in the
  repo, not just one feature's own.
- `online-pipeline-knowledge-init.yml` job (lines 19-23): **same fixed global group** as
  `finalize`, `online-pipeline-finalize` — not a group of its own. Comment at lines 20-22: "Shared,
  fixed group with finalize (not a group of its own): both stages commit straight to `main` and can
  both touch Project Knowledge, so they must never run in parallel on the same repo."

So today there are exactly three distinct concurrency groups in use: two per-slug
(`online-pipeline-userspec-{slug}`, `online-pipeline-implement-{slug}`) and one fixed/global
(`online-pipeline-finalize`, shared by both `finalize` and `knowledge-init` because both write to
`main`). Any new stage that also writes to `main` directly would need to consider joining that
fixed global group (to avoid racing Project Knowledge writes); any new stage that instead works on
a per-feature branch (like a new `feature/{slug}-N` branch per round) would more closely match the
per-slug pattern already used by `implement`.
