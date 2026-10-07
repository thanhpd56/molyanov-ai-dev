# Code Research — online-pipeline-implement-followup

## Problem, confirmed in code

In the `userspec` stage, a comment on the GitHub issue keeps working **after** the spec PR is
already open. `online-pipeline-userspec.yml`'s route job (`skills/online-pipeline/assets/workflows/online-pipeline-userspec.yml:94-126`)
computes `action=continue` for *any* `issue_comment` as long as `work/{slug}/user-spec.md` does not
yet have `status: approved` — it never checks the in-progress status marker. So a user commenting
on the issue after `status: ready_for_review` (spec PR already open) still dispatches
`userspec-turn`, which (per `SKILL.md` Stage: userspec-turn, step 3/8) treats the new comment as the
next interview answer, lets `user-spec-planning` Step 6's own rule route back to validation, and the
open spec PR just updates in place. This is exactly the "keep chatting after the artifact exists"
behavior the user wants — it already exists, but only for the spec stage.

In the `implement` stage this is missing. `online-pipeline-implement.yml`'s route job
(`skills/online-pipeline/assets/workflows/online-pipeline-implement.yml:19-65`) only reacts to an
`issue_comment` event by checking whether `feature/{slug}` exists and whether its status marker
reads exactly `awaiting_decision`:

```
if [ -f "work/$SLUG/logs/working/online-pipeline-status.yml" ] && grep -q '^status: awaiting_decision' ...; then
  echo "action=resume"
else
  echo "action=skip"
fi
```

Once the implement run finishes successfully it writes `status: ready_for_pr`
(`SKILL.md` Stage: implement, step 3) and the workflow opens the code PR
(`online-pipeline-implement.yml:312-324`, "Open the code PR" step). From that point on, *any* later
issue comment — including one relayed from the Slack thread asking for an extra change, a fix, or an
optimization — hits `action=skip` in the route job and nothing happens. This is the exact gap the
user is reporting: "sau khi code xong tạo mr thì ... chat thêm vào slack không thấy chạy thêm gì."

Also note: the route job's `if:` condition
(`skills/online-pipeline/assets/workflows/online-pipeline-implement.yml:20-22`) requires
`github.event.issue.pull_request == null` — i.e. it only ever looks at comments on the **issue**,
never comments posted directly on the PR. This matches the Slack Bridge design: the Worker relays a
Slack thread reply by posting a `gh issue comment` on the originating issue (keyed by the
`channel_id`/`thread_ts` markers in the issue body, `SKILL.md` Naming Contract), not on the PR. So
"chat tiếp qua Slack" always arrives as an issue comment, consistent with how the existing
`userspec-turn` continuation already works.

## Relevant naming/status contract (`skills/online-pipeline/SKILL.md`)

- Status marker: `work/{slug}/logs/working/online-pipeline-status.yml`, single line `status: {value}`.
  Current implement-stage values: `in_progress`, `awaiting_decision`, `ready_for_pr`. No value exists
  today for "PR open, idle, further requests welcome" distinct from `ready_for_pr` itself.
- `ONLINE_PIPELINE_AUTOMATED` signal is only added to `userspec-turn` and `finalize` prompts, never to
  `implement`/`implement-resume` (`SKILL.md` Naming Contract) — the implement stage prompt itself
  carries no such literal signal today, so a new "followup" action would need its own stage-name
  signal the same way `implement-resume` already gets one (`STAGE_NAME` env + prompt header in
  `online-pipeline-implement.yml:209-212,230-242`).
- Concurrency group `online-pipeline-implement-${slug}` with `cancel-in-progress: false`
  (`online-pipeline-implement.yml:71-73`) already queues a second triggering event for the same
  feature instead of racing it — directly reusable for a follow-up comment arriving while a prior
  run (initial implement, or an earlier follow-up) is still in flight.
- `implement-resume`'s own "stale" guard (`online-pipeline-implement.yml:110-123`) re-checks the
  status marker *after* acquiring the concurrency gate, because the route job's snapshot can go stale
  if two comments queue back-to-back. The same re-check pattern will be needed for a new
  `ready_for_pr`-triggered action, for the same reason.
- `relay_to_slack` mirroring in the "Implement with retry" step
  (`online-pipeline-implement.yml:264-276`) already diffs `gh issue view --json comments` before/after
  every `claude -p` attempt and relays any new `github-actions`-authored comment — this is
  stage-agnostic and needs no change to support a new action value.
- `finalize` (`online-pipeline-finalize.yml`) uses `work/completed/{slug}` existence on `main` as its
  own idempotency check (`online-pipeline-finalize.yml:37-48,103-107`). The `implement` route job has
  no equivalent check today — it only looks at `feature/{slug}` branch existence and the status
  marker on that branch. If a code PR has already merged (triggering `finalize`, which does not
  delete `feature/{slug}` itself — only `userspec/{slug}` is deleted via `--delete-branch` in
  `online-pipeline-userspec.yml:169`), a stray late comment on the now-closed issue could still find
  `feature/{slug}` existing with a stale `ready_for_pr` marker. A new followup action keyed only on
  `ready_for_pr` + branch existence would need an explicit "is the code PR still open" check (e.g.
  `gh pr view "feature/$SLUG" --json state`) to avoid firing after merge.

## Reusable precedent: `implement-resume`

The cleanest existing template for "take a new issue comment and keep working on the same open PR
branch" is `implement-resume` itself (`SKILL.md` Stage: implement-resume, and the `resume` branch of
`online-pipeline-implement.yml`'s route/implement jobs). The differences a new "continue after PR
open" action would need from `implement-resume`:

1. Route condition: dispatch when status is `ready_for_pr` (and the code PR is still open) instead of
   `awaiting_decision`.
2. Prompt framing: the new comment is a **new request** (add/change/optimize), not an **answer to a
   previously-asked question** — `implement-resume`'s prompt explicitly frames `COMMENT_BODY` as "the
   answer to the question you asked last time" (`online-pipeline-implement.yml:237-241`), which would
   be factually wrong for this case.
3. Success exit: after a follow-up change, the branch already has an open PR
   (`gh pr list --head "feature/$SLUG" --state open`, reused from the existing "Open the code PR" step
   at `online-pipeline-implement.yml:316-324`) — pushing new commits to `feature/{slug}` updates that
   PR automatically; no second `gh pr create` call should run (the existing step already guards this
   with `if [ -z "$EXISTING" ]`).

## Files relevant to implementation

- `skills/online-pipeline/SKILL.md` — Naming Contract (status marker values) and Stage: implement /
  implement-resume sections; needs a new stage description for the follow-up case.
- `skills/online-pipeline/assets/workflows/online-pipeline-implement.yml` — route job (lines 19-65)
  and the `implement` job's prompt-building block (lines 148-301); needs the new action and prompt
  framing.
- `skills/project-initialization/assets/new-project/.github/workflows/online-pipeline-implement.yml`
  — vendored duplicate that must be kept in sync per `SKILL.md`'s own "Setup Script Maintenance"
  section.
- `.codex/skills/online-pipeline/` and `.codex/skills/project-initialization/.../online-pipeline/` —
  mirrored Codex-runtime copies of the same files (confirmed present via `find`); same dual-runtime
  sync concern applies to any skill-content change (not a code change, so likely handled by the
  existing `sync-to-codex.sh` tooling rather than manual duplication — needs confirming during
  implementation, not blocking for the spec).
