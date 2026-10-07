---
name: online-pipeline
description: |
  Runs the full feature lifecycle (request → interview → approve → implement → PR → finalize)
  through GitHub Actions and GitHub issues/PRs/comments only, with no local terminal or `claude`
  CLI session required for any step. Also switches a feature already in progress between local
  Claude Code CLI execution and this GitHub Actions pipeline, at any point in any stage.

  Use when: "bật online-pipeline", "setup online pipeline", "cài online-pipeline cho repo",
  "chạy user-spec qua GitHub Actions", "online pipeline github actions", "enable online pipeline",
  "chuyển feature này lên online", "chuyển feature này về local", "switch this feature to online",
  "switch this feature to local", "move this feature online", "move this feature back to local"

  This skill has three audiences: (1) a human asking to enable it on a repo — see Setup below; (2)
  `claude -p` running non-interactively inside a GitHub Actions job — see Stages below; (3) a local,
  interactive Claude Code CLI session asked to switch a feature between local and online execution
  — see Hybrid Local/Online Switch below. The workflow YAML always names which stage to run for
  audience (2).
---

# Online Pipeline

Reuses `user-spec-planning`, `code-writing`, and `documentation-writing` unchanged except for the
controlled exceptions listed in [Modified Skills](#modified-skills-and-why) below. This skill
is the GitHub-Actions-specific orchestration layer on top of them: it decides, per stage, what
non-interactive `claude -p` should do, and leaves PR/label/merge mechanics to deterministic bash in
the workflow files (not to the agent), so GitHub state never depends on the agent phrasing a `gh`
command correctly.

## Naming Contract

Every workflow and every stage agree on these derived names — recompute them, never invent a new
scheme:

- `slug` = kebab-case(issue title, truncated to 40 chars) + `-` + issue number (e.g.
  `add-dark-mode-42`). Deterministic from `issue.title` + `issue.number` alone, so any job can
  recompute it without a lookup table. The trailing segment after the last `-` is always the
  issue number.
- Feature folder: `work/{slug}/` (identical to local `user-spec-planning`/`code-writing` usage).
- Spec branch: `userspec/{slug}`, spec PR label: `userspec-spec`.
- Code branch: `feature/{slug}`, code PR label: `userspec-implement`.
- Status marker (written by the agent as its last action, read by the workflow, never read by
  another skill): `work/{slug}/logs/working/online-pipeline-status.yml`, a single line
  `status: {value}`. Values: `in_progress` (default — overwritten to this before every run, so a
  crash leaves no stale success signal), `awaiting_decision` (stopped for a user decision),
  `ready_for_review` (userspec stage only — validation clean, spec PR should be opened/updated),
  `ready_for_pr` (implement stage only — implementation complete; code PR opened on first success,
  or updated in place on a later `implement-followup` round, see Stages below).
- Automation signal: the literal string `ONLINE_PIPELINE_AUTOMATED`, included in every
  `userspec-turn` and `finalize` prompt (the two stages that call a modified skill). Not needed for
  `implement`/`implement-resume`/`implement-followup` — `code-writing` does not check for it.
- Local-placeholder label: `local-placeholder`, applied only to the GitHub issue (never a PR) for a
  feature currently being worked on locally. Created defensively wherever it's used, exactly like
  `userspec-spec`/`userspec-implement`. Removed early in every switch-to-online procedure, right
  before the online artifact that follows it (the `/switch-online` comment in the mid-interview and
  mid-implement cases, the spec or code PR in the userspec-boundary and implement-boundary cases) —
  see Hybrid Local/Online Switch below.
- Switch handoff comment: the literal string `/switch-online`, posted on the issue by a local
  switch-to-online action. Distinct from `ONLINE_PIPELINE_AUTOMATED` above (that signal only ever
  appears inside `userspec-turn` and `finalize` prompts, never `implement`/`implement-resume`/
  `implement-followup`);
  `/switch-online` is read by the `userspec-turn` and `implement-resume` stage prompts themselves,
  not by any route job — route jobs never need to read comment content to dispatch correctly.
- `knowledge-init` label: applied only to the GitHub issue (never a PR) that drives Stage:
  knowledge-init below — by `bootstrap-project.yml` in the separate `control-plane` repo right
  after it vendors secrets onto a freshly bootstrapped repo, or by a manual Slack slash command on
  an existing repo. Created defensively wherever it's applied, exactly like
  `local-placeholder`/`userspec-spec`/`userspec-implement`. The route job in
  `online-pipeline-userspec.yml` ignores any issue/comment carrying it, the same way it already
  ignores `local-placeholder` — the two labels never dispatch to the same stage.
- Slack mapping marker (only present on a feature started via Slack's `/new-feature`, see Slack
  Bridge below): the first line of the issue body, exactly
  `<!-- slack: channel_id={channel_id} thread_ts={thread_ts} -->`. A feature started any other way
  (a plain issue, or the local-placeholder flow) never has this line — that absence is exactly how
  the relay helper below detects "this repo/feature has no Slack thread to mirror into."

The agent communicates with the user exclusively through `gh issue comment {issue_number} --body
"..."` — chat/stdout output is not read by anyone in a CI job. Every stage prompt supplies the
issue number explicitly; use it for every comment. Never call `relay_to_slack` yourself: on a repo
onboarded to Slack (see Slack Bridge below), the workflow mirrors every one of these comments to
the Slack thread deterministically after you exit — see Slack Bridge for why.

## Why Skills Must Be Vendored Into the Target Repo

A GitHub Actions runner has no `~/.claude/skills/` or `~/.claude/agents/` — those only exist on a
developer's own machine. `claude -p` running in a CI job can only use a skill or agent that
physically exists in the checked-out repo's own `.claude/skills/` or `.claude/agents/`. Every
stage here drives `user-spec-planning`, `code-writing`, `documentation-writing`, or `test-master`,
which between them spawn nine reviewer/research agents (`code-researcher`,
`interview-completeness-checker`, `skeptic`, `userspec-quality-validator`,
`userspec-adequacy-validator`, `code-reviewer`, `security-auditor`, `test-reviewer`,
`documentation-reviewer`) — all of it, plus this skill itself, must be vendored into the target
repo's `.claude/skills/` and `.claude/agents/`. Every stage prompt in this file references the
skill it needs by its *vendored* path, `.claude/skills/{name}/SKILL.md` — not this source repo's
own bare `skills/{name}/SKILL.md` layout, which only resolves inside this specific repo.

`scripts/vendor-skills.sh` does this by cloning `https://github.com/thanhpd56/molyanov-ai-dev`
(public, no auth) and copying the exact closure listed above, plus refreshing the three
`.github/workflows/online-pipeline-*.yml` files themselves — it is the single update path for both
the automation logic and the skills/agents driving it. Re-run it anytime to pick up upstream
changes. It refuses to run over uncommitted local edits to anything it would overwrite (commit or
stash first), validates the full closure is present in the source before changing anything (no
partial update), and never commits on its own — review the diff and commit when ready.

The `finalize` stage has one further runner-only dependency beyond skills/agents:
`documentation-writing`'s Feature Finalization Mode runs `~/.claude/scripts/sync-to-codex.sh`
whenever it updates Project Knowledge (the normal path, not an edge case). That script isn't a
skill or agent, so `vendor-skills.sh` doesn't carry it — `online-pipeline-finalize.yml` installs it
directly onto the runner's `$HOME` from the same source repo before invoking `claude -p`.

## Setup (human-facing)

To enable online-pipeline on an existing repository, resolve this skill's directory and run
`scripts/setup-online-pipeline.sh` from the target repo root. It vendors the workflow files and
their full skill/agent closure (via `vendor-skills.sh`, see above), creates the labels the workflows
and the local/online switch apply, offers to set the needed secrets via `gh secret set`, and prints
the manual steps the user must still do in a browser (see the script's own output for the
authoritative, current list).

A project scaffolded by `project-initialization` already has the workflow files in
`.github/workflows/` and a vendored snapshot of this skill closure in `.claude/skills/` /
`.claude/agents/` (see that skill's `assets/new-project/`) — do not run the setup script there;
only the secrets and the manual browser steps remain. That vendored snapshot can still go stale
the same way an existing project's can, so `scripts/vendor-skills.sh` (already present at
`.claude/skills/online-pipeline/scripts/` in a scaffolded project) is the update path there too.

## Slack Bridge (Online Pipeline over Slack)

Optional, additive layer: a repo that has never run this (no `SLACK_RELAY_TOKEN`/`SLACK_WORKER_URL`
secrets set) behaves exactly as before — GitHub issue/PR comments only. A repo that has it gets
every GitHub-side interaction mirrored into a Slack thread, driven by one account-wide Cloudflare
Worker (source lives in a separate `control-plane` repo, never in this one — see
`work/slack-pipeline-interaction/user-spec.md` and that repo's own `README.md` for the Worker's
own setup and request-routing logic; this section only covers what *this* skill's stages do).

**No `on:` trigger changes.** All three workflow YAMLs keep the exact triggers they already have.
The Worker drives interaction by calling the GitHub REST API directly (create issue, post
`/approve` comment, merge a PR) or by relaying into Slack — it never needs a new GitHub Actions
event to fire any of this.

**`relay_to_slack` — the one mirror helper, called only from deterministic workflow YAML bash,
never by the agent:**

```bash
relay_to_slack() {
  local issue_number="$1" text="$2"
  if [ -z "${SLACK_RELAY_TOKEN:-}" ] || [ -z "${SLACK_WORKER_URL:-}" ]; then
    return 0  # repo never onboarded Slack — not an error, just nothing to do
  fi
  local issue_body channel_id thread_ts
  issue_body="$(gh issue view "$issue_number" --json body -q .body)"
  channel_id="$(printf '%s' "$issue_body" | grep -oP '(?<=channel_id=)\S+' || true)"
  thread_ts="$(printf '%s' "$issue_body" | grep -oP '(?<=thread_ts=)\S+' || true)"
  if [ -z "$channel_id" ]; then
    return 0  # this feature has no Slack thread (started directly on GitHub, or local-placeholder)
  fi
  curl -fsS -X POST "$SLACK_WORKER_URL/relay" \
    -H "X-Relay-Token: $SLACK_RELAY_TOKEN" -H "Content-Type: application/json" \
    -d "$(jq -n --arg c "$channel_id" --arg t "$thread_ts" --arg x "$text" \
          '{channel_id:$c, thread_ts:$t, text:$x}')" || true  # relay failure must never fail the run
}
```

`SLACK_RELAY_TOKEN` and `SLACK_WORKER_URL` are repo secrets set by the (now Slack-aware)
`setup-online-pipeline.sh`; every workflow job that runs `claude -p` exports them.

No stage prompt in this file ever calls `relay_to_slack` itself — every one of them (`userspec-turn`,
`implement`/`implement-resume`/`implement-followup`, `knowledge-init`) originally did, by reproducing this bash block via
its own tool calls on every turn, but in production this proved unreliable: the agent would
sometimes post the `gh issue comment` and commit, but skip the relay call, so the message landed on
GitHub and never reached Slack. Every one of these call sites was moved into deterministic workflow
bash instead, each diffing `gh issue view --json comments` before/after the `claude -p` call(s) and
relaying every newly-posted comment whose `.author.login` is exactly `github-actions` (the identity
behind `secrets.GITHUB_TOKEN` — confirmed empirically against real comment data; note this is
*not* `"github-actions[bot]"`, the display name used elsewhere). Filtering by that login excludes a
human's own GitHub comment and the Worker's own mirror of a Slack reply (posted with a real user's
PAT), either of which landing on the issue mid-run would otherwise get echoed straight back into
the same thread.

Telling the agent not to self-relay is prose, not a guarantee — in production the agent kept
calling `relay_to_slack` itself anyway often enough to cause real duplicate Slack posts (the
deterministic relay delivering the message once, the agent's own reproduction delivering it again).
Every `claude -p` invocation in `online-pipeline-userspec.yml`, `online-pipeline-implement.yml`,
and `online-pipeline-knowledge-init.yml` now blanks `SLACK_RELAY_TOKEN`/`SLACK_WORKER_URL` for that
child process specifically (`SLACK_RELAY_TOKEN="" SLACK_WORKER_URL="" claude -p ...`) — the surrounding
shell step keeps the real values for its own deterministic relay call after `claude -p` exits, but
the agent's own environment never has them, so any self-relay attempt is a harmless no-op (the
`relay_to_slack` reference implementation's own `-z` guard) or at worst an unauthorized 401, never
a second real post. This is the actual enforcement; the prose instruction documents intent only.
the same Slack thread.

**Mirror points — all deterministic workflow bash, never agent-executed:**

- `online-pipeline-userspec.yml`'s "Run userspec-turn" step, after the single `claude -p` call
  (interview question / `awaiting_decision` stop), and its "Open or update the spec PR" step's
  closing `gh issue comment` (spec PR ready, `/approve` instructions) — the latter already ran as
  deterministic bash and is unaffected by this change.
- `online-pipeline-implement.yml`'s "Implement with retry" step, after every `claude -p` attempt
  (`awaiting_decision` question or finding), and its "Report technical failure" step after 3 failed
  attempts — the latter already ran as deterministic bash and is unaffected. Its "Push followup
  commit and confirm" step (`implement-followup` only, see Stages below) relays its own
  confirmation comment the same deterministic way, directly — that comment is posted after
  "Implement with retry" already exited, so it falls outside that step's own before/after
  comment-count diffing and needs its own `relay_to_slack` call.
- `online-pipeline-knowledge-init.yml`'s "Run knowledge-init turn with retry" step, after every
  `claude -p` attempt, and its "Report technical failure" step after 3 failed attempts — the
  latter already ran as deterministic bash and is unaffected.
- `online-pipeline-finalize.yml`'s "Report technical failure" step, after 3 failed attempts — this
  one never involved the agent calling `relay_to_slack` either; unaffected.

**The one exception — Slack-only, no GitHub equivalent:** when `finalize` completes successfully,
`online-pipeline-finalize.yml` posts "✅ Feature xong" (or equivalent) via `relay_to_slack` only.
There is no matching `gh issue comment` call for this (the GitHub issue is simply closed/archived
by the finalize commit) and none should be added — this is the single net-new capability in this
mirror set, not a gap to "fix" by also adding a GitHub-side post.

**`!approve` and new-feature/new-project creation are entirely Worker-side**, not something any
stage prompt here does: the Worker computes `slug` from `issue.title` + `issue.number` using the
exact same formula as this file's Naming Contract, looks up the two possible open PRs
(`userspec/{slug}`, `feature/{slug}`), and either posts the real `/approve` comment (spec PR — the
existing route job in `online-pipeline-userspec.yml` handles it unchanged) or calls the merge REST
endpoint directly (code PR — new capability, no workflow trigger needed since
`pull_request: closed` doesn't distinguish merge actor/method). The Worker also filters out any
Slack event carrying `bot_id` before treating it as a relay candidate or an `!approve`, so its own
posts into a thread never loop back as a duplicate `gh issue comment` or a spurious approve. None
of this is implemented in this skill or these workflows — it lives entirely in the control-plane
repo's Worker source.

## Rate-Limit Auto-Switch (control-plane token pool)

Optional, additive on top of the Slack Bridge above — a repo without `SLACK_RELAY_TOKEN`/
`SLACK_WORKER_URL` set simply never matches the Worker-call branch below and keeps today's
behavior (retry the same token on every technical failure). See
`work/control-plane-token-switch/user-spec.md` and the separate `control-plane` repo's own
`worker/src/index.js` for the full contract; this section only covers what the three stages below
do, entirely in deterministic workflow bash — the agent itself never calls this endpoint and is
unaware it exists.

**What triggers it.** After every `claude -p` attempt in all three stages
(`online-pipeline-userspec.yml`, `online-pipeline-implement.yml`,
`online-pipeline-knowledge-init.yml`), the workflow bash checks the attempt's captured stdout/stderr
for one of the two fixed, confirmed-real Claude rate-limit strings: `"You've hit your session
limit"` (5-hour limit) or `"You've hit your weekly limit"` (7-day limit). Only these two fixed
prefixes are matched — never a guessed variant — so a wording change on GitHub's side simply fails
to match instead of mis-firing (see Risk 1 in the user-spec). No match: the attempt is handled
exactly as any other technical failure, unchanged.

**On a match**, the workflow extracts the `"resets {time} ({tz})"` segment from the same output and
calls `POST $SLACK_WORKER_URL/switch-token/auto` with `X-Relay-Token: $SLACK_RELAY_TOKEN` (the same
secret already used for `/relay` — no new secret) and body `{owner, repo, rateLimitResetText}`:

- `{ok:true, alias, token}` — the real token for a different pool alias. The workflow uses it as
  `CLAUDE_CODE_OAUTH_TOKEN` for the *next* `claude` call (a job's own env var is fixed for its whole
  lifetime and never self-refreshes, so every subsequent invocation in the loop is explicitly
  prefixed with the current value held in a shell variable, not the original secret). That next call
  is a **rate-limit resume** (see Stage-specific notes below), not a fresh technical-failure retry:
  it does **not** count against the stage's `ATTEMPTS=3` budget, and none of the stage's reset/clean
  step runs first — the file edits (and any local commit) from the rate-limited attempt are left
  exactly as they were.
- `{ok:false, error:"no_alias_available"}` — every pool token is currently rate-limited. The
  workflow stops immediately instead of burning the remaining attempts on a token known to still be
  limited: it posts a distinct message via `gh issue comment` + the Slack relay ("Mọi token trong
  pool đều đang bị rate-limit...") and exits non-zero, separately from the generic "failed after 3
  attempts" message technical-error exhaustion posts.
  Any other outcome (network error, timeout, 5xx, or any other `error` value such as
  `"set_secret_failed"`) is treated the same as "Worker call failed": the attempt falls back to the
  existing retry-same-token behavior, exactly as if no rate-limit pattern had matched at all.

**Stage-specific notes:**

- All three stages (`implement`/`implement-resume`/`implement-followup`, `knowledge-init`,
  `userspec-turn`) handle a
  rate-limited attempt the same way: a **rate-limit resume**. Instead of resetting and retrying from
  scratch, the workflow re-invokes `claude -p -c`/`--continue` with the new token, in the same
  working directory, with a short prompt announcing the token refresh instead of the stage's normal
  prompt — Claude Code's own session continuity (`-c`) picks up the exact same conversation where it
  left off (see Risk 3/3b in `work/smart-resume-on-token-limit/user-spec.md`: this job's own `claude`
  process is always the only one active in this working directory, and the local session transcript
  carries no credential/account binding, so resuming under a different pool token is safe). This
  loops with no hard cap: every successful switch is one more rate-limit resume, logged as
  `"Rate-limit resume #N — continuing previous session with new token"` to stay visually distinct
  from the stage's normal `"... attempt X/3"` log line; it only stops when the Worker reports
  `SWITCH_EXHAUSTED` (pool exhausted, see above).

  If the `-c` resume call itself fails for a reason **other** than a rate-limit match (e.g. a CLI
  error, "session not found"), that is a normal technical failure: it falls back to the stage's
  existing reset-and-retry-from-scratch behavior and counts against `ATTEMPTS=3`, exactly as before
  this mechanism existed.

  **Known limitation:** the reset baseline (`$LAST_GOOD`/`origin/main`) is captured once before the
  loop and never advances after a successful rate-limit resume. If one or more resumes succeed and
  then a later *plain* technical failure resets, the rollback discards the resumed work too, not
  just the failing attempt's. Accepted for v1 (see Risk 1 in the user-spec) — the common single-
  rate-limit case is unaffected.
- `userspec-turn` previously called `claude -p` exactly once per issue comment with no retry loop at
  all — any technical error failed the whole run immediately. It now has the same
  `ATTEMPTS=3`-with-reset loop as the other two stages. Because a normal continuing interview turn
  leaves the status marker at `in_progress` (not a distinct "done" value), the loop treats "posted
  at least one new issue comment this attempt" as its success signal, on top of `ready_for_review` —
  only an attempt that posted nothing at all and left the status unchanged counts as a technical
  failure worth retrying.

**Rate-limit resume vs. "resume"/`implement-resume`.** These are two unrelated mechanisms that never
interact, despite both using the word "resume": `implement-resume` (`ROUTE_ACTION == 'resume'`) is a
brand-new GitHub Actions job/runner, triggered by a human answering a mid-implementation decision
question via an issue comment — there is no CLI session involved. Rate-limit resume happens entirely
**within** one already-running job's attempt loop, continuing the same local Claude Code CLI session
(`-c`/`--continue`) under a new token. A rate-limit mid-session never changes `ROUTE_ACTION` or the
status marker, and a decision-resume job never has rate-limit session state to continue (it starts
on a fresh runner with no prior local session). `implement-followup` (`ROUTE_ACTION == 'followup'`)
is a third, equally unrelated mechanism: a brand-new job dispatched by a new change request on an
already-open PR, not a continuation of any stopped session either.

**Manual pool management** (`/add-token <alias> <token>`, `/remove-token <alias>`, `/switch-token
[<alias>]`) is entirely Slack-side, implemented in the `control-plane` repo's Worker — not in this
skill or these workflows. `/new-project`-bootstrapped repos also get an initial alias from the pool
at bootstrap time (`bootstrap-project.yml`), falling back to the fixed `control-plane` secret copy
only when the pool is empty.

## Stages

### Stage: userspec-turn

Covers interview, completeness check, draft, and validate — i.e. `user-spec-planning` Steps 1–5.
Triggered once per issue comment (or once on issue open) by `online-pipeline-userspec.yml`, which
retries you up to `ATTEMPTS=3` times on a plain technical failure (see Rate-Limit Auto-Switch
above) — each retry is a fresh attempt from the same branch state, not a resume of a half-finished
one.

1. The workflow has already reset the status marker to `in_progress` before invoking you (on every
   attempt, not just the first); always
   write a final status before exiting, even on a plain interview turn that is neither a stop nor a
   completion (leave it `in_progress` in that case).
2. If `work/{slug}/user-spec.md` does not exist yet: this is the first run for this issue. Call
   `user-spec-planning`'s Start path directly with the given `slug` (do not let it choose its own
   slug) and the issue body as the initial task description — strip the Slack mapping marker line
   (see Naming Contract) first if present; it is metadata for `relay_to_slack`, not part of the
   description.
3. Otherwise: call `user-spec-planning`'s Resume path. Treat the latest issue comment supplied in
   the prompt as the newest user answer — append it through the normal interview loop exactly as a
   live chat reply would be, with one exception: if that comment is exactly `/switch-online`, it is
   a local switch-to-online handoff, not an answer — do not append it as one. Simply continue the
   interview loop as usual and post the next pending question (or next step) as a new comment.
4. Whenever `user-spec-planning` would "ask the user" (interview questions, a completeness-checker
   gap, a `user_decision_required` validation finding), post the question with `gh issue comment`
   instead of chat output — do not also call `relay_to_slack` yourself (Slack Bridge above); the
   workflow mirrors it after you exit. Write `status: awaiting_decision` to the status marker
   *before* this commit, not after:
   commit+push `logs/userspec/interview.yml` and the status marker (and any other changed state) to
   `userspec/{slug}` immediately, then exit 0. Writing status after the push would leave it
   uncommitted and invisible to any later run — each GitHub Actions run is a fresh checkout with no
   memory of this one, including no local disk continuity. This is a normal stop, not a failure.
5. Append `ONLINE_PIPELINE_AUTOMATED` to the context you hand `user-spec-planning` for Step 5
   (validate) so its automated-resume branch activates; it is a no-op for the other steps.
6. Run Steps 1–5 exactly as `user-spec-planning/SKILL.md` defines them, with one exception: **do
   not run Step 6 ("Obtain Approval")**. That step is replaced entirely by the PR + `/approve`
   comment flow below — do not ask for approval in a comment or chat.
7. When Step 5 validation ends clean (all three reviewers `clean`): write `status: ready_for_review`
   to the status marker, then commit and push it with `userspec/{slug}` as usual, and exit 0. The
   workflow opens or updates the spec PR after you exit — do not call `gh pr create` yourself.
8. If an existing spec PR already exists for this branch (e.g. the user commented again after
   review asking for a change, which `user-spec-planning` Step 6's own rule routes back to
   validation), just keep pushing to the same branch; the open PR updates automatically.

### Stage: implement

Triggered once, when the spec PR (branch `userspec/{slug}`, label `userspec-spec`) merges. The
workflow has already created `feature/{slug}` from `main` and checked it out.

1. Read the approved `work/{slug}/user-spec.md` and `decisions.md`. Implement it using the
   `code-writing` skill's normal process (including its review waves) against the current
   repository — nothing online-pipeline-specific changes how `code-writing` itself works.
2. If a reviewer in `code-writing`'s review waves returns a finding with `user_decision_required:
   true` that is not a technical/tooling error: write `status: awaiting_decision` to the status
   marker, then commit it together with the work completed so far to `feature/{slug}` and push,
   then post the question with `gh issue comment {issue_number}` — do not also call
   `relay_to_slack` yourself (Slack Bridge above); the workflow mirrors it after you exit. Then
   exit 0. **Write the status
   marker and commit it before pushing, never after** — the next run that resumes this stop is a
   fresh checkout on a different runner with no disk continuity from this one; it can only see
   `awaiting_decision` if that commit actually reached `origin/feature/{slug}`. A status write that
   happens after the push, or that is never committed, is invisible to that later run and silently
   breaks the resume path. This stop is not a technical failure — the workflow does not count it
   against the 2-retry cap.
3. On successful completion: write `status: ready_for_pr` to the status marker, commit it with all
   remaining work to `feature/{slug}`, push, and exit 0. The workflow opens the code PR (label
   `userspec-implement`) after you exit — do not call `gh pr create` yourself.
4. Any other exit (crash, uncaught tool error, non-zero exit code without a committed status write)
   is a technical failure. The workflow resets `feature/{slug}` to the last pushed commit and
   retries you from there — commit the status marker together with everything else defensively
   before anything risky, so a mid-run crash loses as little as possible.

### Stage: implement-resume

Same as `implement`, except `feature/{slug}` already has a checkpoint commit from a prior
`awaiting_decision` stop, and the prompt supplies the latest issue comment as the answer to the
question you asked last time. Continue from the existing branch state — do not restart the
implementation from `main`. The same three exit outcomes (steps 2–4 above) apply.

If that comment is exactly `/switch-online`, it is a local switch-to-online handoff, not an answer
to any question — this feature was implemented entirely locally up to this point (no prior
`awaiting_decision` stop ever happened online) and the status marker was just created fresh by that
switch action. Run this the same as step 1 of stage `implement` instead: read the approved
`work/{slug}/user-spec.md` and `decisions.md` and continue implementing with `code-writing` against
the branch's current code (the local work already pushed to it) — do not treat the comment as an
answer to a question that was never asked.

### Stage: implement-followup

Triggered by a new issue comment while `feature/{slug}`'s code PR is still `OPEN` and the status
marker already reads `ready_for_pr` — the normal "implementation already finished, PR open" state.
The route job tells this apart from `implement-resume` purely from the status marker value (see
Naming Contract): `awaiting_decision` dispatches `implement-resume` (an answer to a question this
pipeline itself asked); `ready_for_pr` with the PR still open dispatches this stage instead (a new,
unprompted request from the user).

1. Decide first whether the latest issue comment is a genuine request for a code change, or just an
   acknowledgement/closing remark with nothing to act on (e.g. "ok cảm ơn"). If it is **not** a real
   change request: post a short reply with `gh issue comment {issue_number}` and stop there — do
   not run `code-writing`, do not commit anything. The workflow already overwrote the local copy of
   `work/{slug}/logs/working/online-pipeline-status.yml` to `in_progress` before this run started
   (its "Prepare the feature branch" step, same bookkeeping every `implement`/`implement-resume` run
   gets); write `ready_for_pr` back into that local file before exiting so the workflow reads this
   run as a normal stop rather than a technical failure — but do **not** commit or push it: `origin`
   already holds `ready_for_pr` from before this run (that is exactly why this stage was dispatched
   in the first place), so there is nothing new to persist.
2. Otherwise, treat the comment as a brand-new code-change request against the existing
   `feature/{slug}` branch and its current code — never as an answer to a previously asked
   question, even if an earlier `awaiting_decision` round happened at some point in this feature's
   history. Run `code-writing`'s normal process (including its review waves), exactly as stage
   `implement` does in its step 1.
3. The same three exit outcomes as stage `implement` (steps 2–4) apply unchanged: a review-wave
   finding with `user_decision_required: true` writes `status: awaiting_decision`, commits, pushes,
   and asks via comment exactly like step 2 there (the next comment then goes through
   `implement-resume`, not another `implement-followup` round); successful completion writes
   `status: ready_for_pr`, commits all remaining work, and pushes exactly like step 3 there — do
   **not** post a confirmation comment yourself here; the workflow's own "Push followup commit and
   confirm" step posts it, only after verifying your push actually landed on `origin` (see
   `online-pipeline-implement.yml`); any other exit is a technical failure retried by the workflow
   exactly like step 4 there.

### Stage: finalize

Triggered once, when the code PR (branch `feature/{slug}`, label `userspec-implement`) merges. Runs
on `main` directly — there is no branch for this stage.

1. Append `ONLINE_PIPELINE_AUTOMATED` to the context you hand `documentation-writing`'s Feature
   Finalization Mode. Its automated branch skips "ask whether to continue finalization" and always
   continues — the merge itself is the user's confirmation.
2. Run `documentation-writing`'s Feature Finalization Mode exactly as written otherwise: update
   Project Knowledge, move `work/{slug}/` to `work/completed/{slug}/`, commit.
3. Commit straight to `main` (there is no finalize branch) and push. If anything fails before this
   final commit/push, `main` is untouched — the workflow simply reruns this stage from a fresh
   checkout, no reset needed. If it fails after push (e.g. the connection drops right after), the
   next run's `work/completed/{slug}/` existence check (done by the workflow, not by you) already
   detects completion and skips re-running you; you do not need your own idempotency check.

### Stage: knowledge-init

Triggered once per issue comment (or once on issue open) by
`online-pipeline-knowledge-init.yml`, for any issue carrying the `knowledge-init` label. Runs on
`main` directly — there is no branch for this stage, and no feature slug; it drives
`documentation-writing`'s initial-documentation flow for the whole repository, not a single
feature.

As with every other stage, never call `relay_to_slack` yourself here (see Slack Bridge above) —
just post the comment; the workflow mirrors it to Slack deterministically after you exit.

1. Always start by running `create-project-knowledge.md`'s Phase 0 (Start or Resume), regardless of
   how many times this stage has already run for this issue. Phase 0's own repository/
   configuration/`CLAUDE.md`/current-Project-Knowledge inspection is the decisive completion check
   in this automated context: if that inspection shows Project Knowledge is no longer missing, a
   template, or only partially filled — whether because this very flow already finished it, a
   `documentation-writing` full update/audit completed it, or someone edited it by hand — skip the
   interview entirely. Post "Project Knowledge đã được khởi tạo rồi." (or equivalent) with `gh issue
   comment {issue_number}`, then call `gh issue close {issue_number}` yourself. Only after that,
   unconditionally — create
   `work/project-knowledge/interview.yml` with `interview_metadata.status: completed` if it does
   not exist yet, or otherwise just refresh its `interview_metadata.last_updated` timestamp even
   when `status` already reads `completed` and nothing else about it needs to change — and commit
   that file straight to `main` and push **as the last action of this run** (so Worker's own cheap
   check in `control-plane`, which only reads this field, can see the repo is done too). Every hit
   of this shortcut must produce a real commit, with no exception — including a repeat hit on a
   reopened issue or a stray comment arriving after the issue is already closed — so the workflow's
   retry loop (which only trusts a landed commit, never issue/comment state, to tell a correct
   repeat from a crash) always sees one. Do not reverse this order: commit+push is always the last
   action, exactly like step 3 below — never close or comment after it. This check never trusts
   `interview_metadata.status` alone for the *decision* to skip the interview, since a repo can
   reach full Project Knowledge through routes that never set that field — only for what to write
   once the decision is already made from the real inspection above.
2. Otherwise, continue `create-project-knowledge.md` exactly as written, from whatever point its
   own Phase 0 resume logic lands on, using the first run's issue body (strip the Slack mapping
   marker line first, see Naming Contract) or the latest issue comment supplied in the prompt as
   the newest user input, the same way `userspec-turn` treats these two sources.
3. Every point where `create-project-knowledge.md` asks the user anything — a question batch, a
   cycle checkpoint, the final-summary checkpoint, the documentation-approval checkpoint — post it
   with `gh issue comment {issue_number}` instead of chat output, then commit
   `work/project-knowledge/interview.yml` (and nothing else) straight to `main` and push as the
   last action of this run, then exit 0. Do not
   invent any literal signal to skip or auto-confirm a checkpoint — every one of them round-trips
   through a real comment, exactly like `userspec-turn`'s questions.
4. When "Write the Documentation" finishes and the user has approved it: write the real Project
   Knowledge files in the working tree and set `interview_metadata.status: completed`, then post
   the completion comment with `gh issue comment {issue_number}`, then call `gh issue close
   {issue_number}` yourself. Only after all of that, run
   `~/.claude/scripts/sync-to-codex.sh --project "$PWD" --apply` (this turn always touches
   `.claude/**`/`CLAUDE.md`, same requirement as `documentation-writing/SKILL.md`'s Manual Project
   Documentation Sync) and commit everything — the Project Knowledge files plus generated
   `.codex/**`/`AGENTS.md` output — straight to `main` and push **as the last action of this run**;
   do not open a PR or any branch, and do not comment/relay/close after this commit. Skip
   `create-project-knowledge.md`'s own "Return to Documentation Review in the main skill" step
   entirely in this automated context — stop right after that final commit instead of returning to
   that menu.
5. Any run that crashes or exits without the commit and/or issue-close described above is a
   technical failure. The workflow retries it (reset to `origin/main`, try again) up to twice, then
   reports the error via `gh issue comment` + `relay_to_slack` and leaves the issue open — it never
   rolls back `main`. This mirrors `finalize`'s retry loop, not `implement`'s branch-reset one,
   since both stages work straight on `main` with no branch of their own.

## Hybrid Local/Online Switch (local-facing)

Audience: a local, interactive Claude Code CLI session. Any feature in a project with
online-pipeline enabled can move between local execution and this GitHub Actions pipeline at any
point, in any of the three stages, including mid-stage. This section is read only by that local
session — the GitHub Actions jobs above never read it.

### Start path always creates a placeholder issue first

`user-spec-planning/SKILL.md`'s Start path creates an empty GitHub issue (label
`local-placeholder`) for every new feature started interactively on a project with online-pipeline
enabled, and derives `work/{slug}` from that issue's number using this file's own Naming Contract —
see that skill's Start path for the exact mechanics. This exists purely so a switch can happen at
any later moment without first having to construct an equivalent issue/branch/label by hand. While
the issue still carries `local-placeholder`, `online-pipeline-userspec.yml`'s route job ignores it
entirely for both `issues: opened` and `issue_comment: created` — no interview comment is ever
auto-posted while the user is working locally.

### Switch-to-online

Triggered by a natural-language request in the local chat (see this file's frontmatter "Use when").
Determine which case applies from the feature's current local state, then follow that case exactly:

1. **Mid-interview** (`work/{slug}/user-spec.md` does not yet have `status: approved`):
   1. Commit any uncommitted interview/draft state in `work/{slug}/` if not already committed (e.g.
      `chore(userspec): snapshot before switch-to-online for {slug}`).
   2. Push current `HEAD` to `userspec/{slug}` on `origin` — create it if it doesn't exist yet,
      update it in place if it does: `git push origin HEAD:refs/heads/userspec/{slug}`.
   3. Remove the `local-placeholder` label from the issue.
   4. Post `gh issue comment {issue_number} --body "/switch-online"` — no real answer, even if the
      user already answered the pending question out loud; a real answer, if any, is a separate
      comment that follows.
   The next `issue_comment` event (this comment itself) dispatches `userspec-turn` normally; its
   stage prompt recognizes the literal `/switch-online` body and continues the interview loop
   instead of treating it as an answer (see stage `userspec-turn` above).
   If step 4 fails after steps 2–3 already succeeded (label removed, branch pushed, no comment
   landed): do not treat the switch as successful — report the error. Re-apply the
   `local-placeholder` label to the issue before stopping, so `online-pipeline-userspec.yml`'s route
   job keeps skipping events on it until you retry; otherwise a stray comment arriving in the
   meantime (bot notification, an accidental comment, a differently-worded retry) would dispatch
   `userspec-turn`, which would misread it as the real answer to whatever question the interview
   last recorded. See "Comment failure and revert" below for the equivalent handling in case 3.

2. **Userspec boundary** (`status: approved` locally, but `userspec/{slug}` has never existed on
   `origin` and no `feature/{slug}` work has started): push `userspec/{slug}` with the approved
   `user-spec.md`/`decisions.md`, remove the `local-placeholder` label, then open the spec PR
   yourself exactly as `online-pipeline-userspec.yml`'s "Open or update the spec PR" step does
   (`gh pr create --base main --head userspec/{slug} --label userspec-spec ...`, creating the label
   defensively first if missing). The user reviews and comments `/approve` on that PR through the
   normal online flow — no new mechanism needed here.

3. **Mid-implement** (code-writing has started against the approved spec but has not finished):
   1. Check whether `feature/{slug}` already exists on `origin`
      (`git ls-remote --exit-code --heads origin feature/{slug}`) **and remember the answer** — the
      revert procedure below needs this exact fact, and by the time it might run, step 3 has already
      pushed the branch, so re-running this same check then would always report "exists" regardless
      of what was true before this attempt. Do not re-derive it later; carry it forward.
   2. Read the current status marker if `feature/{slug}` already exists online. Unless it already
      reads exactly `status: awaiting_decision`, write `status: awaiting_decision` to
      `work/{slug}/logs/working/online-pipeline-status.yml` — the value
      `online-pipeline-implement.yml`'s route job compares the status marker against to dispatch
      `implement-resume` (as opposed to `implement-followup`, which it dispatches instead when the
      marker reads `ready_for_pr` with the PR still open — see Stages below) — and commit it together with the current
      code state. Dispatch depends on exactly this value every time this case runs, not only the
      first; leaving a stale `ready_for_pr`/`in_progress`/other value in place would make the route
      job silently `action=skip` with no error shown anywhere. If it already reads
      `awaiting_decision`, leave it untouched (it may reflect genuine in-flight online state) and
      just commit the code changes.
   3. Push current `HEAD` to `feature/{slug}` on `origin`:
      `git push origin HEAD:refs/heads/feature/{slug}`.
   4. Remove the `local-placeholder` label from the issue — **before** the next step: the route job
      reads the label from the comment event's own payload snapshot, so removing it after would
      still show the old label to that event.
   5. Post `gh issue comment {issue_number} --body "/switch-online"`.
   If step 5 fails after steps 2–3 already pushed, see "Comment failure and revert" below — do not
   leave the branch at `awaiting_decision` with no comment following it.

4. **Implement boundary** (code-writing finished locally against the approved spec, but
   `feature/{slug}` has never existed on `origin`): push `feature/{slug}` with the finished code,
   remove the `local-placeholder` label from the issue, then open the code PR yourself exactly as
   `online-pipeline-implement.yml`'s "Open the code PR" step does (`gh pr create --base main --head
   feature/{slug} --label userspec-implement ...`, creating the label defensively first if missing).
   The user merges it like any online PR to trigger `finalize` online — no status marker or
   `/switch-online` comment needed for this case.

5. **Finalize**: there is no mid-finalize switch. `finalize` commits straight to `main` with no
   checkpoint, so switching here only ever means choosing where to run it (local, see "Local
   finalize" below, or online — merge the code PR as usual) before starting it, never during it.

#### Push conflicts

If pushing in any of the cases above is rejected because `origin` has commits the local branch
doesn't: `git fetch origin`, then try `git merge` (or `rebase`, consistently with whichever the user
normally prefers) the remote branch into the local one. If it merges cleanly, push again and
continue the case above. If it produces a real conflict, stop and show the conflicted files/hunks to
the user to resolve in chat — never pick a side automatically.

#### Comment failure and revert

This covers both case 1 and case 3 above — the two cases that remove the label and post
`/switch-online` before anything online can confirm the handoff landed.

**Case 1 (mid-interview):** see the revert instruction already given inline in case 1 step 4 above
(re-apply `local-placeholder`). The rest of this subsection is specific to case 3.

**Case 3 (mid-implement):** the real `online-pipeline-implement.yml` route job dispatches
`implement-resume` from branch-existence plus the status marker alone; it never reads comment
content (recognizing `/switch-online` only happens inside the stage prompt, after dispatch already
happened). So if posting the `/switch-online` comment fails (network loss, insufficient `gh`
permission) after the branch+marker push already succeeded: do not treat the switch as successful.
GitHub is now in a state that looks ready to dispatch forever, waiting for a comment that may never
arrive — any unrelated comment landing on that issue in the meantime would be misread as the answer
to a question that was never asked. Revert immediately, using the fact you already determined and
carried forward from case 3 step 1 (do not re-check `git ls-remote` now — the branch exists either
way after step 3's push, so a fresh check cannot tell these two apart):

- If you are confident `feature/{slug}` already existed online before this switch attempt (the fact
  carried forward from case 3 step 1): push the status marker back to whatever value it held before
  this attempt (or `in_progress` if that value itself is unknown) so the branch no longer reads
  `awaiting_decision`.
- If you are confident this switch attempt created `feature/{slug}` for the first time: delete the
  branch from `origin` (`git push origin --delete feature/{slug}`) instead of leaving a
  half-initialized one.
- If you are not confident which of the two is true (e.g. a push-conflict resolution or anything
  else intervened between step 1 and now, long enough that you no longer trust your own carried-
  forward memory of it): default to the first, non-destructive option — reset the marker away from
  `awaiting_decision`, do not delete the branch. Resetting the marker alone is always enough to stop
  the route job from dispatching, whether or not the branch turns out to be one you created fresh;
  the only downside of resetting instead of deleting a branch you did create is a stray near-empty
  branch left on `origin`, a far smaller and already-accepted class of leftover (see "Switch-to-local"
  below on abandoned online branches) than destroying one that held real prior work.

Report the comment failure clearly to the user either way — the switch did not happen.

If this revert step **also** fails (e.g. the network is still down): do not retry in a loop. Report
a loud, explicit error telling the user exactly what to fix by hand — `feature/{slug}` may be stuck
reading `awaiting_decision` with no real handoff behind it; delete the branch or fix the status
marker on GitHub directly before reusing that issue. This is an accepted rare double-failure case
for v1, not a case to build automatic recovery for.

A successful ordinary switch (push then comment, no errors) still has a very small timing gap
between the two — a comment landing in that exact window could theoretically be misread the same
way. This is an accepted rare risk for v1; only an outright comment-post failure gets the explicit
revert above.

### Switch-to-local

Triggered by a natural-language request in the local chat while the feature is currently being
worked on through GitHub (mid-interview-by-comment, or mid-implement possibly at
`awaiting_decision`):

1. Check the local working tree is clean (`git status --porcelain`). If it has uncommitted changes,
   stop immediately and report the error — ask the user to commit or stash first. Never stash
   automatically and never checkout over uncommitted work.
2. If clean: `git fetch origin`, then checkout whichever of `feature/{slug}` or `userspec/{slug}`
   currently exists and is further along (prefer `feature/{slug}` if both exist).
3. Read that branch's state — the status marker if present, or
   `work/{slug}/logs/userspec/interview.yml` otherwise — and report plainly what is unresolved (the
   pending interview question, or the pending implementation decision/finding).
4. Continue immediately in the same local chat: `user-spec-planning`'s Resume path for an
   interview-stage feature, or `code-writing`'s normal process for an implementation-stage one.

Switching to local does not re-arm `local-placeholder` or any other block on the issue — this is an
accepted v1 limitation. If the user switches online again later and then finishes entirely locally
without merging the online PR/branch created along the way, that PR/branch is simply left open;
local finalize below still completes normally regardless.

### Local finalize: closing the placeholder issue

When `documentation-writing`'s Feature Finalization Mode completes locally (unchanged, run exactly
as that skill defines it) for a feature whose slug came from this project's placeholder-issue flow
(online-pipeline enabled when the feature was started): after the finalize commit, close the issue —
issue number is the trailing segment of the slug. Before closing, verify that issue actually is this
feature's placeholder: `gh issue view {issue_number} --json title -q .title`, kebab-case and
truncate it the same way the Naming Contract does, and confirm it matches the slug with the trailing
`-{issue_number}` removed. This guards a narrow but real case — a project that enabled
online-pipeline only after some local-only features already used freely chosen slugs could, purely
by coincidence, have one such slug end in digits that match a real, unrelated issue number; closing
by number alone could close that unrelated issue. If the title doesn't match, skip closing and log
why instead. If the titles do match, run `gh issue close {issue_number}`; if that fails, log/report
it but do not roll back the already-completed finalize — the archive under `work/completed/{slug}/`
stands either way. This applies regardless of whether the feature ever switched online and back; it
is keyed on where finalize itself ran (locally), not on switch history. Finalize running through
`online-pipeline-finalize.yml` is unaffected and keeps its current behavior — no auto-close is added
there.

## Modified Skills and Why

Two of the three changes below are gated on the literal string `ONLINE_PIPELINE_AUTOMATED`
appearing in the prompt context and do not change interactive (local) behavior at all. The third
(Start path) deliberately does change local behavior — that is its entire purpose — but only on a
project that has online-pipeline enabled, and only when called interactively.

- `user-spec-planning/SKILL.md` Step 5 and `assets/interview.yml.template`: when automated and a
  reviewer returns `user_decision_required`, the round number and the pending finding are saved to
  `interview.yml` before commenting and stopping, so the next automated run resumes the same round
  (not round 1). Step 3 is untouched — it runs once, with no round concept, so nothing is lost
  when a session ends there.
- `documentation-writing/SKILL.md` Feature Finalization Mode: when automated, the "feature looks
  incomplete, continue?" question is skipped and finalization always continues.
- `user-spec-planning/SKILL.md` Start path: on a project with online-pipeline enabled, an
  interactive (slug not already supplied) call creates a placeholder GitHub issue and derives the
  slug from it instead of choosing one freely — see Hybrid Local/Online Switch above for why. Not
  gated on `ONLINE_PIPELINE_AUTOMATED`; gated on online-pipeline being enabled and no slug having
  been supplied by the caller instead.

## Setup Script Maintenance

If `assets/workflows/*.yml` changes (new secret, new label, new branch-naming scheme), update this
file's Naming Contract and `scripts/setup-online-pipeline.sh`'s printed instructions together —
they describe the same contract from two angles and will drift silently otherwise. Also copy the
updated files over
`skills/project-initialization/assets/new-project/.github/workflows/` (plain duplicates, not a
build step or symlink — matches how that scaffold already vendors
`.claude/skills/project-knowledge/`) and update the matching steps in that scaffold's `README.md`.

If a stage starts depending on a different skill or agent (the closure listed in
[Why Skills Must Be Vendored](#why-skills-must-be-vendored-into-the-target-repo)), update
`vendor-skills.sh`'s `SKILLS=(...)`/`AGENTS=(...)` arrays and re-run it over
`skills/project-initialization/assets/new-project/.claude/` (that scaffold's copy is a vendored
snapshot like any other project's, not a live link — it goes stale the same way and is refreshed
the same way).
