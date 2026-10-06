# Решения: online-pipeline-github-actions

<!-- Добавляй запись только для существенного решения или отклонения от user-spec.

## Краткое название решения

**Что решили:** решение.
**Почему:** причина.
**Отклонение от user-spec:** нет / что изменилось и почему.
-->

## PAT secret for the `/approve` merge step

**Что решили:** `/approve` merges the spec PR using a dedicated PAT secret (`GH_PAT`), not the
default `GITHUB_TOKEN`. This is a 3rd required manual setup step (create a Personal Access Token
with `repo` scope) in addition to the 2 the user-spec names (install the Claude GitHub App, create
a Claude auth token).
**Почему:** GitHub does not let the default `GITHUB_TOKEN` trigger downstream workflow runs
(anti-recursion protection, confirmed in `claude-code-action`'s own FAQ). If the spec-PR merge used
`GITHUB_TOKEN`, the merge would succeed but the implement workflow (triggered by `pull_request:
closed` on the spec PR) would never fire — silently breaking the acceptance criterion "`/approve` →
PR merged → job implement bắt đầu". Confirmed with the user before implementing.
**Отклонение от user-spec:** да — Constraints/Verification describe exactly 2 manual setup steps;
there are actually 3. `scripts/setup-online-pipeline.sh` and the setup instructions name all 3.

## Vendor the skill/agent closure into each target repo

**Что решили:** `online-pipeline` requires every target project to carry its own vendored copy,
under `.claude/skills/` and `.claude/agents/`, of `user-spec-planning`, `code-writing`,
`documentation-writing`, `test-master`, `online-pipeline` itself, and the nine agents they spawn
(`code-researcher`, `interview-completeness-checker`, `skeptic`, `userspec-quality-validator`,
`userspec-adequacy-validator`, `code-reviewer`, `security-auditor`, `test-reviewer`,
`documentation-reviewer`). A new script, `skills/online-pipeline/scripts/vendor-skills.sh`, does
this by cloning the public `molyanov-ai-dev` repo (no auth needed) and copying that closure, plus
refreshing the three `online-pipeline-*.yml` workflow files — it is also the update mechanism: the
project owner re-runs it anytime to pull the latest version, review the diff, and commit. It
refuses to run over uncommitted changes to anything it would overwrite and validates the full
closure is present in the source before changing anything. `setup-online-pipeline.sh` now calls it
instead of copying workflow files itself. `project-initialization`'s new-project scaffold ships a
static snapshot of the same closure by default, with the same update path documented in its
`README.md`. `online-pipeline-finalize.yml` additionally installs
`~/.claude/scripts/sync-to-codex.sh` directly onto the runner (not vendored as a skill/agent,
fetched from the same source repo) because `documentation-writing`'s Feature Finalization Mode
depends on it whenever it updates Project Knowledge, which is the normal finalize path.
**Почему:** A GitHub Actions runner has no `~/.claude/skills/`, `~/.claude/agents/`, or
`~/.claude/scripts/` — those exist only on a developer's own machine (confirmed: this repo's own
`skills/`/`agents/` directories are the version-controlled source a developer periodically mirrors
into their global `~/.claude/`, not something a CI runner has access to at all). Without vendoring,
every `claude -p` invocation in every online-pipeline workflow would fail to find the skill it was
told to use, and finalize would additionally fail trying to run a sync script that doesn't exist.
This was discovered only after the initial implementation was already reviewed (twice) and
committed — found by the user asking how to test it, surfaced to the user with three options, and
the user chose vendoring. The `sync-to-codex.sh` sub-gap and three vendor-skills.sh correctness
issues (silent loss of uncommitted local edits, no update path for the workflow files themselves,
partial-update risk on an incomplete source) were found by a follow-up code-reviewer pass on this
fix itself and corrected the same way.
**Отклонение от user-spec:** да — none of this (vendoring, `vendor-skills.sh`, the extra
`.claude/skills/`/`.claude/agents/` content in the new-project scaffold, or the
`sync-to-codex.sh` runner install step) is in the approved user-spec. It is a necessary correction
to make the approved behavior actually work in the GitHub Actions environment the whole feature
targets, confirmed with the user before implementing.
