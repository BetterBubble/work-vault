---
title: closing-comments-block3-2026-08-07
type: note
permalink: tacticum/00-board/closing-comments-block3-2026-08-07
---

=== #640
Закрываю: `116252e1` «iva-kmp-brownfield 0.2.2 — agent efficiency (O1-O4)» и следом `486e757a`
(0.2.3), который перенёс батчинг из repo-config фрагментов в тела агентов. Состав по задаче на
месте целиком: Efficiency rules во всех трёх фрагментах (O1 и O3-lite), «Output discipline» в
codegraph-first-navigation, tree-first резолв токенов в coder на трёх поверхностях и в
design-token-usage (O2), §3.1 context pass-through в `tacticum-workflow` на трёх поверхностях (O4).
Правила живы в `origin/main` и сегодня — Zero-resolve rule и batch-правила на месте;
`check_profile_version_discipline.py` зелёный, «OK — 64 profile(s) clean». В проде kmp-инсталляция
живёт уже на 0.12.1, то есть 0.2.2/0.2.3 раскатаны давно.
Что важно для честного прочтения: численные пороги A/B по O1 и O4b достигнуты не были — 137 ходов
при цели ≤90, транскрипт 0.70 МБ при ≤0.45, мульти-tool ходов ноль. Закрываю не вопреки этому:
итерации по O1 остановлены отдельным решением, потолок отнесён к ограничению рантайма, а
upgrade path через bulk-эндпоинты на бэкенде в скоуп этой задачи не входил — он ведётся отдельно,
US #643 («O5 — server-side эффективность MCP-контракта: batch `kb_verify_api_exists(...)`»).
Не проверял: A/B-числа взял из комментариев к задаче, своего замера трейсов не делал.

=== #641
Закрываю: `6eafd422` «port kmp 0.2.4-0.2.6 fixes to rn 0.4.5 / mail 0.6.5 / task-workflow 0.3.5» и
`ebccfcd6`, переиздавший rn 0.4.7 / mail 0.6.7 после инцидента с root-owned файлами на VPS.
Перенесено всё перечисленное и во все три профиля: команды `approve-docs` и `run-implementation`,
`tacticum-workflow` в трёх форматах, оба фрагмента (CLAUDE.md и AGENTS.md с Depth rule), плюс
расширение из комментариев — task-dir resolution в `run-coder`/`run-tester`/`run-test-runner` и
`local-code-intelligence` у rn и mail (у task-workflow codegraph нет, поэтому там его и не
появилось). ADR-0055 лежит в `docs/adr/0055-profile-orchestration-topology-per-cli.md`.
Подтверждение брал не из отчёта, а из боевой базы каталога: у трёх установок — rn, mail и kmp — в
телах `approve-docs` и `run-implementation` есть «top-level session» и нет старого «Hand off to the
tacticum-workflow sub-agent», то есть дефект из описания в проде отсутствует; версии там давно выше
0.4.7/0.6.7, так что остаток «re-pin» из комментариев закрыт.
Не проверял: поведенческой приёмки не было. Живого прогона `/approve-docs` с наблюдением, что в
main-чате спавнятся coder и tester и что fail-loud срабатывает на попытке hand-off, никто не
зафиксировал, автотеста на это в репозитории нет. Закрываю по составу доставки и по состоянию
боевой базы.

=== #663
Закрываю: `4fe722fb` «provision auto-resolves workspace, no user choice». Все три пункта резолюции
в коммите: find-first по всем workspace организации выполняется ДО резолюции workspace и возвращает
`created=false`; появился `Workspace.is_default` с миграцией (в main она под номером
`0036_workspace_is_default.py` — переномерована из 0035 при слиянии), бэкфиллом одно-workspace
организаций и partial unique index; в ошибке `workspace_ambiguous` теперь структурные `candidates`
с `{id, slug, name}`, а `whoami` отдаёт `workspace_id`. Мульти-workspace организациям миграция
дефолт сознательно не проставляет — это оставлено админу и записано в докстринге.
Подтверждение: `pytest tests/workspace/test_provision_installation.py` — 10 passed, включая
`test_provision_find_first_returns_existing_install_across_workspaces`,
`test_provision_creates_in_default_workspace` и
`test_provision_ambiguous_error_carries_structured_candidates`; `test_whoami_mcp.py` вместе с
`test_install_flow.py` — 63 passed, там же e2e-кейс мульти-workspace провижина. И ручной пункт
проверил в боевой базе: у организации с тремя workspace (`base`, `fnf`, `smoke-…` — ровно как в
описании инцидента) `base` помечен `is_default`.
Не проверял: это тесты и состояние базы, а не полевой прогон — живого провижина у пользователя с
мульти-workspace организацией я не наблюдал.

=== #657
Закрываю: `b0094ae0` «context.yaml-first installation_id discipline + serena launcher». 57 файлов,
все пять профилей — web, kmp, rn, mail, go-backend. Тела агентов поправлены во всех трёх форматах
(`agents/`, `agents-codex/`, `agents-copilot/`); из скиллов — kb-navigation у всех пяти,
design-system-discovery и design-token-usage у четырёх фронтовых (у go-backend их нет, там
дизайн-системы не применяются); фрагменты CLAUDE.md, AGENTS.md и copilot-instructions у всех пяти.
Требование «не трогать» соблюдено: `approve-docs` и `run-implementation` в диффе отсутствуют. Web
поднят до 0.1.3.
Смотрел содержательный дифф, а не заголовок: в теле агента стоит шаг «Read `.tacticum/context.yaml`
→ take `installation_id`», следом `kb_discover(installation_id=<id>)` с прямым «do NOT rely on
server auto-resolve», и STOP-правило про то, что молчаливая деградация на Glob/Grep/Serena вместо KB
— нарушение жёсткого правила. Так во всех 17 телах агентов линейки. Старой формулировки «omit it —
the server resolves it» в живых текстах `origin/main` не осталось. Регрессия закрыта оракулом
`assert_no_stale_id_advice`, прогон `test_install_flow.py` зелёный — в тех же 63 passed.
Остаток, названный в тексте задачи — договориться после деплоя бэкенд-фикса, оставлять ли серверный
авто-резолв, — это решение по отдельной issue, а не хвост этого change-set.
Не проверял: подтверждение тестовое и текстовое. Что агент в живой сессии действительно читает
`context.yaml` и не уходит в Glob/Grep, полевым прогоном я не проверял.