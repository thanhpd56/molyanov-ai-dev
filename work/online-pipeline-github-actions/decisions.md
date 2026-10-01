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
