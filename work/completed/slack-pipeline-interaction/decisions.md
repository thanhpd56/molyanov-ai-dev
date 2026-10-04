# Решения: slack-pipeline-interaction

<!-- Добавляй запись только для существенного решения или отклонения от user-spec.

## Краткое название решения

**Что решили:** решение.
**Почему:** причина.
**Отклонение от user-spec:** нет / что изменилось и почему.
-->

## Где физически лежит scaffold control-plane репо внутри molyanov-ai-dev

**Что решили:** новая верхнеуровневая директория `control-plane/` (сиблинг `skills/`, `agents/`,
`scripts/`, `work/`) содержит полный scaffold для singleton control-plane репо: Worker source
(`control-plane/worker/`) и `control-plane/.github/workflows/bootstrap-project.yml` и
`control-plane/README.md` с инструкциями по одноразовому деплою. Пользователь копирует это
содержимое в реально созданный новый GitHub репо (`gh repo create` + push) при одноразовой
настройке аккаунта (Expected Behavior шаг 4).
**Почему:** user-spec прямо запрещает вендорить Worker в `skills/online-pipeline/` и запрещает
3-й репо — значит код должен физически жить "в репо điều phối", которого пока не существует.
`control-plane/` — новая верхнеуровневая директория по аналогии с существующим паттерном
`skills/project-initialization/assets/new-project/` (scaffold → cp → новый репо), но не внутри
какого-либо skill, т.к. Worker не ветуется per-project (в отличие от online-pipeline).
`scripts/sync-to-codex.sh`'s `SOURCE_PATHS=(CLAUDE.md skills agents commands)` не трогает
верхнеуровневые директории вроде `work/`/`scripts/` — `control-plane/` безопасно не участвует в
Claude/Codex sync, это обычный исходный код Worker, а не skill/agent.
**Отклонение от user-spec:** нет — user-spec описывает поведение/содержимое control-plane репо,
но не называет, где именно в molyanov-ai-dev этот scaffold временно живёт до переноса; это чисто
техническое размещение, не меняющее итоговое поведение системы.
