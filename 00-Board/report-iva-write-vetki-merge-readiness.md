---
title: report-iva-write-vetki-merge-readiness
type: note
permalink: tacticum/00-board/report-iva-write-vetki-merge-readiness
status: draft
repo: tacticum-dev
project: iva-write
tags:
- board
- report
- iva-write
- verify
---

# Готовность к мержу: feat/iva-write-mcp-profile и feat/iva-write-rollout-roles (04.08)

Прогон на реальном дереве репозитория, БД поднималась настоящая (docker postgres:16-alpine
из `tests/conftest.py`). Ничего не коммитил, не пушил, не мержил. Базовый замер снят в
отдельном worktree на `origin/main`, дерево снято после замеров.

## Вердикт

**К мержу как есть — НЕ готово ни одна из веток.** Внутри себя обе зелёные, но обе
конфликтуют с текущим `origin/main`, и у `rollout` есть два содержательных блокера
(коллизия версий ролей QA и утечка канала на две новые QA-роли).

## 1. Состояние веток относительно origin/main (f486464, дофетчено 04.08)

| Ветка | ahead | behind | конфликты с origin/main |
|---|---|---|---|
| feat/iva-write-mcp-profile (095db2f) | 8 | 77 | ДА, 6 файлов |
| feat/iva-write-rollout-roles (74e8a6c) | 21 | 73 | ДА, 11 файлов |

Конфликты у mcp-profile: два голдена `iva-role-analyst`, `test_install_flow.py`,
квикстарт аналитика, CHANGELOG и manifest роли аналитика.
У rollout — то же плюс голдены `iva-role-qa` и `iva-role-qa-web` (4 файла).

## 2. Соотношение веток

`feat/iva-write-rollout-roles` содержит `feat/iva-write-mcp-profile` ЦЕЛИКОМ
(`git merge-base --is-ancestor` = 0, merge-base = 095db2f, голова mcp-profile).
Порядок: сначала mcp-profile (она запушена), потом rollout поверх.

## 3. Тесты (tests/catalog + tests/e2e_install)

| Дерево | tests | passed | skipped | failed |
|---|---|---|---|---|
| origin/main (baseline) | 1268 | 1226 | 42 | 0 |
| feat/iva-write-mcp-profile | 1081 | 1081 | 0 | 0 |
| feat/iva-write-rollout-roles | 1114 | 1112 | 2 | 0 |

Числа веток НИЖЕ baseline потому, что ветки отстают от main на 73-77 коммитов, а не
из-за регрессии. Эталон рунбука 1081 совпал точь-в-точь для mcp-profile; для rollout
рунбук называет 1087/1, факт 1112/2 — рунбук устарел на 5 своих же коммитов и на
слияние с main.

## 4. Скрипты дисциплины

- mcp-profile: version discipline OK (59 профилей), mirror sync OK (73/6). Обе зелёные.
- rollout: mirror sync OK; **version discipline ПАДАЕТ** — `iva-role-qa` и
  `iva-role-qa-web` изменены, но версия совпадает с уже занятой в main.

## 5. Коллизия версий (блокер)

main коммитом 77041a7 независимо поднял `iva-role-qa` 0.6.2 → **0.7.0** и
`iva-role-qa-web` 0.2.1 → **0.3.0**. Ветка rollout подняла их в те же самые номера со
СВОИМ содержимым. После мержа сид даст
`version_already_exists_with_different_content`. Нужен пере-бамп (0.7.1/0.3.1) +
записи в CHANGELOG.

## 6. Утечка канала на новые QA-роли (блокер)

В main появились `iva-role-qa-desktop` и `iva-role-qa-mobile`, обе композят лейн
`iva-qa-mcp` — тот самый, куда rollout кладёт `helm-iva-write`. После мержа канал
приезжает им молча: в `_IVA_WRITE_ROLES` и `IVA_WRITE_ROLES` их нет, поэтому e2e будет
ждать ингредиент в `absent` и упадёт, голдены разойдутся, а паки у ролей есть, но про
канал не знают. Все оракулы канала параметризованы ПО списку — сторожа «роль композит
канал, но не в списке» нет, ровно та болезнь молчащего рукописного списка, от которой
main уже заводил сторож в e2e.

## 7. Голдены

В ветках голдены регенерированы и сходятся (e2e зелёный). Но те же ключи
(`repo/.mcp.json`, `repo/CLAUDE.md`) меняет и main — отсюда конфликт. Разрешать руками
нельзя, после мержа обязательна регенерация `E2E_INSTALL_REGEN_GOLDEN=1`.

## 8. Адрес канала

Во всех манифестах обеих веток `url:` = `https://helm.tacticum.ru/mcp/iva-write`.
Старый `mcp.tacticum.ru/iva-write/mcp` встречается только в предупреждающих
комментариях «не воспроизводить». Нарушений нет.

## Что мешает мержу (по порядку)

1. mcp-profile: свести с main, регенерировать голдены аналитика, прогнать.
2. rollout: то же + пере-бамп версий двух QA-ролей + решение по qa-desktop/qa-mobile
   (выдать канал с текстом в паках или явно исключить) + сторож против молчащего списка.

## Observations

- [факт] Обе ветки зелёные внутри себя, но обе конфликтуют с origin/main #verify
- [блокер] iva-role-qa 0.7.0 и iva-role-qa-web 0.3.0 заняты в main другим содержимым #verify
- [блокер] qa-desktop и qa-mobile получают канал записи без текста в паках и без оракулов #verify
- [факт] Адрес канала везде правильный, helm.tacticum.ru/mcp/iva-write #verify