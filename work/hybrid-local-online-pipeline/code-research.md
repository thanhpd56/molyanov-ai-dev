# Code Research: hybrid-local-online-pipeline

## Scope
How the existing `online-pipeline` GitHub Actions automation and the local
`user-spec-planning`/`code-writing` skills work today, to ground a new 2-way local<->online
switching capability.

## GitHub Actions trigger conditions (confirmed, no `push` trigger exists anywhere)

- `skills/online-pipeline/assets/workflows/online-pipeline-userspec.yml:7-11`: `on: issues:
  [opened]` and `issue_comment: [created]`. The `route` job (`:19-20`) filters comments to
  `github.event.comment.user.type != 'Bot'`. `action` (`start`/`continue`/`approve`/`skip`) is
  derived purely from branch existence + a `status: approved` grep (`:69-96`) — no status-marker
  file check for the userspec stage.
- `online-pipeline-implement.yml:7-11`: `on: pull_request: [closed]` and `issue_comment:
  [created]`. Route condition (`:20-22`) is (a) PR merged=true with label `userspec-spec` (first
  implement run), or (b) a non-bot comment on a plain issue (`github.event.issue.pull_request ==
  null`). **Correction (verified directly against the file, line ~61):** condition (b) alone is
  NOT sufficient — the "Determine stage" step greps the on-branch status marker and only sets
  `action=resume` (dispatching implement-resume) when it literally matches `^status:
  awaiting_decision`; any other value (including a plain comment with no marker, or one that says
  `in_progress`) resolves to `action=skip`, and the job does nothing. This is load-bearing for the
  hybrid switch design: any switch-to-online mechanism that posts a comment to wake up
  implement-resume must first ensure the status marker on `feature/{slug}` reads exactly
  `awaiting_decision`, or the comment is silently ignored.
- `online-pipeline-finalize.yml:7-9`: `on: pull_request: [closed]` only, gated (`:22`) on `merged
  == true && contains(labels, 'userspec-implement')`.
- `/approve` comment handling is PR-scoped (not issue-scoped) and merges via `GH_PAT`, not
  `GITHUB_TOKEN`, because the default token can't trigger downstream workflow runs
  (`online-pipeline-userspec.yml:138`; also `work/online-pipeline-github-actions/decisions.md:12-24`).

**Implication for hybrid switching**: no new GitHub Actions trigger type is needed. The existing
`issue_comment` trigger already fires on any non-bot comment for both userspec and implement
stages — a dedicated token (`/switch-online`) added to the *content* check inside the route job
(not to the `on:` event itself) is enough to signal "resume from local handoff," reusing 100% of
existing plumbing. Boundary-of-stage switches (spec approved locally, or implementation finished
locally) can bootstrap the *next* stage by producing the exact PR+label combination
(`userspec/{slug}`+`userspec-spec`, `feature/{slug}`+`userspec-implement`) that the existing
`pull_request: closed` triggers already expect — no new trigger required there either.

## `vendor-skills.sh` / `setup-online-pipeline.sh`

`vendor-skills.sh:27-29` hardcodes the closure: 5 skills (`user-spec-planning code-writing
documentation-writing test-master online-pipeline`), 9 agents, 3 workflow files. Overwrites
`.claude/skills/`, `.claude/agents/`, `.github/workflows/online-pipeline-*.yml`; refuses over
uncommitted changes to those paths. Neither script touches `work/`, and neither has any
"convert a local feature folder into online-pipeline shape" capability today — this is new scope.

`setup-online-pipeline.sh` vendors + creates 2 labels (`userspec-spec`, `userspec-implement`) +
offers 3 secrets (`CLAUDE_CODE_OAUTH_TOKEN`/`ANTHROPIC_API_KEY`, `GH_PAT`) + prints manual
GitHub-App-install steps. No issue-related logic exists here.

## `code-writing/SKILL.md` — zero git discipline

No mention of `git`, `branch`, or commit anywhere in the file. Purely: understand → implement/
verify → up to 2 review waves. No checkpoint/resume concept. All commit/push/status-marker
discipline seen during online `implement`/`implement-resume` is imposed externally by
`online-pipeline/SKILL.md`'s stage instructions, not by `code-writing` itself. A hybrid
switch-to-online mid-implementation must impose that same external discipline from the *local*
side (the switch mechanism, not `code-writing`).

## `init-feature-folder.sh`

`skills/user-spec-planning/scripts/init-feature-folder.sh` — signature `<feature-name>
[work-dir]`, no issue number, no git interaction at all, idempotent (skips existing files), makes
no assumption about current branch. The new "create issue first, compute slug from it" step must
happen *before* this script is invoked, passing it the already-computed slug — the script itself
does not need to change.

## Project scaffold ("asleep by default")

Documented in `skills/project-initialization/assets/new-project/README.md:41-63` (not in
`project-initialization/SKILL.md` itself, which has no online-pipeline mentions). Scaffold ships
the 3 workflow YAMLs plus a vendored skill/agent snapshot; "inert until setup is finished" (3
manual steps: GitHub App, auth secret, `GH_PAT`). Practical detection signal for "online-pipeline
enabled" on a project: presence of `.github/workflows/online-pipeline-userspec.yml`. This is
necessary-but-not-fully-sufficient (secrets/App install aren't detectable from repo files alone),
accepted as a known limitation — same class of gap the original online-pipeline feature already
lives with for its own setup detection.

## No prior art for local `gh issue create`/`gh pr create`

Zero hits for `gh issue create` anywhere in the repo. `gh pr create` exists only inside the
workflow YAMLs themselves (deterministic bash, never agent-driven — `online-pipeline/SKILL.md`
explicitly forbids the automated agent from calling it). The hybrid feature's local-side issue/PR
creation is new, user-triggered, interactive-session behavior — a different trust context from the
automated agent's existing restriction, so it does not conflict with that rule.

## `decisions.md` vs `user-spec.md` (online-pipeline-github-actions precedent)

`work/online-pipeline-github-actions/decisions.md` carries materially new facts not present in
`user-spec.md`'s Accepted Decisions (the `GH_PAT` requirement, and the full vendoring-mechanism
discovery written up post-approval) — i.e. `decisions.md` is not a duplicate of the spec's
Accepted Decisions section; it's where mid-implementation deviations get recorded. Expect the same
pattern here if implementation surfaces something not anticipated in interview.
